# İlk istemci: odaya katılan bot

Resmi headless API sadece host yazmaya izin verir. `node-haxball` ile bir botu başka birinin açtığı odaya **istemci** olarak da bağlayabilirsin, odada normal bir oyuncu gibi görünür, chat'i dinler, mesaj atar, kendi inputunu gönderir.

## Ne zaman kullanırsın

- Kendi odanı dışarıdan izlemek, log'lamak, istatistik toplamak
- Discord ↔ Haxball köprüsü: Discord'dan gelen mesajı oda chatine bastır
- Toplu test: birden fazla bot client'ı odaya alıp davranış simüle etmek
- Resmi headless'in vermediği client-only veriye erişmek (lokal extrapolation state'i, ping bilgisi)

## Önkoşullar

- Katılacağın bir odanın **ID**'si. Oda linki `https://www.haxball.com/play?c=Olnit_iGRWs` ise ID `Olnit_iGRWs`.
- Oda şifreliyse şifresi.
- Oda recaptcha korumalı ise bir client token, bu pratikte kendi odalarına bağlanırken kapatabileceğin bir özellik, başkasının recaptcha'lı odasına bağlanmak için [tokenGenerator](https://github.com/wxyz-abcd/node-haxball/tree/main/tokenGenerator) aracı gerekir. Aşağıdaki örnek korumasız oda varsayar.

## Auth nedir

Haxball her oyuncuyu bir kalıcı anahtar üzerinden tanır. Bu anahtar tarayıcıda localStorage'de tutulur ve "aynı kişi mi" sorusunu cevaplar, ban'lar buna göre tutulur, profil yıldızı buna göre işler. `node-haxball`'da bu anahtarı manuel yönetmen gerekir.

İki yol var:

`Utils.generateAuth()`, Yeni bir anahtar üretir. İlk kullanımda bunu çağır, dönen `authKey` stringini diske yaz, sonraki bağlantılarda aynısını kullan.

`Utils.authFromKey(authKey)`, Önceden ürettiğin `authKey`'i kullanılabilir auth objesine dönüştürür. Bot her seferinde yeni kimlik almasın istiyorsan bunu kullan.

## Tam kod

`client.js`:

```javascript
const fs = require("fs");
const { Room, Utils } = require("node-haxball")();

const ROOM_ID = "Olnit_iGRWs"; // hedef oda ID'si
const ROOM_PASSWORD = null;     // şifresizse null
const KEY_FILE = "./auth.key";

async function getAuth() {
  if (fs.existsSync(KEY_FILE)) {
    const key = fs.readFileSync(KEY_FILE, "utf8").trim();
    const authObj = await Utils.authFromKey(key);
    return { key, authObj };
  }
  const [key, authObj] = await Utils.generateAuth();
  fs.writeFileSync(KEY_FILE, key);
  console.log("Yeni auth key üretildi ve auth.key dosyasına kaydedildi.");
  return { key, authObj };
}

(async () => {
  const { key, authObj } = await getAuth();

  Room.join({
    id: ROOM_ID,
    password: ROOM_PASSWORD,
    authObj: authObj,
  }, {
    storage: {
      player_name: "TR-Bot",
      avatar: "🤖",
      player_auth_key: key,
    },
    onOpen: (room) => {
      console.log(`Odaya bağlandım: ${room.name}`);
      room.sendChat("Selam, ben bir bot.", null);

      room.onPlayerChat = (id, message) => {
        if (id === room.currentPlayerId) return;
        const player = room.getPlayer(id);
        console.log(`<${player.name}> ${message}`);

        if (message.toLowerCase() === "!ping") {
          room.sendChat(`@${player.name} pong`, null);
        }
      };
    },
    onClose: (reason) => {
      console.log("Bağlantı kesildi:", reason.toString());
      process.exit(0);
    },
  });
})();
```

## Çalıştırma

```sh
node client.js
```

Başarılı olursa terminal `Odaya bağlandım: ...` der, hedef oda chat'inde `Selam, ben bir bot.` mesajı görünür. Odadaki oyuncular `!ping` yazınca bot `@isim pong` cevabı verir.

## Önemli ayrıntılar

`room.currentPlayerId`, botun kendi player id'sini verir. Kendi mesajına cevap vermesin diye `onPlayerChat` içinde bu kontrol şart, yoksa sonsuz loop'a girer.

`onPlayerChat` callback'inin parametresi resmi HBInit'in aksine **Player objesi değil int id'dir**. Tam Player verisine ihtiyacın varsa `room.getPlayer(id)` ile alırsın. Aynı kural diğer "id ile gelen" callback'ler için de geçerli, tip detayı [referans/callbacks.md](../referans/callbacks.md) içinde tek tek listelenmiştir.

`player_auth_key` storage içine konulduğunda kütüphane bağlantı sırasında bunu kullanır. Aynı `authKey`'le bağlandığın sürece Haxball backend'i seni aynı kişi olarak tanır, host tarafında banlanırsan başka bir key üretmen gerekir.

`onClose` ile gelen `reason` bir `Errors.HBError` objesidir; `.toString()` insan okunur mesaj verir, programatik karar için `.code` alanına bakılır. Hata kodlarının tam listesi [referans/errors.md](../referans/errors.md) içinde.

## Sınırlar

İstemci botu yazarken aklında bulunsun: host odadan atarsa veya oda kapanırsa client da bağlantısını kaybeder. Otomatik yeniden bağlanma kütüphanenin sorumluluğunda değildir, kendin `setTimeout` ile `Room.join`'i tekrar çağırmalısın. [rehber/03-room-join.md](../rehber/03-room-join.md) reconnect mantığı için bir kalıp gösteriyor.

Birden fazla istemciyi tek script'ten yönetmek için kütüphane sınırlı bir izolasyon sunar, pratikte her client'ı kendi Node process'inde çalıştırmak en istikrarlı yoldur. [tutoriallar/07-istemci-bot.md](../tutoriallar/07-istemci-bot.md) bu örüntüyü genişletilmiş bir bot etrafında işliyor.
