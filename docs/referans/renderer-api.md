# Renderer API

`Renderer`, oyun state'ini görsel bir çıktıya dönüştüren addon türüdür. Browser canvas en sık kullanım; terminal ASCII, image dosyası, even SVG çıktısı da mümkün.

Pratik uygulama [rehber/15-tarayicida-kullanim.md](../rehber/15-tarayicida-kullanim.md) içinde; addon yapısı [addon-api.md](addon-api.md) içinde.

## Constructor

```typescript
abstract class Renderer implements Addon, AllRendererCallbacks {
  static readonly addonType: AddonType;  // = AddonType.Renderer
  constructor(metadata?: any);
}
```

Function constructor pattern:

```javascript
function MyRenderer(API, config) {
  Object.setPrototypeOf(this, Renderer.prototype);
  Renderer.call(this, {
    version: "1.0",
    author: "ben",
    description: "Canvas renderer",
  });

  const that = this;

  this.canvas = config.canvas;
  this.ctx = config.canvas.getContext("2d");

  this.render = function (extrapolatedRoomState) {
    drawFrame(that.ctx, extrapolatedRoomState);
  };
}
```

`name` yok, `allowFlags` yok, Renderer her modda kullanılabilir, isimle aranmaz.

## Render lifecycle

Renderer üç sinyalden veri çeker:

1. **`render(state)`**, Her animation frame'de (browser `requestAnimationFrame` ile, Node'da kütüphanenin default RAF emülasyonu ile). Burada anlık görsel çizilir. State, host'tan gelen son snapshot'ın extrapolate edilmiş halidir.
2. **`AllRendererCallbacks`**, `onTeamGoal`, `onPlayerBallKick`, `onAnnouncement` gibi event'ler. Görselde animasyon (gol slow-motion, tekme partikülü, mesaj toast) tetiklemek için.
3. **`SandboxOnlyCallbacks`**, Sadece sandbox modunda gelen ek event'ler (custom frame counter, snapshot değişiklikleri).

`render`, kütüphane tarafından yüksek frekansta çağrılır. Burada heavy işlem yapma; pre-compute edilmiş değerleri sadece çizdir.

## Koordinat dönüşümü (map ↔ pixel)

Stadium koordinatları ile canvas pixel koordinatları arasında bir scale + offset uygulamak zorundasın. Standard yaklaşım:

```javascript
this.viewWidth = state.stadium.maxViewWidth;
this.viewHeight = this.viewWidth * (this.canvas.height / this.canvas.width);
this.scale = this.canvas.width / this.viewWidth;

function mapToPixel(mapX, mapY) {
  return {
    x: (mapX + that.viewWidth / 2) * that.scale,
    y: (mapY + that.viewHeight / 2) * that.scale,
  };
}
```

`Stadium.maxViewWidth` haritanın görünürlük genişliğidir (oyunun varsayılan kamera genişliği). Bu olmadan oyuncular ekranın dışında kalır.

## Üç hazır renderer

Repo'da kullanıma hazır renderer örnekleri:

- **`defaultRenderer`** (`examples/renderers/defaultRenderer.js`), Resmi Haxball client'ı yakın görsel; çim/beton zemin texture'ları, disc gradient'leri, mesaj balonları. Browser canvas için.
- **`sandboxRenderer`**, Sandbox modu için zenginleştirilmiş debug görseli (vertex/segment id'leri görünür, collision flag'leri renkli).
- **`imageRenderer`**, PNG/JPEG çıkartmak için. Replay analizinde "şu frame'in görüntüsü" almak için.
- **`pixiRenderer`**, PixiJS tabanlı, daha yüksek FPS isteyen senaryolar.
- **`flappyKirbyRenderer`**, Tematik örnek (custom asset'lerle).

İlk projeyi `defaultRenderer`'ı kopyalayarak başlat, kendi davranışını üstüne yaz.

## Render callbacks

Renderer'a özgü görsel event'ler:

```javascript
this.onGameStart = function (byId) { /* maç başlangıç intro animasyonu */ };
this.onTeamGoal = function (teamId, goalId, goal, ballDiscId, ballDisc) {
  this.goalAnimation = { teamId, startTime: Date.now(), duration: 3000 };
};
this.onAnnouncement = function (msg, color, style, sound) {
  this.toasts.push({ msg, color, style, expiresAt: Date.now() + 5000 });
};
this.onPlayerBallKick = function (playerId) {
  this.kickParticle = { playerId, startTime: Date.now() };
};
```

Bu callback'ler `render()` ile aynı obje üzerinde state biriktirir; `render` çağrıldığında bu state'i okuyup çizer. Yani render saf çizim, callback'ler state mutate eder.

## requestAnimationFrame override

Node'da canvas yoktur, kütüphane default bir RAF emülasyonu sağlar. Custom RAF için:

```javascript
Room.create({...}, {
  renderer: new MyRenderer(API, { canvas }),
  // Renderer'a özgü değil, sandbox/replay için commonParams seviyesinde:
  // requestAnimationFrame: customRAF,
});
```

Browser'da `window.requestAnimationFrame` doğal olarak kullanılır.

## Renderer'ı runtime'da değiştirmek

```javascript
room.setRenderer(newRenderer);
```

`setRenderer` mevcut renderer'ın `finalize()`'sını çağırır, yeni renderer'ın `initialize()`'ını çağırır. Plugin değişimi gibi ama renderer için sadece bir tane geçerlidir.

## Sınırlar

- **Tek renderer**: Bir room'da iki renderer paralel çalışmaz.
- **Render frekansı**: Browser ekran refresh rate'ine bağlı (60-144 Hz). Node'da kütüphane default 60 Hz emülasyon yapar.
- **State okuma**: `render(state)` içinde `state.players` artık modify etme, sadece oku ve çiz. Mutation tahmin edilmeyen davranışa neden olur.
