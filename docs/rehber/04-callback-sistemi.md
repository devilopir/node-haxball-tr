# Callback sistemi

`node-haxball`'da event'lere abone olmanın üç yolu vardır. Hangisini seçeceğin botun mimarisine bağlı, ama farklarını bilmen şart, yanlış yere yazılmış callback bazen tetiklenir, bazen tetiklenmez ve neden olduğunu anlaması zor olur.

## 1. Doğrudan atama, `room.onX = fn`

`Room.create`/`Room.join`'in `onOpen`'ında alınan `room` objesine doğrudan callback atanır:

```javascript
onOpen: (room) => {
  room.onPlayerJoin = (player) => {
    console.log("girdi:", player.name);
  };
  room.onPlayerChat = (id, message) => {
    if (message === "!ping") room.sendChat("pong", null);
  };
}
```

Bu yöntem hızlı ve okunabilirdir. Tek bir handler atayabilirsin, aynı isme ikinci kez atama yaparsan ilki kaybolur. Plugin'ler ve RoomConfig'ler aynı zamanda kendi callback'lerini çalıştırır, bu doğrudan atama onların **üzerine** çalışır (yani sen `room.onPlayerJoin = ...` derken plugin'lerin `onPlayerJoin`'leri yine tetiklenir, kendi handler'ın ek olarak çağırılır).

Bot tek dosyalık bir prototip ise burada kal. Komut sayısı arttıkça, kod birden fazla modüle bölünmek istenir hale geldikçe diğer yöntemlere geç.

## 2. RoomConfig, tek noktada event yönetimi

`RoomConfig`, odanın tüm event handler'larını toplu olarak veren bir addon nesnesidir. Kütüphanenin idiomatic pattern'i ES6 class değil, function constructor üzerinedir:

```javascript
const API = require("node-haxball")();
const { RoomConfig, AllowFlags } = API;

function MyConfig() {
  Object.setPrototypeOf(this, RoomConfig.prototype);
  RoomConfig.call(this, {
    version: "1.0",
    author: "ben",
    description: "ana oda konfigi",
    allowFlags: AllowFlags.CreateRoom,
  });

  const that = this;

  this.onPlayerJoin = function (player) {
    that.room.sendAnnouncement(
      `Hoş geldin ${player.name}`, player.id, 0x88FF88, "normal", 0
    );
  };

  this.onPlayerChat = function (id, message) {
    if (message.startsWith("!")) {
      // komut işle
      return false; // mesajı odaya yayma
    }
  };
}

// kullanım:
Room.create({...}, { config: new MyConfig() });
```

Birkaç ayrıntı:

- `that = this` deyimi gerekli çünkü iç handler'larda `this` farklı bir objeyi gösterebilir (örneğin setInterval callback'inde global objeyi). Klasik pre-ES6 closure idiomudur ama kütüphanenin tüm örnekleri bu şekilde yazılmış olduğundan tutarlı kalmak iyi olur.
- `this.room`, RoomConfig framework tarafından `initialize` aşamasında set edilir. Constructor içinde henüz `null`dur, sadece event handler'ların içinden erişilir.
- `allowFlags`, addon'un hangi modlarda (CreateRoom, JoinRoom) aktive edilebileceğini söyler. Bot host olarak çalışıyorsa `CreateRoom`, client botsa `JoinRoom`, ikisinde de işliyorsa `CreateRoom | JoinRoom`.

RoomConfig'ler iki "metod" stilinde yazılabilir. Yukarıdaki "method2", daha temiz, kütüphanenin önerdiği. "method1" ise `onOpen` içinde `room.onX = ...` atamalarını gruplayan helper bir fonksiyon kullanır; `room.onX = fn` doğrudan atamasıyla aynıdır, sadece organize edilmiş halidir. method2 örnekleri için kütüphane reposunun [examples/roomConfigs/method2](https://github.com/wxyz-abcd/node-haxball/tree/main/examples/roomConfigs/method2) klasörü.

Tek bir `config` verilir.

## 3. Plugin'ler, Before/After ve customData zinciri

Plugin'ler modüler kod parçalarıdır. Her plugin kendi `onBeforeX` ve `onAfterX` handler'larını verir, kütüphane bunları sıraya dizip çalıştırır.

```javascript
const API = require("node-haxball")();
const { Plugin, AllowFlags } = API;

function AfkKick() {
  Object.setPrototypeOf(this, Plugin.prototype);
  Plugin.call(this, "afkKick", true, {
    version: "1.0",
    author: "ben",
    description: "AFK kalan oyuncuyu odadan atar",
    allowFlags: AllowFlags.CreateRoom,
  });

  const that = this;
  const lastActive = new Map();

  this.onPlayerJoin = function (player, customData) {
    lastActive.set(player.id, Date.now());
  };

  this.onPlayerInputChange = function (id, value, customData) {
    lastActive.set(id, Date.now());
  };
}

// kullanım:
Room.create({...}, { plugins: [new AfkKick()] });
```

