# Utils ve auth

`Utils` namespace, kütüphanenin yardımcı fonksiyonlarını barındırır. Bot yazarken sık dokunduğun yer burası, auth üretimi, oda listesi, renk/IP dönüşümleri, stadium parse'ı buradadır.

```javascript
const { Utils } = require("node-haxball")();
```

## Auth, kalıcı kimlik

`authObj`, Haxball backend'inin "aynı kişi mi" sorusunu cevapladığı şeydir. Ban'lar, profil yıldızı, oda istatistikleri buna göre tutulur.

```javascript
const [authKey, authObj] = await Utils.generateAuth();
// authKey: kalıcı hex string, sakla
// authObj: tek seferlik bağlantı objesi, kullanımdan sonra at
```

`authKey`'i kaydet, sonraki bağlantılarda aynı kimlikten gel:

```javascript
const authObj = await Utils.authFromKey(savedAuthKey);
```

İkinci bir fonksiyon:

```javascript
const bigInt = Utils.authToNumber(authString);  // auth → BigInt
const auth = Utils.numberToAuth(bigInt);         // BigInt → auth
```

Ban listesinde auth key'leri kompakt saklamak için kullanılır. Pratikte çoğunlukla `Player.auth` stringini olduğu gibi saklamak daha basittir.

## Oda listesi

```javascript
const rooms = await Utils.getRoomList();
```

Resmi Haxball backend'inden public oda listesini çeker. Her `RoomData`:

- `id`, `Room.join`'de kullanılacak
- `name`, oda adı
- `password`, şifre korumalı mı (boolean)
- `players`, `maxPlayers`
- `geo`, `{ lat, lon, flag }`
- `version`, protokol versiyonu

İstatistik/monitoring botu yazıyorsan periyodik çağrı atabilirsin. Backend'i bombardımana tutma: dakikada birkaç çağrıdan fazlası rate limit yer.

```javascript
const myGeo = await Utils.getGeo();
Utils.calculateAllRoomDistances(myGeo, rooms);
// rooms[i].dist artık dolu, sıralama yapabilirsin
rooms.sort((a, b) => a.dist - b.dist);
```

## Geolocation

```javascript
const geo = await Utils.getGeo();
// { code: "TR", lat: 41.0, lon: 29.0, flag: "tr" } gibi
```

Otomatik geo tespiti, sunucunun gerçek konumundan. `Room.create({...})` çağrısında `geo:` alanına bunu vermek liste sıralamasını doğru yapar.

Manuel parse de mümkün:

```javascript
const geo = Utils.parseGeo({ code: "tr", lat: 41, lon: 29 });
const geoFromStr = Utils.parseGeo("tr", { code: "us", lat: 0, lon: 0 });
```

`parseGeo`, eksik alanları doldurur ya da fallback'i kullanır.

## Stadyum dönüşümleri

```javascript
const stadium = Utils.parseStadium(hbsText);    // string → Stadium | undefined
const text = Utils.exportStadium(stadium);       // Stadium → .hbs string
const defaults = Utils.getDefaultStadiums();     // Stadium[]
```

`parseStadium` parse başarısız olursa `undefined` döner. Bot kullanıcı upload'u kabul ediyorsa hatayı yakala:

```javascript
const stadium = Utils.parseStadium(uploadedText);
if (!stadium) {
  room.sendAnnouncement("Geçersiz stadyum dosyası", playerId, 0xFF0000, "normal", 2);
  return;
}
```

`getDefaultStadiums` 10 varsayılan haritayı içerir. İsimlere göre filtrele.

## Renk dönüşümleri

```javascript
const num = Utils.colorToNumber("rgb(255,0,0)");  // 16711680
const str = Utils.numberToColor(16711680);         // "rgba(255,0,0,255)"
```

**Önemli sınır**: `colorToNumber` yalnızca `"rgb(R,G,B)"` formatını destekler. `#ff0000`, `FF0000`, `rgba(R,G,B,A)` gibi diğer formatlar hata fırlatır. Hex kullanmak istiyorsan kendin `parseInt("ff0000", 16)` ile dönüştür:

```javascript
const num = parseInt("ff0000", 16); // 16711680
```

