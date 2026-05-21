# Oyun ve maç kontrolü

Maçın başlatılması, durdurulması, kuralları (skor/süre limiti), kullanılan stadyum ve takım renkleri, hepsi host'un belirlediği şeylerdir. Bu sayfa ilgili API'leri gruplar.

## Maç başlat / durdur / duraklat

```javascript
room.startGame();
room.stopGame();
room.pauseGame();           // toggle: çalıyorsa duraklatır, duraklamışsa devam ettirir
room.isGamePaused();        // boolean
```

`startGame`, oyun **aktif değilse** çalışır. Çalan bir maçı yeniden başlatmak için önce `stopGame()` çağrılır.

`stopGame`, oyunu sonlandırır. Doğal bitiş (skor/süre limiti) ile farkı: doğal bitişte `onGameEnd(winningTeamId)` tetiklenir, manuel stop'ta sadece `onGameStop(byId)` gelir.

`pauseGame`, ortada bir maç varken devreye girer. Duraklatılmış oyunda fizik tick atmaz ama oyuncular bağlı kalır.

## Skor ve süre limiti

```javascript
room.setScoreLimit(value);  // 0 = limitsiz
room.setTimeLimit(value);   // dakika cinsinden, 0 = limitsiz
```

Bu ayarlar maç aktifken değiştirilemez, `startGame`'den önce set edilir. Aktifken çağırılırsa sessizce yok sayılır.

Tipik kombinasyonlar:

- **Lobi/streetfight**: `setScoreLimit(3)`, `setTimeLimit(0)`, ilk 3 atan kazanır, süresiz
- **Resmi maç formatı**: `setScoreLimit(3)`, `setTimeLimit(6)`, 6 dakikada en çok 3 gol
- **Antrenman**: `setScoreLimit(0)`, `setTimeLimit(0)`, limitsiz, manuel durdurulur

## Stadyum

```javascript
room.setCurrentStadium(stadium);
```

Parametre bir `Impl.Stadium.Stadium` objesi, string isim değil. İki kaynak:

```javascript
// Varsayılan stadyumlardan
const big = Utils.getDefaultStadiums().find((s) => s.name === "Big");
room.setCurrentStadium(big);

// .hbs dosyasından
const fs = require("fs");
const stadiumData = fs.readFileSync("./stadyumlarim/futsal-x.hbs", "utf8");
const stadium = Utils.parseStadium(stadiumData);
if (stadium) room.setCurrentStadium(stadium);
```

`Utils.parseStadium`, parse başarısız olursa `undefined` döner, boş kontrolü her zaman yap. `.hbs` dosya formatı bir JSON5 türüdür, [ortak/stadyum-formati.md](../ortak/stadyum-formati.md) içinde detaylanmıştır.

Stadyum maç aktifken değiştirilemez. Aktifken çağırılırsa sessizce yok sayılır; önce `stopGame()` gerekir.

### Varsayılan stadyum listesi

`Utils.getDefaultStadiums()` `Stadium[]` döner. `.name` üzerinden erişilen tanıdık isimler: `"Classic"`, `"Easy"`, `"Small"`, `"Big"`, `"Rounded"`, `"Hockey"`, `"Big Hockey"`, `"Big Easy"`, `"Big Rounded"`, `"Huge"`. Resmi Haxball'da görünen 10 default haritanın hepsi bu listede.

## Takım kilidi

```javascript
room.lockTeams();
```

`teamsLock` flag'ini `true` yapar. Lock açıkken sadece admin oyuncu takım değiştirebilir; default kapalıdır ve oyuncular kendi takımlarını seçer.

`lockTeams` çağrısı kalıcıdır, kilidi kaldırmak için `room.setTeamsLock(false)`... aslında bu metod yok. Lock'u kaldırmak için aynı `lockTeams()` toggle olarak çalışmaz; takım kilidi `room.state` üzerinden değiştirilebilir bir bayrak değildir. Pratikte lock'u kaldırmak istiyorsan `EventFactory.setTeamsLock(false)` ile oluşturup `executeEvent` ile gönderebilirsin, bu düşük seviyeli bir yol; günlük bot kullanımında lockTeams sadece açılır, kapatma ihtiyacı nadiren çıkar.

