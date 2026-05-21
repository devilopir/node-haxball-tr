# Callback referansı

`node-haxball`'da tüm event'ler callback olarak tüketilir. Listeyi mantıksal gruplara böldüm; her grup hangi modlarda (host/client) tetiklenir notu ile.

Tüm callback'lerde `customData` parametresi opsiyoneldir (verilmeyebilir), Before/After chain'inde önceki callback'in dönüş değeri. Pratik handler'da çoğu zaman görmezden gelinir.

Üç türde callback var:

- **`onX(...)`**, basit callback, "şu oldu" haberi
- **`onBeforeX(...)`**, bazı event'lerde mevcut, `false` döndürerek bazı event'leri veto eder
- **`onAfterX(...)`**, `onBeforeX` ile çift, customData chain'ini tamamlar

[rehber/04-callback-sistemi.md](../rehber/04-callback-sistemi.md) Before/After mekanizmasını işliyor.

---

## Yaşam döngüsü (host-only)

- **`onRoomLink(link, customData?)`**, Oda backend tarafından alındı, public link aktif.
- **`onBeforeRoomLink(link)` / `onAfterRoomLink(link, customData?)`**, Before/After çifti.
- **`onBansClear()` / `onBanClear(id)`**, Tüm/tek banlar temizlendi.
- **`onRoomRecaptchaModeChange(on)`**, `room.requireRecaptcha = X` ile değişti.
- **`onRoomTokenChange(token)`**, `room.token = X` ile değişti.

## Oda metadata (host-only)

- **`onAnnouncement(msg, color, style, sound)`**, Bir announcement yapıldı (sen veya bir admin). **Dikkat**: `style` burada **int** olarak gelir (0=normal, 1=bold, 2=italic, 3=small, 4=small-bold, 5=small-italic), ama `room.sendAnnouncement` çağırılırken **string** verilir. Aynı kavram iki tarafta farklı tip kullanır. Bu fark kütüphane içinde serileştirme sırasında oluşur.
- **`onPlayerHeadlessAvatarChange(id, value)`**, Host bir oyuncunun headless avatarını değiştirdi.
- **`onPlayersOrderChange(idList, moveToTop)`**, `reorderPlayers` çağrıldı.
- **`onSetDiscProperties(id, type, data1, data2)`**, Disc özellikleri değişti. `id` etkilenen disc veya player id'si. `type`: `0` = disc, `1` = oyuncu disc'i. `data1: number[]` fizik dizisi sırasıyla `[x, y, xspeed, yspeed, xgravity, ygravity, radius, bCoeff, invMass, damping]`. `data2: int[]` ise `[color, cMask, cGroup]`. Array içinde değişmeyen alanlar `null` olabilir. Toplu hızlı işlem için array formatı, yazma tarafı (`setDiscProperties`) için object formatı kullanılır.
- **`onPingData(array)`**, Ping ölçümleri güncellendi (tüm oyuncular için, oyuncu sırasıyla).
- **`onRoomPropertiesChange(props)`**, `setProperties` çağrıldı. `props` sadece değişen alanları içerir.

## Maç event'leri (host + client)

- **`onGameStart(byId)`**, Maç başladı. `byId` başlatan oyuncu (host için 0).
- **`onGameStop(byId)`**, Maç durduruldu (manuel, doğal bitiş değil).
- **`onGamePauseChange(isPaused, byId)`**, Duraklatıldı / devam ettirildi.
- **`onKickOff()`**, Maç başında veya gol sonrası top ilk vurulduğunda.
- **`onTimeIsUp()`**, Süre limiti doldu, maç doğal sonlanmak üzere.
- **`onGameEnd(winningTeamId)`**, Maç doğal bittikten sonra kazanan ile çağırılır. Beraberlik durumunda 0.
- **`onGameTick()`**, Her tick, saniyede 60 kez. **Pahalı işlem yapma**.
- **`onPositionsReset()`**, Maç başı + her gol sonrası mevkiler resetlendiğinde.
- **`onTeamGoal(teamId, goalId, goal, ballDiscId, ballDisc)`**, Gol oldu. `teamId` gol atan takım, `goal` gol nesnesi (Impl.Stadium.Goal).
- **`onPlayerBallKick(playerId)`**, Bir oyuncu topa vurdu.
- **`onScoreLimitChange(value, byId)`**, `setScoreLimit` çağrıldı.
- **`onTimeLimitChange(value, byId)`**, `setTimeLimit` çağrıldı.
- **`onStadiumChange(stadium, byId)`**, `setCurrentStadium` çağrıldı.
- **`onTeamsLockChange(value, byId)`**, Takım kilidi değişti.
- **`onTeamColorsChange(teamId, value, byId)`**, Takım renkleri.
- **`onKickRateLimitChange(min, rate, burst, byId)`**, Kick rate limit ayarı.

