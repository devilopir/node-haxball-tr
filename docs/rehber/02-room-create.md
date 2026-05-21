# Room.create

Host olarak yeni bir oda kurmak için `Room.create(createParams, commonParams)` çağrılır. Çağrı asenkrondur, backend ile el sıkışma birkaç saniye sürer, başarı `commonParams.onOpen` callback'i ile bildirilir.

## İmza

```typescript
Room.create(createParams: CreateRoomParams, commonParams: CommonNetworkRoomParams): {
  cancel: () => void;
  useRecaptchaToken: (token: string) => void;
}
```

Dönen objenin iki kullanımı vardır. `cancel()`, hâlâ bağlanma aşamasındaysa süreci iptal eder; bağlantı kurulduktan sonra çağırılırsa odayı kapatır. `useRecaptchaToken`, recaptcha gereken durumlarda yeni token sağlamak için kullanılır.

## createParams alanları

Zorunlu alanlar:

- **`name`** (string): Oda adı, oda listesinde görünür.
- **`token`** (string): `https://www.haxball.com/headlesstoken`'den alınan recaptcha token. Süresi geçtiyse bağlantı reddedilir.
- **`maxPlayerCount`** (int): Maksimum oyuncu sayısı.
- **`showInRoomList`** (boolean): Public listede görünsün mü.
- **`geo`** (GeoLocation): `{ code: "tr", lat: 41, lon: 29 }` formatında. Oda listesindeki bayrak ve sıralama bunu kullanır. `Utils.getGeo()` ile sunucunun gerçek konumundan otomatik üretilebilir.

Opsiyonel alanlar:

- **`password`** (string | null): Oda şifresi. `null` ise şifresiz.
- **`noPlayer`** (boolean): `true` ise host kendisini oyuncu olarak yaratmaz. Bot kullanımında neredeyse her zaman `true` istenir.
- **`playerCount`** (int): Oda listesinde sahte oyuncu sayısı göstermek için. Gerçek bağlantıları etkilemez, sadece görseldir.
- **`unlimitedPlayerCount`** (boolean): `maxPlayerCount` sınırını bypass eder, oda istediği kadar oyuncu kabul edebilir. Resmi backend oda listesine yüksek oyuncu sayılı odaları belirli bir eşikten sonra göstermeyebilir; bu kütüphane içinde sabit bir limit değildir, default backend politikasıdır.
- **`fakePassword`** (boolean): `true`/`false` verilirse listede kilit ikonu gerçek durumla bağımsız olarak ayarlanır. `null` ise normal davranır.
- **`onError`** ((err, playerId) => void): Bir istemci bağlantısında exception fırlarsa çağırılır; ardından o oyuncunun bağlantısı kapanır.

## commonParams alanları

Bu obje hem `create` hem `join` arasında ortaktır.

- **`storage`**: Host'un kimliği. `{ player_name, avatar, geo, player_auth_key, crappy_router }`. `noPlayer: true` durumunda dahi `player_name` verilmek zorundadır, protokol gereği. `crappy_router: true`, kötü router'lar için zaman aşımı eşiğini 4'ten 10 saniyeye çıkarır.
- **`config`** (RoomConfig | null): Bir `RoomConfig` addon'u. Detay [09-addon-sistemi.md](09-addon-sistemi.md).
- **`plugins`** (Plugin[]): Plugin listesi.
- **`libraries`** (Library[]): Library listesi.
- **`renderer`** (Renderer | null): Sandbox dışında nadiren kullanılır.
- **`version`** (int): Protokol versiyonu, default 9. Haxball client güncellenirse manuel bumping gerekebilir.
- **`proxyAgent`**: HTTPS proxy agent. IP başına 2 oda limitini aşmak için ana yol. Detay [16-proxy-deploy.md](16-proxy-deploy.md).
- **`identityToken`**: Custom backend'de oyuncu kimliği için. Default Haxball backend'inde ihtimal yoktur.
- **`debugDesync`**: Host'ta `true`, client'ta bir desync handler. Geliştirme aşamasında host/client state farkını yakalamak için.
- **`noPluginMechanism`** (boolean): `true` ise plugin/renderer mekanizması bypass edilir, sadece `room._onXxx` callback'leri çağırılır. Çok yüksek performans gerektiren özel durumlar dışında ihtiyacın olmaz.

## Yaşam döngüsü callback'leri

- **`preInit(room)`**: Room objesi yaratıldıktan **hemen sonra**, addon'lar initialize olmadan önce. GUI bağlamak veya room üzerine custom fonksiyon eklemek için.
- **`onOpen(room)`**: Oda kurulumu başarılı oldu. Bot mantığı buraya yazılır.
- **`onClose(reason)`**: Oda kapandı. `reason` bir `Errors.HBError`. `.toString()` insan okunur mesaj verir.
- **`onConnInfo(state, extraInfo)`**: Bağlantı aşamasındaki state geçişleri. Debug için faydalı, normal kullanımda boş bırakılır.

## Minimal örnek

```javascript
const { Room } = require("node-haxball")();

Room.create({
  name: "Oda",
  token: "thr1.AAAAA...",
  maxPlayerCount: 10,
  noPlayer: true,
  showInRoomList: true,
  geo: { code: "tr", lat: 41, lon: 29 },
}, {
  storage: { player_name: "host" },
  onOpen: (room) => {
    room.onAfterRoomLink = (link) => console.log(link);
  },
  onClose: (reason) => {
    console.error("kapandı:", reason.toString());
  },
});
```

## onOpen ve onAfterRoomLink farkı

`onOpen`, WebRTC bağlantısı kurulup ilk state alındığı anda çağırılır. Bu anda **henüz oda backend listede yayınlanmamış** olabilir; link'e karşı doğrulama gelmemiş, oyuncular bağlanabilecek durumda değildir.

`onAfterRoomLink`, backend "evet, bu link aktif" cevabını verdiğinde tetiklenir. Public listede görünme, public link'in çalışması bu noktadadır.

Sıraya göre:

1. `preInit(room)`, addon init öncesi
2. addon initialize'lar
3. `onOpen(room)`, bağlantı tamam
4. `onBeforeRoomLink` → `onAfterRoomLink`, link backend'de aktif

Event handler'larını `onOpen` içinde atamak güvenlidir. Stadium/limit gibi backend'e gönderilen ayarları `onAfterRoomLink` içinde yapmak ilk-oyuncu race condition'larını engeller.

## Token süresi geçtiyse

`onClose` tetiklenir, `reason.code` `MissingRecaptchaCallbackError` veya benzeri olur. `useRecaptchaToken(newToken)` çağrısı **sadece create sırasında**, henüz oda kurulmadan işe yarar, kurulduktan sonra token süresi konu değildir, bağlantı zaten yapılmıştır.

Pratik yaklaşım: token'ı her başlatmada elle güncelle, prodüksiyonda token rotation mekanizması kur (örneğin Discord'dan komutla token gönder ve botu restart et).

## Birden fazla oda

Tek Node process'inde birden fazla `Room.create` çağrısı yapılabilir. Her birinin kendi `room` objesi olur. Pratikte iki şey seni yakalar: Haxball backend IP başına 2 oda limitler ve aynı process içinde herhangi bir oda crash olursa hepsi düşer. Birden fazla oda işletecekseniz odaları ayrı process'lerde tutmak operasyon açısından daha sağlıklıdır.
