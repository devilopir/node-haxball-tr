# İlk oda: host bot

Bu sayfa, sıfırdan çalışan en kısa host botunu üretir. Kod ~30 satır, kopyala-çalıştır.

## Önkoşullar

- Node.js 18 veya üzeri kurulu, `npm install node-haxball` tamam ([02-kurulum-ve-token.md](02-kurulum-ve-token.md))
- `https://www.haxball.com/headlesstoken` üzerinden alınmış geçerli bir token

## Tam kod

`bot.js`:

```javascript
const { Room } = require("node-haxball")();

const TOKEN = "thr1.AAAAA..."; // kendi token'ınla değiştir

Room.create({
  name: "İlk Odam",
  password: null,
  token: TOKEN,
  maxPlayerCount: 10,
  noPlayer: true,
  showInRoomList: true,
  geo: { code: "tr", lat: 41.0, lon: 29.0 },
}, {
  storage: { player_name: "host" },
  onOpen: (room) => {
    room.onAfterRoomLink = (link) => {
      console.log("Oda hazır:", link);
    };

    room.onPlayerJoin = (player) => {
      console.log(`Katıldı: ${player.name} (id ${player.id})`);
      room.sendAnnouncement(`Hoş geldin ${player.name}!`, player.id, 0x88FF88, "normal", 0);
    };

    room.onPlayerLeave = (player) => {
      console.log(`Ayrıldı: ${player.name}`);
    };
  },
  onClose: (reason) => {
    console.log("Oda kapandı:", reason.toString());
    process.exit(0);
  },
});
```

## Çalıştırma

```sh
node bot.js
```

Beklenen çıktı:

```
Oda hazır: https://www.haxball.com/play?c=XXXXXXXXXX
```

Bu linki tarayıcıda aç, odaya gir. Terminal `Katıldı: ...` yazmalı, oda içinde sana yeşil renkli karşılama mesajı gelmeli.

## Kod ne yapıyor

İlk satır kütüphaneyi yükler. `require("node-haxball")()`, sondaki `()` ihmal edilmemeli; kütüphane bir factory döner, çağırılınca tüm API namespace'i veren objeyi üretir. Buradan sadece ihtiyacın olanı destructure ediyoruz, burada `Room` yeterli.

`Room.create` iki argüman alır. İlk argüman odanın kendisini tanımlar: ismi, şifresi, token'ı, maksimum oyuncu sayısı. `noPlayer: true` host'un kendisini oyuncu olarak yaratmamasını söyler, host'un kalede oturduğu o klasik bug'ı engeller. `geo` alanı zorunludur, oda listesinde hangi ülke bayrağıyla görüneceğini belirler.

İkinci argüman bağlantı seviyesi ayarlardır. `storage.player_name` host kullanıcısının ismidir ama `noPlayer: true` olduğu için zaten görünmez; yine de protokol gerektirdiği için verilir.

`onOpen`, bağlantı kurulup oda backend'de kayıt olduktan sonra çağrılır. Bot mantığı buradan içeri yazılır, event handler'ları doğrudan `room` objesine atanır. `onAfterRoomLink`, oda public listede yayınlandığında ve oyunculara verilecek link hazır olduğunda tetiklenir.

`sendAnnouncement` parametre sırası: mesaj, hedef oyuncu id'si (null verirsen herkese), renk (hex int, `-1` şeffaf), font style (`"normal"`, `"bold"`, `"italic"`, `"small"`, `"small-bold"`, `"small-italic"`), sound (`0` ses yok, `1` chat sesi, `2` bildirim). Burada sadece o oyuncuya yeşil renkli, sessiz olarak yazıyoruz.

`onClose`, oda backend tarafından düşürüldüğünde veya `process` sinyali aldığında çağrılır. Burada hangi sebepten kapandığını basıp process'i kapatıyoruz.

## Token süresi geçtiyse

Çıktıda `Oda hazır` yerine `Oda kapandı: ...captcha...` veya benzer bir hata görürsen token süresi dolmuştur. `https://www.haxball.com/headlesstoken` üzerinden yenisini al, koddaki `TOKEN` değerini değiştir, tekrar çalıştır.

## Sonraki adımlar

Bu bot odayı açar ama hiçbir şey yapmaz, chat'i okumaz, kimseyi kicklemez. Bir sonraki tutorial olan [tutoriallar/01-temel-host-bot.md](../tutoriallar/01-temel-host-bot.md) bu iskeletin üzerine basic moderasyon, oyun başlatma ve admin atama ekler.

Eğer bir bot yazmak yerine başka bir odaya katılan istemci yazmak istiyorsan [04-ilk-istemci.md](04-ilk-istemci.md) onu işliyor.
