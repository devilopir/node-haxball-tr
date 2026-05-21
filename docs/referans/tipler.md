# Tip referansı

`node-haxball`'ın sık karşılaşılan veri yapıları. Her tip için temel alanları listeliyorum; detaylı kullanımlar ilgili rehber sayfasında.

## Player

Oyuncu objesi. `room.players` array'inde, `room.getPlayer(id)` ile lookup edilir.

```typescript
interface Player {
  id: int;                       // bağlantı süresince benzersiz, çıkışta kaybolur
  name: string;                  // kullanıcı adı
  team: Team;                    // { id: 0|1|2, color: int }
  flag: string;                  // ülke kodu (2 harf)
  avatar: string | null;         // kullanıcının kendi avatarı
  headlessAvatar: string | null; // host'un atadığı override
  isAdmin: boolean;
  conn: string;                  // IP bilgisi (numerik temsil)
  auth: string;                  // KALICI kimlik, ban listeleri için anahtar
  avatarNumber: int;             // avatar yoksa gösterilen sayı
  // ... ve diğer alanlar
}
```

**Dikkat:** `auth` değerini her zaman `player.auth` üzerinden okuyun. Oyuncunun adı değişebilir, id'si yeniden bağlantıda yenilenir; auth ise aynı kalır. `player.name` değiştirilebilir, `player.id` her bağlantıda yeni.

## Team

```typescript
interface Team {
  id: int;     // 0 = spec, 1 = red, 2 = blue
  color: int;  // hex renk değeri
}
```

`Team.spec`, `Team.red`, `Team.blue`, `Team.byId` static referansları namespace'te mevcut.

## RoomState

Bir tick anındaki tüm state. `room.state` ile erişilir.

```typescript
interface RoomState {
  name: string;
  players: Player[];
  redScore: int | null;
  blueScore: int | null;
  timeElapsed: int | null;
  timeLimit: int;
  scoreLimit: int;
  stadium: Impl.Stadium.Stadium;
  currentFrameNo: int;
  banList: BanEntry[];
  password: string;
  geo: GeoLocation;
  maxPlayerCount: int16;
  fakePassword: boolean | null;
  fixedPlayerCount: int16 | null;
  showInRoomList: boolean;
  unlimitedPlayerCount: boolean;
  ext: RoomState | null;        // extrapolated versiyon
  copy: () => RoomState;
}
```

Client modunda `state` host'tan gelen son snapshot'tır. `extrapolate(ms, ignoreMultipleCalls)` ile kısa süreli ileri sarım yapılabilir (renderer için).

## GameState

Maç aktifken `room.gameState` non-null. Maç durunca `null`.

İçinde: kalan süre, skor, disc'lerin pozisyonları, `gamePlayState` (BeforeKickOff/Playing/AfterGoal/Ending), aktif oyuncu listesi.

## ScoresObject

`onTeamGoal` doğrudan bunu vermez ama `room.redScore`/`blueScore`/`timeElapsed`/`scoreLimit`/`timeLimit` üzerinden eşdeğerine erişilir.

## Disc

`room.getBall()`, `room.getDisc(id)`, `room.getPlayerDisc(playerId)` döner.

```typescript
class Disc {
  pos: Point;                   // { x, y }
  speed: Point;
  gravity: Point;
  radius: number;
  damping: number;              // 0 < damping <= 1
  invMass: number;              // 0 = sabit, büyük = hafif
  bCoef: number;                // 0 = sıçramaz, 1 = elastik
  color: int;                   // -1 şeffaf, hex int
  cGroup: int;                  // collision group flags
  cMask: int;                   // collision mask flags
}
```

`setDiscProperties(id, props)` ile değiştirilen alanlar farklı isimler kullanır (`xspeed`, `yspeed`, `xgravity`, `ygravity`, `bCoeff`), okurken `disc.speed.x`, yazarken `xspeed`.

## Point

```typescript
class Point {
  x: number;
  y: number;
}
```

## Stadium

[ortak/stadyum-formati.md](../ortak/stadyum-formati.md) detaylı.

