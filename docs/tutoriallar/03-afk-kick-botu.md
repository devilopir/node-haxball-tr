# AFK kick botu

Bir oda yöneten herkesin yaşadığı problem: oyuncu girer, takıma alınır, AFK kalır. Kale boş kalır, maç oynanmaz, herkes rahatsız olur. Bu tutorial input aktivitesini ölçen ve uyarı verip atan bir bot kurar.

## Stratejik kararlar

Önce ne sayacağız:

- **Input** (yön tuşu veya ateş), temel "oynuyor" işareti
- **Chat mesajı**, yazıyor ama spec'te değilse aktivite mi? Karar: takımdayken aktivite sayar, spec'tekiler için chat kıştır olabilir, sayma
- **Spec'te oturmak**, kontrolün altında. Spec'te kalanları kicklemiyoruz, sadece takımdayken AFK olanları
- **Maç dışı**, maç aktif değilse AFK detection kapalı. Lobi sohbeti normal

Uyarı stratejisi:

- 30 saniye aktivite yok → spec'e at, oyuncu fark eder, kendi tarafından geri takıma alınır
- Spec'te 60 saniye sessizlik → kickle

İki aşamalı yaklaşım kati kick'ten daha az rahatsız edicidir.

## Tam kod

`afk-bot.js`:

```javascript
const { Room, Utils } = require("node-haxball")();

const TOKEN = "thr1.AAAAA...";

// Ayarlar
const AFK_TO_SPEC_MS = 30_000;
const AFK_KICK_FROM_SPEC_MS = 60_000;
const CHECK_INTERVAL_MS = 5_000;

// Aktivite durumu
const lastActivity = new Map();   // playerId → ms timestamp
const movedToSpec = new Set();    // playerId, spec'e atılan, kicke aday

Room.create({
  name: "TR | Anti-AFK",
  token: TOKEN,
  maxPlayerCount: 12,
  noPlayer: true,
  showInRoomList: true,
  geo: { code: "tr", lat: 41, lon: 29 },
}, {
  storage: { player_name: "host" },
  onOpen: (room) => attachLogic(room),
  onClose: (reason) => {
    console.log("kapandı:", reason.toString());
    process.exit(0);
  },
});

function attachLogic(room) {
  room.onAfterRoomLink = (link) => {
    console.log("Oda hazır:", link);
    const big = Utils.getDefaultStadiums().find((s) => s.name === "Big");
    if (big) room.setCurrentStadium(big);
    room.setScoreLimit(3);
    room.setTimeLimit(0);
  };

  room.onPlayerJoin = (player) => {
    lastActivity.set(player.id, Date.now());
    ensureAdmin(room);
  };

  room.onPlayerLeave = (player) => {
    lastActivity.delete(player.id);
    movedToSpec.delete(player.id);
    ensureAdmin(room);
  };

  room.onPlayerInputChange = (playerId) => {
    lastActivity.set(playerId, Date.now());
    movedToSpec.delete(playerId);
  };

  room.onPlayerBallKick = (playerId) => {
    lastActivity.set(playerId, Date.now());
    movedToSpec.delete(playerId);
  };

  room.onPlayerChat = (id) => {
    const player = room.getPlayer(id);
    if (!player || player.team.id === 0) return;
    lastActivity.set(id, Date.now());
  };

  room.onPlayerTeamChange = (id, teamId) => {
    if (teamId !== 0) {
      // Birisi onu spec'ten takıma aldı, sayacı sıfırla
      lastActivity.set(id, Date.now());
      movedToSpec.delete(id);
    }
  };

  room.onGameStart = () => {
    // Maç başında herkesin sayacını yenile, "girişte AFK" haksızlığını engelle
    for (const player of room.players) {
      if (player.id === 0) continue;
      lastActivity.set(player.id, Date.now());
    }
    movedToSpec.clear();
  };

  setInterval(() => check(room), CHECK_INTERVAL_MS);
}

function check(room) {
  if (!room.gameState) return; // sadece maç aktifken

  const now = Date.now();

  for (const player of room.players) {
    if (player.id === 0) continue;

    const last = lastActivity.get(player.id) ?? now;
    const idle = now - last;

    if (player.team.id !== 0) {
      // Takımda, AFK_TO_SPEC süresi geçtiyse spec'e at
      if (idle >= AFK_TO_SPEC_MS) {
        room.setPlayerTeam(player.id, 0);
        movedToSpec.add(player.id);
        room.sendAnnouncement(
          `${player.name} AFK kaldı, spec'e alındı`,
          null, 0xFF8888, "italic", 1
        );
        room.sendAnnouncement(
          `${Math.floor(AFK_KICK_FROM_SPEC_MS / 1000)} saniye içinde aktivite vermezsen odadan atılacaksın.`,
          player.id, 0xFFAA00, "bold", 2
        );
        lastActivity.set(player.id, now); // sayacı spec için yeniden başlat
      }
    } else if (movedToSpec.has(player.id)) {
      // Spec'e atıldı, sayacı doldu mu
      if (idle >= AFK_KICK_FROM_SPEC_MS) {
        room.kickPlayer(player.id, "Uzun süre AFK", false);
      }
    }
  }
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

