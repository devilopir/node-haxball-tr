# Room API referansı

`Room` sınıfı, hem host hem client modlarında oda objesini temsil eder. `Room.create` veya `Room.join`'in `onOpen` callback'inde bu objeyi alırsın.

API'ler mantıksal olarak gruplandı; kullanım örnekleri için rehber bölümlerine link verildi.

## Static metodlar

### Room.create(createParams, commonParams)

Yeni bir oda kurar. [rehber/02-room-create.md](../rehber/02-room-create.md).

```
static create(createParams: CreateRoomParams, commonParams: CommonNetworkRoomParams): {
  cancel: () => void;
  useRecaptchaToken: (token: string) => void;
}
```

### Room.join(joinParams, commonParams)

Var olan bir odaya istemci olarak bağlanır. [rehber/03-room-join.md](../rehber/03-room-join.md).

```
static join(joinParams: JoinRoomParams, commonParams: CommonNetworkRoomParams): {
  cancel: () => void;
  useRecaptchaToken: (token: string) => void;
}
```

### Room.sandbox(callbacks, options)

Network'a bağlanmayan offline simülasyon. Stadium test, AI antrenmanı için.

```
static sandbox(callbacks: CommonlyUsedCallbacks & ..., options: SandboxOptions): SandboxRoom
```

### Room.streamWatcher(initialData, callbacks, options)

Streaming server'dan gelen binary state'i oynatır. Nadir kullanım.

## Readonly özellikler

Bağlantı ve durum bilgisi:

