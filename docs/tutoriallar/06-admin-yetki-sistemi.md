# Admin yetki sistemi

Haxball'un yerleşik "admin" bayrağı tek seviyedir: ya admin'sin, ya değilsin. Gerçek bir topluluk yönetiminde rol katmanlarına ihtiyaç vardır, sahip, moderatör, VIP, normal oyuncu. Bu tutorial, auth tabanlı kalıcı rol sistemini kurar.

## Roller

- `owner`: Tüm komutlara erişim, başkalarına rol atayabilir
- `admin`: Kick, ban, maç kontrolü, mute
- `mod`: Kick, mute
- `vip`: Özel avatarlar, renk, ekstra komutlar
- `none`: Default

Roller kalıcıdır (auth'a bağlı, restart sonrası korunur). İlk başlatmada hiçbir owner yoktur, botu başlatan ilk kişi `!claim <şifre>` ile owner olur.

## Tam kod

`admin-bot.js`:

```javascript
const fs = require("fs");
const { Room, Utils } = require("node-haxball")();

const TOKEN = "thr1.AAAAA...";
const CLAIM_PASSWORD = "abc123"; // ilk owner olmak için
const ROLES_FILE = "./roles.json";

// Rol hiyerarşisi (yüksek değer = daha yetkili)
const ROLE_RANK = {
  none: 0,
  vip: 1,
  mod: 2,
  admin: 3,
  owner: 4,
};

const roles = new Map(); // auth → role

function loadRoles() {
  if (!fs.existsSync(ROLES_FILE)) return;
  const data = JSON.parse(fs.readFileSync(ROLES_FILE, "utf8"));
  for (const [auth, role] of Object.entries(data)) roles.set(auth, role);
}

function saveRoles() {
  fs.writeFileSync(ROLES_FILE, JSON.stringify(Object.fromEntries(roles), null, 2));
}

function getRole(auth) {
  return roles.get(auth) ?? "none";
}

function setRole(auth, role) {
  if (role === "none") roles.delete(auth);
  else roles.set(auth, role);
  saveRoles();
}

function hasRole(player, minRole) {
  if (!player) return false;
  return ROLE_RANK[getRole(player.auth)] >= ROLE_RANK[minRole];
}

// Komut sistemi
const commands = new Map();

function defineCommand(name, opts) {
  commands.set(name, {
    name,
    help: opts.help ?? "",
    minRole: opts.minRole ?? "none",
    handler: opts.handler,
  });
}

function reply(room, id, msg, error = false) {
  room.sendAnnouncement(
    msg, id,
    error ? 0xFF8888 : 0xCCCCCC,
    error ? "italic" : "small", 0
  );
}

// Komutlar

defineCommand("help", {
  help: "Komut listesi",
  handler: (room, byId) => {
    const player = room.getPlayer(byId);
    const myRole = getRole(player.auth);
    const myRank = ROLE_RANK[myRole];

    const lines = [...commands.values()]
      .filter((c) => myRank >= ROLE_RANK[c.minRole])
      .map((c) => `!${c.name}, ${c.help}${c.minRole !== "none" ? ` [${c.minRole}+]` : ""}`)
      .sort();
    reply(room, byId, lines.join("\n"));
  },
});

defineCommand("role", {
  help: "Kendi rolünü gösterir",
  handler: (room, byId) => {
    const player = room.getPlayer(byId);
    reply(room, byId, `Rolün: ${getRole(player.auth)}`);
  },
});

defineCommand("claim", {
  help: "İlk owner ol. Kullanım: !claim <şifre>",
  handler: (room, byId, args) => {
    const existingOwners = [...roles.values()].filter((r) => r === "owner");
    if (existingOwners.length > 0) {
      return reply(room, byId, "Owner zaten atanmış.", true);
    }
    if (args[0] !== CLAIM_PASSWORD) {
      return reply(room, byId, "Yanlış şifre.", true);
    }
    const player = room.getPlayer(byId);
    setRole(player.auth, "owner");
    room.setPlayerAdmin(byId, true);
    reply(room, byId, "Owner olarak atandın.");
  },
});

defineCommand("setrole", {
  help: "Bir oyuncuya rol ver. Kullanım: !setrole <id> <role>",
  minRole: "owner",
  handler: (room, byId, args) => {
    const targetId = parseInt(args[0]);
    const role = args[1]?.toLowerCase();
    if (isNaN(targetId) || !ROLE_RANK.hasOwnProperty(role)) {
      return reply(room, byId, "Kullanım: !setrole <id> <none|vip|mod|admin|owner>", true);
    }
    const target = room.getPlayer(targetId);
    if (!target) return reply(room, byId, "Oyuncu bulunamadı.", true);

    setRole(target.auth, role);
    syncHaxballAdmin(room);
    reply(room, byId, `${target.name} → ${role}`);
    if (role !== "none") {
      reply(room, targetId, `Sana ${role} rolü verildi.`);
    }
  },
});

defineCommand("roles", {
  help: "Tüm rolleri listele (sadece auth son 6 karakter görünür)",
  minRole: "admin",
  handler: (room, byId) => {
    const lines = [...roles.entries()]
      .filter(([, r]) => r !== "none")
      .map(([auth, role]) => `...${auth.slice(-6)}, ${role}`);
    reply(room, byId, lines.join("\n") || "Atanmış rol yok");
  },
});

defineCommand("kick", {
  help: "Oyuncuyu odadan atar. Kullanım: !kick <id> [sebep]",
  minRole: "mod",
  handler: (room, byId, args) => {
    const targetId = parseInt(args[0]);
    if (isNaN(targetId)) return reply(room, byId, "Geçerli id gir.", true);

    const byPlayer = room.getPlayer(byId);
    const target = room.getPlayer(targetId);
    if (!target) return reply(room, byId, "Oyuncu bulunamadı.", true);

    // Daha yüksek rolü olan kişiyi kickleyemez
    if (ROLE_RANK[getRole(target.auth)] >= ROLE_RANK[getRole(byPlayer.auth)]) {
      return reply(room, byId, "Bu oyuncuyu kickleyemezsin (yeterli yetkin yok).", true);
    }

    room.kickPlayer(targetId, args.slice(1).join(" ") || "Mod kararı", false);
  },
});

defineCommand("ban", {
  help: "Oyuncuyu banlar. Kullanım: !ban <id> [sebep]",
  minRole: "admin",
  handler: (room, byId, args) => {
    const targetId = parseInt(args[0]);
    if (isNaN(targetId)) return reply(room, byId, "Geçerli id gir.", true);

    const byPlayer = room.getPlayer(byId);
    const target = room.getPlayer(targetId);
    if (!target) return reply(room, byId, "Oyuncu bulunamadı.", true);

    if (ROLE_RANK[getRole(target.auth)] >= ROLE_RANK[getRole(byPlayer.auth)]) {
      return reply(room, byId, "Bu oyuncuyu banlayamazsın.", true);
    }

    room.kickPlayer(targetId, args.slice(1).join(" ") || "Admin kararı", true);
  },
});

defineCommand("avatar", {
  help: "Avatarını değiştir. Kullanım: !avatar <emoji>",
  minRole: "vip",
  handler: (room, byId, args) => {
    const emoji = args[0] ?? "";
    room.setPlayerAvatar(byId, emoji.slice(0, 2), true);
    reply(room, byId, "Avatar güncellendi.");
  },
});

function syncHaxballAdmin(room) {
  // Mod ve üstü kişilere otomatik Haxball admin'i ver
  for (const player of room.players) {
    if (player.id === 0) continue;
    const shouldBeAdmin = hasRole(player, "mod");
    if (player.isAdmin !== shouldBeAdmin) {
      room.setPlayerAdmin(player.id, shouldBeAdmin);
    }
  }
}

// Chat handler
function handleChat(room, id, message) {
  if (!message.startsWith("!")) return false;
  const parts = message.slice(1).trim().split(/\s+/);
  const name = parts[0].toLowerCase();
  const args = parts.slice(1);

  const cmd = commands.get(name);
  if (!cmd) {
    reply(room, id, `Bilinmeyen komut: !${name}`, true);
    return true;
  }

  const player = room.getPlayer(id);
  if (!hasRole(player, cmd.minRole)) {
    reply(room, id, `Bu komut için ${cmd.minRole}+ rolü gerekir.`, true);
    return true;
  }

  try {
    cmd.handler(room, id, args);
  } catch (err) {
    console.error(`Komut '${name}' hata verdi:`, err);
    reply(room, id, "Komut hata verdi.", true);
  }
  return true;
}

// Bot
loadRoles();

Room.create({
  name: "TR | Yetki Sistemi",
  token: TOKEN,
  maxPlayerCount: 12,
  noPlayer: true,
  showInRoomList: true,
  geo: { code: "tr", lat: 41, lon: 29 },
}, {
  storage: { player_name: "host" },
  onOpen: (room) => {
    room.onAfterRoomLink = (link) => console.log("link:", link);

    room.onPlayerJoin = (player) => {
      const role = getRole(player.auth);
      if (role !== "none") {
        reply(room, player.id, `Hoş geldin ${player.name}! (rol: ${role})`);
      }
      syncHaxballAdmin(room);
    };

    room.onPlayerLeave = () => syncHaxballAdmin(room);

    room.onPlayerChat = (id, message) => {
      if (handleChat(room, id, message)) return false;
    };
  },
  onClose: (reason) => process.exit(0),
});
```

## Çalıştırma

`node admin-bot.js`. İlk owner ol:

1. Odaya gir
2. `!claim abc123` yaz (şifreyi koddaki ile uyumlu olarak değiştir)
3. Owner olarak atandın, Haxball admin'i de otomatik geldi
4. Diğer oyunculara rol atamak için: `!setrole 5 mod`

`roles.json` diske yazılır, bot restart sonrası roller korunur.

## Tasarım notları

**Rol hiyerarşisi**: `ROLE_RANK` ile sayısal sıralama, rol karşılaştırması bir int karşılaştırması haline gelir. Yeni rol eklemek için ROLE_RANK'a entry ekleyip, ilgili komutlara `minRole`'u yaz.

**`!claim` tek seferlik**: Mevcut owner varsa komut reddedilir. Owner kaybedildiyse (bot sahibi auth.key'ini kaybetti) `roles.json`'u elle silmek gerekir. Bu kasıtlı bir güvenlik, kimse rastgele owner olamasın.

**`syncHaxballAdmin`**: Rol değişiklikleri ile Haxball'un kendi admin bayrağı senkronize tutulur. Mod ve üstü herkes Haxball admin'i olur. Aksi takdirde rolün olsa bile takım/start/stop yapamazsın çünkü Haxball'un içsel admin kontrolü ayrı.

**Yetki silsilesi koruması**: Mod, admin'i kickleyemez. Admin, owner'ı banlayamaz. `ROLE_RANK[target] >= ROLE_RANK[by]` kontrolü her güçlü komutta var. Olmasaydı bir admin owner'ı banlayabilirdi, kötü.

**Auth tabanlı saklama**: Rolü `player.name`'e değil `player.auth`'a bağlamak şart. İsim değiştirilebilir, auth aynı kişiyi temsil eder.

**`syncHaxballAdmin` her join'de**: Birisi rolüyle eşleşmeyen Haxball admin durumunda olabilir (örneğin bot restart oldu, ilk gelen oyuncu otomatik admin oldu ama aslında rol "none"). Join'de senkronize ediyoruz.

## Genişletmek

**Discord entegrasyonu**: Rol atamayı Discord'dan yapmak. Bot Discord'a bağlı, `/setrole <auth> <role>` komutu Discord tarafından alınıp Node'a iletiliyor. Karmaşıklığa değer mi diye düşün, küçük topluluk için chat içi yeterli.

**Geçici rol**: `setRole` zamanlı versiyon. `vip` rolü 30 gün geçerli, sonrası `none`. `roles.json` schema'sını `{ role, expiresAt }` objesine genişlet.

**Audit log**: Kim kime hangi rolü ne zaman verdi? `roles_log.jsonl` append-only dosyaya her değişikliği yaz:

```javascript
function setRole(auth, role, byAuth = "system") {
  // ... mevcut kod
  fs.appendFileSync(
    "./roles_log.jsonl",
    JSON.stringify({ ts: Date.now(), auth, role, by: byAuth }) + "\n"
  );
}
```

Geriye doğru sorgu, "kim suistimal etti" analizi için.

## Doğrulama

- `player.auth`, Player interface (line 1607 civarı)
- `room.setPlayerAdmin(id, value)`, line 7965
- `room.setPlayerAvatar(id, value, headless)`, line 7657
- `room.kickPlayer(id, reason, isBanning)`, line 7980
