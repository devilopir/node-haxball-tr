# Stadyum dosyası ve disc fiziği

Haxball'un fiziksel dünyası vertex'ler, segment'ler, plane'ler, goal'lar, disc'ler ve bunlar arasındaki joint'lerden oluşur. Bir `.hbs` dosyası bu bütünün serialize edilmiş halidir.

## Stadium objesi nedir

`Impl.Stadium.Stadium` ana sınıftır. Önemli alanları:

- **`vertices`** (`Vertex[]`): Sabit noktalar. Genellikle duvar köşeleri, kale direkleri.
- **`segments`** (`Segment[]`): İki vertex arasındaki çizgi. Duvarlar bunlardır.
- **`planes`** (`Plane[]`): Sonsuz düzlemler. Saha dışı sınırlar genelde plane'dir.
- **`goals`** (`Goal[]`): Gol çizgileri. Top buradan geçince gol sayılır.
- **`discs`** (`Disc[]`): Hareketli daireler. Top, kale direkleri, dekorasyon.
- **`joints`** (`Joint[]`): İki disc'i birbirine bağlayan elastik bağlantılar.
- **`redSpawnPoints`, `blueSpawnPoints`** (`Point[]`): Maç başında oyuncuların spawn olduğu yerler.
- **`playerPhysics`** (`PlayerPhysics`): Oyuncu disc'lerinin fizik parametreleri (kütle, radius, kick gücü).
- **`name`** (string): Stadyum adı.
- **`width`, `height`** (number): Saha boyutları.
- **`fullKickOffReset`** (boolean): Gol sonrası top dışındaki disc'lerin de resetlenip resetlenmediği.

Çoğu durumda bu yapıya el atmazsın, stadium'u olduğu gibi kullanır, sadece set edersin.

## Stadium yükleme

İki kaynak vardır.

**Varsayılanlar**:

```javascript
const big = Utils.getDefaultStadiums().find((s) => s.name === "Big");
room.setCurrentStadium(big);
```

Default isimler: `"Classic"`, `"Easy"`, `"Small"`, `"Big"`, `"Rounded"`, `"Hockey"`, `"Big Hockey"`, `"Big Easy"`, `"Big Rounded"`, `"Huge"`.

**`.hbs` dosyasından**:

```javascript
const fs = require("fs");
const text = fs.readFileSync("./stadyumlar/futsal-x4.hbs", "utf8");
const stadium = Utils.parseStadium(text);

if (!stadium) {
  console.error("Stadium parse edilemedi: dosya bozuk veya format yanlış");
  return;
}

room.setCurrentStadium(stadium);
```

`parseStadium` başarısız olursa `undefined` döndürür, null kontrolü her zaman yap.

## Stadyum dışa aktarma

Mevcut stadium'u `.hbs` formatında string'e dönüştür:

```javascript
const text = Utils.exportStadium(room.stadium);
fs.writeFileSync("./current.hbs", text);
```

Tipik kullanım: sandbox modunda bir haritayı düzenledikten sonra diske yazma.

## .hbs dosya formatı

`.hbs`, JSON5 türevi bir text formattır. Detay [ortak/stadyum-formati.md](../ortak/stadyum-formati.md) içinde; orada üst seviye yapı, koordinat sistemi, çarpışma grupları ve bilinen alanlar listelenmiştir.

## Disc objesi

`room.state.discs` veya `room.getDiscs()` array döndürür. Index 0 her zaman top'tur. Diğer disc'ler stadium'a göre değişir, kale direkleri ekstra disc olarak modellenmiş olabilir.

Bir `Disc` objesinin yazılabilir alanları:

- **`pos.x`, `pos.y`** (number): Pozisyon
- **`speed.x`, `speed.y`** (number): Hız
- **`gravity.x`, `gravity.y`** (number): Yerçekimi vektörü (genelde 0,0)
- **`radius`** (number): Yarıçap
- **`invMass`** (number): 1/kütle. 0 = sonsuz kütle (sabit), büyük değer hafif disc
- **`bCoef`** (number): Sıçrama katsayısı, 0=sıçramaz, 1=elastik
- **`damping`** (number): Yavaşlama (1=hiç yavaşlamaz, daha düşük=daha çabuk durur)
- **`color`** (int): -1=şeffaf, 0-16777215=hex renk
- **`cMask`, `cGroup`** (int): Çarpışma flag'leri

