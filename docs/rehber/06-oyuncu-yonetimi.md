# Oyuncu yönetimi

Oyuncuya yönelik moderasyon işlemleri host-only metodlardır. Client modunda çağırılırlarsa sessizce reddedilir (admin yetkisi olsan bile bunlar host'un kararıdır).

## Oyuncu objesi

`room.players` bir `Player[]` döndürür. Tek bir id için `room.getPlayer(id)`. Her `Player`:

- `id` (int), Benzersiz, bağlantı süresince sabit. Oyuncu çıkıp tekrar girerse yeni id alır.
- `name` (string), Oyuncu adı, kullanıcı tarafından değiştirilebilir.
- `team` (Team), `{ id, color }` objesi. `team.id` 0=spec, 1=red, 2=blue.
- `flag` (string), Ülke kodu.
- `avatar` (string | null), Kullanıcının kendi avatar emojisi.
- `headlessAvatar` (string | null), Host'un atadığı avatar (kullanıcı avatarını geçersiz kılar).
- `isAdmin` (boolean), Admin yetkisi var mı.
- `conn` (string), Bağlantı bilgisi (IP'nin numerik formatı). [11-utils-ve-auth.md](11-utils-ve-auth.md) bunu IP'ye çevirme fonksiyonunu işliyor.
- `auth` (string), Auth key hash'i. Aynı kullanıcının farklı session'larında sabit kalır, ban listeleri için anahtar bunudur.

## Takım yönetimi

```javascript
room.setPlayerTeam(playerId, teamId); // 0=spec, 1=red, 2=blue
```

Maç aktifken takım değiştirme `teamsLock` ayarına bağlıdır. Default açıkken oyuncular admin yardımı olmadan kendi takımını değiştirebilir; lock edilince sadece admin/host değiştirebilir.

```javascript
room.lockTeams();   // teamsLock = true
room.autoTeams();   // spec'tekileri sırayla red/blue'ya yerleştir
room.randTeams();   // tüm oyuncuları rastgele iki takıma böl
room.resetTeams();  // herkesi spec'e al
room.resetTeam(1);  // sadece kırmızıyı spec'e al
```

`autoTeams` ile `randTeams` arasındaki fark: autoTeams sadece spec'te olanları dağıtır, hâlihazırda takımda olanları rahatsız etmez. randTeams herkesi karıştırır.

## Admin yetkisi

```javascript
room.setPlayerAdmin(playerId, true);  // admin ver
room.setPlayerAdmin(playerId, false); // admin al
```

Admin'in yapabilecekleri client tarafında otomatik kontrol edilir (kick, takım değiştirme, start/stop, vb.). Bot bir komut sistemi üzerinden admin'siz oyunculara da yetki tanımak istiyorsa bunu kendi izin sisteminde tutmalıdır, Haxball protokolünde "yarı admin" yoktur.

Pratik bir kalıp: admin gidince yeni admin atamak. [tutoriallar/01-temel-host-bot.md](../tutoriallar/01-temel-host-bot.md) içindeki `ensureAdmin` bu işi yapar.

## Kick ve ban

```javascript
room.kickPlayer(playerId, reason, isBanning);
```

- `reason` (string | null): Oyuncuya gösterilen mesaj. `null` ise default mesaj.
- `isBanning` (boolean): `true` ise oyuncu banlanır (aynı auth ile geri giremez), `false` ise sadece kicklenir.

### Ban listesi

Banlanan oyuncular `room.banList` üzerinden takip edilir. Yapısı `BanEntry[]`, her entry'de:

- `id` (BanEntryId): Ban entry'sinin id'si (oyuncu id'si değil).
- `type` (BanEntryType): `Player` (auth ile), `IP`, vb.
- `value`: Tipe göre auth string, IP bilgisi vs.

Ban'ı kaldırmak için iki yol:

```javascript
room.clearBan(playerId);     // belirli oyuncunun ban'ını kaldır
room.clearBans();             // tüm banları kaldır
room.removeBan(banEntryId);   // belirli ban entry'sini sil
```

`clearBan(playerId)`, oyuncunun **şu anki session id'si** ile aranır. Banladığın oyuncu zaten çıktı, id'si artık geçerli değil, bu durumda `removeBan(banEntryId)` veya `clearBans()` kullanılır.

Default backend'in ban listesi bot süreci boyunca tutulur, restart'ta sıfırlanır. Kalıcı ban listesi istiyorsan auth key'leri kendin diske yaz ve `onPlayerJoin`'de kontrol et:

```javascript
const fs = require("fs");
const BAN_FILE = "./bans.json";
let bannedAuths = new Set(
  fs.existsSync(BAN_FILE) ? JSON.parse(fs.readFileSync(BAN_FILE, "utf8")) : []
);

room.onPlayerJoin = (player) => {
  if (bannedAuths.has(player.auth)) {
    room.kickPlayer(player.id, "Kalıcı ban", false);
    return;
  }
};

function permanentBan(player, reason) {
  bannedAuths.add(player.auth);
  fs.writeFileSync(BAN_FILE, JSON.stringify([...bannedAuths]));
  room.kickPlayer(player.id, reason, false);
}
```

Burada `false` veriyoruz çünkü Haxball'un kendi ban listesini de doldurmaya gerek yok, biz `onPlayerJoin`'de yakalıyoruz.

## Avatar yönetimi

```javascript
room.setPlayerAvatar(playerId, "🤖", true);  // headless avatar
room.setPlayerAvatar(playerId, "🤖", false); // client avatar
```

Üçüncü parametre kritik. `true` ise "headless avatar" set edilir, host'un atadığı, kullanıcının kendi avatarını override eden işaret. `false` ise normal kullanıcı avatarı set edilir, kullanıcı bunu kendi tarafından değiştirebilir.

Avatar string'i en fazla 2 karakter olabilir (emoji genelde tek karakter sayılır). Boş string `""` vermek avatarı kaldırır.

Tipik kullanım: takım kaptanına ⚡, en çok gol atana ⚽ gibi durumsal işaret koymak, bunlar için `true` (headless) verirsin, kullanıcının kendi seçtiği avatar override edilir.

Host'un kendi avatarı için `room.setAvatar(value)` ayrı bir metoddur, `noPlayer: true` ile zaten host görünmez.

## Sohbet ile etkileşim

```javascript
room.sendChat(msg, targetId);             // chat mesajı, host'un adıyla
room.sendAnnouncement(msg, targetId, color, style, sound);
```

`sendChat` ile `sendAnnouncement` farkı:

- `sendChat`: Normal chat olarak görünür, "host" adıyla geçer. 140 karakter limiti var.
- `sendAnnouncement`: Renkli, vurgulu sistem mesajı. 1000 karakter, hedef oyuncuya özel gönderilebilir, ses ayarı var.

Sistem mesajları için (komut cevabı, uyarı, hata) `sendAnnouncement` tercih edilir, kalabalık chat'te dikkat çeker, ayrıca bot mesajını "kim demiş bunu" karmaşasından ayırır.

## Kick rate limit

Kötü niyetli admin'ler veya bug'lı bot'ların kitle kick atmasını engellemek için:

```javascript
room.setKickRateLimit(min, rate, burst);
```

- `min` (int): Minimum aralık (tick cinsinden, 60 = 1 saniye)
- `rate` (int): Sürdürülebilir hız
- `burst` (int): Anlık burst limiti

Default değerler genelde yeterli; ayar ihtiyacı olağandışı bot trafiği gözlemlediğinde devreye girer.

## Hızlı referans

| İşlem | Metod |
|---|---|
| Takıma al | `setPlayerTeam(id, teamId)` |
| Admin ver/al | `setPlayerAdmin(id, isAdmin)` |
| Kickle | `kickPlayer(id, reason, false)` |
| Banla | `kickPlayer(id, reason, true)` |
| Tek oyuncunun ban'ını kaldır | `clearBan(id)` |
| Tüm banları temizle | `clearBans()` |
| Avatar ata | `setPlayerAvatar(id, "🤖", true)` |
| Avatar kaldır | `setPlayerAvatar(id, "", true)` |
| Spec'tekileri dağıt | `autoTeams()` |
| Tümünü karıştır | `randTeams()` |
| Takımları kilitle | `lockTeams()` |