`Plugin.call(this, name, isActive, metadata)`, ikinci argüman `true` ise plugin oda kurulduğu anda otomatik aktive olur. `false` verirsen `room.setPluginActive("afkKick", true)` ile manuel açılır; runtime'da kapatılıp açılabilen plugin'ler için yararlı.

Birden fazla plugin verilebilir; aynı event'e yazılmış tüm plugin handler'ları sıra ile çağırılır.

### Before/After ve customData

Bir event ateşlendiğinde sıra şudur:

1. Tüm plugin'lerin `onBeforeX` handler'ları, eklendikleri sırayla
2. Kütüphanenin kendi iç işlemi
3. Tüm plugin'lerin `onAfterX` (veya basit `onX`) handler'ları, eklendikleri sırayla
4. `RoomConfig`'in `onX` handler'ı
5. `room.onX` doğrudan atanan handler

Her aşamada bir callback `undefined` dışında bir değer döndürürse, o değer `customData` parametresi olarak bir sonraki callback'e aktarılır. Bu, eklentiler arası gevşek bir iletişim kanalıdır:

```javascript
function StatTracker() {
  Object.setPrototypeOf(this, Plugin.prototype);
  Plugin.call(this, "statTracker", true, { /* metadata */ });

  this.onBeforePlayerChat = function (id, message) {
    return { isCommand: message.startsWith("!") };
  };
}

function CommandRouter() {
  Object.setPrototypeOf(this, Plugin.prototype);
  Plugin.call(this, "commandRouter", true, { /* metadata */ });

  this.onAfterPlayerChat = function (id, message, customData) {
    if (customData?.isCommand) {
      // komut işleme...
    }
  };
}
```

`customData` mekanizması zorunlu değildir; çoğu plugin handler'ı bunu görmezden gelir.

### Veto: `false` döndürmek

`onBeforeX` handler'ı `false` döndürürse, bazı event'ler iptal edilir. Bu davranış event-spesifiktir; `onBeforePlayerChat` `false` döndürürse mesaj yayınlanmaz (chat moderasyon için en yaygın kullanım). Tüm event'ler veto edilebilir değildir, özellikle "bir şey oldu, sen yorumla" tipli event'lerde (örneğin `onAfterGameStart`) `false` etkisizdir.

## Hangisini ne zaman

| Senaryo | Yöntem |
|---|---|
| Tek dosyalık küçük bot | `room.onX = fn` |
| Tek dosyalık ama birkaç event çoksa, organize etmek istiyorsan | RoomConfig |
| Bot mantığı parçalara bölünebilir (AFK, istatistik, komutlar ayrı modüller) | Plugin'ler |
| Topluluk plugin'i yazıyorsan, başka botlarla birlikte çalışsın | Plugin |
| Birkaç plugin'i tek bot içinde birleştireceksen | RoomConfig + plugins listesi (config ana mantık, plugin'ler ek modüller) |

Üçü aynı anda kullanılabilir. Plugin'ler kendi handler'larını çalıştırır, RoomConfig kendi metodlarını, `room.onX` doğrudan atananı. Aynı event'e üçü de yazarsa üçü de çağırılır.

## "Common" callback'ler, basit kullanım

Tüm Before/After çiftine sahip olmayan, "şu oldu, haber veriyorum" tipli callback'ler de vardır. `onPlayerJoin`, `onPlayerLeave`, `onPlayerChat`, `onGameTick` bu gruptadır. Bir tane parametre listesi ile çalışırlar, `customData` zinciri ile uyumludurlar ama veto edilemezler.

Pratikte ihtiyacın olan callback'lerin çoğu bu kategoride. Before/After ayrımına ancak chat moderasyon (`onBeforePlayerChat` ile mesajı engelleme) veya custom RoomConfig pattern'ları için ihtiyaç duyarsın.

## Callback isimleri ve parametre listeleri

Tam liste [referans/callbacks.md](../referans/callbacks.md) içinde. Sık kullanılanlar:

- `onPlayerJoin(player, customData?)`
- `onPlayerLeave(player, reason, isBanned, byId, customData?)`, `reason` null ise kendi ayrılmış, string ise kicklenmiş; `isBanned` ban edildi mi
- `onPlayerChat(id, message, customData?)`, `id`, Player objesi değil
- `onPlayerTeamChange(id, teamId, byId, customData?)`
- `onPlayerAdminChange(id, isAdmin, byId, customData?)`
- `onGameStart(byId, customData?)`
- `onGameStop(byId, customData?)`
- `onTeamGoal(teamId, goalId, goal, ballDiscId, ballDisc, customData?)`
- `onGameEnd(winningTeamId, customData?)`, resmi HBInit'in `onTeamVictory`'sinin karşılığı, sadece kazanan takım id'sini verir
- `onPositionsReset(customData?)`
- `onStadiumChange(stadium, byId, customData?)`
- `onAfterRoomLink(link, customData?)`
- `onGameTick(customData?)`, her tick, saniyede 60 kez. **Burada yoğun işlem yapma**, ana event loop'u boğar.

`byId`, eylemi tetikleyen oyuncunun id'sidir; host kendi atadıysa `0` olur (`noPlayer: true` ise host'a karşılık 0 yine geçerlidir).
