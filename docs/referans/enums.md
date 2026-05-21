# Enum referansı

`node-haxball` ile gelen tüm enum tipleri ve değerleri. API namespace'inden destructure edilir:

```javascript
const { OperationType, CollisionFlags, Direction, ... } = require("node-haxball")();
```

## OperationType

Protokol seviyesindeki tüm event tipleri. `EventFactory` ve `onBeforeOperationReceived` callback'inde kullanılır.

| Değer | İsim | Anlam |
|---|---|---|
| 0 | `SendAnnouncement` | Announcement gönder (host only) |
| 1 | `SendChatIndicator` | Typing göstergesi |
| 2 | `CheckConsistency` | Senkron kontrolü |
| 3 | `SendInput` | Input state |
| 4 | `SendChat` | Chat mesajı |
| 5 | `JoinRoom` | Oyuncu girişi (host only) |
| 6 | `KickBanPlayer` | Kick/ban |
| 7 | `StartGame` | Maç başlat |
| 8 | `StopGame` | Maç durdur |
| 9 | `PauseResumeGame` | Duraklat/devam |
| 10 | `SetGamePlayLimit` | Skor + süre limiti |
| 11 | `SetStadium` | Stadium değiştir |
| 12 | `SetPlayerTeam` | Takım atama |
| 13 | `SetTeamsLock` | Takım kilidi |
| 14 | `SetPlayerAdmin` | Admin yetkisi |
| 15 | `AutoTeams` | Spec'tekileri dağıt |
| 16 | `SetPlayerSync` | Sync mod |
| 17 | `Ping` | Ping mesajı |
| 18 | `SetAvatar` | Kullanıcı avatarı |
| 19 | `SetTeamColors` | Takım rengi |
| 20 | `ReorderPlayers` | Liste sırası |
| 21 | `SetKickRateLimit` | Kick rate limit |
| 22 | `SetHeadlessAvatar` | Headless avatar (admin override) |
| 23 | `SetDiscProperties` | Disc/oyuncu disc özelliği |
| 24 | `CustomEvent` | Custom JSON event |
| 25 | `BinaryCustomEvent` | Custom binary event |
| 26 | `SetPlayerIdentity` | Custom identity |

## VariableType

`defineVariable` içinde tip belirtmek için:

| Değer | İsim |
|---|---|
| 0 | `Void` |
| 1 | `Boolean` |
| 2 | `Integer` |
| 3 | `Number` |
| 4 | `String` |
| 5 | `Color` |
| 6 | `CollisionFlags` |
| 7 | `Coordinate` |
| 8 | `Team` |
| 9 | `TeamWithSpec` |
| 10 | `BgType` |
| 11 | `CameraFollow` |
| 12 | `KickOffReset` |
| 13 | `Flag` |
| 14 | `File` |
| 15 | `PlayerId` |
| 16 | `Keys` |
| 17 | `Progress` |
| 18 | `PlayerPositionInGame` |

## ConnectionState

`onConnInfo` callback'inde gelen state:

| Değer | İsim | Anlam |
|---|---|---|
| -1 | `TryingReverseConnection` | NAT traversal için ters bağlantı deneniyor |
| 0 | `ConnectingToMaster` | Backend'e bağlanıyor |
| 1 | `ConnectingToPeer` | Host'a WebRTC ile bağlanıyor (client) |
| 2 | `AwaitingState` | İlk state bekleniyor |
| 3 | `Active` | Bağlantı tamam |
| 4 | `ConnectionFailed` | Hata |

## AllowFlags

Addon hangi modda kullanılabilir:

| Değer | İsim |
|---|---|
| 1 | `JoinRoom` |
| 2 | `CreateRoom` |

`AllowFlags.JoinRoom | AllowFlags.CreateRoom` ile ikisini birden ver.

## AddonType

Addon türlerini sayısallaştırır:

| Değer | İsim |
|---|---|
| 0 | `RoomConfig` |
| 1 | `Plugin` |
| 2 | `Renderer` |
| 3 | `Library` |

## CollisionFlags

Disc'lerin çarpışma gruplarını tanımlar. Bit flag, `|` ile birleşir.

