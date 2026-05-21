# Sözlük

Haxball ve `node-haxball` ile ilgili terimler. Tanımlar dökümantasyonun ihtiyacı için yazıldı, akademik tanım değil.

## addon

`node-haxball`'ın `RoomConfig`, `Plugin`, `Library`, `Renderer` ortak adı. Hepsi `Addon` base'inden türer ve odaya bağlanabilir kod birimlerini temsil eder.

## admin

Haxball odasındaki yetki bayrağı. Admin oyuncu start/stop, takım değiştirme, kick yapabilir. node-haxball'da `player.isAdmin` boolean ile takip edilir, `room.setPlayerAdmin(id, value)` ile atanır.

## allowFlags

Bir addon'un hangi modlarda (CreateRoom/JoinRoom) kullanılabileceğini söyleyen bit flag. Addon constructor'ında metadata içinde verilir.

## announcement

`sendAnnouncement` ile yollanan sistem mesajı. Normal chat'ten farklı: renkli, vurgulu, hedefli (sadece bir oyuncuya gösterilebilir), 1000 karakter limiti.

## asist

Top sahibinin gol atmadan önce ona pas atan oyuncuya kütüphane veya bot tarafından atanan kredi. Haxball'un kendi protokolünde "asist" kavramı yok, bot `onPlayerBallKick` zincirini tutarak hesaplar.

## auth (authKey, authObj)

Oyuncunun Haxball backend'inde kalıcı kimliği. `authKey` saklanan stringtir, `authObj` `Room.join`'e geçirilen kullanım nesnesidir. Ban listeleri ve profil yıldızı auth'a dayanır. Player.auth alanından okunur.

## backend

Haxball'un kendi sunucuları. Oda listesi, oda kaydı, oyuncu kimliği burada yönetilir. `node-haxball` backend'e doğrudan konuşur, kütüphanenin kendi backend'i yoktur.

## ban

