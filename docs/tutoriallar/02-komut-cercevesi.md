# Komut çerçevesi

Bir bot yazdıkça `!help`, `!mute`, `!kick`, `!stats` derken handler'lar `onPlayerChat`'in içinde büyür ve kontrolden çıkar. Bu tutorial, yeni komut eklemenin tek satır olduğu, izin kontrollü, otomatik help mesajı üreten bir komut sistemini sıfırdan kurar.

Sonunda ~150 satır, çalışan, genişletmesi kolay bir iskelet olacak.

## Tam kod

`bot.js`:

```javascript
const { Room, Utils } = require("node-haxball")();

const TOKEN = "thr1.AAAAA...";

const commands = new Map();
const adminAuths = new Set(); // başlangıçta boş, ilk gelen admin olur

function defineCommand(name, opts) {
  commands.set(name, {
    name,
    help: opts.help ?? "",
    adminOnly: opts.adminOnly ?? false,
    handler: opts.handler,
  });
}

defineCommand("help", {
  help: "Mevcut komutları gösterir",
  handler: (room, byId, args) => {
    const lines = [...commands.values()]
      .filter((c) => !c.adminOnly || isAdmin(room, byId))
      .map((c) => `!${c.name}, ${c.help}`)
      .sort();
    room.sendAnnouncement(lines.join("\n"), byId, 0xCCCCCC, "small", 0);
  },
});

defineCommand("ping", {
  help: "Bot ayakta mı diye kontrol",
  handler: (room, byId) => {
    room.sendAnnouncement("pong", byId, 0xCCFFCC, "small", 0);
  },
});

defineCommand("kick", {
  help: "Bir oyuncuyu kickler. Kullanım: !kick <id> [sebep]",
  adminOnly: true,
  handler: (room, byId, args) => {
    const targetId = parseInt(args[0]);
    if (isNaN(targetId)) {
      return reply(room, byId, "Geçerli bir id ver. Kullanım: !kick <id> [sebep]", true);
    }
    const target = room.getPlayer(targetId);
    if (!target) {
      return reply(room, byId, `Oyuncu bulunamadı: ${targetId}`, true);
    }
    const reason = args.slice(1).join(" ") || "Admin kararı";
    room.kickPlayer(targetId, reason, false);
  },
});

defineCommand("ban", {
  help: "Bir oyuncuyu banlar. Kullanım: !ban <id> [sebep]",
  adminOnly: true,
  handler: (room, byId, args) => {
    const targetId = parseInt(args[0]);
    if (isNaN(targetId)) {
      return reply(room, byId, "Geçerli bir id ver. Kullanım: !ban <id> [sebep]", true);
    }
    const reason = args.slice(1).join(" ") || "Admin kararı";
    room.kickPlayer(targetId, reason, true);
  },
});

defineCommand("clearbans", {
  help: "Tüm banları kaldırır",
  adminOnly: true,
  handler: (room, byId) => {
    room.clearBans();
    reply(room, byId, "Tüm banlar kaldırıldı.");
  },
});

defineCommand("rr", {
  help: "Maçı yeniden başlatır",
  adminOnly: true,
  handler: (room) => {
    if (room.gameState) room.stopGame();
    setTimeout(() => room.startGame(), 500);
  },
});

defineCommand("players", {
  help: "Oda içindeki oyuncuları listeler",
  handler: (room, byId) => {
    const lines = room.players
      .filter((p) => p.id !== 0)
      .map((p) => {
        const team = p.team.id === 0 ? "Spec" : p.team.id === 1 ? "Red" : "Blue";
        const admin = p.isAdmin ? " ★" : "";
        return `[${p.id}] ${p.name} (${team})${admin}`;
      });
    room.sendAnnouncement(lines.join("\n"), byId, 0xCCCCCC, "small", 0);
  },
});

function isAdmin(room, playerId) {
  if (playerId === 0) return true;
  const player = room.getPlayer(playerId);
  return player?.isAdmin || adminAuths.has(player?.auth);
}

function reply(room, playerId, message, error = false) {
  room.sendAnnouncement(
    message,
    playerId,
    error ? 0xFF8888 : 0xCCCCCC,
    error ? "italic" : "small",
    0
  );
}

function handleChat(room, id, message) {
  if (!message.startsWith("!")) return;

  const parts = message.slice(1).trim().split(/\s+/);
  const name = parts[0].toLowerCase();
  const args = parts.slice(1);

  const cmd = commands.get(name);
  if (!cmd) {
    reply(room, id, `Bilinmeyen komut: !${name} (!help dene)`, true);
    return true;
  }

  if (cmd.adminOnly && !isAdmin(room, id)) {
    reply(room, id, "Bu komut için admin yetkisi gerekir.", true);
    return true;
  }

  try {
    cmd.handler(room, id, args);
  } catch (err) {
    console.error(`Komut '${name}' hata verdi:`, err);
    reply(room, id, "Komut çalıştırılırken bir hata oluştu.", true);
  }

  return true;
}

Room.create({
  name: "TR | Komut Sistemi",
  token: TOKEN,
  maxPlayerCount: 12,
  noPlayer: true,
  showInRoomList: true,
  geo: { code: "tr", lat: 41, lon: 29 },
}, {
  storage: { player_name: "host" },
  onOpen: (room) => {
    room.onAfterRoomLink = (link) => {
      console.log("Oda hazır:", link);
      const big = Utils.getDefaultStadiums().find((s) => s.name === "Big");
      if (big) room.setCurrentStadium(big);
      room.setScoreLimit(3);
      room.setTimeLimit(3);
    };

    room.onPlayerJoin = (player) => {
      // İlk gelen admin olur
      const nonHost = room.players.filter((p) => p.id !== 0);
      if (!nonHost.some((p) => p.isAdmin)) {
        room.setPlayerAdmin(player.id, true);
        adminAuths.add(player.auth);
      }
      reply(room, player.id, `Hoş geldin ${player.name}! Komutlar için !help.`);
    };

    room.onPlayerLeave = () => {
      const nonHost = room.players.filter((p) => p.id !== 0);
      if (nonHost.length > 0 && !nonHost.some((p) => p.isAdmin)) {
        nonHost.sort((a, b) => a.id - b.id);
        room.setPlayerAdmin(nonHost[0].id, true);
      }
    };

    room.onPlayerChat = (id, message) => {
      if (handleChat(room, id, message)) return false; // chat'e basma
    };
  },
  onClose: (reason) => {
    console.log("kapandı:", reason.toString());
    process.exit(0);
  },
});
```

