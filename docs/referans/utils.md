# Utils namespace referansı

`Utils`, kütüphanenin yardımcı fonksiyonlarını barındırır. Auth üretimi, oda listesi sorgulama, renk/IP dönüşümleri, stadium parse, geo işlemleri burada.

```javascript
const { Utils } = require("node-haxball")();
```

Pratik örnekler için [rehber/11-utils-ve-auth.md](../rehber/11-utils-ve-auth.md).

## Auth

### Utils.generateAuth()

```
Utils.generateAuth(): Promise<[string, Auth]>
```

Yeni bir auth key + auth objesi üretir. İlk kullanımda çağrılır, `authKey` saklanır. Internal hata olursa Promise reject olur.

### Utils.authFromKey(authKey)

```
Utils.authFromKey(authKey: string): Promise<Auth>
```

Saklanmış `authKey`'den auth objesi yeniden üretir. Sonraki bağlantılarda aynı kimlikle gelmek için.

### Utils.authToNumber(auth)

```
Utils.authToNumber(auth: string): BigInt
```

Auth stringini BigInt'e dönüştürür. Kompakt saklama için.

### Utils.numberToAuth(auth)

```
Utils.numberToAuth(auth: BigInt): string
```

`authToNumber`'ın tersi.

## Oda listesi ve geolocation

### Utils.getRoomList()

```
Utils.getRoomList(): Promise<RoomData[]>
```

Resmi backend'den public oda listesini çeker. Her `RoomData`: `id`, `name`, `password` (boolean), `players`, `maxPlayers`, `geo`, `version`, `details`, `dist?` (calculateAllRoomDistances çağrılmışsa).

### Utils.calculateAllRoomDistances(geo, list)

```
Utils.calculateAllRoomDistances(geo: GeoLocation, list: RoomData[]): void
```

`list`'in her elemanına `.dist` field'i ekler (verilen `geo`'ya mesafe).

### Utils.getGeo()

```
Utils.getGeo(): Promise<GeoLocation>
```

Sunucunun gerçek konumunu döndürür. `Room.create({...}).geo` için kullanışlı.

### Utils.parseGeo(geoStr?, fallback?, retNull?)

```
Utils.parseGeo(geoStr?: object | string, fallback?: object, retNull?: boolean): GeoLocation
```

Yarı tanımlanmış geo objesini normalize eder. `retNull: true` ise eksik girişlerde null döner.

## Stadium

### Utils.parseStadium(text)

```
Utils.parseStadium(textDataFromHbsFile: string): Impl.Stadium.Stadium | undefined
```

`.hbs` dosyasının metni → parsed Stadium. Başarısızsa `undefined`.

### Utils.exportStadium(stadium)

```
Utils.exportStadium(stadium: Impl.Stadium.Stadium): string
```

Parsed stadium → `.hbs` text. Sandbox modunda düzenlenmiş haritayı diske yazmak için.

### Utils.getDefaultStadiums()

```
Utils.getDefaultStadiums(): Impl.Stadium.Stadium[]
```

Resmi Haxball'un 10 varsayılan haritası. İsimle filtrele:

```javascript
const big = Utils.getDefaultStadiums().find((s) => s.name === "Big");
```

## Renk dönüşümleri

### Utils.colorToNumber(color)

```
Utils.colorToNumber(color: string): int
```

> **⚠️ Sınırlı format desteği**: Sadece `"rgb(R,G,B)"` formatı kabul edilir. `#ff0000`, `"ff0000"`, `rgba(...)` formatları **hata fırlatır** (`Cannot read properties of null`). Hex string'lerden int üretmek için `parseInt("ff0000", 16)` kullan.

```javascript
Utils.colorToNumber("rgb(255,0,0)"); // 16711680
```

Hex string'lerden int üretmek istiyorsan `parseInt("ff0000", 16)` kullan.

### Utils.numberToColor(number)

```
Utils.numberToColor(number: int): string
```

Int → `"rgba(r,g,b,a)"` formatında CSS string. Alpha kanalı 0-255 aralığındadır.

```javascript
Utils.numberToColor(16711680); // "rgba(255,0,0,255)"
```

### Utils.hexStrToNumber(str)

```
Utils.hexStrToNumber(str: string): string
```

Hex string'i normalleştirilmiş hex string'e çevirir (whitespace/prefix temizliği). Dönüş tipi `string`, parse edilmiş `int` değil. Sayısal değer için bunu `parseInt(result, 16)` ile çözmen gerekir.

## IP dönüşümleri

### Utils.ipToNumber(ip)

```
Utils.ipToNumber(ip: string): NumericIPv4 | NumericIPv6 | null
```

IPv4 veya IPv6 stringini numerik temsile çevirir. Geçersizse `null`.

### Utils.numberToIp(ip)

```
Utils.numberToIp(ip: NumericIPv4 | NumericIPv6): string | null
```

Tersi.

### Utils.byteArrayToIp(data)

```
Utils.byteArrayToIp(data: Uint8Array): string
```

Byte array temsilinden IP stringine.

### Utils.compareBits(value1, value2, bitMask)

```
Utils.compareBits(value1: NumericIPv4, value2: NumericIPv4, bitMask: NumericIPv4): boolean
Utils.compareBits(value1: NumericIPv6, value2: NumericIPv6, bitMask: NumericIPv6): boolean
```

İki IP'nin maskeli karşılaştırması, IP range ban'ı için kullanışlı.

## Input ve key state

### Utils.keyState(dirX, dirY, kick)

```
Utils.keyState(dirX: Direction, dirY: Direction, kick: boolean): int
```

Yön ve ateş bilgisinden int input state üretir. `Direction` sadece üç değer alır: `-1` (Backward), `0` (Still), `1` (Forward). Detay [`Direction` enum'unda](enums.md#direction).

### Utils.reverseKeyState(state)

```
Utils.reverseKeyState(state: int): { dirX: Direction, dirY: Direction, kick: boolean }
```

`keyState`'in tersi: int → `{ dirX, dirY, kick }` objesi. `dirX`/`dirY` değerleri yine -1/0/1.

## Promise yardımcısı

### Utils.promiseWithTimeout(promise, msec)

```
Utils.promiseWithTimeout(promise: Promise, msec: int32): Promise
```

Verilen promise'e timeout ekler. Süre dolarsa reject olur.

## Tick koordinasyonu

### Utils.runAfterGameTick(callback, ticks)

```
Utils.runAfterGameTick(callback: () => void, ticks: int): void
```

`ticks` adet game tick'i sonra callback'i çalıştırır. Oyun duraklatılırsa sayaç da durur, `setTimeout`'tan farkı bu.

## Token yardımcıları

### Utils.refreshRoomToken(params)

```
Utils.refreshRoomToken(params: RefreshRoomTokenParams): Promise<RefreshRoomTokenReturnValue>
```

`params`: `{ token?, rcr? }`. Recaptcha response (`rcr`) elde etmek tarayıcı gerektirir.

### Utils.generateRoomId(params)

```
Utils.generateRoomId(params: GenerateRoomIdParams): Promise<GenerateRoomIdReturnValue>
```

Oda kurmadan önce oda ID'sini önceden almak için. `{ roomId, newToken }` döner.
