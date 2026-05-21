# Replay

Haxball, oyunun tüm event'lerini `.hbr` formatında byte dizisi olarak kaydedebilir. `node-haxball`, bu dosyaları okuyup analiz etmenin (ve düzenlemenin) tam API'sini sunar.

## Kayıt almak

```javascript
room.startRecording();         // kayıt başlat, true/false döner (zaten kayıttaysa false)
const bytes = room.stopRecording();  // kayıt durdur, Uint8Array verir
room.isRecording();            // boolean
```

Kayıt başladığı andan durduğu ana kadar olan her event yakalanır. Maç başlamadan kayıt başlatabilirsin, boş bir oda kaydı da geçerli .hbr olur, sadece içeride event yoktur.

```javascript
const fs = require("fs");

room.startRecording();
room.onGameStart = () => {
  console.log("kayıt başladı, oyun başladı");
};

// Maç bitince diske yaz
room.onGameEnd = (winningTeamId) => {
  const bytes = room.stopRecording();
  if (bytes) {
    const filename = `replays/${Date.now()}_team${winningTeamId}_won.hbr`;
    fs.writeFileSync(filename, Buffer.from(bytes));
    room.startRecording(); // bir sonraki maç için
  }
};
```

## Replay yapısı

`.hbr` dosyası bir `ReplayData` objesinin binary temsilidir:

```typescript
ReplayData {
  roomData: RoomState;                      // başlangıç state (stadium, oyuncular...)
  events: (HaxballEvent | {frameNo: int})[]; // sıralı event listesi
  goalMarkers: {teamId, frameNo}[];          // gol işaretleri
  totalFrames: int;                          // toplam tick sayısı (60 tick = 1 saniye)
  version: int;                              // şu an sadece 3 destekleniyor
}
```

`frameNo` her tick'i tek tek sayar. 60 fps maç ise 1 saniyelik replay 60 frame'dir.

## readAll, tüm replay'i belleğe yükle

Replay'i analiz etmek için:

```javascript
const fs = require("fs");
const { Replay } = require("node-haxball")();

const bytes = fs.readFileSync("./replays/match1.hbr");
const replay = Replay.readAll(new Uint8Array(bytes));

console.log(`Toplam frame: ${replay.totalFrames}`);
console.log(`Toplam süre: ${(replay.totalFrames / 60).toFixed(1)}s`);
console.log(`Gol sayısı: ${replay.goalMarkers.length}`);
console.log(`Oyuncu sayısı: ${replay.roomData.players.length}`);

for (const goal of replay.goalMarkers) {
  const minute = Math.floor(goal.frameNo / 60 / 60);
  const second = Math.floor(goal.frameNo / 60) % 60;
  const team = goal.teamId === 1 ? "Red" : "Blue";
  console.log(`${minute}:${second.toString().padStart(2, "0")} ${team} gol`);
}
```

Bu sayfa kadar bilgiyle replay'den gol özetleri, takım kazanma istatistikleri çıkarabilirsin. Detaylı bir tutorial [tutoriallar/08-replay-analizi.md](../tutoriallar/08-replay-analizi.md) içinde.

## read, non-blocking, callback'li okuma

Büyük replay'lerde tüm event'leri belleğe yüklemek yerine, oyun yeniden oynatılıyormuş gibi callback'lerle adım adım okumak isteyebilirsin:

```javascript
const reader = Replay.read(bytes, {
  onGameTick: () => {
    // her tick için (60/s)
  },
  onTeamGoal: (teamId, goalId, goal, ballDiscId, ballDisc) => {
    console.log(`gol: ${teamId}`);
  },
  onPlayerJoin: (player) => {
    console.log(`girdi: ${player.name}`);
  },
}, {
  requestAnimationFrame: null,  // default
  cancelAnimationFrame: null,
});
```

`reader` bir `AsyncReplayReader` döndürür. Bunu kontrol etmek için play/pause/seek methodları bulunur, tipik kullanım sandbox modunda replay'i "oynatmak". Sadece veri çıkarmak istiyorsan `readAll` daha basit.

## Trim, replay kesme

Bir maçın sadece "ilk yarısını" ya da "son 30 saniyeyi" istiyorsan:

```javascript
const replay = Replay.readAll(bytes);

// Sadece son 1800 frame (30 saniye)
Replay.trim(replay, {
  beginFrameNo: replay.totalFrames - 1800,
  endFrameNo: replay.totalFrames - 1,
});

const trimmedBytes = Replay.writeAll(replay);
fs.writeFileSync("./replays/highlight.hbr", Buffer.from(trimmedBytes));
```

`trim` senkronon, `trimAsync` ise Promise döner, çok uzun replay'ler için async tercih edilir, ana event loop'u bloklamaz.

## writeAll, `.hbr` dosyası üretme

```javascript
const bytes = Replay.writeAll(replayData);
fs.writeFileSync("./out.hbr", Buffer.from(bytes));
```

`readAll` ile parse edilmiş bir replay'i tekrar binary'e çevirir. Replay üzerinde değişiklik yaptıktan sonra (trim, event silme/ekleme) dosyayı yeniden kaydetmek için.

Custom event'ler ekleyerek "replay'i editlemek" teorik olarak mümkün ama pratik kullanımı az; Haxball client'ı standart event'leri bekler, yapısı bozulmuş bir replay oynatılırken hata verir.

## Periyodik kayıt rotasyonu

Botunun her maçı kaydetmesini istiyorsan tipik kalıp:

```javascript
const RECORDS_DIR = "./replays";
fs.mkdirSync(RECORDS_DIR, { recursive: true });

room.onAfterRoomLink = () => {
  if (!room.isRecording()) room.startRecording();
};

room.onGameEnd = (winningTeamId) => {
  const bytes = room.stopRecording();
  if (bytes) {
    const stamp = new Date().toISOString().replace(/[:.]/g, "-");
    const file = `${RECORDS_DIR}/${stamp}.hbr`;
    fs.writeFileSync(file, Buffer.from(bytes));
    console.log(`Replay kaydedildi: ${file}`);
  }
  // Bir sonraki maç için yeni kayıt
  setTimeout(() => room.startRecording(), 1000);
};
```

Disk tüketimine dikkat, yoğun bir oda günde gigabytes alabilir. Eski replay'leri silen cron veya bot içi bir cleanup gerekir.

## Sınırlar

- `version`: Sadece v3 destekleniyor. Eski replay dosyaları parse edilemez.
- Stream watcher ile karıştırma: `Room.streamWatcher`, replay değil, canlı binary stream içindir.
- `roomData` snapshot'tır, replay başlangıcındaki state'tir. Replay süresince state değişikliklerini events'ten manuel uygulaman gerekir.
- `goalMarkers` sadece gollerin meta'sıdır; gol sahibi/asisti yoktur, bunu çıkarmak için event listesini tarayıp top sahibini takip etmen gerekir. Pratiği [tutoriallar/08-replay-analizi.md](../tutoriallar/08-replay-analizi.md) içinde.