## Disc'i değiştirme, host-only

```javascript
room.setDiscProperties(discId, {
  x: 0,
  y: 0,
  xspeed: 0,
  yspeed: 0,
});
```

Verilmeyen alanlar değişmez, `xspeed` vermezsen mevcut hızı korur.

Pratik örnekler:

**Topu merkeze sıfırla**:

```javascript
room.setDiscProperties(0, { x: 0, y: 0, xspeed: 0, yspeed: 0 });
```

**Topu büyüt**:

```javascript
room.setDiscProperties(0, { radius: 20 });
```

**Düşük yerçekimi**:

```javascript
room.setDiscProperties(0, { ygravity: -0.05 });
```

Top normalde gravitysiz hareket eder, bunu ayarlayınca sürekli yukarı doğru itilir.

## Oyuncu disc'i

Oyuncuların kendileri de birer disc'tir. `setPlayerDiscProperties` ile manipüle edilir:

```javascript
room.setPlayerDiscProperties(playerId, {
  invMass: 0.3,         // ağırlaştır (default ~0.5)
  radius: 20,           // büyült
  bCoef: 0.5,
});
```

Tipik kullanım: kalede kalecinin "ağır" olması, pas vurulduğunda yerinden oynamasın diye `invMass` küçültülür.

`onPlayerJoin` sırasında çağırılırsa oyuncu odaya girer girmez ayar uygulanır; ama oyuncu disc'i sadece maç aktifken vardır, maç başlamadan önce çağırırsan etkisi olmaz. Doğru zaman `onPlayerJoin` değil, `onPlayerDiscCreated` veya maç başında `onGameStart`'ın hemen sonrası.

## Top sahibi tespiti

`onPlayerBallKick(playerId)` callback'i sana son topa vuranı verir. Bu pratikte "top sahibi" demek değildir, top kimden son sektiyse o. Asist takibi için bu yeterlidir:

```javascript
let lastKickerByTeam = { 1: null, 2: null };
let beforeLastKickerByTeam = { 1: null, 2: null };

room.onPlayerBallKick = (playerId) => {
  const player = room.getPlayer(playerId);
  const teamId = player.team.id;
  if (teamId === 0) return;

  beforeLastKickerByTeam[teamId] = lastKickerByTeam[teamId];
  lastKickerByTeam[teamId] = playerId;
};

// Full imza: onTeamGoal(teamId, goalId, goal, ballDiscId, ballDisc, customData?)
// Burada sadece teamId yeterli; goalId/goal stadium analizi için, ballDisc top disc'i.
room.onTeamGoal = (teamId, goalId, goal, ballDiscId, ballDisc) => {
  const scorer = lastKickerByTeam[teamId];
  const assister = beforeLastKickerByTeam[teamId];
  if (scorer && assister && scorer !== assister) {
    console.log(`Gol: ${room.getPlayer(scorer).name}, asist: ${room.getPlayer(assister).name}`);
  }
};

room.onGameStart = () => {
  lastKickerByTeam = { 1: null, 2: null };
  beforeLastKickerByTeam = { 1: null, 2: null };
};
```

Detaylı istatistik tutorial'ı [tutoriallar/05-istatistik-takibi.md](../tutoriallar/05-istatistik-takibi.md) bu kalıbı genişletiyor.

## Collision flags

Disc'lerin hangi disc'lerle çarpışacağı `cMask` ve `cGroup` flag'leri ile belirlenir. Default flag'ler `CollisionFlags` enum'unda: `ball`, `red`, `blue`, `redKO`, `blueKO`, `wall`, vs.

İki disc çarpışır eğer: `A.cMask & B.cGroup !== 0` ve `B.cMask & A.cGroup !== 0`.

Pratik kullanım nadirdir; özel stadyum davranışları (örneğin sadece kırmızıya geçirgen duvar) için ayarlanır. Default Haxball stadyumları zaten doğru flag'lerle gelir.