```typescript
class Stadium {
  vertices: Vertex[];
  segments: Segment[];
  planes: Plane[];
  goals: Goal[];
  discs: Disc[];
  joints: Joint[];
  redSpawnPoints: Point[];
  blueSpawnPoints: Point[];
  playerPhysics: PlayerPhysics;
  defaultStadiumId: int;        // 255 = custom
  maxViewWidth: number;
  cameraFollow: CameraFollow;
  canBeStored: boolean;
  fullKickOffReset: boolean;
  name: string;
  width: number;
  height: number;
  bg: BackgroundObject;
}
```

## PlayerPhysics

Stadyumun oyuncu disc'lerine uyguladığı fizik:

```typescript
class PlayerPhysics {
  radius: number;
  bCoef: number;
  invMass: number;
  damping: number;
  acceleration: number;
  kickingAcceleration: number;
  kickingDamping: number;
  kickStrength: number;
  kickback: number;             // tekme sonrası geri tepme
  // ...
}
```

## Vertex / Segment / Plane / Goal / Joint

Stadium'un fiziksel yapı taşları.

```typescript
class Vertex {
  pos: Point;
  bCoef: number;
  cGroup: int;
  cMask: int;
}

class Segment {
  v0: int;                      // vertex id
  v1: int;
  bias: number;
  curve: number;                // 0 = düz, >0 = bombeli
  vis: boolean;
  // bCoef, cGroup, cMask
}

class Plane {
  normal: Point;                // birim vektör
  dist: number;
  // bCoef, cGroup, cMask
}

class Goal {
  p0: Point;
  p1: Point;
  team: Team;                   // hangi takımın kalesi
}

class Joint {
  d0: int;                      // disc id
  d1: int;
  length: number | null;
  strength: number | null;
  color: int;
}
```

## BanEntry

```typescript
interface BanEntry {
  id: BanEntryId;               // entry'nin kendi id'si (oyuncu id'si değil!)
  type: BanEntryType;
  value: any;                   // type'a göre değişir (auth, ip, range)
  // ...
}
```

## GeoLocation

```typescript
interface GeoLocation {
  code?: string;                // "tr", "us", ...
  lat: number;
  lon: number;
  flag?: string;
}
```

`Utils.parseGeo(input, fallback)` ile normalize edilir.

## RoomData

`Utils.getRoomList()`'in döndürdüğü objelerin tipi:

```typescript
type RoomData = {
  id: string;                   // Room.join'de kullanılır
  name: string;
  password: boolean;            // şifre korumalı mı (string DEĞIL, boolean)
  players: int;
  maxPlayers: int;
  geo: GeoLocation;
  version: int;
  details?: string;
  dist?: number;                // calculateAllRoomDistances çağrıldıysa
};
```

## Auth

```typescript
interface Auth {
  // kütüphane internal'ı, sen sadece olduğu gibi geçirirsin
}
```

`Utils.generateAuth()` `[authKey: string, authObj: Auth]` döner. `authKey`'i kaydet, `authObj`'i `Room.join`'de kullan.

## HaxballEvent

`EventFactory` fonksiyonlarının döndürdüğü event objesi. İç yapısı event tipine göre değişir. `room.executeEvent(event, byId)` ile tetiklenir.

## UpdatedRoomProps

`onRoomPropertiesChange` callback'inde gelen property listesi:

```typescript
type UpdatedRoomProps = {
  name?: string | null;
  password?: string | null;
  fakePassword?: boolean | null;
  geo?: GeoLocation | null;
  playerCount?: int | null;
  maxPlayerCount?: int | null;
};
```

Sadece **değişen** alanlar dolu gelir, diğerleri `undefined`.

## Errors.HBError

```typescript
class HBError {
  code: ErrorCodes;             // int değer
  // toString() insan okunur Türkçe/İngilizce mesaj
}
```

[errors.md](errors.md) tam liste.

## TeamColors

```typescript
class TeamColors {
  angle: int;
  text: int;                    // forma desenindeki text rengi
  inner: int[];                 // 3 ana renk
}
```

`setTeamColors(teamId, angle, ...colors)` ile değiştirilir.
