# Room.join

`Room.join(joinParams, commonParams)` var olan bir odaya istemci olarak bağlanır. Dönen objenin yapısı `Room.create` ile aynıdır, `cancel()` ve `useRecaptchaToken()` içerir.

## İmza

```typescript
Room.join(joinParams: JoinRoomParams, commonParams: CommonNetworkRoomParams): {
  cancel: () => void;
  useRecaptchaToken: (token: string) => void;
}
```

## joinParams alanları

- **`id`** (string): Oda ID'si. `https://www.haxball.com/play?c=Olnit_iGRWs` linkindeki `c=` sonrası kısmı.
- **`password`** (string | null): Oda şifreli ise şifresi.
- **`token`** (string | null): Oda recaptcha korumalı ise client token. Default Haxball backend'inde temiz bir üretim yolu yok; [tokenGenerator](https://github.com/wxyz-abcd/node-haxball/tree/main/tokenGenerator) NW.js aracı gerekir. Korumasız odalarda boş bırakılır.
- **`authObj`** (Auth): `Utils.generateAuth()` veya `Utils.authFromKey()` ile üretilmiş zorunlu obje.

## Auth: kimlik ve kalıcılık

Haxball backend'i kullanıcıları `authObj` üzerinden tanır. Aynı auth ile bağlandığın sürece sen "aynı kişisin"; ban'lar buna göre tutulur, profil yıldızı buna göre işler.

İki seçenek var. İlk seferde yeni kimlik üret:

```javascript
const [authKey, authObj] = await Utils.generateAuth();
// authKey'i diske yaz, bir daha üretme
```

Sonraki seferlerde aynı anahtardan tekrar oluştur:

```javascript
const authObj = await Utils.authFromKey(savedAuthKey);
```

`authKey` bir hex stringdir. `localStorage`'a benzer şekilde botun veri klasöründe sakla. Kaybedersen yeni kimlik olursun, banlandıysan bu işine yarar, kimliğin değer kazandıysa kaybetmek istemezsin.

`storage.player_auth_key` alanına da aynı `authKey`'i yazman önerilir; bazı durumlarda kütüphane bunu da kullanır.

## commonParams

Yapı [02-room-create.md](02-room-create.md) ile aynıdır. Client tarafında dikkat edilecek noktalar:

- **`config`**: Client'ta da RoomConfig kullanılabilir, sadece host-only callback'ler (örneğin `onAfterPlayerKicked`) tetiklenmez.
- **`plugins`**: Plugin'ler client modunda kısıtlı çalışır; host yetkilisi gereken işlemler (`setPlayerAdmin` vb.) sessizce reddedilir.
- **`debugDesync`**: Client'ta bir `(hostRoomState, clientRoomState) => void` callback verebilirsin. Host'un da `debugDesync: true` set etmiş olması gerekir; aksi durumda yalnız çalışmaz.

## Reconnect kalıbı

`onClose`'tan dönen botu otomatik geri bağlama, kütüphanenin sorumluluğunda değildir. Bir kalıp:

```javascript
const fs = require("fs");
const { Room, Utils } = require("node-haxball")();

const ROOM_ID = "Olnit_iGRWs";
const KEY_FILE = "./auth.key";
const RECONNECT_DELAY_MS = 5000;
const MAX_BACKOFF_MS = 60000;

let backoff = RECONNECT_DELAY_MS;

async function getAuth() {
  if (fs.existsSync(KEY_FILE)) {
    return Utils.authFromKey(fs.readFileSync(KEY_FILE, "utf8").trim())
      .then((authObj) => ({ authObj, key: fs.readFileSync(KEY_FILE, "utf8").trim() }));
  }
  const [key, authObj] = await Utils.generateAuth();
  fs.writeFileSync(KEY_FILE, key);
  return { authObj, key };
}

async function connect() {
  const { authObj, key } = await getAuth();

  Room.join({
    id: ROOM_ID,
    authObj: authObj,
  }, {
    storage: { player_name: "TR-Bot", player_auth_key: key },
    onOpen: (room) => {
      console.log("bağlandı");
      backoff = RECONNECT_DELAY_MS; // başarılıysa backoff sıfırla
      attachHandlers(room);
    },
    onClose: (reason) => {
      console.log(`bağlantı koptu (${reason.toString()}), ${backoff/1000}s sonra tekrar denenecek`);
      setTimeout(connect, backoff);
      backoff = Math.min(backoff * 2, MAX_BACKOFF_MS);
    },
  });
}

function attachHandlers(room) {
  room.onPlayerChat = (id, message) => {
    if (id === room.currentPlayerId) return;
    console.log(`<${room.getPlayer(id).name}> ${message}`);
  };
}

connect();
```

Üstel backoff kritik. Sabit `setTimeout(connect, 5000)` ile sürekli yeniden bağlanma denemesi, Haxball backend tarafından "rate limited" olarak algılanır ve IP'n geçici banlanır. Her başarısızlıkta süreyi iki katına çıkar, başarılı bağlantıda sıfırla.

## Connection state takibi

Bağlantı kurulurken arada birçok state'ten geçilir (`ConnectionState` enum'u: `Idle`, `Connecting`, `ConnectingToPeer`, `AwaitingState`, vb.). Debug için `onConnInfo` handler'ı ekleyebilirsin:

```javascript
onConnInfo: (state, extraInfo) => {
  console.log("connection state:", state, extraInfo);
}
```

Çoğu durumda gerek yoktur, bağlantı sessizce kurulur ya da `onClose` ile sebep verilerek düşer.

## Aynı oda, birden fazla client

Tek process'te birden fazla `Room.join` çağrılabilir; her biri ayrı `authObj` ile çağrılırsa farklı oyuncu olarak görünür. Pratikte kararlılığı düşüktür, bağlantılardan biri kopunca diğerleri etkilenebilir. Yük testi ya da çoklu bot senaryosunda her client'ı kendi process'inde tutmak önerilir.
