# Temel host bot

Bu tutorial, [baslangic/03-ilk-oda.md](../baslangic/03-ilk-oda.md) iskeletinin üzerine günlük kullanım için gereken temel davranışı ekler:

- Boş odaya ilk gireni admin yap, admin gidince yeni admin ata
- Maç ayarlarını (stadyum, skor, süre) açılışta belirle
- Yeterli oyuncu olduğunda oyunu otomatik başlat
- Maç bitince oyuncuları otomatik dengele
- Temel `!help` komutu

Sonunda ~80 satır, ekstra paket gerektirmeyen, tek dosyalık çalışan bir bot olacak.

## Tam kod

`bot.js`:

```javascript
const { Room, Utils } = require("node-haxball")();

const TOKEN = "thr1.AAAAA..."; // kendi token'ınla değiştir
const MIN_PLAYERS_TO_START = 2;

Room.create({
  name: "TR | Temel Bot",
  password: null,
  token: TOKEN,
  maxPlayerCount: 12,
  noPlayer: true,
  showInRoomList: true,
  geo: { code: "tr", lat: 41.0, lon: 29.0 },
}, {
  storage: { player_name: "host" },
  onOpen: (room) => attachLogic(room),
  onClose: (reason) => {
    console.log("Oda kapandı:", reason.toString());
    process.exit(0);
  },
});

function attachLogic(room) {
  room.onAfterRoomLink = (link) => {
    console.log("Oda hazır:", link);
    const big = Utils.getDefaultStadiums().find((s) => s.name === "Big");
    if (big) room.setCurrentStadium(big);
    room.setScoreLimit(3);
    room.setTimeLimit(3);
  };

  room.onPlayerJoin = (player) => {
    ensureAdmin(room);
    if (countNonSpectators(room) === 0 && countNonHost(room) >= MIN_PLAYERS_TO_START) {
      tryStart(room);
    }
  };

  room.onPlayerLeave = () => {
    ensureAdmin(room);
  };

  room.onGameEnd = (winningTeamId) => {
    setTimeout(() => rebalance(room), 3000);
  };

  room.onPlayerChat = (id, message) => {
    if (message === "!help") {
      room.sendAnnouncement(
        "Komutlar: !help, bu mesaj.",
        id, 0xCCCCCC, "small", 0
      );
      return false;
    }
  };
}

function ensureAdmin(room) {
  const players = room.players.filter((p) => p.id !== 0);
  if (players.length === 0) return;
  if (players.some((p) => p.isAdmin)) return;
  players.sort((a, b) => a.id - b.id);
  room.setPlayerAdmin(players[0].id, true);
}

function countNonHost(room) {
  return room.players.filter((p) => p.id !== 0).length;
}

function countNonSpectators(room) {
  return room.players.filter((p) => p.id !== 0 && p.team.id !== 0).length;
}

function tryStart(room) {
  rebalance(room);
  room.startGame();
}

function rebalance(room) {
  const pool = room.players.filter((p) => p.id !== 0).slice();
  pool.sort(() => Math.random() - 0.5);
  pool.forEach((p, i) => {
    room.setPlayerTeam(p.id, (i % 2) + 1);
  });
}
```

## Çalıştırma

```sh
node bot.js
```

Çıktıda oda linki belirir. Linki paylaş, en az iki oyuncu girince oyun otomatik başlar; biri gol atınca maç biter ve oyuncular yeni takımlara karıştırılır.

## Kod ne yapıyor

**`attachLogic`**, `onOpen`'dan çağırılır, tüm event handler'ları tek bir fonksiyona toplar. `Room.create`'i kısa tutmak için ayırmak okunabilirliği artırır.

**`onAfterRoomLink`** içindeki stadium ve limit ayarları, oda public listede göründükten **sonra** yapılır. Daha erken yapmak teknik olarak çalışır ama bazen ilk oyuncu girmeden önce backend tarafında reset olabilir; `onAfterRoomLink` güvenli noktadır.