`node afk-bot.js`. Token'ı değiştir. İki sekme aç, ikincide bekle (input verme). 30 saniye sonra spec'e atılırsın, ek 60 saniye sonra odadan kicklenirsin.

## Mantığın açıklaması

**İki aşama, iki Map**: `lastActivity` herkes için zamanı tutar. `movedToSpec` ise yalnız "spec'e atılan AFK adayı" için flag. Bunun nedeni: normal bir oyuncu kendi isteğiyle spec'e geçtiğinde onu kicklememeliyiz. Sadece "biz onu spec'e attıysak" geri sayım başlamalı.

**Üç aktivite kaynağı**: `onPlayerInputChange` tuş state değişimi (hareket/ateş), `onPlayerBallKick` topa vurma, `onPlayerChat` yazılı mesaj. Üçü de "oyuncu yaşıyor" sinyali. Chat sadece takımdayken sayılır, spec'teki yazışmaları sayarsak AFK detection işe yaramaz.

**`onPlayerTeamChange` sayaç sıfırlaması**: Birisi spec'ten takıma döndürülürse, sayacı sıfırla. Bu admin atamasıyla veya teamsLock kapalıyken kendi seçimiyle olabilir. Aksi takdirde "girişte 1 dakika spec'te kaldı, takıma geçti, hemen AFK'lendi" haksızlığı olur.

**`onGameStart` reset**: Maç başında herkesin sayacı sıfırlanır. Maç biti ile yeni maç arasında geçen sürede AFK olarak görünmesin diye.

**`check` sadece `gameState` varken**: Maç durduğunda AFK detection kapalı. Maç beklenirken oyuncu kahve içiyor olabilir, normal.

**`setInterval(5s)`**: Her oyuncuyu her tick'te kontrol etmek gereksiz, saniye seviyesinde hassasiyet AFK için yeterli. 5 saniye iyi bir denge.

**`lastActivity.set(player.id, now)` spec'e atınca**: Spec'e atıldıktan sonra kick için sayaç yeniden başlar. Aksi takdirde takımdayken biriken sayaç + spec sayacı toplam olur, beklenenden hızlı kicklenir.

## Bilinen sınırlar

**`onPlayerInputChange` frekansı**: Tuş state'i değiştiğinde tetiklenir, sürekli sağa giden bir oyuncu için sayım nispeten seyrek olur (input zaten sabit), ama yön değiştirip duran biri için saniyede 10+ event olabilir. Sayaç sadece bellekte tutulduğu için bu yük sorun değil.

**Yarış durumu**: Aynı tick'te `onPlayerInputChange` ile `check` çakışırsa, oyuncu yanlışlıkla spec'e atılabilir. Buffer 5 saniye olduğu için pratikte olmaz, sıkı timing senaryolarında olabilir.

**Bilerek AFK kalıp atılmama**: 25 saniyede bir tuşa basan bir bot kicklenmez. AFK detection mükemmel değildir, kötü niyetli birine karşı moderasyona devretmek gerekir.

## Genişletmek

Komut sistemi ekle ([02-komut-cercevesi.md](02-komut-cercevesi.md)), `!afk <id>` ile bir oyuncuyu manuel uyarmaya yarayan ekstra komut:

```javascript
defineCommand("afk", {
  help: "Bir oyuncuyu AFK olarak işaretler",
  adminOnly: true,
  handler: (room, byId, args) => {
    const targetId = parseInt(args[0]);
    if (!isNaN(targetId)) {
      lastActivity.set(targetId, 0); // hemen AFK sayılır
    }
  },
});
```

Bir sonraki `check()` tetiklenmesinde oyuncu spec'e atılır.

## Doğrulama

`src/index.d.ts` üzerinden cross-check:

- `onPlayerInputChange(id, value, customData?)` (line 2609)
- `onPlayerBallKick(playerId, customData?)` (line 2161)
- `onPlayerTeamChange(id, teamId, byId, customData?)` (line 2451)
- `room.gameState`, RoomBase readonly
- `setPlayerTeam(id, teamId)` (line 7955)
- `kickPlayer(id, reason, isBanning)` (line 7980)
