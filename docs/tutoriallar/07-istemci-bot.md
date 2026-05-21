# İstemci botu

[baslangic/04-ilk-istemci.md](../baslangic/04-ilk-istemci.md) bağlanan minimal bir client gösterdi. Bu tutorial üstüne daha pratik özellikler ekler: otomatik reconnect, chat komutları, Discord köprüsü için iskelet, ve birkaç AI hareket örneği.

## Use case: Lig duyurucusu

Senaryo: senin değil, başkasının açtığı bir lig odasında bir bot oturuyor, oradaki olayları (gol, kim kazandı, kim girdi/çıktı) Discord'a aktarıyor. Host yetkisi olmadan, sadece izleyici olarak.

## Tam kod

`client-bot.js`:

```javascript
const fs = require("fs");
const { Room, Utils } = require("node-haxball")();

const TARGET_ROOM_ID = "Olnit_iGRWs"; // hedef oda
const ROOM_PASSWORD = null;
const KEY_FILE = "./client-auth.key";
const RECONNECT_MIN_MS = 5_000;
const RECONNECT_MAX_MS = 120_000;

// Opsiyonel: Discord webhook
const DISCORD_WEBHOOK = process.env.DISCORD_WEBHOOK ?? null;

let backoff = RECONNECT_MIN_MS;

async function getAuth() {
  if (fs.existsSync(KEY_FILE)) {
    const key = fs.readFileSync(KEY_FILE, "utf8").trim();
    return { key, authObj: await Utils.authFromKey(key) };
  }
  const [key, authObj] = await Utils.generateAuth();
  fs.writeFileSync(KEY_FILE, key);
  console.log("Yeni auth.key üretildi");
  return { key, authObj };
}

async function notifyDiscord(text) {
  if (!DISCORD_WEBHOOK) return;
  try {
    await fetch(DISCORD_WEBHOOK, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ content: text }),
    });
  } catch (err) {
    console.error("Discord webhook hatası:", err.message);
  }
}

async function connect() {
  const { key, authObj } = await getAuth();

  Room.join({
    id: TARGET_ROOM_ID,
    password: ROOM_PASSWORD,
    authObj,
  }, {
    storage: {
      player_name: "TR-Watcher",
      avatar: "📷",
      player_auth_key: key,
    },
    onOpen: (room) => {
      console.log(`Bağlandı: ${room.name}`);
      backoff = RECONNECT_MIN_MS;
      attachHandlers(room);
      notifyDiscord(`📡 İzleyici **${room.name}** odasına bağlandı`);
    },
    onClose: (reason) => {
      const message = reason?.toString() ?? "bilinmeyen sebep";
      console.log(`Bağlantı koptu: ${message}, ${backoff / 1000}s sonra tekrar`);
      notifyDiscord(`🔌 Bağlantı koptu: ${message}`);
      setTimeout(connect, backoff);
      backoff = Math.min(backoff * 2, RECONNECT_MAX_MS);
    },
    onConnInfo: (state, extra) => {
      // detaylı debug için, normalde sessiz
      // console.log("connection:", state, extra);
    },
  });
}

function attachHandlers(room) {
  let lastKicker = null; // { auth, name, teamId }
  let prevLastKicker = null;
  const scoreState = { red: 0, blue: 0 };

  room.onPlayerJoin = (player) => {
    notifyDiscord(`➕ **${player.name}** odaya katıldı`);
  };

  room.onPlayerLeave = (player) => {
    notifyDiscord(`➖ **${player.name}** ayrıldı`);
  };

  room.onGameStart = (byId) => {
    // byId 0: sistem/host tetikledi
    const byName = byId === 0 ? "host" : room.getPlayer(byId)?.name ?? "?";
    notifyDiscord(`▶ **Maç başladı** (${byName})`);
    lastKicker = null;
    prevLastKicker = null;
    scoreState.red = 0;
    scoreState.blue = 0;
  };

  room.onPlayerBallKick = (playerId) => {
    const p = room.getPlayer(playerId);
    if (!p) return;
    prevLastKicker = lastKicker;
    lastKicker = { auth: p.auth, name: p.name, teamId: p.team.id };
  };

  room.onTeamGoal = (teamId) => {
    const team = teamId === 1 ? "Red" : "Blue";
    scoreState.red = room.redScore ?? scoreState.red;
    scoreState.blue = room.blueScore ?? scoreState.blue;

    let line = `⚽ **${team} gol**, ${scoreState.red}-${scoreState.blue}`;
    if (lastKicker && lastKicker.teamId === teamId) {
      line += `\n  Gol: ${lastKicker.name}`;
      if (prevLastKicker && prevLastKicker.auth !== lastKicker.auth && prevLastKicker.teamId === teamId) {
        line += `, asist: ${prevLastKicker.name}`;
      }
    } else if (lastKicker) {
      line += `\n  Kendi kalesine: ${lastKicker.name}`;
    }
    notifyDiscord(line);
  };

  room.onGameEnd = (winningTeamId) => {
    const team = winningTeamId === 0 ? "Beraberlik" : winningTeamId === 1 ? "Red" : "Blue";
    notifyDiscord(`🏁 **Maç bitti**, ${team} (${scoreState.red}-${scoreState.blue})`);
  };

  room.onPlayerChat = (id, message) => {
    if (id === room.currentPlayerId) return;
    const sender = room.getPlayer(id);
    if (!sender) return;

    // Discord'a chat aktarımı (opsiyonel)
    // notifyDiscord(`<${sender.name}> ${message}`);

    // Bot'a hitap edilen komutlar
    if (message.toLowerCase() === "!bot") {
      room.sendChat(`Selam ${sender.name}, ben izleyici.`, null);
    }
  };
}

connect();
```

