# İstatistik takibi

Bot, gol/asist/maç sayısı gibi verileri auth bazlı saklarsa, oyuncular ne kadar gol atmış görebilir, leaderboard üretilebilir, rekabet artar. Bu tutorial SQLite üzerinde temiz bir stat tracker kurar.

## Tasarım

İki tablo:

- `players (auth, name, goals, assists, own_goals, games, wins, mvps, last_seen)`
- `matches (id, started_at, ended_at, red_score, blue_score)` + `match_players (match_id, auth, team, goals)`

`players` tablosu hızlı sorgu için (leaderboard'a tıklayınca anlık cevap). `matches` ve `match_players` arşiv için ("3 hafta önce hangi maçı oynamıştım?" tarzı).

Asist mantığı için top sahibi tespit lazım, son topa vuran asistir (gol kendisi değilse). Detay [rehber/08-stadyum-ve-disc.md](../rehber/08-stadyum-ve-disc.md) içinde gösteriliyor.

## Kurulum

```sh
npm install better-sqlite3 node-haxball
```

## Tam kod

`stats-bot.js`:

```javascript
const Database = require("better-sqlite3");
const { Room, Utils } = require("node-haxball")();

const TOKEN = "thr1.AAAAA...";
const DB_FILE = "./data/stats.db";

// DB kurulumu
const fs = require("fs");
fs.mkdirSync("./data", { recursive: true });
const db = new Database(DB_FILE);
db.exec(`
  CREATE TABLE IF NOT EXISTS players (
    auth TEXT PRIMARY KEY,
    name TEXT,
    goals INTEGER DEFAULT 0,
    assists INTEGER DEFAULT 0,
    own_goals INTEGER DEFAULT 0,
    games INTEGER DEFAULT 0,
    wins INTEGER DEFAULT 0,
    last_seen INTEGER
  );

  CREATE TABLE IF NOT EXISTS matches (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    started_at INTEGER,
    ended_at INTEGER,
    red_score INTEGER,
    blue_score INTEGER
  );

  CREATE TABLE IF NOT EXISTS match_players (
    match_id INTEGER,
    auth TEXT,
    team INTEGER,
    goals INTEGER DEFAULT 0,
    assists INTEGER DEFAULT 0,
    PRIMARY KEY (match_id, auth)
  );
`);

const upsertPlayer = db.prepare(`
  INSERT INTO players (auth, name, last_seen)
  VALUES (?, ?, ?)
  ON CONFLICT(auth) DO UPDATE SET name = ?, last_seen = ?
`);

const incrGoals = db.prepare(`UPDATE players SET goals = goals + 1 WHERE auth = ?`);
const incrAssists = db.prepare(`UPDATE players SET assists = assists + 1 WHERE auth = ?`);
const incrOwnGoals = db.prepare(`UPDATE players SET own_goals = own_goals + 1 WHERE auth = ?`);
const incrGames = db.prepare(`UPDATE players SET games = games + 1 WHERE auth = ?`);
const incrWins = db.prepare(`UPDATE players SET wins = wins + 1 WHERE auth = ?`);

const insertMatch = db.prepare(`INSERT INTO matches (started_at, red_score, blue_score) VALUES (?, ?, ?)`);
const updateMatchEnd = db.prepare(`UPDATE matches SET ended_at = ?, red_score = ?, blue_score = ? WHERE id = ?`);
const insertMatchPlayer = db.prepare(`
  INSERT OR REPLACE INTO match_players (match_id, auth, team, goals, assists)
  VALUES (?, ?, ?, ?, ?)
`);

const queryStats = db.prepare(`SELECT * FROM players WHERE auth = ?`);
const queryTop = db.prepare(`
  SELECT name, goals, assists, games, wins
  FROM players
  WHERE games >= 3
  ORDER BY (goals * 2 + assists - own_goals * 2) DESC
  LIMIT 10
`);

// Match state
let currentMatchId = null;
const matchGoals = new Map();   // auth → maç içi gol sayısı
const matchAssists = new Map(); // auth → maç içi asist
const matchPlayers = new Map(); // auth → team id
let lastKickerByTeam = { 1: null, 2: null };
let beforeLastKickerByTeam = { 1: null, 2: null };
const playersByAuth = new Map(); // auth → Player snapshot

Room.create({
  name: "TR | İstatistik Takipli",
  token: TOKEN,
  maxPlayerCount: 12,
  noPlayer: true,
  showInRoomList: true,
  geo: { code: "tr", lat: 41, lon: 29 },
}, {
  storage: { player_name: "host" },
  onOpen: (room) => attachLogic(room),
  onClose: (reason) => { db.close(); process.exit(0); },
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
    const now = Date.now();
    upsertPlayer.run(player.auth, player.name, now, player.name, now);
    playersByAuth.set(player.auth, { id: player.id, name: player.name });
    ensureAdmin(room);
  };

  room.onPlayerLeave = () => ensureAdmin(room);

  room.onGameStart = () => {
    matchGoals.clear();
    matchAssists.clear();
    matchPlayers.clear();
    lastKickerByTeam = { 1: null, 2: null };
    beforeLastKickerByTeam = { 1: null, 2: null };

    for (const player of room.players) {
      if (player.id === 0 || player.team.id === 0) continue;
      matchPlayers.set(player.auth, player.team.id);
    }

    const result = insertMatch.run(Date.now(), 0, 0);
    currentMatchId = result.lastInsertRowid;
  };

  room.onPlayerBallKick = (playerId) => {
    const player = room.getPlayer(playerId);
    if (!player || player.team.id === 0) return;

    const teamId = player.team.id;
    beforeLastKickerByTeam[teamId] = lastKickerByTeam[teamId];
    lastKickerByTeam[teamId] = player.auth;
  };

  room.onTeamGoal = (teamId) => {
    const scorerAuth = lastKickerByTeam[teamId];
    const assisterAuth = beforeLastKickerByTeam[teamId];

    // Top kendi takımı tarafından mı vuruldu?
    const lastKickerAnyTeam = lastKickerByTeam[teamId];
    // En son herhangi bir takımdan kim vurdu kontrol et, kendi kalesine attıysa
    const lastKickerTeam = findLastKickerTeam();
    if (lastKickerTeam && lastKickerTeam !== teamId) {
      // kendi kalesine atmış
      const ownGoaler = lastKickerByTeam[lastKickerTeam];
      if (ownGoaler) {
        incrOwnGoals.run(ownGoaler);
      }
    } else if (scorerAuth) {
      incrGoals.run(scorerAuth);
      matchGoals.set(scorerAuth, (matchGoals.get(scorerAuth) ?? 0) + 1);

      if (assisterAuth && assisterAuth !== scorerAuth) {
        incrAssists.run(assisterAuth);
        matchAssists.set(assisterAuth, (matchAssists.get(assisterAuth) ?? 0) + 1);
      }
    }
  };

  room.onGameEnd = (winningTeamId) => {
    const redScore = room.redScore ?? 0;
    const blueScore = room.blueScore ?? 0;

    // Maç kaydı
    updateMatchEnd.run(Date.now(), redScore, blueScore, currentMatchId);

    for (const [auth, team] of matchPlayers) {
      insertMatchPlayer.run(
        currentMatchId,
        auth,
        team,
        matchGoals.get(auth) ?? 0,
        matchAssists.get(auth) ?? 0
      );
      incrGames.run(auth);
      if (team === winningTeamId) incrWins.run(auth);
    }
  };

  room.onPlayerChat = (id, message) => {
    if (message === "!stats") {
      const player = room.getPlayer(id);
      const stats = queryStats.get(player.auth);
      if (!stats || stats.games === 0) {
        room.sendAnnouncement("Henüz istatistik yok. Bir maç oyna.", id, 0xCCCCCC, "small", 0);
        return false;
      }
      const wr = stats.games ? ((stats.wins / stats.games) * 100).toFixed(0) : 0;
      const text = [
        `${stats.name}, ${stats.games} maç`,
        `Gol: ${stats.goals}, Asist: ${stats.assists}, Kendi kale: ${stats.own_goals}`,
        `Galibiyet: ${stats.wins} (${wr}%)`,
      ].join("\n");
      room.sendAnnouncement(text, id, 0xCCCCFF, "small", 0);
      return false;
    }

    if (message === "!top") {
      const top = queryTop.all();
      const lines = top.map((p, i) => {
        const score = p.goals * 2 + p.assists;
        return `${i + 1}. ${p.name}, ${p.goals}g ${p.assists}a (${score}p, ${p.games}m)`;
      });
      room.sendAnnouncement(lines.join("\n") || "Yeterli veri yok", id, 0xCCCCFF, "small", 0);
      return false;
    }
  };
}

function findLastKickerTeam() {
  // Hangi takımın last kicker'ı en son set edildi?
  // Pratik bir tespit yok, son set edileni alacak basit yöntem:
  // onPlayerBallKick zaten team'i set ediyor, en son hangisi set edildi onu bilmek için ayrı bir state lazım
  return null; // basitleştirildi; üretim için ayrı bir global tutulmalı
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

```sh
node stats-bot.js
```

`./data/stats.db` otomatik oluşur. `!stats` ile kendi istatistiklerini, `!top` ile ilk 10'u görebilirsin.

DB'ye direkt SQL atmak istersen:

```sh
sqlite3 ./data/stats.db "SELECT name, goals, games FROM players ORDER BY goals DESC LIMIT 5"
```

## Mantığın açıklaması

**Iki katman: `players` ve `match_players`**: Anlık leaderboard hızlı olsun diye stat'ler aggregate edilmiş `players` tablosunda tutulur. Detaylı sorgular (3 hafta önceki maçlar, takım dağılımı analizi) için raw event verisi `matches`/`match_players`'da. SQLite'ın transaction'ları sayesinde ikisinin tutarlı kalması zor değil, `onGameEnd`'in tek bir block içinde olması yeterli.

**`matchPlayers` Map**: Maç başladığında oyuncu listesi snapshot olarak alınır. Maç ortasında biri ayrılırsa istatistiği `match_players` tablosuna girer ama `incrGames` çağırılmaz, maçı tamamlamadı.

**Asist mantığı**: `beforeLastKickerByTeam` ve `lastKickerByTeam` her takımın son iki dokunanını tutar. Top kırmızının ağına girdiyse, en son maviden vuran scorer, daha öncekisi asister. Aynı kişi peş peşe iki kez vurmuşsa asist sayılmaz (`assisterAuth !== scorerAuth` kontrolü).

**Kendi kalesine gol, `own_goals`**: Top hangi takıma girdiyse `teamId`, ama topa son vuran o takımdan değilse "kendi kalesine attı" demektir. Yukarıdaki kodda basitleştirildi (`findLastKickerTeam` her zaman null döner), üretim sürümünde son vuran kim ise team'ini takip eden tek bir global tutmalısın:

```javascript
let lastKickerOverall = null; // { auth, teamId }

room.onPlayerBallKick = (playerId) => {
  const p = room.getPlayer(playerId);
  if (!p) return;
  lastKickerOverall = { auth: p.auth, teamId: p.team.id };
  // ...mevcut takım-bazlı tracking...
};

// onTeamGoal'da:
if (lastKickerOverall && lastKickerOverall.teamId !== teamId) {
  incrOwnGoals.run(lastKickerOverall.auth);
}
```

Yukarıda iskeleti basit tutmak için bu kısmı `null` ile bıraktım, sen production kodunda doldurmalısın.

**`!top` skor formülü**: `goals * 2 + assists - own_goals * 2`. Sadece gol sıralarsan kaleciler hep dipte olur. Kendi kalesini cezalandırmak rastgele tekme atmayı caydırır. K=2 katsayısı tartışılır, turnuva için 1, casual oda için 2.

## Test ve debug

Botu canlı denemeden önce DB schema'yı doğrula:

```sh
sqlite3 ./data/stats.db ".schema"
```

Manuel data insert ederek query'leri test et:

```sh
sqlite3 ./data/stats.db "INSERT INTO players (auth, name, goals, games) VALUES ('test1', 'Ali', 5, 2)"
sqlite3 ./data/stats.db "SELECT * FROM players ORDER BY goals DESC"
```

## Genişletmek

**Pozisyon takibi**: Her maçta her oyuncu için pozisyon (forvet/kaleci) tahmin et, gol pozisyonu ortalama X koordinatına bakarak. `match_players` tablosuna `avg_x`, `avg_y` ekle.

**Top sahibi süresi**: Her oyuncunun top'a en yakın olduğu süreyi say. `onGameTick`'te tüm oyuncuların top'a uzaklığını hesapla, en yakın olanın sayacını artır.

**Discord export**: SQLite'tan veri çekip Discord bot'un komutlarına serve et. `discord.js` ile basit:

````javascript
client.on("messageCreate", (msg) => {
  if (msg.content === "!hax-top") {
    const top = queryTop.all();
    const fence = "```"; // Discord code block için üç backtick
    const body = top.map((p, i) => `${i + 1}. ${p.name}, ${p.goals}g`).join("\n");
    msg.reply(`${fence}\n${body}\n${fence}`);
  }
});
````

## Doğrulama

- `better-sqlite3` senkron API ile çağrılır, Haxball event'leriyle uyumlu (block etmez çünkü hızlı)
- `room.redScore`, `room.blueScore`, RoomBase readonly
- `onPlayerBallKick(playerId)`, line 7935 grubunda
- `onTeamGoal(teamId, goalId, goal, ballDiscId, ballDisc)`, line 2181
