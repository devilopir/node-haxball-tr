# Addon API referansı

Tüm addon türleri `Addon` base'inden türer. Bu sayfa base'i ve dört spesifik tipi (RoomConfig, Plugin, Library, Renderer) referans olarak listeliyor.

Pratik kullanım için [rehber/09-addon-sistemi.md](../rehber/09-addon-sistemi.md) ve [rehber/10-plugin-yazma.md](../rehber/10-plugin-yazma.md).

## Addon base

Tüm addon'ların ortak interface'i:

```typescript
interface Addon {
  readonly room: Room;          // initialize sonrası geçerli
  defineMetadata(metadata?: any): void;
  defineVariable(variable: Variable): any;
  setVariableGUIProps(varName: string, ...vals: { name, value }[]): void;
  initialize(): void;
  finalize(): void;
}
```

### `room`

Addon hangi room'a bağlıysa onu verir. Constructor anında `null`dur; `initialize()` çağrılana kadar erişilmez.

### `defineMetadata(metadata?)`

Constructor'da otomatik çağrılır, default implementasyon boştur. GUI ortamında bu objeyi göstermek isteyenler override eder.

```typescript
metadata = {
  name: string;
  version: number;
  author: string;
  description: string;
  allowFlags: int;              // AllowFlags.CreateRoom | AllowFlags.JoinRoom
}
```

### `defineVariable(variable)`

Runtime'da değiştirilebilir parametre tanımlar:

```typescript
this.defineVariable({
  name: "myParam",
  description: "Bot davranışı için ayar",
  type: VariableType.Integer,
  value: 60,
  range: { min: 10, max: 600, step: 10 },
});
```

Tanımlandıktan sonra `this.myParam` ile okunur/yazılır. Değişiklikler `onVariableValueChange` callback'ini tetikler.

### `initialize()`

Addon aktive edildiğinde / oda kurulurken / addon eklendiğinde çağırılır. `this.room` artık geçerlidir, setup buraya yazılır:

```typescript
this.initialize = function () {
  this.intervalId = setInterval(...);
  // library lookups
  const cmd = this.room.librariesMap.commands;
};
```

### `finalize()`

Addon deaktive edildiğinde / oda kapanırken / addon kaldırıldığında. Cleanup için:

```typescript
this.finalize = function () {
  clearInterval(this.intervalId);
  // dosya flush, event listener kaldırma
};
```

## RoomConfig

Bir odanın "ana mantığı". Her oda için en fazla 1 RoomConfig olur. Tüm `AllRoomConfigCallbacks`'i destekler (Before/After çiftler dahil).

```typescript
abstract class RoomConfig implements Addon, AllRoomConfigCallbacks {
  static readonly addonType: AddonType;  // = AddonType.RoomConfig
  constructor(metadata?: any);
}
```

Function constructor pattern (kütüphane konvensiyonu):

```javascript
function MyConfig() {
  Object.setPrototypeOf(this, RoomConfig.prototype);
  RoomConfig.call(this, {
    version: "1.0",
    author: "ben",
    description: "...",
    allowFlags: AllowFlags.CreateRoom,
  });

  this.onPlayerJoin = function (player) {
    this.room.sendAnnouncement("Selam", player.id, 0xCCCCCC, "normal", 0);
  };
}

// Kullanım:
Room.create({...}, { config: new MyConfig() });
```

## Plugin

Modüler özellik birimi. Bir odada birden fazla plugin olabilir. `AllPluginCallbacks` tipi callback'leri destekler.

```typescript
abstract class Plugin implements Addon, AllPluginCallbacks {
  static readonly addonType: AddonType;  // = AddonType.Plugin
  constructor(name: string, active?: boolean, metadata?: any);
}
```

Constructor parametre sırası diğer addon'lardan farklı: `name` ilk, `active` ikinci, `metadata` üçüncü.

```javascript
function MyPlugin() {
  Object.setPrototypeOf(this, Plugin.prototype);
  Plugin.call(this, "myPlugin", true, {
    version: "1.0",
    author: "ben",
    description: "...",
    allowFlags: AllowFlags.CreateRoom,
  });

  this.onPlayerJoin = function (player, customData) {
    // ...
  };
}

// Kullanım:
Room.create({...}, { plugins: [new MyPlugin()] });
```

