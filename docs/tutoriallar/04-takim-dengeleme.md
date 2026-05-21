# Takım dengeleme

`autoTeams()` ve `randTeams()` Haxball'un kendi takım dağıtım metodlarıdır ama oyuncu becerisini umursamazlar. Bu tutorial üç farklı dengeleme stratejisini implement eder: basit oda rotasyonu, ELO temelli, ve "kazanan kalır" (winner stays).

## Üç strateji

**1. Skor temelli rotasyon**: Maç bittiğinde kazanan takım dağılır, kaybeden kalır. Kaybeden oyuncuların moralini bozmasın diye değil, "kazanan değişsin" mantığında. Discord toplulukları için tipik kullanım.

**2. ELO**: Her oyuncuya bir skor verilir, maç sonucu bu skoru günceller. Yeni maç başlarken oyuncular ELO toplamı eşit olan iki takıma bölünür. Adil ama her oyuncunun ELO'su olmayan sezon başlangıçlarında işe yaramaz.

**3. Winner stays**: Kazanan takım sabit kalır, kaybedenler dışarı atılır, sıradan oyuncular alınır. Klasik streetfight modu, sıraya girmiş oyuncularla işler.

Bu tutorial üçünü de yazıyor, ortak iskelet aynı, fark fonksiyon mantığında.

## Tam kod

`balance-bot.js`:

