# node-haxball-tr

[`node-haxball`](https://github.com/wxyz-abcd/node-haxball) için Türkçe dökümantasyon ve tutorial koleksiyonu. Hedef okuyucu: JavaScript temelleri olan, kendi Haxball odasını ya da botunu yazmak isteyen Türk geliştirici.

`node-haxball`, Haxball'un protokolünü Node.js'te baştan implement eden bir kütüphane. Resmi headless API'nin (HBInit) yapabildiği her şeyi yapar, ek olarak istemci botları, replay okuma, sandbox simülasyonu, plugin/library/renderer addon sistemi ve düşük seviyeli protokol erişimi sunar.

## İçerik

**baslangic/**, Sıfırdan ilk çalışan odaya kadar.

- [Haxball nedir, headless host nedir](baslangic/01-haxball-nedir.md)
- [Kurulum ve token alma](baslangic/02-kurulum-ve-token.md)
- [İlk oda: host bot](baslangic/03-ilk-oda.md)
- [İlk istemci: odaya katılan bot](baslangic/04-ilk-istemci.md)

**rehber/**, Konu odaklı, kavramları derinleştiren bölümler.

- [Mimari: host, client, standalone, sandbox](rehber/01-mimari.md)
- [Room.create](rehber/02-room-create.md)
- [Room.join](rehber/03-room-join.md)
- [Callback sistemi](rehber/04-callback-sistemi.md)
- [Oyun akışı event'leri](rehber/05-event-akisi.md)
- [Oyuncu yönetimi: takım, admin, kick, ban](rehber/06-oyuncu-yonetimi.md)
- [Oyun kontrolü: skor, süre, stadyum](rehber/07-oyun-akisi.md)
- [Stadyum dosyası ve disc fiziği](rehber/08-stadyum-ve-disc.md)
- [Addon sistemi](rehber/09-addon-sistemi.md)
- [Plugin yazma](rehber/10-plugin-yazma.md)
- [Utils ve auth](rehber/11-utils-ve-auth.md)
- [Veri saklama](rehber/12-veri-saklama.md)
- [Replay](rehber/13-replay.md)
- [EventFactory ve protokol katmanı](rehber/14-eventfactory-ve-protokol.md)
- [Tarayıcıda kullanım](rehber/15-tarayicida-kullanim.md)
- [Proxy, deploy ve IP limiti](rehber/16-proxy-deploy.md)
- [Sandbox ve stream watcher](rehber/17-sandbox-ve-stream.md)
- [Yerelleştirme (Language API)](rehber/18-yerellestirme.md)

**tutoriallar/**, Her biri çalışan, kopyalanabilir tam bot.

- [Temel host bot](tutoriallar/01-temel-host-bot.md)
- [Komut çerçevesi](tutoriallar/02-komut-cercevesi.md)
- [AFK kick botu](tutoriallar/03-afk-kick-botu.md)
- [Takım dengeleme](tutoriallar/04-takim-dengeleme.md)
- [İstatistik takibi](tutoriallar/05-istatistik-takibi.md)
- [Admin yetki sistemi](tutoriallar/06-admin-yetki-sistemi.md)
- [İstemci botu](tutoriallar/07-istemci-bot.md)
- [Replay analizi](tutoriallar/08-replay-analizi.md)

**referans/**, API yüzeyinin tamamı, aramak için.

- [Room API](referans/room-api.md)
- [Callback listesi](referans/callbacks.md)
- [Utils namespace](referans/utils.md)
- [Enum'lar](referans/enums.md)
- [Tipler](referans/tipler.md)
- [Addon base sınıfları](referans/addon-api.md)
- [EventFactory](referans/eventfactory.md)
- [Replay API](referans/replay-api.md)
- [Query namespace](referans/query.md)
- [Renderer API](referans/renderer-api.md)
- [Hata kodları](referans/errors.md)

**ortak/**

- [.hbs stadyum dosya formatı](ortak/stadyum-formati.md)
- [Sık yapılan hatalar](ortak/sik-yapilan-hatalar.md)
- [Sözlük](ortak/sozluk.md)

## Nasıl gezinilir

Daha önce Haxball host script yazmadıysan: `baslangic/` klasörünü sırayla oku, sonra `tutoriallar/01-temel-host-bot.md`'yi kendi token'ınla çalıştır. Buradan sonra rehber bölümlerini ihtiyaç duydukça aç.

Resmi HBInit API'sinden geliyorsan: [rehber/01-mimari.md](rehber/01-mimari.md) ve [rehber/02-room-create.md](rehber/02-room-create.md) farkları net gösterir. API isimleri çoğunlukla benzer ama callback sözleşmesi ve addon mantığı farklı.

Belirli bir fonksiyonun davranışını arıyorsan referansla başla; her referans sayfası mantıksal olarak gruplanmıştır.

## Versiyon

Doküman `node-haxball@2.3.0` üzerinden hazırlanmıştır. API uyumsuz değişiklikleri kütüphanenin [GitHub release sayfasında](https://github.com/wxyz-abcd/node-haxball/releases) takip edebilirsin.

## Katkı

Hata, eksik bilgi, daha iyi örnek önerisi için issue açabilirsin. Kod örneklerinde API isimleri kütüphanenin `src/index.d.ts` dosyasına karşı doğrulanmıştır; uyumsuzluk gördüğünde repo'nun güncel sürümüyle karşılaştır, hâlâ tutarsızsa bildir.

## Lisans

Doküman içeriği MIT lisansı altındadır. Kütüphanenin kendisi için [node-haxball deposunun LICENSE dosyasına](https://github.com/wxyz-abcd/node-haxball/blob/main/LICENSE) bakın.