`numberToColor` çıktısında alpha kanalı **0-255** aralığındadır (0-1 değil). `-1` değeri beyaza eşdeğer dönüşür (Haxball protokolünde -1 şeffaf anlamına gelir ama `numberToColor` bunu CSS string olarak göstermez).

`sendAnnouncement` ve `setTeamColors` int renk ister; `0xFFFFFF` (16777215) beyaz, `0x000000` siyah, `-1` şeffaf, bu int değerleri doğrudan ver.

## IP dönüşümleri

`Player.conn` bir IP'nin numerik temsilidir. Loglama için string'e çevirmek istersen:

```javascript
const ipStr = Utils.byteArrayToIp(Buffer.from(player.conn, "hex"));
// veya
const ipNum = Utils.ipToNumber("192.168.1.1");
const back = Utils.numberToIp(ipNum);  // "192.168.1.1"
```

IPv4 ve IPv6 ikisi de destekleniyor. Aynı IP üzerinden gelen multiple bağlantıları tespit etmek için bu çevrim kullanılır.

## Input ve key state

Oyuncu input'u (yön + ateş) bir int olarak kodlanır. Bot bir input vermek isterse (sandbox veya client modu):

```javascript
// dirX, dirY: -1 (Backward), 0 (Still), 1 (Forward)
const state = Utils.keyState(1, -1, true); // sağa-yukarı + tekme
room.setKeyState(state);
```

Tersine:

```javascript
const { dirX, dirY, kick } = Utils.reverseKeyState(state);
```

`Direction` enum'u sadece üç değere sahiptir: `Backward = -1`, `Still = 0`, `Forward = 1`. Aynı set hem X hem Y ekseninde kullanılır, `Math.sign(delta)` ile direkt yön hesaplanır.

## runAfterGameTick

Belli sayıda tick sonra bir callback çalıştırmak için:

```javascript
Utils.runAfterGameTick(() => {
  room.startGame();
}, 60); // 60 tick = 1 saniye sonra
```

`setTimeout`'tan farkı: game tick'lerine bağlı, oyun duraklatıldığında bu sayaç da durur. Oyun mantığıyla senkron kalmak gerektiğinde `setTimeout`'tan daha doğru.

## promiseWithTimeout

```javascript
const result = await Utils.promiseWithTimeout(somePromise, 5000);
```

Promise zaman aşımı ekler, 5 saniye içinde çözülmezse reject olur. `Utils.getRoomList()` veya `Utils.generateAuth()` gibi ağ çağrıları için yararlı.

## Token rotation

Backend yeni token üretmek için:

```javascript
const result = await Utils.refreshRoomToken({
  token: oldToken,
  rcr: recaptchaResponse,
});
// result.code, result.value
```

`rcr` (recaptcha response) elde etmek için tarayıcıda recaptcha çözmek gerekir, Node tarafında otomatik değildir. Pratik kullanım: Discord botu tarayıcıdan token alıp, çağrıyla yenileyip Node bot'a iletir.

```javascript
const { roomId, newToken } = await Utils.generateRoomId({
  token: yourToken,
  proxyAgent: null,
});
```

Oda kurmadan önce oda ID'sini önceden almak için. Nadiren ihtiyaç olur; oda linkini önceden bilmen gerekiyorsa.

## Hızlı referans tablosu

| İhtiyaç | Fonksiyon |
|---|---|
| Yeni auth üret | `Utils.generateAuth()` |
| Auth key'den objeyi oluştur | `Utils.authFromKey(key)` |
| Public oda listesi | `Utils.getRoomList()` |
| Sunucu geo'su | `Utils.getGeo()` |
| Geo objeyi normalize et | `Utils.parseGeo(input, fallback)` |
| .hbs metnini parse et | `Utils.parseStadium(text)` |
| Stadium → .hbs metni | `Utils.exportStadium(stadium)` |
| Varsayılan haritalar | `Utils.getDefaultStadiums()` |
| Hex/CSS string → renk int | `Utils.colorToNumber(str)` |
| Renk int → CSS string | `Utils.numberToColor(n)` |
| Promise'e timeout ekle | `Utils.promiseWithTimeout(p, ms)` |
| IP int ↔ string | `Utils.ipToNumber(s)` / `Utils.numberToIp(n)` |
| Tick sonrası callback | `Utils.runAfterGameTick(cb, ticks)` |