`active: false` ile başlattığında plugin yüklenir ama callback'leri çağrılmaz. `room.setPluginActive("myPlugin", true)` ile sonra açılır.

### Plugin runtime metodları (room üzerinden)

- `room.setPluginActive(name, active)`, toggle
- `room.addPlugin(plugin)`, yeni plugin ekle
- `room.removePlugin(plugin)`, sil
- `room.updatePlugin(index, newPlugin)`, değiştir
- `room.movePlugin(currentIndex, newIndex)`, sıra değiştir (callback execution sırası)
- `room.plugins`, `Plugin[]`
- `room.pluginsMap[name]`, name ile erişim

## Library

Diğer addon'ların paylaşılan kütüphanesi. Kendi başına event handler'a sahip değil, tipik olarak yardımcı fonksiyonlar sunar.

```typescript
abstract class Library implements Addon {
  static readonly addonType: AddonType;  // = AddonType.Library
  constructor(name: string, metadata?: any);
}
```

Plugin/RoomConfig'lerden **önce** initialize olur, böylece onlar Library'lere güvenle erişebilir.

```javascript
function CommandLib() {
  Object.setPrototypeOf(this, Library.prototype);
  Library.call(this, "commands", {
    version: "1.0",
    author: "ben",
    description: "Komut sistemi",
    allowFlags: AllowFlags.CreateRoom,
  });

  const handlers = new Map();

  // Public API
  this.add = function (cmd) {
    handlers.set(cmd.name, cmd);
  };

  this.handle = function (room, byId, message) {
    // ...
  };
}

// Plugin içinden:
this.initialize = function () {
  const cmdLib = this.room.librariesMap.commands;
  cmdLib?.add({ name: "ping", handler: () => ... });
};
```

### Library runtime metodları

- `room.addLibrary(library)`
- `room.moveLibrary(index, newIndex)`
- `room.updateLibrary(index, newLibrary)`
- `room.libraries`, `Library[]`
- `room.librariesMap[name]`, name ile erişim

## Renderer

Görsel render katmanı. Browser canvas, terminal, image vb. Bir odada en fazla 1 Renderer olur.

```typescript
abstract class Renderer implements Addon, AllRendererCallbacks {
  static readonly addonType: AddonType;  // = AddonType.Renderer
  constructor(metadata?: any);
}
```

Render callback'lerine ek olarak `AllRendererCallbacks` (host event'leri + game event'leri + custom event'ler) destekler.

```javascript
function MyRenderer() {
  Object.setPrototypeOf(this, Renderer.prototype);
  Renderer.call(this, {
    name: "myRenderer",
    version: "1.0",
    author: "ben",
    description: "...",
  });

  this.render = function (extrapolatedState) {
    // canvas'a çiz, image üret, vb.
  };
}

Room.create({...}, { renderer: new MyRenderer() });
// veya:
room.setRenderer(new MyRenderer());
```

Repo'nun `examples/renderers/` klasöründe production örnekleri (defaultRenderer, pixiRenderer, imageRenderer, sandboxRenderer, flappyKirbyRenderer).

## AddonType enum

`addon.constructor.addonType` veya `addonInstance.constructor.addonType` ile sınıf türü öğrenilir:

```typescript
enum AddonType {
  RoomConfig = 0,
  Plugin = 1,
  Renderer = 2,
  Library = 3,
}
```

Custom addon yönetim koduna runtime tip kontrolü gerektiğinde işe yarar.

## Bir addon'un yaşam döngüsü özet

```
new MyAddon()                  ← constructor, this.room null
  ↓
Room.create({...}, { plugins: [myAddon] }) veya room.addPlugin(myAddon)
  ↓
addon.initialize()             ← this.room artık geçerli
  ↓
... event callback'ler çağırılır ...
  ↓
room.removePlugin(myAddon) veya oda kapanır
  ↓
addon.finalize()
```
