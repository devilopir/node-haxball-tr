# Sık yapılan hatalar

Bot yazarken yaşadığın "neden çalışmıyor bu" durumlarının tipik nedenleri ve çözümleri.

## "Oda kuruluyor sonra hemen düşüyor"

**Token süresi dolmuş**. `https://www.haxball.com/headlesstoken` token'ı birkaç dakika içinde geçersiz olur. Bot başlatırken yeni token al, koda yapıştır, hemen başlat.

**Aynı IP'de zaten 2 oda var**. Haxball backend IP başına 2 oda izin verir. Üçüncü oda kurma denemesi reddedilir. [rehber/16-proxy-deploy.md](../rehber/16-proxy-deploy.md) çözümler.

**Versiyon uyumsuzluğu**. Haxball client güncellendi ama kütüphanenin `version: 9` ayarı eski. `Room.create({...}, { version: 10 })` ile elle override et veya kütüphaneyi güncelle.

## "Oda var, ben bağlanamıyorum"

**Şifre yanlış**. `Room.join({ ..., password: "secret" })`, case-sensitive.

**Banlanmışsın**. Auth değiştir veya host'tan ban'ı kaldırmasını iste.

**Recaptcha gerekli**. Oda `requireRecaptcha: true` ise client token gerekir. Detay [baslangic/04-ilk-istemci.md](../baslangic/04-ilk-istemci.md).

**Connection state takılmış**. `onConnInfo` ekleyip hangi aşamada kaldığını gör. Genelde `ConnectingToPeer` aşamasında WebRTC problemi varsa NAT/firewall sorunudur.

## "Bot bağlandı, hiçbir event tetiklenmiyor"

**Handler atamasını yanlış yere yazdın**. `Room.create({...}, { ... })`'in **dışında** `room.onPlayerJoin = ...` atayamazsın çünkü `room` henüz var değil. Atamalar `onOpen` callback'inin içinde yapılır.

**RoomConfig dahili olarak handler'ları override etti**. RoomConfig ve doğrudan `room.onX = fn` atama birlikte çalışır ama yazımın doğru olduğundan emin ol. [rehber/04-callback-sistemi.md](../rehber/04-callback-sistemi.md).

**Plugin'in `allowFlags`'i yanlış**. Host bot için `AllowFlags.JoinRoom` flag'i olan plugin yüklenmez. `AllowFlags.CreateRoom` ya da `CreateRoom | JoinRoom` olmalı.

## "`player.team.id` yerine `player.team` int sandım"

`Player.team` int **değil**, `Team` objesidir. Karşılaştırma:

```javascript
// Yanlış
if (player.team === 1) { ... }

// Doğru
if (player.team.id === 1) { ... }
```

Bu hata sessizdir, `player.team !== 0` daima `true` olur (obje ≠ sayı), spec'tekiler de "takımdaymış gibi" görünür. [rehber/06-oyuncu-yonetimi.md](../rehber/06-oyuncu-yonetimi.md) Player yapısı.

## "`onPlayerChat`'te `player.name` `undefined`"

`onPlayerChat` callback'i Player objesi değil, **int id** alır:

```javascript
// Yanlış
room.onPlayerChat = (player, message) => { ... player.name ... };

// Doğru
room.onPlayerChat = (id, message) => {
  const player = room.getPlayer(id);
  // player.name
};
```

Resmi HBInit API'sinden gelenler bu farkı sık atlar.

## "`setDefaultStadium('Big')` çalışmıyor"

node-haxball'da bu metod yok. `Utils.getDefaultStadiums()` ile parsed stadyumu alıp `setCurrentStadium(stadium)` ile set edersin:

```javascript
const big = Utils.getDefaultStadiums().find((s) => s.name === "Big");
room.setCurrentStadium(big);
```

## "`sendAnnouncement` style int verdim, çalışmıyor"

`sendAnnouncement` çağrısının style parametresi **string**. Callback observation'da int olarak gelir ama yazarken string verilir:

```javascript
// Yanlış
room.sendAnnouncement("test", id, 0xFFFFFF, 0, 0);

// Doğru
room.sendAnnouncement("test", id, 0xFFFFFF, "normal", 0);
```

Geçerli değerler: `"normal"`, `"bold"`, `"italic"`, `"small"`, `"small-bold"`, `"small-italic"`.

## "`onTeamVictory` tetiklenmiyor"