## Takım renkleri

```javascript
room.setTeamColors(teamId, angle, color1, color2, color3);
```

- `teamId`: 1=red, 2=blue.
- `angle` (int): Forma desen açısı, derece.
- `color1, 2, 3` (int): Üç renk (hex int formatında).

```javascript
room.setTeamColors(1, 45, 0xff0000, 0xffffff, 0xff8888); // kırmızı takım için
```

Renk hex formatından geçer (`Utils.colorToNumber("#ff0000")` ile string → int dönüşümü yapılır). Bot'un bir takım kimliği vurgulaması için kullanışlı: turnuva sırasında ev sahibi takım rengini farklılaştırmak gibi.

## Oda properties

Listede görünen oda ayarlarını değiştirmek için `setProperties`:

```javascript
room.setProperties({
  name: "Yeni Oda Adı",
  password: "secret",        // null vermek şifreyi kaldırır
  maxPlayerCount: 16,
  geo: { code: "tr", lat: 41, lon: 29 },
  fakePassword: false,
  playerCount: null,         // sahte sayaç kapat
  unlimitedPlayerCount: false,
  showInRoomList: true,
});
```

Sadece verdiğin alanlar değişir, diğerleri olduğu gibi kalır. Backend'e propagate edilir, oda listesinde de güncellenir.

İki property özel olarak doğrudan setter ile değiştirilir:

```javascript
room.token = "thr1.YENI_TOKEN...";   // token rotation
room.requireRecaptcha = true;        // recaptcha gereksinimini aç
```

`token` değişikliği özel: token expired bildirimi geldiğinde botu restart etmeden token değiştirip recovery yapmak için. Pratik bir kullanım: Discord'dan komutla yeni token alıp `room.token = newToken` set etmek.

## Otomatik takım atama

`autoTeams()`, spec'te bekleyen oyuncuları sırayla red ve blue'ya yerleştirir. Hangi oyuncuların hangi takıma gideceği, oda içindeki sıraya göre belirlenir.

```javascript
room.onPlayerJoin = (player) => {
  if (countNonHostPlayers(room) >= 4 && !room.gameState) {
    room.autoTeams();
    room.startGame();
  }
};
```

`randTeams()`, spec'te olsun ya da olmasın **tüm** oyuncuları rastgele iki takıma dağıtır. autoTeams'ten farkı budur.

## Disc fiziği müdahalesi

Bot, oyunun ortasında topa veya oyunculara müdahale edebilir. Detay [08-stadyum-ve-disc.md](08-stadyum-ve-disc.md) içinde ama özet:

```javascript
room.setDiscProperties(0, { x: 0, y: 0, xspeed: 0, yspeed: 0 }); // top merkezde, durgun
room.setPlayerDiscProperties(playerId, { invMass: 0.5 });        // oyuncunun ağırlığı yarı
```

Disc id 0 standart stadyumlarda toptur; diğer disc'ler stadyuma göre değişir.

## Tek tablo özet

| İşlem | Metod |
|---|---|
| Maç başlat | `startGame()` |
| Maç bitir | `stopGame()` |
| Duraklat / devam | `pauseGame()` |
| Skor limiti | `setScoreLimit(n)` |
| Süre limiti (dk) | `setTimeLimit(n)` |
| Stadyum (parsed) | `setCurrentStadium(stadium)` |
| Spec'tekileri dağıt | `autoTeams()` |
| Tümünü karıştır | `randTeams()` |
| Takımları kilitle | `lockTeams()` |
| Takım rengi | `setTeamColors(team, angle, c1, c2, c3)` |
| Adı/şifre değiştir | `room.setProperties({ name, password })` |
| Token yenile | `room.token = "thr1..."` |