| Bit | İsim | Açıklama |
|---|---|---|
| 1 | `ball` | Top |
| 2 | `red` | Kırmızı takım disc'i |
| 4 | `blue` | Mavi takım disc'i |
| 8 | `redKO` | Kırmızı kickoff sırasında aktif |
| 16 | `blueKO` | Mavi kickoff sırasında aktif |
| 32 | `wall` | Duvar |
| 64 | `kick` | Tekme grubu (oyuncu disc'leri) |
| 128 | `score` | Skor flag'i |
| 256 ... 134217728 | `free1` ... `free20` | Custom kullanım için 20 boş slot (2^8 ... 2^27) |
| 268435456 | `c0` | Custom 0 (2^28) |
| 536870912 | `c1` | Custom 1 (2^29) |
| 1073741824 | `c2` | Custom 2 (2^30) |
| -2147483648 | `c3` | Custom 3 (sign bit, 2^31) |

`cMask` "ben hangi gruplarla çarpışırım", `cGroup` "ben hangi gruptanım". İki disc çarpışır eğer her ikisinin grup-mask AND işlemi sıfırdan farklı bir bit verirse.

## Direction

Input yönü. Sadece üç değer var, hem X hem Y ekseni için aynısı kullanılır.

| Değer | İsim |
|---|---|
| -1 | `Backward` |
| 0 | `Still` |
| 1 | `Forward` |

Pratikte yön hesapları `Math.sign(delta)` ile yapılır, sonuç direkt -1/0/+1 olur.

## CameraFollow

Stadium içinde tanımlı kamera davranışı:

| Değer | İsim |
|---|---|
| 0 | `None` |
| 1 | `Player` |

## BackgroundType

Stadium arka plan texture'ı:

| Değer | İsim |
|---|---|
| 0 | `None` |
| 1 | `Grass` |
| 2 | `Hockey` |

## GamePlayState

Maç içi alt durumlar:

| Değer | İsim | Anlam |
|---|---|---|
| 0 | `BeforeKickOff` | Maç başı, top'a daha vurulmadı |
| 1 | `Playing` | Aktif oyun |
| 2 | `AfterGoal` | Gol sonrası kısa duraklama |
| 3 | `Ending` | Maç son saniyeleri |

## PlayerPositionInGame

Oyuncunun maç içindeki **futbol mevkii** (futsal/football manager modu için). Spec ile player ayrımı için bu enum değil, `player.team` (`Team.spec` / `red` / `blue`) kullanılır.

| Değer | İsim | Anlam |
|---|---|---|
| 0 | `None` | Mevki atanmamış |
| 1 | `GK` | Goalkeeper (kaleci) |
| 2 | `SW` | Sweeper (libero) |
| 3 | `WBL` | Wing-back left |
| 4 | `DL` | Defender left |
| 5 | `DC` | Defender center |
| 6 | `DR` | Defender right |
| 7 | `WBR` | Wing-back right |
| 8 | `DML` | Defensive midfielder left |
| 9 | `DMC` | Defensive midfielder center |
| 10 | `DMR` | Defensive midfielder right |
| 11 | `ML` | Midfielder left |
| 12 | `MC` | Midfielder center |
| 13 | `MR` | Midfielder right |
| 14 | `AML` | Attacking midfielder left |
| 15 | `AMC` | Attacking midfielder center |
| 16 | `AMR` | Attacking midfielder right |
| 17 | `FL` | Forward left |
| 18 | `FC` | Forward center |
| 19 | `FR` | Forward right |
| 20 | `ST` | Striker |

## BanEntryType

Ban entry'sinin neye dayandığı:

| Değer | İsim | Anlam |
|---|---|---|
| 0 | `Player` | Anlık player id'sine bağlı (oyuncu odadayken) |
| 1 | `IP` | IP veya IP range (IPv4 ya da IPv6) |
| 2 | `Auth` | Auth string |

`IP` tipinin `value` alanı IPv4 ise `NumericIPv4`, IPv6 ise `NumericIPv6` (BigInt) olur. Range için ayrı `IPv4Range` / `IPv6Range` type'ları kullanılır. `addPlayerBan`, `addIpBan`, `addAuthBan` farklı tipler üretir.

## ErrorCodes

`Errors.HBError`'ün `.code` alanı. Tam liste için [errors.md](errors.md).
