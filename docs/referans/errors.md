# Hata kodları

`Errors.HBError`, `node-haxball`'da hata sinyali için kullanılan ortak sınıftır. Her instance şu alanları taşır:

- `code: ErrorCodes`, int değer, hata türü (bu sayfadaki tabloya karşılık gelir)
- `params?: any[]`, bazı kodlar ek parametre içerir (örneğin "X karakteri çok uzun" durumunda asıl uzunluk değeri)
- `toString(): string`, current `Language` ayarına göre okunabilir mesaj

Standart Error sınıfından **türemez**; bu yüzden `instanceof Error` `false` döner. `instanceof HBError` kullan.

```javascript
const { Errors } = require("node-haxball")();
const { ErrorCodes, HBError } = Errors;

room = Room.create({...}, {
  onClose: (reason) => {
    if (reason instanceof HBError) {
      console.log("kod:", reason.code, "anlam:", reason.toString());
    }
  },
});
```

## Bağlantı hataları

| Kod | İsim | Anlam |
|---|---|---|
| 0 | `Empty` | Hata yok, normal state |
| 1 | `ConnectionClosed` | Bağlantı kapandı |
| 2 | `GameStateTimeout` | İlk state beklenirken timeout |
| 3 | `RoomClosed` | Oda kapatıldı (host tarafından) |
| 4 | `RoomFull` | Oda dolu, girilemez |
| 5 | `WrongPassword` | Şifre hatalı |
| 6 | `BannedBefore` | Bu auth banlı, giriş reddedildi |
| 7 | `IncompatibleVersion` | Protokol versiyonu uyuşmazlığı |
| 8 | `FailedHost` | Host bağlantısı kuruldu ama bir şey ters gitti |
| 9 | `Unknown` | Tanımsız hata |
| 10 | `Cancelled` | `cancel()` çağırıldı |
| 11 | `FailedPeer` | WebRTC peer hatası |
| 12 | `KickedNow` | Az önce kicklendi |
| 13 | `Failed` | Generic başarısız |
| 14 | `MasterConnectionError` | Backend'e bağlanılamıyor |

## Stadium ve parse

| Kod | İsim | Anlam |
|---|---|---|
| 15 | `StadiumParseError` | Genel stadium parse hatası |
| 16 | `StadiumParseSyntaxError` | JSON5 syntax hatası |
| 17 | `StadiumParseUnknownError` | Beklenmeyen yapı |
| 18 | `ObjectCastError` | Tip dönüşüm hatası |
| 19 | `TeamColorsReadError` | Stadium dosyasında takım renkleri satırında beklenenden fazla değer var (parse sırasında) |

## Encoding/byte işlemleri

20-27 arası UTF-8 ve buffer ile ilgili düşük seviyeli hatalar. Custom binary event üretmiyorsan görme ihtimalin düşük.

## Veri uzunluğu / format

| Kod | İsim | Anlam |
|---|---|---|
| 28 | `BadColorError` | Renk değeri aralık dışı |
| 29 | `BadTeamError` | Geçersiz team id |
| 30 | `StadiumLimitsExceededError` | Stadium sınırı (vertex/segment sayısı vb.) |
| 31 | `MissingActionConfigError` | Action config eksik |
| 32 | `UnregisteredActionError` | Bilinmeyen action |
| 33 | `MissingImplementationError` | Bir method implemente edilmemiş |
| 34 | `AnnouncementActionMessageTooLongError` | Announcement >1000 karakter |
| 35 | `ChatActionMessageTooLongError` | Chat >140 karakter |
| 36 | `KickBanReasonTooLongError` | Kick reason çok uzun |
| 37 | `ChangeTeamColorsInvalidTeamIdError` | setTeamColors'a geçersiz teamId |

## Token / auth

| Kod | İsim | Anlam |
|---|---|---|
| 38 | `MissingRecaptchaCallbackError` | Recaptcha token gerekli (oda kurarken/girerken) |
| 41 | `JoinRoomNullIdAuthError` | `Room.join` için id ya da auth eksik |
| 51 | `AuthFromKeyInvalidIdFormatError` | `authFromKey`'e geçersiz format |
| 56 | `AuthBannedError` | Auth blacklist'te |

## Replay

| Kod | İsim | Anlam |
|---|---|---|
| 39 | `ReplayFileVersionMismatchError` | v3 değil, parse edilemez |
| 40 | `ReplayFileReadError` | Dosya okunurken hata (bozuk byte vb.) |

## Oyuncu metadata sınırları

| Kod | İsim | Anlam |
|---|---|---|
| 42 | `PlayerNameTooLongError` | İsim çok uzun |
| 43 | `PlayerCountryTooLongError` | Country code >2 karakter |
| 44 | `PlayerAvatarTooLongError` | Avatar >2 karakter |
| 45 | `PlayerJoinBlockedByMPDError` | "modifyPlayerData" tarafından engellendi |
| 46 | `PlayerJoinBlockedByORError` | "onBeforeOperationReceived" tarafından engellendi |

## Addon sistemi

| Kod | İsim | Anlam |
|---|---|---|
| 47 | `PluginNotFoundError` | İsimle plugin lookup başarısız |
| 48 | `PluginNameChangeNotAllowedError` | Plugin'in name'i runtime'da değiştirilemez |
| 49 | `LibraryNotFoundError` | İsimle library lookup başarısız |
| 50 | `LibraryNameChangeNotAllowedError` | Library'nin name'i runtime'da değiştirilemez |
| 60 | `PluginAlreadyExistsError` | Aynı name ile ikinci kez ekleme |
| 61 | `LibraryAlreadyExistsError` | Aynı name ile ikinci kez ekleme |

## Proxy / backend

| Kod | İsim | Anlam |
|---|---|---|
| 55 | `BadActorError` | Şüpheli istemci davranışı |
| 57 | `NoProxyIdentityProblem` | Proxy identity sorunu (gözlem) |
| 58 | `NoProxyIdentitySolution` | Proxy identity sorunu (önerilen çözüm) |
| 59 | `FailedToCreateRoom` | Backend oda kurmayı reddetti |
| 62 | `RateLimitReached` | Backend rate limit yedi |
| 63 | `UnknownMessageType` | Bilinmeyen protokol mesajı |

## Pratik kullanım

```javascript
const { Errors } = API;
const { ErrorCodes } = Errors;

room = Room.create({...}, {
  onClose: (reason) => {
    switch (reason.code) {
      case ErrorCodes.MissingRecaptchaCallbackError:
        console.error("Token süresi geçti, yenile");
        break;
      case ErrorCodes.RateLimitReached:
        console.error("Çok hızlı reconnect, daha uzun bekle");
        backoff *= 4;
        break;
      case ErrorCodes.IncompatibleVersion:
        console.error("Protokol uyumsuz, kütüphaneyi güncelle");
        process.exit(1);
        break;
      default:
        console.error("Bağlantı koptu:", reason.toString());
    }
  },
});
```

`reason.toString()` current `Language` ayarına göre okunabilir mesaj döner. Default İngilizce; Türkçe çeviri için özel bir `Language` alt sınıfı yazıp `Language.current = new MyTurkce(API)` ile aktive edilir (repo'da `examples/languages/` örnekleri var).
