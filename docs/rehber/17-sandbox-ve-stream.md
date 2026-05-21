# Sandbox ve stream watcher

`Room.sandbox` ve `Room.streamWatcher`, network'e bağlanmadan oyun state'i ile çalışmak için kullanılan iki ek mod. Sandbox tamamen lokal simülasyon, stream watcher ise binary streaming sunucusundan gelen veriyi tüketmek için.

## Room.sandbox

```typescript
Room.sandbox(callbacks, options): SandboxRoom
```

Bir kullanıcı bağlantısı kurmaz, Haxball backend'ine konuşmaz. Tamamen offline bir simülasyon başlatır.

```javascript
const sandbox = Room.sandbox({
  onPlayerJoin: (player) => console.log("girdi", player.name),
  onTeamGoal: (teamId) => console.log("gol", teamId),
}, {
  controlledPlayerId: 1,
  delayedInit: false,
  requestAnimationFrame: null,
  cancelAnimationFrame: null,
});
```

`callbacks`: Plugin/Renderer'da kullanılan `CommonlyUsedCallbacks & CustomCallbacks & SandboxOnlyCallbacks`. `onGameTick`, `onTeamGoal`, `onPlayerBallKick` gibi event'leri normal Room'daki gibi alır.

`options`:

- `controlledPlayerId: int | null`, Hangi oyuncunun input'unu kontrol edeceğin. `setKeyState` çağrısı bu oyuncuya gider.
- `delayedInit: boolean`, `true` ise sandbox manuel `initialize()` çağrısı bekler. `false` ise constructor anında başlar.
- `requestAnimationFrame`, `cancelAnimationFrame`, Custom RAF override (Node'da default emülasyon kullanılır).

## SandboxRoom API

`SandboxRoom`, `Room`'un alt kümesini sunar (network'e ihtiyacı olmayan metodlar).

**Var olan metodlar**: `setCurrentStadium`, `setScoreLimit`, `setTimeLimit`, `startGame`, `stopGame`, `setGamePaused`, `setPlayerTeam`, `setPlayerAdmin`, `kickPlayer`, `autoTeams`, `setTeamsLock`, `setDiscProperties`, `setPlayerAvatar`, `setKickRateLimit`, `setTeamColors`, `setPlayerSync`, `reorderPlayers`, `setPlayerIdentity`, `sendAnnouncement`, `sendCustomEvent`, `sendBinaryCustomEvent`, `setKeyState`, `executeEvent`, `extrapolate`, `startRecording`, `stopRecording`, `isRecording`.

**Sandbox-özel metodlar**:

- `runSteps(n: int)`, Simülasyonu `n` tick ileri sarar. Sandbox'ta saat manuel; bu çağrılmazsa zaman ilerlemez.
- `setSimulationSpeed(coefficient)`, Otomatik tick speed (1 = normal, 2 = 2x hızlı, 0 = durdurulmuş).
- `playerJoin(id, name, flag, avatar, conn, auth)`, Sahte bir oyuncu odaya ekler.
- `playerLeave(id)`, Oyuncu çıkar.
- `playerChat(id, message)`, Sahte chat mesajı tetikler.
- `playerInput(id, value)`, Sahte input gönderir.
- `setPlayerChatIndicator(id, active)`, Typing göstergesi.
- `sendPingData(array)`, Ping verisi sahte.
- `takeSnapshot(): RoomState`, Anlık state'in derin kopyası. Replay-able test için.
- `useSnapshot(snapshot)`, Daha önce alınan snapshot'a geri döner.
- `applyEvent(event, byId)`, Bir `HaxballEvent`'i lokal olarak uygula.
- `destroy()`, Sandbox'ı tamamen kapatır.

**Olmayan metodlar** (network gerektirir): `sendChat`, `getPlayer`, `getBall`, `getDiscs`, `lockTeams`, `resetTeams`, `randTeams`, `clearBan`, `clearBans`, `addPlayerBan`, `addIpBan`, `addAuthBan`, `setProperties`, `setConfig`, `addPlugin`. State doğrudan `room.state` üzerinden okunur.

## Tipik sandbox kullanımı

```javascript
const { Room, Utils } = require("node-haxball")();

const big = Utils.getDefaultStadiums().find((s) => s.name === "Big");

const sandbox = Room.sandbox({
  onPlayerBallKick: (id) => console.log(`${id} topa vurdu`),
  onTeamGoal: (teamId) => console.log(`takım ${teamId} gol`),
}, {
  controlledPlayerId: 1,
  requestAnimationFrame: null,
  cancelAnimationFrame: null,
});

sandbox.setCurrentStadium(big);
sandbox.playerJoin(1, "test-oyuncu", "tr", null, "", "fake-auth-1");
sandbox.setPlayerTeam(1, 1);
sandbox.startGame();

// Manuel 60 tick (1 saniye) sar:
sandbox.runSteps(60);

// State kontrolü:
console.log("oyuncu pozisyonu:", sandbox.state.players.find(p => p.id === 1));
```

Sandbox runtime'da çok hızlı çalışır, gerçek saatten bağımsız ilerlediği için 60 saniyelik bir maç birkaç saniyede simüle edilebilir.

## Snapshot ile zaman yolculuğu

```javascript
const snap = sandbox.takeSnapshot();
sandbox.runSteps(300);
// Bir şey değişti, beğenmediysen geri al:
sandbox.useSnapshot(snap);
```

Test senaryolarında "şu state'i kur, davranışı dene, baştan başla" döngüsü için.

## Room.streamWatcher

```typescript
Room.streamWatcher(initialStreamData, callbacks, options): StreamWatcherRoom
```

Özel binary streaming server'dan gelen state akışını oynatır. Pratik kullanım az; kütüphanenin [streamingServer](https://github.com/wxyz-abcd/node-haxball/tree/main/streamingServer) projesi bunu kullanır.

`initialStreamData: Uint8Array`, Streaming server'ın ilk paketi.

`StreamWatcherRoom` özellikleri SandboxRoom'unkilerle örtüşür ama daha azdır, input/playerJoin gibi mutasyon metodları yoktur, sadece izleyici state'i tutar.

## Hangi mod ne zaman

| Senaryo | Mod |
|---|---|
| Network'lü canlı bot | `Room.create` veya `Room.join` |
| Stadyum tasarımını fizik açısından test | `Room.sandbox` |
| Bot AI'yı offline antrenman | `Room.sandbox` (`runSteps` ile hızlı) |
| Birim test (deterministik) | `Room.sandbox` (`takeSnapshot`/`useSnapshot`) |
| Replay analizi | `Replay.read`/`Replay.readAll`, mod değil |
| Custom streaming server izleyicisi | `Room.streamWatcher` |
