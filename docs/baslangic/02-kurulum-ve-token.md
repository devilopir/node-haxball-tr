# Kurulum ve token alma

## Node.js sürümü

`node-haxball@2.3.0`, package.json'ında `node >= 16.9.0` ister. Pratikte 18 LTS veya 20 LTS önerilir, 16 LTS desteği biten sürümdür, 18+ önerilir. Sürümünü kontrol et:

```sh
node --version
```

Çıktı `v18.x.x` veya üzeri olmalı. Değilse [nodejs.org/download](https://nodejs.org/download) üzerinden veya `nvm` ile güncelle.

## Paket kurulumu

Boş bir klasör aç, içinde:

```sh
npm init -y
npm install node-haxball
```

Bu kadar. WebRTC implementasyonu olarak `node-datachannel` otomatik kurulur, derleme adımı vardır, Linux'ta `build-essential`, Windows'ta Visual Studio Build Tools, macOS'ta ise Xcode Command Line Tools gerekebilir. Kurulum sırasında C++ hatası alırsan ilk bakacağın yer budur.

## npm install nedir, kısa not

`npm`, Node.js ile birlikte gelen paket yöneticisidir. `npm install X`, X paketini `node_modules/` klasörüne indirir ve `package.json` içindeki `dependencies` listesine yazar. Klasörü başka bir makineye taşırsan `npm install` (X olmadan) bu listedeki tüm bağımlılıkları geri kurar. `node_modules/` her zaman `.gitignore` içinde olur, repo'ya commit edilmez.

## Headless token alma

Bir oda kurmak için Haxball backend'ine bir kez recaptcha çözümünden gelen token göndermek zorundasın. Bu token oda kayıt anında doğrulanır; bağlantı kurulduktan sonra unutulur.

Adımlar:

1. Tarayıcıyla `https://www.haxball.com/headlesstoken` adresine git.
2. "Verify you are human" recaptcha'sını çöz.
3. Sayfada görünen `thr1.XXXXX...` formatındaki stringi kopyala.
4. Bu string'i kodundaki `token: "..."` alanına yapıştır.

Token birkaç dakika sonra geçersiz olur. Oda kurulumu başarısız olursa ve hata "captcha required" ya da benzer mesaj döndürürse token süresi geçmiştir, yeni token al, koddaki değeri değiştir, tekrar dene.

Token'ı koda hardcode etmek geliştirme aşamasında pratik, üretime taşırken bir environment variable veya ayrı bir config dosyası kullan. Versiyon kontrolüne token sızdırma, public bir token saniyeler içinde başkası tarafından kullanılır ve sen kullanamaz hale gelirsin.

## İstemci olarak bağlanmak için token

Recaptcha korumalı bir odaya `Room.join` ile katılmak istiyorsan ayrı bir client token gerekir. Bunu doğrudan elde etmenin temiz bir yolu Haxball'un resmi backend'inde yoktur; deponun [tokenGenerator](https://github.com/wxyz-abcd/node-haxball/tree/main/tokenGenerator) klasöründeki NW.js tabanlı araç kullanılabilir. Çoğu kullanım için (oda kurmak, normal oyunculara hizmet vermek) bu token'a ihtiyacın olmaz; sadece kendi botunla recaptcha korumalı bir odaya girmek istediğinde devreye girer.

## İlk çalıştırma testi

Kütüphanenin yüklendiğini doğrulamak için:

```sh
node -e "const api = require('node-haxball')(); console.log('version:', api.version);"
```

Bir version stringi yazdırırsa kurulum tamam. Hata verirse genelde `node-datachannel`'ın derlenememesidir; npm install çıktısını yukarı kaydır, eksik build tool'larını gör.

Buradan sonra [03-ilk-oda.md](03-ilk-oda.md) ile ilk botu çalıştırabilirsin.
