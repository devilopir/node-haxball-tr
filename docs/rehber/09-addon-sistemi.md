# Addon sistemi

`node-haxball` mantığı modüler tutmak için dört tip addon sunar. Hangisini ne için kullanacağın projeye bağlı, bazıları interchangeable değil, doğru olanı seçmek farklılığı bilmekten geçer.

## Dört tip

| Addon | Ne işe yarar | Bir room'da kaç tane |
|---|---|---|
| **RoomConfig** | Odanın ana event handler katmanı, "ana mantık" | 1 |
| **Plugin** | Modüler özellik (AFK kick, istatistik, komutlar) | Çok |
| **Library** | Diğer addon'ların kullandığı paylaşılan kod (permission, command parser) | Çok |
| **Renderer** | Görsel render (browser GUI veya custom output) | 1 |

Hepsi aynı `Addon` base'inden türer, yani hepsinde `this.room`, `initialize()`, `finalize()`, `defineVariable()` aynı şekilde çalışır. Farkları event callback yelpazesindedir.

## Constructor imzaları

```javascript
// RoomConfig: sadece metadata (name yok!)
RoomConfig.call(this, { version, author, description, allowFlags });

// Plugin: name (positional), isActive, metadata
Plugin.call(this, "myPlugin", true, { version, author, description, allowFlags });

// Library: name (positional), metadata
Library.call(this, "myLib", { version, author, description, allowFlags });

// Renderer: sadece metadata (name yok)
Renderer.call(this, { version, author, description });
```

`name`, sadece `Plugin` ve `Library` constructor'ında **positional** olarak verilir (ilk argüman). Bu `room.pluginsMap[name]` ve `room.librariesMap[name]` üzerinden erişim için gereklidir. `RoomConfig` ve `Renderer`'a name verilmez; tek tane oldukları için isimle aranmazlar.

## Boilerplate iskeleti

```javascript
const API = require("node-haxball")();
const { Plugin, AllowFlags } = API;

function MyPlugin() {
  Object.setPrototypeOf(this, Plugin.prototype);
  Plugin.call(this, "myPlugin", true, {
    version: "1.0",
    author: "ben",
    description: "Bot için bir şey yapan plugin",
    allowFlags: AllowFlags.CreateRoom,
  });

  const that = this;
  let internalState = {};

  this.initialize = function () {
    // Plugin aktive edildiğinde
    internalState = {};
  };

  this.finalize = function () {
    // Plugin deaktive edildiğinde
  };

  this.onPlayerJoin = function (player, customData) {
    // ...
    that.room.sendAnnouncement("Selam " + player.name, player.id, 0xCCCCCC, "normal", 0);
  };
}
```

Class syntax (`class extends Plugin`) teknik olarak çalışır ama kütüphanenin tüm örnekleri function-constructor stilinde yazılmıştır. Tutarlılık için aynısını kullan; toplulukla kod paylaşırken sürtüşme yaşamazsın.

## AllowFlags

```javascript
AllowFlags.CreateRoom    // sadece host bot olarak çalışan oda
AllowFlags.JoinRoom      // sadece client bot olarak katılan
AllowFlags.CreateRoom | AllowFlags.JoinRoom  // ikisinde de
```

Addon ne için tasarlandıysa o flag'i ver. Yanlış modda kullanılan addon'lar yüklenmez ve hata atar. Default behavior açısından host-only callback'leri olan plugin'leri `JoinRoom`'da kullanmak işe yaramaz; flag bu yanlış kullanımı yükleme aşamasında yakalar.

## Addon'lar arası iletişim, librariesMap, pluginsMap

Bir addon başka bir addon'a referansa nasıl ulaşır?

```javascript
this.onPlayerChat = function (id, message) {
  const cmdLib = that.room.librariesMap.commands;
  if (cmdLib) cmdLib.handle(id, message);
};
```

`room.librariesMap[name]` ve `room.pluginsMap[name]` mevcut addon instance'larına isimle erişir. Bu, command parser gibi yardımcı kodu bir Library içine koyup, command-handler plugin'lerinden ona erişmek için kullanılan yaygın bir pattern.

Optional chaining (`?.`) önemli: bağlı plugin/library yüklenmemiş olabilir, sıkı çağrı yaparsan crash olur.