- `isHost: boolean`, host mu, client mı
- `currentPlayerId: uint16`, bu bağlantının player id'si (host için 0)
- `currentPlayer: Player`, bu bağlantının Player objesi
- `state: RoomState`, anlık tam state (oyuncular, disc'ler, skor)
- `stateExt: RoomState | null`, extrapolate edilmiş state (client'ta render için)
- `gameState: GameState | null`, maç aktif değilse `null`
- `gameStateExt: GameState | null`, extrapolate gameState
- `sdp: string`, WebRTC SDP (debug için)
- `currentFrameNo: int`, kaç tick geçti

Oda metadata:

- `name: string`, oda adı
- `link: string`, public link
- `password: string`, şu an ki şifre
- `geo: { lat, lon, flag }`
- `maxPlayerCount: int16`
- `fakePassword: boolean | null`
- `fixedPlayerCount: int16 | null`
- `showInRoomList: boolean`
- `unlimitedPlayerCount: boolean`
- `timeLimit: int`, dakika, 0 = limitsiz
- `scoreLimit: int`, 0 = limitsiz
- `stadium: Impl.Stadium.Stadium`, şu anki stadyum
- `players: Player[]`, oda içindeki oyuncular

Skor (maç aktifken):

- `redScore: int | null`
- `blueScore: int | null`
- `timeElapsed: int | null`, saniye

Ban listesi:

- `banList: BanEntry[]`

Addon'lar:

- `config: RoomConfig`
- `renderer: Renderer | null`
- `plugins: Plugin[]`
- `pluginsMap: object`, `{ [name]: Plugin }`
- `libraries: Library[]`
- `librariesMap: object`, `{ [name]: Library }`

## Yazılabilir özellikler

Bazı property'ler doğrudan setter ile değiştirilir, addon ve callback zinciri tetikler:

- `room.token = "thr1.NEW_TOKEN"`, token rotation
- `room.requireRecaptcha = true | false`, recaptcha mode
- `room.debugDesync = true | callback | null`, desync debug

Diğer property'ler için `setProperties()` veya ilgili setter metodu kullanılır.

## Oyuncu yönetimi

- `getPlayer(id: uint16): Player`, id ile lookup
- `setPlayerTeam(playerId, teamId)`, 0=spec, 1=red, 2=blue
- `setPlayerAdmin(playerId, isAdmin)`
- `setPlayerAvatar(id, value, headless)`. `headless: true` oyuncunun **headless avatarını** ayarlar (host tarafından sabitlenir, oyuncu kendisi değiştiremez); `false` normal kullanıcı avatarı ayarlanır. İki katman birbirinden bağımsızdır; ikisi de ayarlıysa headless öncelikli görünür
- `kickPlayer(playerId, reason, isBanning)`
- `reorderPlayers(idList, moveToTop)`, liste sırasını değiştirir
- `setPlayerIdentity(id, data, targetId?)`, custom backend identity

[rehber/06-oyuncu-yonetimi.md](../rehber/06-oyuncu-yonetimi.md) detaylı kullanım.

## Ban yönetimi

- `addPlayerBan(playerId): BanEntryId | null`, anlık oyuncuyu banla
- `addIpBan(...ips): (BanEntryId | null)[]`, IP/range ile banla
- `addAuthBan(...auths): (BanEntryId | null)[]`, auth string ile banla
- `removeBan(id: BanEntryId): boolean`, ban entry'sini sil
- `clearBans()`, tümünü temizle
- `clearBan(playerId)`, belirli oyuncunun ban'ını kaldır (anlık id ile)

## Oyun kontrolü

- `startGame()`, maç başlat
- `stopGame()`, maç durdur (doğal bitiş değil)
- `pauseGame()`, duraklat/devam (toggle)
- `isGamePaused(): boolean`
- `setScoreLimit(value)`, 0 = limitsiz
- `setTimeLimit(value)`, dakika, 0 = limitsiz
- `setCurrentStadium(stadium)`, parsed Stadium objesi
- `lockTeams()`
- `autoTeams()`, spec'tekileri dağıt
- `randTeams()`, herkesi karıştır
- `resetTeams()`, herkesi spec'e
- `resetTeam(teamId)`, sadece bir takımı spec'e
- `setTeamColors(teamId, angle, ...colors)`
- `changeTeam(teamId)`, kendi takımını değiştir (client modunda anlamlı)

[rehber/07-oyun-akisi.md](../rehber/07-oyun-akisi.md) detaylı.

## Mesajlaşma

- `sendChat(msg, targetId)`, chat olarak, host adıyla
- `sendAnnouncement(msg, targetId, color, style, sound)`, sistem mesajı
- `setChatIndicatorActive(active)`, typing göstergesi

`sendAnnouncement` parametre detayları:

- `msg`: ≤1000 karakter
- `targetId`: `null` = herkes, int = sadece o oyuncuya
- `color`: -1 = şeffaf, int hex (0xFFFFFF gibi)
- `style`: `"normal"`, `"bold"`, `"italic"`, `"small"`, `"small-bold"`, `"small-italic"`
- `sound`: 0=ses yok, 1=chat sesi, 2=highlight

## Disc fiziği

- `getBall(extrapolated?): Disc`, top (disc 0)
- `getDisc(discId, extrapolated?): Disc`
- `getDiscs(extrapolated?): Disc[]`
- `getPlayerDisc(playerId, extrapolated?): Disc`
- `setDiscProperties(discId, properties: SetDiscPropertiesParams)`. `properties` opsiyonel alanlardan oluşan bir objedir, sadece verilen alanlar değişir. Alanlar: `x, y, xspeed, yspeed, xgravity, ygravity, radius, bCoeff, invMass, damping, color, cMask, cGroup` (her biri `number | null`)
- `setPlayerDiscProperties(playerId, properties: SetDiscPropertiesParams)`, aynı obje yapısı oyuncu disc'i için

[rehber/08-stadyum-ve-disc.md](../rehber/08-stadyum-ve-disc.md).

## Oda ayarları

- `setProperties(params)`, name, password, geo, playerCount, maxPlayerCount, fakePassword, unlimitedPlayerCount, showInRoomList
- `setKickRateLimit(min, rate, burst)`
- `setHandicap(handicap)`, yapay lag (test için)
- `setUnlimitedPlayerCount(on)`
- `setFakePassword(value)`
- `setSync(value)`, sync mod (advanced)
- `setAvatar(avatar)`, host'un kendi avatarı

## Addon yönetimi

- `setConfig(roomConfig)`, RoomConfig'i runtime'da değiştir
- `mixConfig(roomConfig)`, başka bir config ile birleştir (handler'lar zincire eklenir)
- `addPlugin(plugin)`, `removePlugin(plugin)`, `updatePlugin(index, newPlugin)`, `movePlugin(index, newIndex)`
- `setPluginActive(name, active)`, runtime aktif/pasif
- `addLibrary(library)`, `moveLibrary`, `updateLibrary`
- `setRenderer(renderer)`

[rehber/09-addon-sistemi.md](../rehber/09-addon-sistemi.md) ve [rehber/10-plugin-yazma.md](../rehber/10-plugin-yazma.md).

## Replay

- `startRecording(): boolean`, kayıt başlat, başarılı/değil
- `stopRecording(): Uint8Array | null`, kaydı durdur, byte array al
- `isRecording(): boolean`

[rehber/13-replay.md](../rehber/13-replay.md).

## Streaming

- `startStreaming(params): StartStreamingReturnValue | null`
- `stopStreaming(): void`

Pratik kullanım az; özel streaming sunucu altyapısı için.

## Düşük seviye

- `executeEvent(event, byId)`, `EventFactory` ile üretilmiş event'i tetikle
- `executeEventWithTarget(event, targetId)`, sadece bir oyuncuya gönder
- `clearEvents()`, pending event kuyruğunu temizle
- `sendCustomEvent(type, data, targetId?)`, custom protocol event
- `sendBinaryCustomEvent(type, data, targetId?)`
- `getKeyState(): int`, şu anki input state
- `setKeyState(state, instant?)`, input gönder (sandbox/client'ta)
- `extrapolate(ms, ignoreMultipleCalls): RoomState`. Anlık state'i `ms` milisaniye ileriye doğru tahmin eder ve hesaplanmış yeni `RoomState`'i döner; ayrıca orijinal RoomState'in `ext` field'ını da bu yeni state'le doldurur. `ignoreMultipleCalls: true` ise aynı tick'te ikinci çağrılarda yeniden hesap yapmaz (önbellek). Esasen renderer'lar için, raw bot mantığında nadiren gerekir

[rehber/14-eventfactory-ve-protokol.md](../rehber/14-eventfactory-ve-protokol.md).

## Bağlantı yönetimi

- `leave()`, odadan ayrıl (client) veya odayı kapat (host)
