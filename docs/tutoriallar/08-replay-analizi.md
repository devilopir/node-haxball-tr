# Replay analizi

Saatlerce maç oynanan bir odadan birikmiş `.hbr` dosyalarını manuel açmak verimsiz. Bu tutorial bir replay'in event'lerini parse edip "en çok kim gol atmış", "kim asistçi", "her maç ortalama kaç dakika sürmüş" gibi sorulara cevap üretir.

[rehber/13-replay.md](../rehber/13-replay.md) replay API'sinin nasıl çalıştığını anlatıyor; o bilinerek geliniyor.

## Senaryo

`./replays/` klasörü dolu. Her dosya bir maç. Hepsini gez, sonuçları bir CSV'ye dök, sonra başka bir araçla (Excel, sqlite, Pandas) analiz et.

## Tam kod

`analyze.js`:

```javascript
const fs = require("fs");
const path = require("path");
const { Replay, OperationType } = require("node-haxball")();

const REPLAYS_DIR = "./replays";
const OUTPUT_CSV = "./analysis.csv";

const files = fs.readdirSync(REPLAYS_DIR).filter((f) => f.endsWith(".hbr"));
console.log(`${files.length} replay bulundu`);

const rows = [];

for (const file of files) {
  console.log(`İşleniyor: ${file}`);
  try {
    const bytes = fs.readFileSync(path.join(REPLAYS_DIR, file));
    const replay = Replay.readAll(new Uint8Array(bytes));
    const analysis = analyzeReplay(replay);
    rows.push({
      file,
      ...analysis,
    });
  } catch (err) {
    console.error(`  HATA: ${err.message}`);
  }
}

// CSV yaz
const headers = ["file", "duration_s", "total_goals", "red_score", "blue_score", "winner", "players", "top_scorer"];
const lines = [headers.join(",")];
for (const row of rows) {
  lines.push(headers.map((h) => csvCell(row[h])).join(","));
}
fs.writeFileSync(OUTPUT_CSV, lines.join("\n"));
console.log(`Çıktı: ${OUTPUT_CSV}`);

function analyzeReplay(replay) {
  const duration_s = Math.round(replay.totalFrames / 60);
  const total_goals = replay.goalMarkers.length;

  // Red ve blue gol sayısı
  let red_score = 0;
  let blue_score = 0;
  for (const goal of replay.goalMarkers) {
    if (goal.teamId === 1) red_score++;
    else if (goal.teamId === 2) blue_score++;
  }

  const winner = red_score > blue_score ? "Red"
    : blue_score > red_score ? "Blue"
    : "Draw";

  // Başlangıç state'inden oyuncuları al
  const initialPlayers = replay.roomData.players ?? [];
  const playerNames = initialPlayers
    .filter((p) => p.id !== 0)
    .map((p) => p.name)
    .join("|");

  // Event'leri tara: gol/asist/owngol
  const scorers = trackScorers(replay);
  const topScorer = [...scorers.entries()]
    .sort((a, b) => b[1] - a[1])[0];

  return {
    duration_s,
    total_goals,
    red_score,
    blue_score,
    winner,
    players: playerNames,
    top_scorer: topScorer ? `${topScorer[0]}(${topScorer[1]})` : "",
  };
}

function trackScorers(replay) {
  const scorers = new Map();
  let lastKicker = null; // playerId

  for (const event of replay.events) {
    if (!event.type) continue;

    if (event.type === OperationType.SendInput) {
      // input event'leri ignore edilir
    } else if (event.byId !== undefined && event.type === OperationType.SendChat) {
      // chat events
    }

    // Bu basit implementasyon, gerçek replay'lerde gol event'i için
    // CustomEvent ya da Set türü event'lere bakmak gerekebilir.
    // Pratik analiz için goalMarkers + initialPlayers genelde yeterli.
  }

  // goalMarkers her gol için frameNo veriyor; gol sahibi
  // ayrı bir event olarak yer almıyor, top sahibi tahminiyle
  // tespit edilir. Bu, sandbox/replay simülasyonu gerektirir.
  return scorers;
}

function csvCell(v) {
  if (v == null) return "";
  const s = String(v);
  if (s.includes(",") || s.includes('"') || s.includes("\n")) {
    return `"${s.replace(/"/g, '""')}"`;
  }
  return s;
}
```

## Gol sahibi tespitinin pratik yolu

`goalMarkers` sadece "kim takım kaç frame'inde gol attı"yı verir, kim attı bilgisini vermez. Asıl gol sahibi tespiti için iki seçenek:

**1. Replay'i oynatıp top sahibi takibi**

`Replay.read` bir `AsyncReplayReader` döndürür. Reader üzerinde `reader.state` anlık `RoomState`'i verir; `setSpeed`, `setTime`, `setCurrentFrameNo` ile oynatma kontrol edilir.

```javascript
const reader = Replay.read(bytes, {
  onPlayerBallKick: (playerId) => {
    prevLastKicker = lastKicker;
    lastKicker = playerId;
  },
  onTeamGoal: (teamId) => {
    const scorerId = lastKicker;
    const player = reader.state.players.find((p) => p.id === scorerId);
    if (player) {
      goals.push({ teamId, scorer: player.name, frame: reader.getCurrentFrameNo() });
    }
  },
}, {});