## RoomConfig vs Plugin

İki addon tipi de event handler tutar. Pratik fark:

**RoomConfig**:

- Tek tane verilir, "ana mantık" olarak konumlandırılır
- `Room.create({...}, { config: new MyConfig() })` ile bağlanır
- Bot mantığının çekirdek kısmı (oda açılışı, ana state, ana callback'ler)

**Plugin**:

- Birden fazla verilir, `plugins: [new A(), new B(), new C()]`
- Runtime'da `setPluginActive(name, false)` ile devre dışı bırakılabilir
- Bağımsız özellikler (AFK detection, chat moderasyon, stat tracker)

Net kural yok. "Ana mantık" RoomConfig, "yan özellikler" Plugin sezgisel ayrımdır.

## Library, kütüphane bileşeni

Library, kendi başına bir bot mantığı yürütmez. Diğer addon'ların kullandığı kod parçasıdır. Tipik library'ler:

- **commands**: `!komut arg1 arg2` formatını parse eder, handler kayıt API'si sunar
- **permissions**: Hangi oyuncunun hangi komutu kullanabileceğini yönetir
- **localStorage**: JSON dosyasına persistance
- **gui**: Renderer'ın görsel API'si

Kütüphane repo'sundaki `examples/libraries/` klasöründe production'a yakın örnekler var. Komut sistemi ve permission yönetimi tipik olarak library'den çalınır; sıfırdan yazmak yerine bunları kullanmak hızlandırır.

Library yüklenirken Plugin/RoomConfig'ten **önce** init olur. Bu sayede plugin'ler initialize'da library'ye güvenle erişebilir.

## Renderer

Render bir render API'sini soyutlar, browser canvas, terminal ASCII, image dosyası vb. Node tarafında host bot için neredeyse hiç kullanılmaz. İhtiyaç tipik olarak:

- Browser-based bot (oyunu canvas'a çizmek)
- Sandbox modunda haritayı görselleştirmek
- Replay'i video gibi izlemek (image renderer)

Tüm renderer örnekleri repo'nun `examples/renderers/` klasöründe.

## defineVariable, runtime config

Addon içinde değiştirilebilir bir parametre tanımlamak için:

```javascript
this.defineVariable({
  name: "afkTimeoutSeconds",
  description: "Bir oyuncuyu AFK için ne kadar tolere edelim",
  type: VariableType.Integer,
  value: 60,
  range: { min: 10, max: 600, step: 10 },
});
```

Tanımlanan variable, addon objesi üzerinde property olarak okunur/yazılır: `this.afkTimeoutSeconds`. GUI olmayan Node ortamında metadata (description, range) kullanılmaz, sadece value tutulur. GUI environment'larında bunlar slider/input olarak görünür.

Variable'lar değiştiğinde `onVariableValueChange` callback'i tetiklenir; runtime'da config değişikliklerine tepki vermek için kullanılır.

## Lifecycle: initialize ve finalize

```javascript
this.initialize = function () {
  // addon aktive edildiğinde / oda kurulurken
};

this.finalize = function () {
  // addon deaktive edildiğinde / oda kapanırken
};
```

`initialize` içinde `this.room` artık geçerlidir. Constructor'da değildir, handler atamaları constructor'da yapılır ama `this.room`'a ihtiyaç duyan setup (timer başlatma, library lookup) `initialize`'a girer.

`finalize` cleanup için: timer'ları temizle, dosyaya state yaz, event listener'ları kaldır.

## Hangisini ne zaman özeti

- **Tek dosyalık bot, hiçbir şeyi yeniden kullanmayacaksın**: RoomConfig + birkaç inline plugin
- **Çok özelliği olan bir bot**: RoomConfig (ana) + Plugin'ler (özellikler) + Library'ler (paylaşılan kod)
- **Topluluk için yazıyorsan**: Plugin (insanlar farklı botlara ekleyebilsin)
- **Görsel render**: Renderer
- **Sandbox testler**: Sadece RoomConfig veya hiç addon, manuel handler atama yeterli

Detaylı plugin yazma kılavuzu [10-plugin-yazma.md](10-plugin-yazma.md) içinde.