## Çarpışma event'leri

- **`onCollisionDiscVsDisc(discId1, discPlayerId1, discId2, discPlayerId2)`**, İki disc çarpıştı. `discPlayerId` -1 ise top/sabit disc, değilse o oyuncunun disc'i.
- **`onCollisionDiscVsSegment(discId, discPlayerId, segmentId)`**, Disc bir duvar segment'ine çarptı.
- **`onCollisionDiscVsPlane(discId, discPlayerId, planeId)`**, Disc bir plane'e çarptı.

Çarpışma callback'leri saniyede çok kez tetiklenir, dikkatli kullan.

## Oyuncu event'leri (host + client)

- **`onPlayerJoin(player)`**, Oyuncu odaya girdi. Player full obje olarak verilir.
- **`onPlayerLeave(player, reason, isBanned, byId)`**, Oyuncu çıktı/atıldı. `reason: null` ise oyuncu kendi ayrıldı; `reason: string` ise kicklendi/banlandı ve string verilen sebeptir. `isBanned: true` ban, `false` kick'tir. `byId` eylemi tetikleyen oyuncunun id'si (host için 0).
- **`onPlayerChat(id, message)`**, Chat mesajı geldi. `id` int (Player değil). Return `false` mesajı bastırır.
- **`onPlayerInputChange(id, value)`**, Tuş state'i değişti (saniyede çok kez tetiklenebilir). AFK detection için kullanılır.
- **`onPlayerChatIndicatorChange(id, value)`**, Typing göstergesi.
- **`onPlayerSyncChange(playerId, value)`**, Oyuncunun sync durumu değişti.
- **`onPlayerTeamChange(id, teamId, byId)`**, Takım değiştirildi.
- **`onPlayerAdminChange(id, isAdmin, byId)`**, Admin atandı/alındı.
- **`onPlayerAvatarChange(id, value)`**, Kullanıcı kendi avatarını değiştirdi.
- **`onPlayerObjectCreated(player)`**, Player objesi yaratıldı (host'ta join sonrası).
- **`onPlayerDiscCreated(player)`**, Player disc'i (kapanmış maçta yoktur, yeni maçta yaratılır).
- **`onPlayerDiscDestroyed(player)`**, Player disc'i kaldırıldı (maç bitince).
- **`onAutoTeams(playerId1, teamId1, playerId2, teamId2, byId)`**, `autoTeams()` çağrıldı. **`playerId2` ve `teamId2` `null` olabilir** (spec'te tek oyuncu kalmışsa). Kullanıcı kodu bu durumu kontrol etmeli, aksi takdirde `null` üzerinden işlem yapılırsa hata atar.

## İstemci event'leri (client-only)

- **`onPingChange(instantPing, averagePing, maxPing)`**, Senin ping değerlerin güncellendi.

## Local event'leri (lokal API çağrıları)

- **`onHandicapChange(value)`**, `setHandicap` çağrıldı.
- **`onRoomRecordingChange(value)`**, `startRecording`/`stopRecording`.

## API event'leri (addon yönetimi)

- **`onPluginActiveChange(plugin)`**, `setPluginActive` çağrıldı.
- **`onConfigUpdate(oldConfig, newConfig)`**, `setConfig`.
- **`onRendererUpdate(old, new)`**, `setRenderer`.
- **`onPluginAdd/Move/Update/Remove`**, Plugin liste değişiklikleri.
- **`onLibraryAdd/Move/Update/Remove`**, Library liste değişiklikleri.
- **`onLanguageChange(abbr)`**, `Language.current = X` ile.
- **`onVariableValueChange(addon, varName, oldValue, newValue)`**, Bir addon variable'ı değişti.

## Bağlantı kapanışı

- **`onClose(reason)`**, `Room.create`/`Room.join`'in `commonParams`'ında verilir, oda kapandığında çağırılır.

## Özet, sık kullanılanlar

Çoğu bot şu callback'lerle iş görür:

```
onAfterRoomLink
onPlayerJoin
onPlayerLeave
onPlayerChat
onPlayerInputChange
onPlayerTeamChange
onPlayerAdminChange
onGameStart
onGameStop
onGameEnd
onTeamGoal
onPlayerBallKick
onPositionsReset
```

Diğerleri özel kullanım, Renderer yazıyorsan çarpışma callback'leri, plugin debugging yapıyorsan API event'leri, vb.