// Sonuna kadar oynat (maxFrameNo'ya seek et)
reader.setCurrentFrameNo(reader.maxFrameNo);
```

`setCurrentFrameNo`/`setTime` senkron çağrılmış görünür ama içinde event'leri tek tek işler. Callback'ler bu işlemin parçası olarak tetiklenir.

**2. Sadece `goalMarkers` ile kabaca analiz**

İlk versiyonda olduğu gibi, takım skorları ve maç süresi çıkarılabilir ama bireysel gol/asist bilgisi yoktur. Çoğu istatistik için bu yeterli, "geçen ay 200 maç oynanmış, ortalama 5.2 gol, %52'sini red kazanmış" gibi metalar bunla çıkar.

İkisinin arasında tercih ediyorsan: özet rapor için #2, oyuncu profili için #1. Bu tutorial'da #2 ile başladık çünkü daha basit ve `Replay.readAll` kullanır.

## Sandbox ile detaylı analiz

`Replay.read` ile bir oynatıcı yaratıp event'leri callback'lerle takip etmek tam analiz için en güvenilir yol:

```javascript
const fs = require("fs");
const path = require("path");
const { Replay, Utils } = require("node-haxball")();

function deepAnalyze(filePath) {
  const bytes = fs.readFileSync(filePath);

  let lastKicker = null;
  let prevLastKicker = null;
  const goals = [];
  const assists = new Map();
  const goalCount = new Map();

  const reader = Replay.read(new Uint8Array(bytes), {
    onPlayerBallKick: (playerId) => {
      if (lastKicker !== playerId) {
        prevLastKicker = lastKicker;
      }
      lastKicker = playerId;
    },
    onTeamGoal: (teamId) => {
      if (lastKicker !== null) {
        const player = reader.state.players.find((p) => p.id === lastKicker);
        if (player) {
          goals.push({ teamId, scorer: player.name, frame: reader.getCurrentFrameNo() });
          goalCount.set(player.name, (goalCount.get(player.name) ?? 0) + 1);

          if (prevLastKicker !== null && prevLastKicker !== lastKicker) {
            const assister = reader.state.players.find((p) => p.id === prevLastKicker);
            if (assister) {
              assists.set(assister.name, (assists.get(assister.name) ?? 0) + 1);
            }
          }
        }
      }
    },
  }, {});

  // Sonuna kadar seek et; callback'ler bu sırada tetiklenir
  reader.setCurrentFrameNo(reader.maxFrameNo);
  reader.destroy?.();

  return { goals, goalCount, assists };
}
```

Reader'ın tam metod listesi `Replay.AsyncReplayReader`: `state`, `gameState`, `maxFrameNo`, `extrapolate`, `getSpeed`, `setSpeed`, `getTime`, `setTime`, `getCurrentFrameNo`, `setCurrentFrameNo`, `length`, `destroy`. Tam imzalar için repo'nun [examples/api_structure/replayReader.js](https://github.com/wxyz-abcd/node-haxball/blob/main/examples/api_structure/replayReader.js) örneği ve `src/index.d.ts:8703+`.

## CSV çıktısı üzerinde sorgu

`analysis.csv` üretildikten sonra sqlite ile sorgu:

```sh
sqlite3 :memory: <<EOF
.mode csv
.import analysis.csv replays
SELECT
  COUNT(*) as match_count,
  AVG(duration_s) as avg_duration,
  AVG(total_goals) as avg_goals,
  SUM(CASE WHEN winner = 'Red' THEN 1 ELSE 0 END) as red_wins,
  SUM(CASE WHEN winner = 'Blue' THEN 1 ELSE 0 END) as blue_wins
FROM replays;
EOF
```

Veya Pandas:

```python
import pandas as pd
df = pd.read_csv("analysis.csv")
print(df.describe())
print(df.groupby("winner").size())
```

## Genişletmek

**Toplu oyuncu istatistiği**: Tüm replay'leri gezdikten sonra oyuncu bazlı toplam çıkar. Her replay'in `goals` listesini birleştir, `groupBy(scorer)` yap.

**Maç eğilimi**: Hangi saatte daha çok oynanmış, hangi gün daha rekabetçi (skor farkı küçük), dosya adındaki timestamp'lerden hesaplanabilir.

**Gol haritası**: `onPlayerBallKick`'ten önceki birkaç frame'de top'un pozisyonunu kaydet, hangi noktalardan gol atılmış görselleştir. ASCII heatmap veya gerçek bir görselleştirme kütüphanesi ile.

## Doğrulama

- `Replay.readAll(uint8Array): ReplayData`, line 8894
- `Replay.read(uint8Array, callbacks, options): AsyncReplayReader`, line 8854
- `replay.goalMarkers`, line 8875
- `replay.totalFrames`, line 8880
- `replay.roomData`, line 8862 (RoomState)
- `OperationType.SendInput` ve diğer enum değerleri, index.d.ts line 15-205

Replay analizi statik veri üzerinde çalıştığı için canlı bot bağlantısı gerekmez. Birkaç gerçek `.hbr` dosyası elinde varsa hemen test edebilirsin.