## Çalıştırma

`node bot.js`. Token'ı kendi token'ınla değiştir. İlk oyuncu girince admin olur, `!help` yazar, mevcut komutları görür. `!players` ile odadakileri listeler.

## Tasarım kararları

**`commands` Map'i**: Komutlar isimle erişilen merkezi bir yere kayıt edilir. Yeni komut eklemek için `defineCommand` çağırmak yeterli. Diğer dosyalardan da bu Map'e ekleme yapılabilir; modüler komut kaynakları için temel.

**`handler(room, byId, args)` imzası**: Üç parametreyle handler'a her şey verilir. `args` zaten `string[]` olarak parse edilmiş, handler kendisi parse yapmaz, sadece dönüştürür (`parseInt`, vs.).

**`reply()` yardımcı**: Her komut "şuna cevap ver" gerektirir. Renk, style, sound default'larıyla tek bir helper bunu özetler. Hata renkli kırmızı, normal cevap gri.

**`isAdmin` çift kontrol**: `player.isAdmin` Haxball'un kendi flag'i, `adminAuths` set ise botun kalıcı admin listesi. İkisinden biri yeterli, Haxball'un admin'i çıktığında yeniden girince admin olarak gelmez (yeni id), ama auth'u aynıdır; `adminAuths` bu sürekliliği sağlar.

**`startsWith("!")` filtresi**: Komut olmayan mesajları erkenden eliminate eder. Performans için değil okunabilirlik için, handler içinde her mesajda `parts.split` çağırmaktansa.

**`return false` ile chat'i bastırma**: Komut mesajı işlendiyse herkese gösterme. Bu özellikle admin komutları için önemli (`!ban`, `!kick` herkese görünmesin). Komut bulunamadığında da bastırıyoruz, hatalı yazımlar chat'i kirletmesin.

**`try/catch` handler etrafında**: Bir komut handler'ında bug varsa botun komple çökmesini engeller. Loga yazar, kullanıcıya generic hata mesajı gösterir.

**`!rr` `setTimeout(500)`**: `stopGame` ve `startGame`'i ardışık çağırırsan ikincisi göz ardı edilebilir, backend henüz "oyun durdu" state'ine geçmemiş olur. 500ms gecikme bunu önler.

## Genişletmek

Komut eklemek bir `defineCommand` çağrısı:

```javascript
defineCommand("rng", {
  help: "1-100 arası rastgele sayı",
  handler: (room, byId) => {
    const n = Math.floor(Math.random() * 100) + 1;
    reply(room, byId, `Rastgele: ${n}`);
  },
});
```

Modüler hale getirmek için komut tanımlarını ayrı dosyalara taşı:

```javascript
// commands/admin.js
module.exports = (defineCommand) => {
  defineCommand("kick", { ... });
  defineCommand("ban", { ... });
};

// bot.js
require("./commands/admin")(defineCommand);
require("./commands/stats")(defineCommand);
```

İzin sistemini detaylandırmak için: `adminAuths` set yerine farklı roller (`ownerAuths`, `modAuths`, `vipAuths`), her komut için `requiredRole`. Detay [tutoriallar/06-admin-yetki-sistemi.md](06-admin-yetki-sistemi.md) içinde.

## Doğrulama

Botun kütüphane API'lerine karşı statik kontrolü:

- `room.kickPlayer(id, reason, isBanning)`, line 7980 imzası
- `room.clearBans()`, line 7600
- `room.gameState`, RoomBase'de readonly property
- `room.players`, line 7435 readonly Player[]
- `room.getPlayer(id)`, line 7985
- `player.team.id`, Team objesinin id alanı

Canlı test için kendi token'ınla bağla, en az iki oyuncuyla `!help`, `!kick`, `!ban` davranışını gör.
