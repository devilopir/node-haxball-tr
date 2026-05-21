# Veri saklama

`node-haxball` veri saklama API'si sunmaz, bu kütüphanenin konusu değil, sen seçeceksin. Bu sayfa bot context'inde işe yarayan üç pratik yaklaşımı kıyaslıyor.

Saklamak isteyeceğin tipik veri:

- Auth key ↔ oyuncu profili eşlemesi
- İstatistik: gol, asist, maç sayısı, oynama süresi
- Kalıcı ban listesi
- Admin/role atamaları
- Plugin runtime config'i

## 1. JSON dosyası, basit, küçük veri için

```javascript
const fs = require("fs");

const FILE = "./data/stats.json";
let stats = fs.existsSync(FILE)
  ? JSON.parse(fs.readFileSync(FILE, "utf8"))
  : {};

function save() {
  fs.writeFileSync(FILE, JSON.stringify(stats, null, 2));
}

room.onTeamGoal = (teamId) => {
  // ... gol sahibini bul
  stats[playerAuth] = stats[playerAuth] || { goals: 0 };
  stats[playerAuth].goals++;
  save();
};
```

Artıları: sıfır bağımlılık, debug için doğrudan dosyayı açıp okuyabilirsin, küçük veri için yeterli.

Eksileri: her `save()` çağrısı tüm dosyayı baştan yazar. 1000 girişlik bir JSON'u her gol için yeniden yazmak saçma. Belirli bir boyuttan sonra (~yüzlerce KB) yavaşlar. Aynı dosyaya iki process aynı anda yazıyorsa data corruption.

Pratik düzeltme, periyodik flush:

```javascript
let dirty = false;
let saveTimeout = null;

function markDirty() {
  dirty = true;
  if (!saveTimeout) {
    saveTimeout = setTimeout(() => {
      if (dirty) {
        fs.writeFileSync(FILE, JSON.stringify(stats));
        dirty = false;
      }
      saveTimeout = null;
    }, 5000); // 5 saniyede bir flush
  }
}
```

Process crash ederse son 5 saniyeyi kaybedersin, istatistik için kabul edilebilir, kritik veri için değil.

JSON tek dosyalık botlar, oyuncu sayısı <100 olan ev sahipleri için en mantıklı yaklaşım. Daha büyük operasyon başlıyorsa SQLite'a geç.

## 2. SQLite, orta ölçek için varsayılan seçim

