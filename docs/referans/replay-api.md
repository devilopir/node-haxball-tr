# Replay API referansı

`Replay` namespace, `.hbr` dosyalarını okuma, düzenleme ve yazma işlevini sağlar.

Pratik kullanım: [rehber/13-replay.md](../rehber/13-replay.md). Tutorial: [tutoriallar/08-replay-analizi.md](../tutoriallar/08-replay-analizi.md).

```javascript
const { Replay } = require("node-haxball")();
```

## Replay.readAll(uint8Array)

```
Replay.readAll(uint8Array: Uint8Array): ReplayData
```

Tüm replay dosyasını belleğe yükler, parse edilmiş `ReplayData` döner. Boyut sınırı yok (büyük replay'lerde RAM'e dikkat).

```javascript
const bytes = fs.readFileSync("./match.hbr");
const replay = Replay.readAll(new Uint8Array(bytes));
console.log(`${replay.totalFrames} frame, ${replay.goalMarkers.length} gol`);
```

## Replay.read(uint8Array, callbacks, options)

```
Replay.read(
  uint8Array: Uint8Array,
  callbacks: CommonlyUsedCallbacks,
  options: ReplayReadOptions
): AsyncReplayReader
```

Non-blocking oynatıcı. Event'ler callback'lerle gerçek zamanda akar (oynatma kontrolü ile). Çoğu zaman `readAll` yeterli; `read` özellikle sandbox'ta replay oynatmak istediğinde devreye girer.

`callbacks`: Plugin/Renderer'da kullanılan `CommonlyUsedCallbacks` setini destekler, `onTeamGoal`, `onPlayerBallKick`, `onGameTick`, vb.

`options`:

- `requestAnimationFrame`: Custom RAF (default: kütüphane sağlar)
- `cancelAnimationFrame`: Custom CAF

Dönen `AsyncReplayReader` ile play/pause/seek kontrolü yapılır (detaylı dokümantasyon kütüphanenin kendi wiki'sinde).

## Replay.writeAll(replayData)

```
Replay.writeAll(replayData: ReplayData): Uint8Array
```

`ReplayData` objesini `.hbr` binary'ye serialize eder. Trim ya da edit sonrası diske kaydetmek için.

```javascript
const bytes = Replay.writeAll(replay);
fs.writeFileSync("./trimmed.hbr", Buffer.from(bytes));
```

## Replay.trim(replayData, params?)

```
Replay.trim(
  replayData: ReplayData,
  params?: { beginFrameNo?: int, endFrameNo?: int }
): void
```

Replay'i in-place trimler. `beginFrameNo` default 0, `endFrameNo` default `totalFrames - 1`. İkisi de inclusive.

```javascript
const replay = Replay.readAll(bytes);
Replay.trim(replay, { beginFrameNo: 0, endFrameNo: 3600 }); // ilk 60 saniye
const out = Replay.writeAll(replay);
```

## Replay.trimAsync(replayData, params?)

```
Replay.trimAsync(replayData, params?): Promise<void>
```

`trim`'in async versiyonu. Çok uzun replay'lerde ana event loop'u bloklamamak için.

```javascript
await Replay.trimAsync(replay, { beginFrameNo: 1000, endFrameNo: 5000 });
```

## ReplayData yapısı

```typescript
class ReplayData {
  roomData: RoomState;                            // başlangıç state
  events: (HaxballEvent | { frameNo: int })[];    // sıralı event listesi
  goalMarkers: { teamId: int, frameNo: int }[];   // gol işaretleri
  totalFrames: int;
  version: int;                                   // şu an v3
}
```

### `roomData`

Replay başlangıcındaki tam state. Stadium, oyuncular, skor (genelde 0), takım dağılımı.

### `events`

Tüm event'ler kronolojik. Her event'in `frameNo` alanı vardır. Replay'i yeniden oynatırken bu listenin sırasına göre işle.

Eventless frame'ler için `{ frameNo: int }` formatında boş entry yer alabilir, bazı tick'lerde sadece zaman geçer, hiç event olmaz.

### `goalMarkers`

Her gol için `teamId` ve `frameNo`. Skor sıralamasını çıkarmak için pratik. Gol sahibi/asisti **bu yapıda yok**, onun için event listesini taramak veya replay'i sandbox'ta oynatmak gerekir.

### `totalFrames`

Toplam tick sayısı. 60 ile böl: saniye cinsinden süre.

### `version`

Sadece `3` destekleniyor. Eski Haxball replay'leri (v1/v2) parse edilemez.

## Replay üzerinde özel işlemler

Yeni event eklemek için `events`'e push yapıp `frameNo`'yu set et:

```javascript
const announce = EventFactory.sendAnnouncement("Maç Başlıyor", 0xFFFFFF, 1, 2);
announce.frameNo = 0;
replay.events.unshift(announce);
```

Sırayı korumak için `events.sort((a, b) => a.frameNo - b.frameNo)` lazımdır.

Event silmek:

```javascript
replay.events = replay.events.filter((e) => e.type !== OperationType.SendChat);
```

`totalFrames` ve `goalMarkers` event sayısıyla otomatik eşitlenmez, manuel düzeltme gerekir bu tip filtre sonrası.

## Sınırlar

- **Sadece v3**: Eski format desteklenmiyor
- **Boyut**: `readAll` belleğe alır, çok uzun (saatlik) replay'lerde sıkıntılı olabilir
- **Şifreli replay yok**: Haxball replay'i şifrelemez, bot da etmez
- **Custom event'ler**: `customEvent` event'leri replay'de yer alır ama resmi Haxball client tarafından gösterilmez
