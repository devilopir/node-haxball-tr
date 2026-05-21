# Yerelleştirme

`Language` namespace, kütüphanenin ürettiği hata mesajlarını (özellikle `Errors.HBError.toString()`) farklı dillere çevirmek için kullanılır.

Default `Language.current` `null`'dur, hata mesajları İngilizce olarak gelir. Türkçe çeviri için kendi `Language` alt sınıfını yazarsın.

## Language sınıfı

```typescript
class Language {
  static current: Language | null;
  static resolveText(s: TemplateString, arr: any[]): string | null;

  abbr: string;
  api: {
    errors: NumberToTemplateStringMap;
  };

  constructor(abbr: string, metadata?: any);
}
```

- `Language.current`, global aktif language; doğrudan atanır.
- `Language.resolveText(template, params)`, Template string'i parametrelerle doldurur. Placeholders: `$1, $2, ...` ve basit `?` operatörü.
- `abbr`, İki harfli dil kodu (`"tr"`, `"en"`).
- `api.errors`, Hata kodlarını template string'lere eşleyen map.

## Türkçe çeviri örneği

```javascript
const { Language, Errors } = require("node-haxball")();
const { ErrorCodes } = Errors;

function TurkceLanguage() {
  Object.setPrototypeOf(this, Language.prototype);
  Language.call(this, "tr", {
    name: "Türkçe",
    author: "ben",
  });

  this.api = {
    errors: {
      [ErrorCodes.Empty]: "",
      [ErrorCodes.ConnectionClosed]: "Bağlantı kapandı",
      [ErrorCodes.GameStateTimeout]: "Oyun state'i zaman aşımına uğradı",
      [ErrorCodes.RoomClosed]: "Oda kapatıldı",
      [ErrorCodes.RoomFull]: "Oda dolu",
      [ErrorCodes.WrongPassword]: "Yanlış şifre",
      [ErrorCodes.BannedBefore]: "Bu hesap odadan banlı",
      [ErrorCodes.IncompatibleVersion]: "Protokol versiyonu uyumsuz",
      [ErrorCodes.MasterConnectionError]: "Backend'e bağlanılamıyor",
      [ErrorCodes.MissingRecaptchaCallbackError]: "Recaptcha token gerekli",
      [ErrorCodes.PlayerNameTooLongError]: "Oyuncu adı çok uzun ($1 karakter)",
      [ErrorCodes.RateLimitReached]: "Çok sık deneme, rate limit doldu",
      // ... gerisi
    },
  };
}

Language.current = new TurkceLanguage();
```

Artık `error.toString()` Türkçe mesaj döner:

```javascript
room = Room.create({...}, {
  onClose: (reason) => {
    console.log(reason.toString());  // "Bağlantı kapandı" gibi
  },
});
```

## Template placeholders

`$1`, `$2`, ... pozisyonel parametre yer tutucuları. `?` operatörü basit koşul içindir:

```javascript
errors: {
  [ErrorCodes.PlayerNameTooLongError]: "İsim çok uzun, en fazla $1 karakter olabilir",
  [ErrorCodes.RoomFull]: "Oda dolu ($1 / $2)",
}
```

Hatanın `params` array'i `$1`, `$2`'ye sırasıyla geçer. `resolveText` çağrısında array geçilir; kütüphane bunu otomatik yapar `toString()` içinde.

## Birden fazla dil

```javascript
const tr = new TurkceLanguage();
const en = new EnglishLanguage(); // veya repo'nun examples/languages/englishLanguage.js'i

// Kullanıcı tercihine göre değiştir:
function setLang(lang) {
  Language.current = lang === "tr" ? tr : en;
}
```

`onLanguageChange(abbr)` callback'i değişimi gözlemler.

## Resmi örnek

Repo'nun [examples/languages/englishLanguage.js](https://github.com/wxyz-abcd/node-haxball/blob/main/examples/languages/englishLanguage.js) dosyası tüm hata kodları için resmi İngilizce metinleri içerir. Türkçe için bunu copy-paste edip metinleri çevirmek hızlı yoldur. Detaylı dosya yapısı için kütüphane reposunun `examples/languages/` klasörü.

## Sınırlar

- Sadece hata mesajları çevrilir, kütüphanenin kendisi (callback isimleri, type field adları) değişmez.
- `Language.current = null` set edilirse `toString()` raw template'i veya generic mesaj döner.
- Bir Addon olmadığı için addon listesine eklenmez; doğrudan global `Language.current` atanır.