`better-sqlite3` paketi senkron API ile rahat kullanılır (Haxball'un async kompleksitesini sürüklemeden):

```sh
npm install better-sqlite3
```

```javascript
const Database = require("better-sqlite3");
const db = new Database("./data/bot.db");

db.exec(`
  CREATE TABLE IF NOT EXISTS players (
    auth TEXT PRIMARY KEY,
    name TEXT,
    goals INTEGER DEFAULT 0,
    assists INTEGER DEFAULT 0,
    games INTEGER DEFAULT 0,
    minutes_played REAL DEFAULT 0,
    last_seen INTEGER
  )
`);

const upsert = db.prepare(`
  INSERT INTO players (auth, name, last_seen)
  VALUES (?, ?, ?)
  ON CONFLICT(auth) DO UPDATE SET name = ?, last_seen = ?
`);

const incrGoals = db.prepare(`UPDATE players SET goals = goals + 1 WHERE auth = ?`);

room.onPlayerJoin = (player) => {
  const now = Date.now();
  upsert.run(player.auth, player.name, now, player.name, now);
};

room.onTeamGoal = (teamId) => {
  const scorer = findScorer(); // logic
  if (scorer) incrGoals.run(scorer.auth);
};
```

Artıları: tek dosyada DB, transaction desteği, indexler, hızlı (senkron API, mikrosaniye gecikme). 100k+ oyuncu için bile rahat. SQL sorgu ile manuel inceleme: `sqlite3 bot.db "SELECT * FROM players ORDER BY goals DESC LIMIT 10"`.

Eksileri: SQL biliyor olman gerekir. Schema migration manuel.

Tipik tablolar:

- `players (auth, name, ...stats)`, oyuncu profili
- `matches (id, started_at, ended_at, red_score, blue_score, ...)`, maç logu
- `match_players (match_id, auth, team, goals, assists)`, maç başına oyuncu performansı
- `bans (auth, reason, banned_at, banned_by)`, kalıcı ban
- `admins (auth, role)`, yetki listesi

[tutoriallar/05-istatistik-takibi.md](../tutoriallar/05-istatistik-takibi.md) bu yapıyı çalışan bir botta gösteriyor.

## 3. Redis, dağıtık veya yüksek frekanslı

Birden fazla oda farklı process'lerde çalışıyorsa ve aralarında veri paylaşacaklarsa, Redis pratik bir orta katmandır. Tek oda için overkill.

```sh
npm install redis
```

```javascript
const { createClient } = require("redis");
const redis = createClient({ url: "redis://localhost:6379" });
await redis.connect();

room.onTeamGoal = async (teamId) => {
  await redis.hIncrBy(`player:${playerAuth}`, "goals", 1);
};
```

Artıları: pub/sub ile cross-process iletişim, atomik counter'lar, TTL desteği (örneğin "5 dakikalık AFK warning state'i").

Eksileri: Redis server ayrı çalıştırılır, deploy karmaşası artar. Network round-trip her çağrı için 1ms+ gecikme, yüksek frekanslı yazmaya çok dikkat.

Pratik bot için Redis cazip değildir. Birden fazla oda işleten lig sistemi, Discord bot ile data paylaşımı, leaderboard görüntüleyen web arayüz gibi senaryolarda devreye girer.

## Auth key ile profil eşleştirme

Oyuncuyu kalıcı olarak tanımanın tek doğru yolu `player.auth` üzerinden gitmektir. `player.name` değiştirilebilir, `player.id` her bağlantıda yeni; auth ise aynı kişi olduğun sürece sabittir.

```javascript
const player = room.getPlayer(id);
const profile = db.prepare("SELECT * FROM players WHERE auth = ?").get(player.auth);
```

`player.auth` undefined olabilir nadir durumlarda (bağlantı kurulurken). `onPlayerJoin` içinde her zaman set olur, sonradan değişmez.

## Backup ve veri kaybı

Hangi yöntemi seçersen seç, dosya silinme/corruption riski var. İki temel önlem:

**Periyodik kopya**: Cron veya bot içi timer ile günlük backup. SQLite için `.backup` komutu hot backup verir.

**Append-only log**: Stats değişiklikleri JSON yerine line-delimited JSON dosyasına append edilir (her satır bir event). Tek seferde "stats yeniden hesapla" işlemi tüm logu işler. Data dosyası bozulsa bile log'tan rebuild yapılabilir.

```javascript
const fs = require("fs");
const logStream = fs.createWriteStream("./data/events.jsonl", { flags: "a" });

function logEvent(type, data) {
  logStream.write(JSON.stringify({ ts: Date.now(), type, ...data }) + "\n");
}

room.onTeamGoal = (teamId, goalId, goal, ballDiscId, ballDisc) => {
  logEvent("goal", { teamId });
};
```

`fs.appendFile` async, `createWriteStream` daha verimli ama process crash'inde son birkaç event kaybolabilir (buffer'da).

## Hangisini ne zaman özeti

| Veri büyüklüğü ve karmaşıklığı | Seçim |
|---|---|
| <10 admin auth listesi, ban listesi | JSON dosyası |
| Oyuncu profili + istatistik, tek bot | SQLite |
| Birden fazla bot, ortak liderlik tablosu | SQLite (paylaşılan dosya) veya PostgreSQL |
| Cross-process state (mute süresi, sıra) | Redis |
| Audit log, replay-able event stream | Append-only JSONL |

Kararsızsan SQLite'tan başla, JSON kadar düşük sürtünmeli, on bin satıra kadar problem çıkarmaz.