Oda yöneticisinin bir oyuncuyu kalıcı engellemesi. `kickPlayer(id, reason, true)` ile yapılır. Ban listesi oda hayatınca tutulur, restart sonrası sıfırlanır (kendi disk persistance'ın olmadıkça).

## bCoef (bouncing coefficient)

Disc'in sıçrama katsayısı. 0=durdurur, 1=elastik (enerji kaybı yok). Genelde 0.5 civarı normal his verir.

## Before/After callback

node-haxball'a özgü çift handler. `onBeforeX` event'ten önce, `onAfterX` sonra çağırılır. `onBeforeX`'in dönüş değeri `onAfterX`'e `customData` olarak geçirilir. Plugin chain'i için kullanılır.

## callback

Bir event olduğunda kütüphanenin çağırdığı kullanıcı fonksiyonu. `room.onPlayerJoin = (player) => {...}` ataması en yaygın şekildir.

## cGroup, cMask

Collision group ve collision mask. İki disc çarpışır eğer her ikisinin grup-mask birleşimi bir bit'i paylaşıyorsa. Stadium'da bazı objelerin belli disc'lerle çarpışmamasını sağlar (örneğin sadece kırmızıyı geçiren duvar).

## checksum (stadium)

Stadium objesinin içeriğine göre hesaplanan hash. Custom stadyumlar için sıfır olmayan bir değerdir, varsayılanlar için `null`. Kütüphanenin `Stadium.calculateChecksum()` metodu ile elde edilir.

## client

Bir odaya katılan, hosting yapmayan bağlantı. `Room.join` ile başlatılır. Client fizik simülasyonunu host'tan alır, kendisi koşturmaz.

## customData

Plugin Before/After chain'inde callback'ler arası geçirilen serbest formatta veri. Bir önceki callback'in döndürdüğü değer, sonrakine bu parametre olarak ulaşır.

## damping

Disc'in hız kaybetme katsayısı. 1=hiç yavaşlamaz (top default), 0.96 oyuncu için normal, 0 anında durur.

## disc

Haxball'un fiziksel dünyasındaki yuvarlak nesneler. Top, oyuncular, bazı dekoratif objeler hepsi disc'tir. Index 0 her zaman top.

## EventFactory

Düşük seviye protokol mesajları üretmek için namespace. Çoğu bot kullanmaz. Replay editing ve custom protokol için.

## extrapolation

Client'ın host'tan gelen state'i gelecek tick'lere doğru tahmin etmesi. Lag hissini azaltır, renderer'lar bunu kullanır. `room.extrapolate(ms)`.

## fakePassword

`true` set edilirse oda listesinde kilit ikonu gösterilir ama oda gerçekten şifresiz olur. Vice versa. Karşı oyuncuyu yanıltmak için.

## fullKickOffReset

Stadium ayarı. Gol sonrası sadece top mu yoksa tüm disc'ler de mi reset olur, bazı stadyumlarda gol sonrası oyuncular spawn point'lerine geri ışınlanır.

## game tick

Oyun simülasyonunun bir adımı. Default 60 tick/saniye. `onGameTick` callback'i her tick'te tetiklenir.

## geo (GeoLocation)

Oda veya oyuncunun konumu. `{ code, lat, lon, flag }`. Oda listesinde sıralama buna göre, oyuncu bayrağı `code`'dan gelir.

## host

Oda'yı kuran ve sunucu rolünü oynayan bağlantı. `Room.create` ile başlatılır. Oda host'un Node process'inde yaşar; host çökerse oda biter.

## HBError

Kütüphanenin tüm hatalarını sarmalayan sınıf. `code` (ErrorCodes enum'undan int) ve `toString()` mesajı vardır.

## .hbr

Haxball replay dosyası uzantısı. Binary format. `Replay.readAll(bytes)` ile parse edilir.

## .hbs

Haxball stadium dosyası uzantısı. JSON5 türevi text. `Utils.parseStadium(text)` ile parse.

## headless

Görsel arayüzü olmayan, sadece oda barındıran Haxball script'i. node-haxball'ın doğal hedefi.

## headless avatar

Host tarafından bir oyuncuya atanan avatar. Oyuncunun kendi avatarını override eder. `setPlayerAvatar(id, value, true)`.

## invMass

1/kütle. Disc'in fiziksel "ağırlığının" tersi. 0 = sonsuz kütle (sabit), büyük değer = hafif disc. Oyuncuların default değeri ~0.5.

## kick (input)

Tekme tuşu (default `X`). `setKeyState(state, true)` ile state'in 5. biti aktif edilirse tekme atılır.

## kickoff

Maç başı veya gol sonrası top'un kickoff dairesine konup karşılıklı dizilmiş oyuncuların beklediği an. `onKickOff` callback'i bunu işaretler.

## Library

Diğer addon'ların kullandığı yardımcı kod birimi. Plugin'ler initialize edilmeden önce yüklenir, böylece onlardan bağımsız erişilebilir. `room.librariesMap` ile lookup.

## maç (game)

Bir `startGame` ile başlayıp ya doğal bitiş (skor/süre limiti) ya da `stopGame` ile sonlanan oyun periyodu.

## OperationType

Protokol event tiplerinin numerik karşılıklarını barındıran enum. `SendChat=4`, `StartGame=7`, vb.

## Plugin

Modüler özellik addon'u. Birden fazla yüklenebilir, runtime'da aktif/pasif edilebilir. AFK detection, chat moderasyon gibi şeyler plugin olarak yazılır.

## proxy (HttpsProxyAgent)

Bot trafiğini farklı bir IP üzerinden çıkarmak için kullanılan ara sunucu. IP başına 2 oda limitini aşmak için ana yol.

## reconnect

Bağlantı koptuktan sonra yeniden bağlanma. Kütüphane otomatik yapmaz; `onClose` callback'inde `setTimeout` ile manuel.

## recaptcha

Haxball'un otomasyon saldırılarına karşı oda kurma sırasında istediği human-verification. `headlesstoken` sayfasından çözülüp token alınır.

## reconnect backoff

Bağlantı koptuğunda yeniden denemeden önce bekleme süresi. Sabit yerine üstel (5s → 10s → 20s → ... 120s) kullanılır, backend rate limit'i tetiklemesin diye.

## Renderer

Görsel render addon'u. Browser canvas, terminal ASCII, image gibi çıktıları üretmek için. Node host bot'larında nadiren kullanılır.

## room link

`https://www.haxball.com/play?c=XYZ` formatında public bağlantı. Oda backend'de kayıt olduktan sonra `onAfterRoomLink` ile gelir.

## RoomConfig

Bir odanın ana mantığını tutan addon. Tek tane verilir. Plugin'lerden farkı: oda'nın "çekirdek" event handler katmanı olması.

## RoomState

Bir tick anındaki tüm state objesi. Oyuncular, disc pozisyonları, skor, stadyum. `room.state` ile okunur.

## sandbox

Network'a bağlanmayan offline simülasyon. `Room.sandbox(...)`. Stadium test, AI antrenmanı için.

## spawn point

Maç başında oyuncuların görüneceği koordinatlar. Her takım için ayrı liste, stadium dosyasında tanımlı.

## spec / spectator (team 0)

İzleyici grubu. Oynayamayanlar, beklemede olanlar burada. Team id `0`.

## stadium

Sahanın fiziksel modeli. Vertex, segment, plane, goal, disc, joint, spawn point'lerden oluşur. `.hbs` formatında saklanır.

## state extrapolation

Bkz. extrapolation.

## team

Kırmızı (id=1), mavi (id=2), spec (id=0). `Player.team` bir `Team` objesidir (`{ id, color }`).

## teamsLock

Takım kilidi. Açıkken oyuncular kendi takımlarını değiştiremez, sadece admin/host değiştirir. `lockTeams()`.

## tick

Bkz. game tick.

## token

Recaptcha sonrası alınan, oda kurma için backend'e gönderilen string. Birkaç dakika sonra expire olur. Headless token: host oda kurmak için. Client token: bazı oda'lara katılmak için.

## trait (stadium)

Stadium dosyasında tanımlanan ortak özellik şablonu. Vertex/segment'lerden `"trait": "ballArea"` ile referans edilir.

## WebRTC

Tarayıcılarda peer-to-peer iletişim için kullanılan teknoloji. Haxball client'ı host'a WebRTC (Web Real-Time Communication) ile bağlanır. `node-haxball`'da `node-datachannel` paketi browser'sız WebRTC sağlar.
