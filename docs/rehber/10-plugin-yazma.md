# Plugin yazma

Bu sayfa pratik: sıfırdan çalışan bir Plugin yazıp odaya bağlamayı, runtime config eklemeyi, persistance kullanmayı, başka plugin'lerle iletişimi göstereceğim.

Boilerplate yapısı için [09-addon-sistemi.md](09-addon-sistemi.md) önce okunmalı; orada anlatılan `Object.setPrototypeOf` + `Plugin.call` deyimini varsayıyorum.

## Anatomi: pratik AFK detection plugin'i

`afkKickPlugin.js`:

```javascript
module.exports = function (API) {
  const { Plugin, AllowFlags, VariableType } = API;

  Object.setPrototypeOf(this, Plugin.prototype);
  Plugin.call(this, "afkKick", true, {
    version: "1.0",
    author: "ben",
    description: "Belirli süre input vermeyen oyuncuları odadan atar",
    allowFlags: AllowFlags.CreateRoom,
  });

  const that = this;

  // Runtime ayarlanabilir parametre
  this.defineVariable({
    name: "timeoutSeconds",
    description: "Oyuncu kaç saniye aktivite vermezse atılır",
    type: VariableType.Integer,
    value: 60,
    range: { min: 10, max: 600, step: 10 },
  });

  this.defineVariable({
    name: "checkOnlyDuringGame",
    description: "Sadece maç süresince mi AFK kontrolü yapılsın",
    type: VariableType.Boolean,
    value: true,
  });

  let lastActivity = new Map();
  let intervalId = null;

  this.initialize = function () {
    intervalId = setInterval(check, 5000);
  };

  this.finalize = function () {
    if (intervalId) clearInterval(intervalId);
    lastActivity.clear();
  };

  this.onPlayerJoin = function (player) {
    lastActivity.set(player.id, Date.now());
  };

  this.onPlayerLeave = function (player) {
    lastActivity.delete(player.id);
  };

  this.onPlayerInputChange = function (playerId) {
    lastActivity.set(playerId, Date.now());
  };

  this.onPlayerBallKick = function (playerId) {
    lastActivity.set(playerId, Date.now());
  };

  this.onPlayerChat = function (id) {
    lastActivity.set(id, Date.now());
  };

  function check() {
    if (that.checkOnlyDuringGame && !that.room.gameState) return;

    const now = Date.now();
    const limitMs = that.timeoutSeconds * 1000;

    for (const player of that.room.players) {
      if (player.id === 0 || player.team.id === 0) continue;
      const last = lastActivity.get(player.id) ?? now;
      if (now - last > limitMs) {
        that.room.kickPlayer(player.id, `${that.timeoutSeconds}s AFK`, false);
      }
    }
  }
};
```

Kullanım:

```javascript
const API = require("node-haxball")();
const { Room } = API;
const afkPluginFactory = require("./afkKickPlugin");

const afkPlugin = new afkPluginFactory(API);

Room.create({...}, {
  ...,
  plugins: [afkPlugin],
});
```

Birkaç şey hakkında bilinçli ol:

**Factory pattern**: Plugin dosyasını `module.exports = function (API) { ... }` ile yazıyoruz, kütüphanenin gösterdiği konvensiyon bu. `new` ile çağırınca constructor olarak çalışır.

**`that = this`**: Closure içindeki callback'ler `this` değil `that` üzerinden plugin objesine ulaşır. `setInterval`'in callback'i farklı `this` ile çağırıldığı için kritik.

**`that.room`**: Constructor anında `null`dur. Sadece `initialize`'dan sonra geçerli, yukarıda interval `initialize`'da başlatıldı, doğru yer.