```javascript
const fs = require("fs");
const { Room, Utils } = require("node-haxball")();

const TOKEN = "thr1.AAAAA...";
const MIN_PLAYERS = 2;
const MAX_TEAM_SIZE = 4;
const STRATEGY = "elo"; // "rotation" | "elo" | "winner_stays"
const ELO_FILE = "./elo.json";

// ELO state
const elo = new Map(); // auth → ELO score
const DEFAULT_ELO = 1000;
const K = 32;

function loadElo() {
  if (!fs.existsSync(ELO_FILE)) return;
  const data = JSON.parse(fs.readFileSync(ELO_FILE, "utf8"));
  for (const [auth, score] of Object.entries(data)) elo.set(auth, score);
}

function saveElo() {
  fs.writeFileSync(ELO_FILE, JSON.stringify(Object.fromEntries(elo), null, 2));
}

function getElo(player) {
  return elo.get(player.auth) ?? DEFAULT_ELO;
}

function setElo(auth, score) {
  elo.set(auth, score);
}

function updateElo(redPlayers, bluePlayers, redScore, blueScore) {
  // Standart Elo: takım ortalamasıyla
  const redAvg = avg(redPlayers.map(getElo));
  const blueAvg = avg(bluePlayers.map(getElo));
  const redExpected = 1 / (1 + Math.pow(10, (blueAvg - redAvg) / 400));
  const blueExpected = 1 - redExpected;
  const redActual = redScore > blueScore ? 1 : redScore < blueScore ? 0 : 0.5;
  const blueActual = 1 - redActual;
  for (const p of redPlayers) {
    setElo(p.auth, Math.round(getElo(p) + K * (redActual - redExpected)));
  }
  for (const p of bluePlayers) {
    setElo(p.auth, Math.round(getElo(p) + K * (blueActual - blueExpected)));
  }
  saveElo();
}

function avg(arr) {
  return arr.length ? arr.reduce((a, b) => a + b, 0) / arr.length : 0;
}

// Dengeleme stratejileri

function rebalanceRotation(room, lastWinner) {
  const playing = room.players.filter((p) => p.id !== 0 && p.team.id !== 0);
  const queue = room.players.filter((p) => p.id !== 0 && p.team.id === 0);

  // Kazananı dağıt
  const winners = playing.filter((p) => p.team.id === lastWinner);
  const losers = playing.filter((p) => p.team.id !== lastWinner);

  const all = [...losers, ...winners, ...queue];
  all.sort(() => Math.random() - 0.5);
  fillTeams(room, all);
}

function rebalanceElo(room) {
  const pool = room.players.filter((p) => p.id !== 0).slice();
  pool.sort((a, b) => getElo(b) - getElo(a));

  const teamSize = Math.min(MAX_TEAM_SIZE, Math.floor(pool.length / 2));
  const red = [];
  const blue = [];

  // Snake draft: en iyiler dönüşümlü
  for (let i = 0; i < teamSize * 2; i++) {
    if (i % 4 < 2) red.push(pool[i]);
    else blue.push(pool[i]);
  }

  red.forEach((p) => room.setPlayerTeam(p.id, 1));
  blue.forEach((p) => room.setPlayerTeam(p.id, 2));
  pool.slice(teamSize * 2).forEach((p) => room.setPlayerTeam(p.id, 0));
}

function rebalanceWinnerStays(room, lastWinner) {
  const playing = room.players.filter((p) => p.id !== 0 && p.team.id !== 0);
  const queue = room.players.filter((p) => p.id !== 0 && p.team.id === 0);
  const winners = playing.filter((p) => p.team.id === lastWinner);
  const losers = playing.filter((p) => p.team.id !== lastWinner);

  // Kazanan takımı koru, kaybedenleri spec'e at
  losers.forEach((p) => room.setPlayerTeam(p.id, 0));

  // Spec'tekileri kaybedenin yerine al, en uzun bekleyene öncelik
  const needed = winners.length;
  const newTeamId = lastWinner === 1 ? 2 : 1;
  for (let i = 0; i < Math.min(needed, queue.length); i++) {
    room.setPlayerTeam(queue[i].id, newTeamId);
  }
}

function fillTeams(room, list) {
  const teamSize = Math.min(MAX_TEAM_SIZE, Math.floor(list.length / 2));
  list.slice(0, teamSize).forEach((p) => room.setPlayerTeam(p.id, 1));
  list.slice(teamSize, teamSize * 2).forEach((p) => room.setPlayerTeam(p.id, 2));
  list.slice(teamSize * 2).forEach((p) => room.setPlayerTeam(p.id, 0));
}

// Bot
loadElo();

Room.create({
  name: `TR | Dengeli Maç (${STRATEGY})`,
  token: TOKEN,
  maxPlayerCount: 12,
  noPlayer: true,
  showInRoomList: true,
  geo: { code: "tr", lat: 41, lon: 29 },
}, {
  storage: { player_name: "host" },
  onOpen: (room) => attachLogic(room),
  onClose: (reason) => process.exit(0),
});

function attachLogic(room) {
  room.onAfterRoomLink = (link) => {
    console.log("link:", link);
    const big = Utils.getDefaultStadiums().find((s) => s.name === "Big");
    if (big) room.setCurrentStadium(big);
    room.setScoreLimit(3);
    room.setTimeLimit(0);
  };

  room.onPlayerJoin = (player) => {
    ensureAdmin(room);
    tryStart(room);
  };

  room.onPlayerLeave = () => ensureAdmin(room);

  room.onGameEnd = (winningTeamId) => {
    // ELO güncelle
    if (STRATEGY === "elo") {
      const red = room.players.filter((p) => p.id !== 0 && p.team.id === 1);
      const blue = room.players.filter((p) => p.id !== 0 && p.team.id === 2);
      const redScore = room.redScore ?? 0;
      const blueScore = room.blueScore ?? 0;
      updateElo(red, blue, redScore, blueScore);
    }

    setTimeout(() => {
      if (STRATEGY === "rotation") rebalanceRotation(room, winningTeamId);
      else if (STRATEGY === "elo") rebalanceElo(room);
      else if (STRATEGY === "winner_stays") rebalanceWinnerStays(room, winningTeamId);
      tryStart(room);
    }, 3000);
  };

  room.onPlayerChat = (id, message) => {
    if (message === "!elo") {
      const player = room.getPlayer(id);
      const score = getElo(player);
      room.sendAnnouncement(`Senin ELO: ${score}`, id, 0xCCCCFF, "small", 0);
      return false;
    }
    if (message === "!top") {
      const top = [...elo.entries()].sort((a, b) => b[1] - a[1]).slice(0, 10);
      const lines = top.map(([auth, score], i) => `${i + 1}. ...${auth.slice(-6)}, ${score}`);
      room.sendAnnouncement(lines.join("\n"), id, 0xCCCCFF, "small", 0);
      return false;
    }
  };
}

function tryStart(room) {
  if (room.gameState) return;
  const ready = room.players.filter((p) => p.id !== 0 && p.team.id !== 0);
  if (ready.length < MIN_PLAYERS) return;
  room.startGame();
}

function ensureAdmin(room) {
  const nonHost = room.players.filter((p) => p.id !== 0);
  if (nonHost.length === 0) return;
  if (nonHost.some((p) => p.isAdmin)) return;
  nonHost.sort((a, b) => a.id - b.id);
  room.setPlayerAdmin(nonHost[0].id, true);
}
```

