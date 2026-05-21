# Mimari: host, client, standalone, sandbox

`node-haxball` aynı kod tabanından dört farklı çalışma modu sunar. Hangisini kullanacağın botun ne yapacağına bağlıdır.

## Host

`Room.create(...)` ile başlatılır. Backend'e yeni bir oda kaydı yaptırır, oyuncuların bağlanacağı public bir link üretir, oyun simülasyonunu (fizik tick'leri, çarpışmalar, gol algılama) bu Node process'inde koşturur.

Bir botun yetki sınırı genelde host modundadır. Skor limiti ayarlamak, oyuncuyu takım değiştirmek, banlamak, bunların hepsi host tarafından yapılabilir. Resmi HBInit ile yazabildiğin her şey burada karşılığını bulur.

Host olduğun andan itibaren tüm trafik senden geçer. 10 oyuncu, saniyede 60 tick × ~1 KB state ≈ 600 KB/s gönderim. Sunucu kapasitesi planlanırken bunu unutma; küçük bir VPS bile rahat kaldırır ama IP başına 2 oda limiti seni nakliyat sorunuyla daha hızlı karşılaştırır.

## Client

`Room.join(...)` ile başlatılır. Var olan bir odaya normal oyuncu olarak bağlanır. Host'un gönderdiği state'i alır, kendi inputunu host'a gönderir.

Client modunda fiziksel simülasyonun **otoritesi sende değil**. `room.state` okunur, host'un son gönderdiği snapshot'tır. Lokal olarak `room.extrapolate(ms)` ile kısa süreli ileri sarım yapılabilir (renderer'lar bunu kullanır) ama bu sadece tahmindir, host'un dediği geçerlidir.

Kick, ban, takım değiştirme gibi host yetkili işlemleri client'tan tetikleyemezsin, eğer odanın admin'iysen `room.kickPlayer(...)` çağırabilirsin, bu host'a admin operasyonu olarak iletilir. Admin değilsen çağrı sessizce reddedilir.

## Standalone (sandbox)

`Room.sandbox(...)` ile başlatılır. Backend'e bağlanmaz, oda kaydı yapmaz, network trafiği üretmez. Tamamen lokal bir oyun simülasyonudur.

Kullanım alanları:

- Stadyum tasarımını test etmek: gerçek bir oda açmadan çarpışmaları, fiziği görmek
- Bot AI'ı offline antrenmanı: input → state ilişkisini hızlandırılmış simülasyonla öğretmek
- Birim testler: belirli bir state'i kurup belirli bir input verince sonucun ne olduğunu deterministik olarak ölçmek

Sandbox'ın saat kontrolü manueldir. `runSteps(n)` ile n tick ilerletilir; gerçek zaman beklenmez. Bu, normal saniyede 60 tick'lik oyunu istediğin hıza çıkarmana izin verir.

## Stream watcher

`Room.streamWatcher(initialStreamData, ...)` ile başlatılır. Özel bir kullanım: bir streaming server'dan alınan binary state akışını oynatır. Pratikte sadece [streamingServer](https://github.com/wxyz-abcd/node-haxball/tree/main/streamingServer) projesinde devreye giriyor; normal bot/host işleri için ihtiyacın olmaz.

## Hangisini ne zaman

| İhtiyaç | Mod |
|---|---|
| Oda açıp moderasyon yapacaksın | Host |
| Var olan odaya izleyici/asistan bot bağlayacaksın | Client |
| Discord ↔ Haxball köprüsü | Client (chat dinleyici), Host (oda da senin değilse client yeterli) |
| Stadyum dosyasını fizik açısından test edeceksin | Sandbox |
| Bot karar mantığını gerçek odayı bağlamadan unit-test edeceksin | Sandbox |
| Replay analizi | Replay API (`Replay.readAll`), mod değil, bkz. [13-replay.md](13-replay.md) |

İki modu aynı process'te birleştirmek teknik olarak mümkün ama önerilmez. Host olarak çalışan bir bot için izleme/raporlama gerekiyorsa ya o botun içine logic'i göm, ya da ayrı bir client process'i bağlat.

## Ortak yapı taşları

Hangi modda olursan ol, kütüphane şu parçaları sunar:

- **Room**: Modu temsil eden ana obje. Event handler'ları ve state burada.
- **RoomState**: Bir tick anındaki tüm state (oyuncular, disc'ler, skor). Hem host hem client'ta okunur.
- **Callback sistemi**: Event'lere abone olmanın iki yolu (`room.onXxx = fn` ve `Callback.add`/`RoomConfig`). [04-callback-sistemi.md](04-callback-sistemi.md) farkı işliyor.
- **Addon'lar**: Plugin, RoomConfig, Library, Renderer, bot mantığını modüler parçalara ayırmak için. [09-addon-sistemi.md](09-addon-sistemi.md).
- **Utils**: Auth üretimi, oda listesi sorgulama, geolocation, stadium parse, yardımcı fonksiyonlar. [11-utils-ve-auth.md](11-utils-ve-auth.md).

Bu yapı taşları her modda aynı isimle çalışır, sadece **hangi callback'lerin tetikleneceği** ve **hangi metodların available olduğu** moda göre değişir. Host-only callback'ler (örneğin `onAfterRoomLink`, `onBanClear`) client modunda hiç gelmez. Sandbox modu ise daha küçük bir metod kümesi sunar: `setCurrentStadium`, `startGame`, `setPlayerTeam`, `setDiscProperties` gibi temel metodlar var ama `getPlayer`, `getBall`, `lockTeams`, `clearBans` gibi yardımcılar yoktur, bunlar sadece network'e bağlı Room (`Room.create` / `Room.join`) objesinde bulunur. Sandbox'ta state'e doğrudan `room.state.players` üzerinden erişilir. Detay [04-callback-sistemi.md](04-callback-sistemi.md) içinde.