**Üç aktivite kaynağı birlikte**: `onPlayerInputChange` (tuş state'i), `onPlayerBallKick` (topa vurma) ve `onPlayerChat` (yazışma). Sadece input takip edersen yazışan ama hareket etmeyen oyuncu haksız atılır.

**`gameState` kontrolü**: Maç dışında oyuncular sohbet edebilir, AFK kicki rahatsız edici olur. `room.gameState` null ise maç aktif değildir.

## Plugin'i runtime'da yönetmek

Aktif/pasif:

```javascript
room.setPluginActive("afkKick", false);
room.setPluginActive("afkKick", true);
```

Variable değiştir:

```javascript
const plugin = room.pluginsMap.afkKick;
plugin.timeoutSeconds = 120;
plugin.checkOnlyDuringGame = false;
```

Listele:

```javascript
console.log(room.plugins.map((p) => p.name));
```

Runtime'da plugin değiştirme/güncelleme:

```javascript
room.addPlugin(newPlugin);
room.removePlugin(plugin);
room.updatePlugin(oldIndex, newPlugin);
room.movePlugin(currentIndex, newIndex);  // sıra önemli
```

Sıra önemli çünkü Before/After zincirinde aynı event'e abone plugin'lerin handler'ları liste sırasıyla çağırılır.

## Persistance, diske yazma

Plugin'in state'ini restart sonrası korumak için JSON dosyası:

```javascript
const fs = require("fs");
const path = require("path");

const DATA_FILE = path.resolve(__dirname, "afkKick.json");

let banList = new Set();

this.initialize = function () {
  if (fs.existsSync(DATA_FILE)) {
    banList = new Set(JSON.parse(fs.readFileSync(DATA_FILE, "utf8")));
  }
};

function persistBan(auth) {
  banList.add(auth);
  fs.writeFileSync(DATA_FILE, JSON.stringify([...banList]));
}
```

Disk yazma maliyetli, her gol için yazma! Saniyeye birkaç yazımdan fazlasını yapacaksan ya in-memory buffer + periyodik flush, ya da SQLite gibi gerçek bir DB. [12-veri-saklama.md](12-veri-saklama.md) seçenekleri daha detaylı işliyor.

## Başka plugin'lere bağımlılık

Bir plugin başka bir plugin'in veya library'nin işlevine ihtiyaç duyabilir:

```javascript
this.initialize = function () {
  const cmdLib = that.room.librariesMap.commands;
  if (!cmdLib) {
    console.warn("afkKick plugin'i 'commands' library'sine ihtiyaç duyuyor");
    return;
  }

  cmdLib.add({
    name: "afk",
    helpText: "Mevcut AFK ayarını gösterir",
    handler: (byId) => {
      that.room.sendAnnouncement(
        `AFK timeout: ${that.timeoutSeconds}s`,
        byId, 0xCCCCCC, "small", 0
      );
    },
  });
};
```

Library yüklü değilse plugin sessizce çalışmaya devam eder, sadece komut kaydı yapmaz. Hard dependency olsaydı `initialize` içinde throw ederdin.

## Plugin testi

Plugin'in mantığını izole etmek için bağlı ihtiyacı (`this.room`) mock edebilirsin:

```javascript
const fakeRoom = {
  players: [
    { id: 1, name: "p1", team: { id: 1 } },
    { id: 2, name: "p2", team: { id: 2 } },
  ],
  gameState: {},
  kickPlayer: (id, reason) => console.log(`kick: ${id} ${reason}`),
};

const plugin = new (require("./afkKickPlugin"))(API);
plugin.room = fakeRoom;
plugin.timeoutSeconds = 1;
plugin.initialize();
// state'i kur, davranışı gözlemle
```

Tam kapsamlı entegrasyon testi için `Room.sandbox` modu uygundur, gerçek Haxball backend'ine bağlanmadan plugin davranışını koşturursun. [01-mimari.md](01-mimari.md) sandbox tanıtımı, [03-room-join.md](03-room-join.md) yan başlığı sandbox kullanımı için referans.

## Sık karşılaşılan hatalar

**`that.room` constructor'da `null`**: Sadece initialize sonrası geçerli.

**`Map`'i clear etmemek**: `finalize`'da Map'i sıfırlamazsan, plugin update'lerinde eski state taşar.

**`onPlayerLeave`'de Map'ten silmemek**: Hayalet entry'ler birikir, uzun süreçte memory leak.

**`setInterval`'i temizlememek**: `finalize`'da `clearInterval` yoksa, plugin re-init'inde iki interval birden çalışır.

**Aynı plugin'i iki kere eklemek**: Plugin'lerin unique name'i olmalı; ikinci ekleme hata atar.

**`AllowFlags` yanlış**: Host bot için `JoinRoom` flag'i verilirse plugin yüklenmez. Konvensiyon: yazdığın plugin'in hangi modda mantıklı olduğunu netleştir, doğru flag'i ver.