**Stadyum seçimi**, Resmi HBInit'in `setDefaultStadium("Big")` kısayolu node-haxball'da yoktur. Onun yerine `Utils.getDefaultStadiums()` parsed bir `Stadium[]` döndürür, içinden ismini eşleştirip `setCurrentStadium(stadium)` ile atarsın. Kendi `.hbs` dosyanı yüklemek için [rehber/08-stadyum-ve-disc.md](../rehber/08-stadyum-ve-disc.md) `Utils.parseStadium` örneğini gösteriyor.

**`ensureAdmin`**, `id !== 0` filtresi hostu listeden çıkarır (host id'si 0'dır, `noPlayer: true` olsa bile state içinde görünebilir). En düşük id'ye sahip oyuncuyu admin yapar; bu, "odaya en erken giren admin olur" sezgisine uyar. Daha karmaşık bir yetki sistemi için [06-admin-yetki-sistemi.md](06-admin-yetki-sistemi.md) bakılır.

**Otomatik başlatma**, `onPlayerJoin` içinde "aktif maç yok ve yeterli oyuncu var mı" kontrolü yapılır. `countNonSpectators` aktif maçı, `countNonHost` toplam oyuncuyu sayar. `MIN_PLAYERS_TO_START` 2 verilmiş; lobi tarzı bir oda için 4 veya 6 daha mantıklı olabilir.

**`rebalance`**, Oyuncuları rastgele iki takıma böler. `(i % 2) + 1` ifadesindeki `+1`, takım id'lerinin `0 = spectator`, `1 = red`, `2 = blue` olmasındandır. Dikkat: `Player.team` int değil `Team` objesidir, takım id'sine `player.team.id` ile erişilir, `setPlayerTeam`'in **ikinci argümanı** ise int id alır. Kütüphanenin bu çift tipli tasarımı yeni başlayanları yakalar. Daha akıllı dengeleme (skor geçmişine bakan, ELO temelli) [04-takim-dengeleme.md](04-takim-dengeleme.md) içinde işleniyor.

**Callback parametresi int id**, node-haxball'da `onPlayerChat`, `onPlayerInputChange`, `onPlayerAdminChange` gibi "bir oyuncuya ait" callback'ler resmi HBInit'in aksine **Player objesi değil int id** geçer. İhtiyaç olursa `room.getPlayer(id)` ile çevirilir. `onPlayerJoin` ve `onPlayerLeave` ise tam Player objesi alır, istisna budur.

**`onPlayerChat` ile `return false`**: Bir chat handler'dan `false` döndürmek mesajın diğer oyunculara iletilmemesini söyler. Komut mesajlarını chat'te göstermek istemiyorsak komutu işledikten sonra `false` döneriz. Normal mesajlarda `return` yazılmaz, default `undefined` ile mesaj normal akışında devam eder.

**`setTimeout(rebalance, 3000)`**, Maç bittikten 3 saniye sonra dengeleme. Gerçek bir maçtan sonra oyunculara "kazandık/kaybettik" anını yaşatacak kadar süre tanımak deneyimi yumuşatır. Oyuncular bu süre içinde takım değiştirebilir, sorun değil; `rebalance` zaten herkesi yeniden böler.

## Test ve genişletme

Statik kontrol için API isimleri `node-haxball@2.3.0`'in `src/index.d.ts` dosyasına karşı doğrulandı:

- `room.players` (line 4836): `state.players`'a kısa yol
- `room.setPlayerAdmin`, `room.setPlayerTeam`, `room.startGame`, `room.setCurrentStadium`, `room.setScoreLimit`, `room.setTimeLimit`: line 7842-7965 aralığında
- `Utils.getDefaultStadiums()` (line 4528): parsed varsayılan stadyumları döndürür
- `onPlayerJoin`, `onPlayerLeave`, `onGameEnd`, `onPlayerChat`, `onAfterRoomLink`: callback listesinde mevcut
- `sendAnnouncement` imzası: `(msg, targetId, color, style, sound)`, line 7729

Canlı test için kendi token'ınla çalıştırıp en az bir gerçek bağlantı yapmayı öneriyorum; `process.exit` çağrılarına dikkat et, production'da `onClose`'ta otomatik kapatmamalı, restart loop'una almak daha iyidir.

Sonraki adım [02-komut-cercevesi.md](02-komut-cercevesi.md) ile genişleyebilir bir komut sistemi kurmaktır; tek `!help` ile kalmak istemiyorsan oradan devam et.
