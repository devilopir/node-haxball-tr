# Haxball ve headless host

Haxball, tarayıcıda çalışan iki boyutlu bir top oyunudur. Normal kullanımda iki oyuncu odaya girer, biri "host" olur ve oyun trafiği bu host'un tarayıcısı üzerinden akar. Host odadan çıkarsa oda kapanır.

Headless host, bu rolü oynayan bir tarayıcı script.idir. Görsel arayüzü yoktur, kendisi oynamaz, sadece odayı ayakta tutar ve oyunculara ev sahipliği eder. Resmi sürüm `https://www.haxball.com/headless` adresinde çalışır ve `HBInit(...)` çağrısıyla bir `room` objesi döndürür. Bot, lig, turnuva, otomatik moderasyon, bunların hepsi bu room objesinin üzerine yazılan event handler'larla yapılır.

`node-haxball`, bu headless host mantığını Node.js'te baştan implement eder. Tarayıcıya bağımlı değildir, doğrudan Haxball'un WebRTC tabanlı protokolüne konuşur. Aynı kütüphane host, client veya standalone simülasyon olarak kullanılabilir; resmi headless'in yapmadığı şeyleri (replay okuma, istemci otomasyonu, sandbox) da kapsar.

## Mimari

Bir odanın hayatında üç taraf vardır.

**Backend**: Haxball'un kendi sunucuları. Oda listesini tutar, yeni oda kayıtlarını alır, eşleştirme yapar. Bu sunuculara doğrudan bir API yoktur, `getRoomList()` gibi fonksiyonlar HTTP üzerinden onlara konuşur.

**Host**: Odayı yaratan taraf. Oyunun fiziksel simülasyonunu (top, oyuncu pozisyonları, çarpışmalar) tick tick koşturur. Oyuncular host'a WebRTC ile bağlanır. Host odadan çıkarsa oda biter.

**Client**: Odaya katılan oyuncular. Host'tan aldığı state'i lokal olarak extrapolate eder, kendi inputlarını host'a gönderir. `node-haxball`'da Room.join ile bir bot'u client olarak bağlamak da mümkündür.

Resmi headless API'de sadece host tarafı vardır. `node-haxball`'da hem host (`Room.create`), hem client (`Room.join`), hem de hiçbir yere bağlı olmayan offline simülasyon (`Room.sandbox`) seçenekleri bulunur.

## Bilinmesi gereken sınırlar

**IP başına 2 oda**: Haxball backend'i aynı IP'den en fazla iki public odanın açık olmasına izin verir. Üçüncüsünü kurmaya çalışırsan reddedilir. Üretimde birden fazla oda işletilecekse proxy ya da farklı sunucular gerekir; [rehber/16-proxy-deploy.md](../rehber/16-proxy-deploy.md) detayları işliyor.

**Token expire**: Oda kurmak için recaptcha token gerekir. Token birkaç dakika içinde geçersiz olur, kurulum başarısız olursa süre aşımıdır, kod hatası değildir. [02-kurulum-ve-token.md](02-kurulum-ve-token.md) süreci anlatıyor.

**Versiyon eşleşmesi**: Oyuncuların istemcisi ile host arasında protokol versiyonu uyuşmalıdır. `node-haxball` default olarak `version: 9` kullanır; Haxball client güncellenirse bu sayı güncellenene kadar oda boş görünür. Hızlı düzeltme `Room.create` çağrısında `version` parametresini elle vermektir.

**Tek thread**: Node.js tek event loop'ta çalışır. Yoğun hesaplama (örneğin her tick'te kompleks analiz) yapılacaksa worker thread'e taşımak gerekir. Tick rate sabit 60 Hz'dir.

## node-haxball ile resmi HBInit farkı

Resmi HBInit ile yazılmış kodu birebir taşıyamazsın. Önemli farklar:

| Kavram | Resmi HBInit | node-haxball |
|---|---|---|
| Başlatma | `HBInit({...})` çağrısı bir room döner | `Room.create({...}, {...})` çağrısı asenkron, `onOpen` callback ile room verir |
| Oyuncu objesi | `room.getPlayer(id)` her zaman aynı objeyi döndürür | `room.state.players` üzerinden okunur, immutable snapshot mantığı vardır |
| Event'ler | `room.onPlayerJoin = fn` doğrudan atama | `RoomConfig` üzerinden tanımlanır, ya da `room.onPlayerJoin = fn` atanır (her ikisi destekli) |
| Plugin | Yok, modülariteyi sen yaparsın | `Plugin` base sınıfı, addon sistemi var |
| İstemci tarafı | Yok, headless sadece host | `Room.join` ile client bot yazılabilir |
| Replay | Sadece `startRecording`/`stopRecording` byte verir | Aynısı + `Replay.read/readAll/trim` ile okuma ve düzenleme |

Geçişi düşünüyorsan [rehber/01-mimari.md](../rehber/01-mimari.md) ve [rehber/04-callback-sistemi.md](../rehber/04-callback-sistemi.md) iki API'nin felsefe farkını netleştirir.
