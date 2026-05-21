# .hbs stadyum dosya formatı

Haxball'un stadyum dosyaları `.hbs` uzantısı taşır. İçerik insanca okunur metindir, JSON5 alt kümesi. JSON5 yorum satırlarını, trailing comma'ları ve tek tırnaklı string'leri destekler.

Bot tarafında `Utils.parseStadium(text)` ile parse, `Utils.exportStadium(stadium)` ile dışa aktarma yapılır. Bu sayfa formatın yapısını gösterir; tipik olarak bir stadium editor (örneğin haxball-stadium-editor) ile düzenlersin, bot sadece yükler.

## Üst seviye yapı

```json5
{
  "name": "Futsal X4",
  "width": 580,
  "height": 270,

  "spawnDistance": 300,
  "bg": { "type": "grass", "width": 550, "height": 240, "kickOffRadius": 75, "cornerRadius": 0 },
  "playerPhysics": { ... },
  "ballPhysics": "disc0",

  "vertexes": [ ... ],
  "segments": [ ... ],
  "planes": [ ... ],
  "goals": [ ... ],
  "discs": [ ... ],
  "joints": [ ... ],
  "redSpawnPoints": [ ... ],
  "blueSpawnPoints": [ ... ],

  "traits": { ... }
}
```

## Koordinat sistemi

- Merkez `(0, 0)` sahanın ortası
- Pozitif X sağa, pozitif Y aşağı
- `width` ve `height` toplam saha kapsamı
- Birim "Haxball unit", yaklaşık 1 piksel olmasa da tarayıcıda render edilirken bu mantıkta ölçeklenir

## Vertices

Sabit noktalar:

```json5
[
  { "x": 0, "y": -240, "bCoef": 0.5, "cMask": ["ball"] },
  { "x": 0, "y": 240, "bCoef": 0.5, "cMask": ["ball"] }
]
```

- `bCoef`: Sıçrama katsayısı, 0=durdurur, 1=elastik
- `cMask`: Hangi gruplarla çarpışır (string array veya int)
- `cGroup`: Kendi grubu (vertex'ler için anlamsız, segment ve duvar için anlamlı)

## Segments

İki vertex arasındaki çizgi, duvarlar:

```json5
[
  { "v0": 0, "v1": 1, "curve": 0, "vis": true, "bCoef": 0.5, "cMask": ["ball"] }
]
```

- `v0`, `v1`: Vertex array index'i
- `curve`: 0=düz çizgi, >0=bombeli (derece cinsinden)
- `vis`: Görünür mü (false ise duvar var ama çizilmez, sahanın yanları)
- `bias`: İç/dış yön ayrımı

## Planes

Sonsuz düzlemler:

```json5
[
  { "normal": [0, 1], "dist": -240, "bCoef": 1, "cMask": ["red", "blue"] }
]
```

- `normal`: Düzlemin normal vektörü (birim)
- `dist`: Düzlemden orijine olan uzaklık

## Goals

Gol çizgileri:

```json5
[
  { "p0": [-580, -90], "p1": [-580, 90], "team": "red" }
]
```

- `p0`, `p1`: Çizginin iki ucu
- `team`: `"red"` veya `"blue"` (hangi takımın kalesi)

Top bu çizgiyi geçince `onTeamGoal` tetiklenir.

## Discs

Hareketli daireler. Index 0 her zaman **top**'tur:

```json5
[
  // Index 0 = top
  { "pos": [0, 0], "radius": 10, "invMass": 1, "bCoef": 0.5, "cGroup": ["ball"], "cMask": ["all"], "color": "FFFFFF" },
  // Diğerleri: kale direkleri, dekorasyon
  { "pos": [-580, -90], "radius": 8, "invMass": 0, "color": "FF0000" }
]
```

- `invMass: 0`: Sabit, hareket etmez
- `damping`: 1=hiç yavaşlamaz (top)
- `color`: Hex string ("FF0000") veya CSS

## Joints

Disc'leri elastik bağlar (örneğin file görseli için):

```json5
[
  { "d0": 1, "d1": 2, "length": null, "strength": null, "color": "FFFFFF" }
]
```

`length: null` = mevcut mesafeyi koruyarak rijit. Sayı verilirse o mesafeye doğru elastik kuvvet.

## Spawn points

Maç başında oyuncuların görüneceği yerler:

```json5
"redSpawnPoints": [ [-50, 0], [-50, -50], [-50, 50] ],
"blueSpawnPoints": [ [50, 0], [50, -50], [50, 50] ]
```

Spawn point sayısı takım kapasitesinden az olabilir, kütüphane fazla oyuncuları aynı noktaya spawn eder.

## Player physics

Sahanın oyuncu disc'lerine uyguladığı fizik:

```json5
"playerPhysics": {
  "radius": 15,
  "bCoef": 0.5,
  "invMass": 0.5,
  "damping": 0.96,
  "acceleration": 0.1,
  "kickingAcceleration": 0.07,
  "kickingDamping": 0.96,
  "kickStrength": 5,
  "kickback": 0
}
```

- `acceleration`: Normal koşma ivmesi
- `kickingAcceleration`: Tekme tuşu basılıyken ivme (genelde daha düşük)
- `kickStrength`: Tekme kuvveti
- `damping`: Hız kaybetme katsayısı (1=hiç, 0=anında durur)

## Background

```json5
"bg": {
  "type": "grass",        // "none" | "grass" | "hockey"
  "width": 550,
  "height": 240,
  "kickOffRadius": 75,
  "cornerRadius": 0
}
```

Görsel arka plan, fizik için kullanılmaz, sadece render.

## Collision flags

`cMask` ve `cGroup` int olabilir veya string array olabilir:

```json5
// İkisi de geçerli
"cMask": ["ball", "red"],
"cMask": 3  // bit flag: 1 (ball) | 2 (red)
```

String formatı: `["ball", "red", "blue", "redKO", "blueKO", "wall", "all", "kick", "score", "c0", "c1", "c2", "c3"]`.

## Traits

Stadium-spesifik kısayolları tanımlar:

```json5
"traits": {
  "ballArea": { "bCoef": 1, "cMask": ["ball"] },
  "goalNet": { "vis": true, "bCoef": 0.1, "cMask": ["ball"] }
}
```

Vertex/segment içinde `"trait": "ballArea"` yazıldığında trait alanları otomatik uygulanır. DRY için yardımcı.

## Default stadium isimleri

`Utils.getDefaultStadiums()` 10 varsayılan stadium döner. İsimleri:

`Classic`, `Easy`, `Small`, `Big`, `Rounded`, `Hockey`, `Big Hockey`, `Big Easy`, `Big Rounded`, `Huge`.

## Parse hatası

`Utils.parseStadium` `undefined` döndürürse, dosya bozuk veya format dışı demektir. Hata kodları (`ErrorCodes.StadiumParseError`, `StadiumParseSyntaxError`) tipik:

- JSON5 syntax hatası (kapanmamış parantez, virgül eksik)
- Beklenmedik alan isim (örneğin `vertexes` yerine `vertices` yazmak)
- Negatif `radius` veya geçersiz collision flag

Online editör veya örnek stadium'ları başlangıç olarak al, sıfırdan yazma, el kasması olur.

## Stadyum kaynakları

- [Haxmaps.com](https://haxmaps.com), topluluk stadyum arşivi (yüzlerce alternatif)
- [Stadium Editor (resmi)](https://www.haxball.com/stadium-editor), basit web editör
- Repo'nun `examples` klasöründe parse edilebilir örnekler