node-haxball'da `onTeamVictory` callback'i **yok**. Karşılığı `onGameEnd(winningTeamId)`. Resmi HBInit'in `onTeamVictory(scores)` callback'i node-haxball'da farklı isimle ve farklı parametre listesiyle.

## "`autoTeams` herkesi spec'e atıyor"

`autoTeams` sadece **spec'te olanları** dağıtır. Maç aktifken ve oyuncular takımda iken çağırırsan hiçbir şey yapmaz. Tüm oyuncuları karıştırmak için `randTeams()`.

## "`setPlayerTeam(id, 1)` bir kez işliyor ama ikinci kez işlemiyor"

Maç aktifken takım değiştirme `teamsLock`'a tabi. Default kapalıyken oyuncular kendileri değiştirebilir ama bot karışırsa rate limit yer (özellikle aynı tick'te birden fazla `setPlayerTeam` çağırırsan). Kontrol ettiğinden emin ol: `room.gameState` aktif mi.

## "JSON dosyası bozuldu"

Botun crash anında veya çift process'in aynı dosyaya yazması durumunda. Çözüm:

1. `fs.writeFileSync`'ın atomik olmadığını bil, partial write riski var
2. Önce `.tmp` dosyaya yaz, sonra `rename` ile değiştir (atomic on POSIX)
3. Yüksek frekanslı yazma yapıyorsan SQLite'a geç. [rehber/12-veri-saklama.md](../rehber/12-veri-saklama.md).

## "`onGameTick` her tick log basıyor, console boğuldu"

`onGameTick` saniyede 60 kez çağırılır. İçinde `console.log` koyma, terminal bombardımanı.

Periyodik iş için sayaç:

```javascript
let tickCount = 0;
room.onGameTick = () => {
  tickCount++;
  if (tickCount % 60 === 0) {
    // saniyede bir
  }
};
```

## "Reconnect döngüsünde IP banlanıyorum"

Sabit `setTimeout(connect, 5000)` ile sürekli denersen Haxball backend rate limit yer ve `RateLimitReached` hatasıyla geçici banlar. Üstel backoff kullan:

```javascript
let backoff = 5000;
const MAX = 120000;

onClose: (reason) => {
  setTimeout(connect, backoff);
  backoff = Math.min(backoff * 2, MAX);
}
onOpen: (room) => {
  backoff = 5000; // başarıda sıfırla
}
```

Detay [rehber/03-room-join.md](../rehber/03-room-join.md).

## "node-datachannel install hata veriyor"

`npm install node-haxball` sırasında `node-datachannel` C++ derleme adımına girer. Eksik build tool varsa hata verir.

- **Linux**: `sudo apt install build-essential python3`
- **Windows**: Visual Studio Build Tools (C++ workload)
- **macOS**: `xcode-select --install`

## "`Utils.parseStadium` undefined döndürdü"

Dosya bozuk veya format dışı. `parseStadium` exception fırlatmaz, sessizce `undefined` verir, `if (!stadium)` ile kontrol et. Hatanın detayını öğrenmek için kütüphanenin internal logger'ı bilgi vermez; manuel parse (JSON5 ile) deneyip hatayı orada yakalayabilirsin.

## "Plugin runtime'da `this` yanlış"

Function constructor pattern'ında `this` callback içlerinde değişebilir. Konvansiyon:

```javascript
function MyPlugin() {
  Object.setPrototypeOf(this, Plugin.prototype);
  Plugin.call(this, "myPlugin", true, { ... });

  const that = this; // <-- önemli

  this.onPlayerJoin = function (player) {
    that.room.sendAnnouncement(...); // this değil that
  };
}
```

`that = this` deyimi kütüphanenin tüm örneklerinde var, tutarlılık için aynısını kullan.

## "Webhook spam yedim, Discord rate limit"

Discord webhook'larına saniyede ~30 mesajı geçince rate limit. Yoğun bot için ya batch'leme yap (5 saniye topla, tek mesaj olarak yolla), ya da queue kullan (`p-queue` paketi yardımcı).

## "`exit` ile process kapatınca DB hata veriyor"

`onClose` içinde `process.exit(0)` çağırırsan, açık dosya/DB handle'ları temizlenmemiş olabilir. Önce `db.close()` ya da `fs.closeSync(fd)`, sonra exit:

```javascript
onClose: (reason) => {
  db.close();
  logStream.end();
  process.exit(0);
}
```

Veya daha güvenli: `process.on("SIGINT", cleanup)` ile signal handler kur, cleanup'tan sonra exit.