## Çalıştırma

`STRATEGY` değişkenini istediğin moda al, `node balance-bot.js`. Üç moddan birini deneyimle:

- `"rotation"`: Maç bittikçe karışır
- `"elo"`: İlk maç random, sonrakiler dengeli. `!elo` ile skorunu, `!top` ile sıralama gör. `elo.json` diske yazılır
- `"winner_stays"`: Kazanan kalır, kaybedenler spec'e atılır, sırada bekleyen girer

## Kritik detaylar

**ELO yarış koşulu**: `updateElo` `onGameEnd`'in **içinde** çağırılıyor. `setTimeout(3s)`'in dışında yapmak gerekli çünkü 3s sonra `rebalanceElo` çağırılırken oyuncu listesi karışmış oluyor. Hesap eski state ile, dengeleme yeni state ile.

**`room.redScore`, `room.blueScore`**: Maç bitmeden önceki son skorları verir. `onGameEnd` çağırıldığında bunlar hâlâ erişilebilir.

**Snake draft**: `rebalanceElo`'da en iyi 4 oyuncuya bakacak olursak draft sırası `red, red, blue, blue, blue, blue, red, red, ...` olmalı, en iyi iki kişi farklı takımlara değil aynı takıma gider, sonra bir sonraki iki kişi karşı takıma. ELO toplamını daha eşit dağıtır. Mevcut implementasyon basitleştirilmiş; saf snake için `[red, blue, blue, red]` örüntüsü.

**`MAX_TEAM_SIZE`**: 4v4 üst sınırı. Oda büyürse fazlalık spec'te kalır.

**`setTimeout(3000)`**: Maç biti ile yeniden başlangıç arasında 3 saniye. Oyuncuların skoru sindirmesi, "iyiydik" diyebilmesi için. Daha kısa rahatsız edici, daha uzun bot ölmüş gibi durur.

**`!top` auth maskeleme**: Auth full değil, son 6 karakter. Auth gizli kalsın, kim olduğu tahmin edilemesin. Gerçek isim göstermek için her auth'un son `Player.name`'ini de kaydetmen gerekir.

## Genişletmek

**Per-position ELO**: Forvet ELO'su ve kaleci ELO'su ayrı. `Player`'ın pozisyonunu kaydet, ona göre güncelle. Daha karmaşık ama "ben sadece kalede iyiyim" oyuncusunu daha doğru değerlendirir.

**Maç günlüğü**: Her maç sonu maçın özet bilgisini (skorlar, oyuncu listeleri, ELO değişimleri) JSON line append et. Sonra "geçen hafta en çok maç oynayan kim" tarzı sorgular yapılabilir. [12-veri-saklama.md](../rehber/12-veri-saklama.md) format önerileri.

**Yarış sezonu**: ELO 30 günde bir reset olur. Bot başlatılırken `elo.json`'un yaşına bakar, eskiyse arşivler ve sıfırdan başlar.

## Doğrulama

- `room.redScore`, `room.blueScore`, RoomBase readonly
- `room.setPlayerTeam(id, teamId)`, line 7955
- `onGameEnd(winningTeamId, customData?)`, line 2191
- ELO formülü standart, K=32 satranç FIDE'ninki, Haxball için iyi başlangıç
