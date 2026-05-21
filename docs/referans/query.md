# Query namespace

`Query`, stadyum geometrisi üzerinde **map koordinatına göre** sorgular yapmak için kullanılan yardımcı fonksiyonlardır. Map editor, collision debug, interaktif stadyum aracı yazıyorsan bu API olmadan yapamazsın.

```javascript
const { Query } = require("node-haxball")();
```

Tüm fonksiyonlar bir `roomState` ve bir `Point` (`{ x, y }`) alır; bazıları ek `threshold` (mesafe toleransı) parametresi alır. İki varyant döner: `*AtMapCoord` parsed obje verir (yoksa `null`), `*IndexAtMapCoord` index verir (yoksa `-1`).

## Vertex sorguları

```
Query.getVertexAtMapCoord(roomState, point, threshold): Vertex | null
Query.getVertexIndexAtMapCoord(roomState, point, threshold): int
```

`point`'e en yakın vertex'i bulur. `threshold` mesafesinin içindeki vertex var ise onu döner; yoksa null/-1.

## Segment sorguları

```
Query.getSegmentAtMapCoord(roomState, point, threshold): Segment | null
Query.getSegmentIndexAtMapCoord(roomState, point, threshold): int
```

Duvar segment'lerine yakınlık sorgusu. Düz çizgi segment'leri için noktanın çizgiye olan dik mesafesi `threshold`'un içinde olmalıdır.

## Goal sorguları

```
Query.getGoalAtMapCoord(roomState, point, threshold): Goal | null
Query.getGoalIndexAtMapCoord(roomState, point, threshold): int
```

Gol çizgilerine yakınlık. Kale çizgisini editörde tıklamak için.

## Plane sorguları

```
Query.getPlaneAtMapCoord(roomState, point, threshold): Plane | null
Query.getPlaneIndexAtMapCoord(roomState, point, threshold): int
```

Sonsuz düzlemlere yakınlık. Saha kenarı (out-of-bounds) sınırları gibi plane'leri sorgulamak için.

## Joint sorguları

```
Query.getJointAtMapCoord(roomState, point, threshold): Joint | null
Query.getJointIndexAtMapCoord(roomState, point, threshold): int
```

Disc'leri birleştiren elastik bağları sorgular.

## Disc sorguları

```
Query.getDiscAtMapCoord(roomState, point): Disc | null
Query.getDiscIndexAtMapCoord(roomState, point): int
```

**Dikkat**: `threshold` yok. Disc'lerin kendi `radius`'u kullanılır, point disc'in radius'u içindeyse hit kabul edilir. Top index 0'dır; sandbox/editor için en sık `getDiscAtMapCoord` kullanılır.

## Spawn point sorguları

```
Query.getSpawnPointIndexAtMapCoord(roomState, point, threshold): int
```

Red/Blue spawn point listesinde noktaya en yakın olanın index'ini döner. Hangi takıma ait olduğu ayrıca `roomState.stadium.redSpawnPoints` / `blueSpawnPoints` listelerine bakarak çözülür.

## Tipik kullanım: editör tıklama

```javascript
const { Query, Room, Utils } = require("node-haxball")();

const sandbox = Room.sandbox({}, { controlledPlayerId: 0, requestAnimationFrame: null, cancelAnimationFrame: null });
sandbox.setCurrentStadium(Utils.getDefaultStadiums()[0]);

function pickObjectAtClick(x, y) {
  const point = { x, y };
  const disc = Query.getDiscAtMapCoord(sandbox.state, point);
  if (disc) return { type: "disc", obj: disc };

  const vertex = Query.getVertexAtMapCoord(sandbox.state, point, 8);
  if (vertex) return { type: "vertex", obj: vertex };

  const segment = Query.getSegmentAtMapCoord(sandbox.state, point, 8);
  if (segment) return { type: "segment", obj: segment };

  return null;
}
```

`threshold` değeri canvas pixel boyutuna göre seçilir; 8-10 piksel tipik bir "kullanıcı tıklamasının hassasiyeti" değeridir.

## Performans notu

Bu fonksiyonlar her çağrıda tüm geometriyi taradığı için 60 fps render loop'unda her tick çağırma, sadece kullanıcı etkileşimi anında çağır (`mousedown`, `click`). Geometri çok büyükse (binlerce vertex), uzaysal index (R-tree, grid) düşünülebilir ama default haritalar için bu önlem gereksizdir.
