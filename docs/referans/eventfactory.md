# EventFactory referansı

`EventFactory` namespace, Haxball protokolündeki düşük seviyeli event mesajlarını üretmek için kullanılır. Üretilen `HaxballEvent` objesi `room.executeEvent(event, byId)` ile tetiklenir.

Pratik kullanım rehberi: [rehber/14-eventfactory-ve-protokol.md](../rehber/14-eventfactory-ve-protokol.md). Çoğu bot bu API'yi kullanmaz; üst seviye `room.startGame()` çağrısı yeterlidir.

```javascript
const { EventFactory } = require("node-haxball")();
```

## Generic factory

### EventFactory.create(type)

```
EventFactory.create(type: OperationType): HaxballEvent | undefined
```

Verilen `OperationType` için boş bir event üretir. Alanlar manuel doldurulur. Spesifik factory fonksiyonları daha pratiktir.

### EventFactory.createFromStream(reader)

```
EventFactory.createFromStream(reader: Impl.Stream.Reader): HaxballEvent
```

Binary stream'den (replay parsing, networking debug için) tek event okur.

## Spesifik factory fonksiyonları

Her event tipi için fonksiyon mevcut. Hepsi serialize edilmiş `HaxballEvent` objesi döner.

### Maç kontrolü

- `startGame(): StartGameEvent`
- `stopGame(): StopGameEvent`
- `pauseResumeGame(paused: boolean): PauseResumeGameEvent`
- `setScoreLimit(value: int): SetLimitEvent`
- `setTimeLimit(value: int): SetLimitEvent`
- `setStadium(stadium: Impl.Stadium.Stadium): SetStadiumEvent`

### Oyuncu

- `joinRoom(id, name, flag, avatar, conn, auth): JoinRoomEvent`
- `kickBanPlayer(id, reason, ban): KickBanPlayerEvent`
- `setPlayerTeam(playerId, teamId): SetPlayerTeamEvent`
- `setPlayerAdmin(playerId, value): SetPlayerAdminEvent`
- `setPlayerSync(value: boolean): SetPlayerSyncEvent`
- `setPlayerIdentity(id, data): IdentityEvent`
- `autoTeams(): AutoTeamsEvent`
- `setTeamsLock(value): SetTeamsLockEvent`
- `reorderPlayers(playerIdList, moveToTop): ReorderPlayersEvent`

### Mesajlaşma

- `sendChat(msg: string): SendChatEvent`
- `sendAnnouncement(msg, color, style, sound): SendAnnouncementEvent`
- `sendChatIndicator(active: uint8): SendChatIndicatorEvent`

### Input

- `sendInput(input: uint32): SendInputEvent`

### Avatar ve görsel

- `setAvatar(value: string): SetAvatarEvent`
- `setHeadlessAvatar(id, avatar): SetHeadlessAvatarEvent`
- `setTeamColors(teamId, colors): SetTeamColorsEvent`

### Disc

- `setDiscProperties(id, data): SetDiscPropertiesEvent`
- `setPlayerDiscProperties(id, data): SetDiscPropertiesEvent`

### Ayarlar

- `setKickRateLimit(min, rate, burst): SetKickRateLimitEvent`

### Protokol

- `ping(pings: int32[]): PingEvent`
- `checkConsistency(data: ArrayBuffer): ConsistencyCheckEvent`

### Custom

- `customEvent(type: uint32, data: object): CustomEvent`
- `binaryCustomEvent(type: uint32, data: Uint8Array): BinaryCustomEvent`

`customEvent` JSON serileştirilebilir veri için, `binaryCustomEvent` raw byte için. Sadece `node-haxball` kullanan istemciler bu mesajları alır; resmi Haxball client görmezden gelir.

## Event'i tetiklemek

```javascript
const event = EventFactory.startGame();
room.executeEvent(event, byId);
```

`byId` eylemi tetikleyen oyuncunun id'si. Host kendi tetiklediyse `0`.

Spesifik bir oyuncuya hedeflenmiş custom event:

```javascript
const event = EventFactory.customEvent(1000, { hello: "world" });
room.executeEventWithTarget(event, targetPlayerId);
```

## HaxballEvent yapısı

Her event tipinin kendi alanları vardır ama ortak yapı:

```typescript
interface HaxballEvent {
  type: OperationType;
  byId?: uint16;                // tetikleyen oyuncu id
  frameNo?: int;                // replay parsing'de set olur
  // tip-spesifik alanlar
}
```

Replay editing senaryolarında `frameNo`'yu manuel set etmen gerekir:

```javascript
const announce = EventFactory.sendAnnouncement("Maç Başlıyor", 0xFFFFFF, 1, 2);
announce.frameNo = 0;
replay.events.unshift(announce);
```

## onBeforeOperationReceived ile etkileşim

Bir event ağdan geldiğinde önce `onBeforeOperationReceived(type, msg, globalFrameNo, clientFrameNo)` callback'i çağırılır. `false` döndürmek event'i sessizce reddeder:

```javascript
room.onBeforeOperationReceived = (type, msg, gFrame, cFrame) => {
  if (type === OperationType.SendChat && containsBannedWord(msg.msg)) {
    return false; // mesajı protokol seviyesinde engelle
  }
};
```

Bu düşük seviye filtre; `onBeforePlayerChat` ile aynı işi daha az tehlikeli yapabilirsin. EventFactory + executeEvent + onBeforeOperationReceived üçlüsü, normal API yetersiz kaldığında protokole müdahale için.

## Sandbox'ta event'leri "fake" tetiklemek

Sandbox modunda kütüphane test için `FakeEventTriggers` interface'ini sunar. `room.fakeXxx` benzeri metodlarla event'leri yokmuş gibi (network'a gitmeden) tetikleyip handler'ları çalıştırabilirsin. Daha çok unit test için.