## Çalıştırma

```sh
DISCORD_WEBHOOK="https://discord.com/api/webhooks/..." node client-bot.js
```

Webhook olmadan da çalışır, sadece terminal log'lar.

## Bilinçli kararlar

**Üstel backoff (`5s → 10s → 20s → ... 120s`)**: Sabit kısa interval reconnect, Haxball backend tarafından rate limit yer ve sonunda IP geçici banlanır. Backoff artması saatlerce kapalı kalsa bile dakika başına 1-2 deneme civarına oturur.

**`backoff` sıfırlama `onOpen`'da**: Başarılı bağlantıda sayacı sıfırla. Birkaç gün sürdü, bağlandı, sonra koptu, yeni süreç 5s'ten başlasın, 120s'lik bekleyişe takılmasın.

**`fetch` Node 18+ built-in**: Node 18'den itibaren global `fetch` var, ekstra paket lazım değil. Daha eski Node kullanıyorsan `node-fetch` veya `axios` ekle.

**Discord webhook ile rate limit**: Bir webhook'a saniyede 30+ mesaj atarsan Discord rate limit yer. Yoğun chat aktarımı yapıyorsan ya batch'le ya da kuyruk koy. Gol bildirimi, join/leave: doğal limit zaten düşük, sorun olmaz.

**Auth dosyası**: `client-auth.key`. Host bot'unun auth'undan ayrı tut, bot'ların farklı kimlik göstermesi mantıklıdır. `.gitignore`'a koy.

**`!bot` komutu**: Botun "varlığını" doğrulamak için minik selam mesajı. Daha çok komut eklemek için [02-komut-cercevesi.md](02-komut-cercevesi.md) iskeletini buraya da koyabilirsin.

## Kendi inputunu vermek

Botu bağlı bıraktıktan sonra kendi hareketini vermek istersen `setKeyState` ile yapılır:

```javascript
// Topa doğru git örneği
room.onGameTick = () => {
  const me = room.getPlayer(room.currentPlayerId);
  if (!me || me.team.id === 0) return;

  const myDisc = room.getPlayerDisc(me.id);
  const ball = room.getBall();
  if (!myDisc || !ball) return;

  const dx = ball.pos.x - myDisc.pos.x;
  const dy = ball.pos.y - myDisc.pos.y;
  const dist = Math.hypot(dx, dy);

  // Direction değerleri: -1 (Backward), 0 (Still), +1 (Forward)
  // Her iki eksen için aynı set kullanılır
  const dirX = Math.abs(dx) < 5 ? 0 : Math.sign(dx);
  const dirY = Math.abs(dy) < 5 ? 0 : Math.sign(dy);

  // Top yakındaysa vur
  const kick = dist < (myDisc.radius + ball.radius + 5);

  room.setKeyState(Utils.keyState(dirX, dirY, kick));
};
```

`Direction` enum'unun yalnız üç değeri vardır: `Backward = -1`, `Still = 0`, `Forward = 1`. İsimler X ve Y eksenlerini ayırmaz, sen `Math.sign` ile -1/0/+1 hesaplarsın, kütüphane bunu yön olarak yorumlar.

Bu tam bir "AI" değil, yalnızca topa koşar, kalecisi yok, takım mantığı yok. Pratik bir başlangıç noktası. Detaylı bot AI'sı için repo'nun `examples/plugins/autoPlay_*.js` dosyaları çok daha derin örnekler veriyor.

**Bot etiği**: Halk içinde başka odalarda bot oynatmak hoş karşılanmaz. Lig odalarında banlanırsın. Kendi odanda veya antrenman için kullan.

## Birden fazla client

Tek process'te birden fazla `Room.join` kararsızdır. İhtiyacın olursa her client'ı ayrı process olarak çalıştır:

```sh
node client-bot.js  # terminal 1
node client-bot.js  # terminal 2 (farklı auth key dosyası)
```

Auth key'i environment variable'dan alarak iki instance'a farklı kimlik verebilirsin:

```javascript
const KEY_FILE = process.env.AUTH_KEY_FILE ?? "./client-auth.key";
const BOT_NAME = process.env.BOT_NAME ?? "TR-Watcher";
```

```sh
AUTH_KEY_FILE=./bot1.key BOT_NAME="Bot-1" node client-bot.js &
AUTH_KEY_FILE=./bot2.key BOT_NAME="Bot-2" node client-bot.js &
```

## Doğrulama

- `Room.join(joinParams, commonParams)`, line 8297
- `Utils.generateAuth()`, `Utils.authFromKey()`, line 4342, 4351
- `room.currentPlayerId`, RoomBase readonly
- `room.getPlayerDisc(id)`, line 7990 grubunda
- `room.getBall()`, same group
- `Utils.keyState(dirX, dirY, kick)`, line 4463
- `setKeyState(state)`, line 7700 civarı

Canlı test: kendi token'ınla yarattığın bir oda ID'sini `TARGET_ROOM_ID`'ye yaz, başka terminalde host bot çalıştır, client'i bağla. Webhook ayarlamadıysan da terminal log'larında join/leave/gol mesajları görmelisin.
