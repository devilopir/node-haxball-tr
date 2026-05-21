# Tarayıcıda kullanım

`node-haxball` Node.js'te tasarlanmıştır ama aynı kod tabanı tarayıcıda da çalışabilir. Bunun nedeni Haxball protokolünün doğal olarak tarayıcıdan tasarlanmış olması, kütüphane sadece protokolün katmanlarını implement eder.

Tarayıcıda neden çalıştırmak istersin: GUI host script'i, in-browser sandbox editor, replay görüntüleyici, görsel bot kontrol paneli.

> **⚠️ Üretim uyarısı**: Aşağıda anlatılan public proxy sunucusu (`node-haxball.onrender.com`) bakım ekibi tarafından her ay yenileniyor, **yenileme penceresinde birkaç gün ölü kalıyor**. Bir lig, bot, veya servis hizmeti veriyorsan bu proxy'ye güvenme. Kendi proxy'ni host et (repo'nun `streamingServer/` klasöründe örnek backend var) veya browser extension yolunu kullan.

## CORS engeli

Haxball'un backend sunucuları (oda kayıt, listeleme, token kontrolü) browser'dan gelen direct request'lere CORS başlığı koymaz. Tarayıcı bu yüzden bağlantıyı reddeder. İki çözüm var.

## 1. Proxy server üzerinden

Kütüphane bakım ekibi `node-haxball.onrender.com` adresinde bir public proxy sunuyor:

```html
<html>
  <head>
    <script src="https://cdn.jsdelivr.net/gh/wxyz-abcd/node-haxball@latest/examples_web/src/vendor/json5.min.js"></script>
    <script src="https://cdn.jsdelivr.net/gh/wxyz-abcd/node-haxball@latest/examples_web/src/vendor/pako-jszip.min.js"></script>
    <script src="https://cdn.jsdelivr.net/gh/wxyz-abcd/node-haxball@latest/src/api.js"></script>
  </head>
  <body>
    <script>
      const API = abcHaxballAPI(window, {
        proxy: {
          WebSocketUrl: "wss://node-haxball.onrender.com/",
          HttpUrl: "https://node-haxball.onrender.com/rs/",
        }
      });
      const { Room } = API;

      Room.create({...}, {...});
    </script>
  </body>
</html>
```

Kütüphane Node'da `require("node-haxball")()` ile aldığın aynı API'yi tarayıcıda `abcHaxballAPI(window, config)` ile alır. Geri kalan kullanım birebir aynıdır.

Proxy sunucusu **her ay yenileniyor**, bazen birkaç gün ölü kalıyor. Üretimde güvenmek riskli; kendi proxy'ni de host edebilirsin (kaynak kodu repo'da `streamingServer` ve örnek backend rehberlerinde).

## 2. Browser extension ile

Eğer botu sadece sen kullanacaksan, browser extension yöntemi daha temizdir. `haxballOriginModifier`, repo'nun kendi browser uzantısıdır: Chrome'a yüklersin, header'ları değiştirip CORS sorununu çözer. Proxy ihtiyacın kalmaz.

```html
<script>
  const API = abcHaxballAPI(window); // proxy config'i olmadan
  const { Room } = API;
  Room.create({...}, {...});
</script>
```

Kurulum talimatı [haxballOriginModifier README'sinde](https://github.com/wxyz-abcd/node-haxball/tree/main/haxballOriginModifier).

## Eksik özellikler

Tarayıcıda **çalışmayan** birkaç şey:

- **`proxyAgent`**: `https-proxy-agent` tarayıcıda mevcut değil. Custom proxy üzerinden bağlantı sadece Node tarafında.
- **`fs` tabanlı persistance**: `localStorage`, `IndexedDB` veya server-side storage kullanmalısın.
- **WebRTC implementasyonu**: Node'da `node-datachannel`, browser'da native. Kütüphane otomatik seçer ama performans karakteristikleri farklıdır.

## Renderer önemi

Browser'da çalışırken çoğunlukla bir Renderer da kullanmak isteyeceksin, oyunu canvas'a çizmek için. Node'da Renderer nadiren kullanılır, browser'da neredeyse her zaman.

```javascript
const canvas = document.getElementById("game");

Room.join({...}, {
  ...,
  renderer: new defaultRenderer(API, {
    canvas: canvas,
    images: {
      grass: grassImg,
      concrete: concreteImg,
      concrete2: concrete2Img,
      typing: typingImg,
    },
    paintGame: true,
  }),
});
```

`defaultRenderer` repo'nun `examples/renderers/` klasöründedir. Image dosyalarını ya kendin sağlarsın, ya da Haxball client'ın asset'lerini extract edersin.

## Pratik akış

Kendi tarayıcı tabanlı Haxball client'ını mı yazıyorsun? Şu sırayla ilerlersin:

1. Chrome extension'ı kur (`haxballOriginModifier`)
2. Boş HTML sayfasına `api.js` ve gerekli vendor script'lerini ekle
3. `Room.join({...})` ile herhangi bir public odaya bağlan, "selam" mesajı at, bağlantı çalışıyor mu doğrula
4. Canvas oluştur, `defaultRenderer` ekle, oyunu görsel olarak gör
5. Custom davranış (örneğin "topa otomatik yaklaşan" bot mantığı) ekle

Replay görüntüleyici yapacaksan `Replay.read` ile callback'leri renderer'a bağlarsın. Replay frame frame oynar.

## Mobil

Browser teknik olarak Android/iOS mobil tarayıcıda da çalışır ama performans sıkıntılı. WebRTC mobil tarayıcılarda Node'a göre daha az kararlı; uzun süreli host bot için mobil tercih edilmez. Casual kullanım (kısa süreli oda izleme) sorunsuz.

## Güvenlik notu

Token'ı tarayıcıda kullanırken HTML kaynağında **görünür** olur. Public bir web sayfasında token bırakırsan saniyeler içinde başkası kullanır. Token'ı her zaman runtime input olarak al (sayfada form veya prompt), kaynak kodda hardcode etme.

Aynısı `authKey` için geçerli. Bot'un kalıcı kimliği değerliyse, tarayıcıda saklamak yerine kullanıcının kendi cihazında tutsun (`localStorage` ile ama sayfayı public yayınlama).
