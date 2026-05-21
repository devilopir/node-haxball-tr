# EventFactory ve protokol katmanı

Haxball içindeki tüm aksiyon, oyuncular arasında WebRTC üzerinden serialize edilen event mesajlarıyla iletilir. Üst seviye `room.startGame()` veya `room.kickPlayer()` çağrılarının altında, bu çağrıların `HaxballEvent` objelerine dönüştürülüp ağa yollanması vardır.

`EventFactory`, bu düşük seviyeli event objelerini doğrudan üretmek için sunulan API'dir. Günlük bot kullanımında ihtiyaç duymazsın, `room.startGame()` direkt çağırmak yeterlidir. Bu sayfanın hedefi protokol seviyesinde özelleştirme veya analiz yapmak isteyen geliştiriciler.

## Ne işe yarar

İki ana kullanım senaryosu:

**Custom replay üretmek**: Bir replay'i editlerken event listesine eleman eklemek için. Örneğin var olan replay'in başına bir announcement event'i ekleyip "intro" görüntüsü olarak göstermek.

**Sahte event tetiklemek**: Test için ya da custom davranış için, kütüphanenin "bu event geldi" gibi davranmasını sağlamak. Sandbox modunda anlamlı.

## EventFactory.create, boş event üret

Bir `OperationType` enum değerine göre boş event yaratır:

```javascript
const { EventFactory, OperationType } = API;
const event = EventFactory.create(OperationType.StartGame);
// event'in alanları boş, manuel doldurulur
```

Çoğunlukla kullanmazsın, spesifik factory fonksiyonları daha pratiktir.

## Spesifik factory fonksiyonları

Her event tipi için bir factory metodu vardır:

```javascript
EventFactory.startGame();
EventFactory.stopGame();
EventFactory.pauseResumeGame(paused);
EventFactory.sendChat(msg);
EventFactory.sendAnnouncement(msg, color, style, sound);
EventFactory.sendInput(input);
EventFactory.sendChatIndicator(active);

EventFactory.joinRoom(id, name, flag, avatar, conn, auth);
EventFactory.kickBanPlayer(id, reason, ban);
EventFactory.setPlayerAdmin(playerId, value);
EventFactory.setPlayerTeam(playerId, teamId);
EventFactory.setPlayerSync(value);
EventFactory.setAvatar(value);
EventFactory.setHeadlessAvatar(id, avatar);
EventFactory.setKickRateLimit(min, rate, burst);
EventFactory.setTeamsLock(value);
EventFactory.setTeamColors(teamId, colors);
EventFactory.setScoreLimit(value);
EventFactory.setTimeLimit(value);
EventFactory.setStadium(stadium);
EventFactory.setDiscProperties(id, data);
EventFactory.setPlayerDiscProperties(id, data);

EventFactory.autoTeams();
EventFactory.reorderPlayers(playerIdList, moveToTop);

EventFactory.ping(pings);
EventFactory.checkConsistency(data);
EventFactory.customEvent(type, data);
EventFactory.binaryCustomEvent(type, data);
EventFactory.setPlayerIdentity(id, data);
```

Hepsi serialize edilmiş, ağa gönderilmeye hazır bir `HaxballEvent` objesi döner.

## Custom event, kendi protokol mesajını yarat

`customEvent` ve `binaryCustomEvent`, kullanıcı tarafında modified client (`node-haxball` kullanan başka bir bağlantı) varsa, ona özel mesaj göndermek için kullanılır:

```javascript
const myCustomType = 1000; // 32-bit int, kendi protokol versiyonun
const data = { hello: "world" };

room.sendCustomEvent(myCustomType, data, targetPlayerId);
```

Karşı taraf `onCustomEvent(type, data, byId)` callback'i ile alır. Resmi Haxball client'ı bunları görmezden gelir; sadece `node-haxball` kullanan istemciler işler.

Pratik kullanım: bot-to-bot iletişim, custom oyun modları (Haxball'un kendisinde olmayan kurallar), ek state senkronizasyonu.

## Stream üzerinden parse, `createFromStream`

```javascript
EventFactory.createFromStream(reader);
```

Bir `Impl.Stream.Reader` objesinden okuyup tek bir `HaxballEvent` üretir. Replay parsing veya networking düzeyinde debug için kullanılır. Normal bot mantığında ihtiyacın olmaz.

## Event'i ateşlemek

`EventFactory` yarattığın event'i otomatik göndermez. Göndermek için:

```javascript
const event = EventFactory.startGame();
room.executeEvent(event, byId);  // byId = eylemi yapan oyuncunun id'si (host için 0)
```

Bu, `room.startGame()` çağrısı ile **fonksiyonel olarak aynıdır**, sadece daha düşük seviye. Pratik fark yoktur, sadece daha fazla yazmak gerekir.

Gerçek değeri custom davranış senaryolarında ortaya çıkar, örneğin replay editörü yazıyorsan event listesine elle event eklemek için:

```javascript
const replay = Replay.readAll(bytes);
const introEvent = EventFactory.sendAnnouncement(
  "Maç Başlıyor",
  0xFFFFFF, 1, 2
);
introEvent.frameNo = 0;
replay.events.unshift(introEvent);
const out = Replay.writeAll(replay);
```

## OperationType enum

`OperationType` tüm event tiplerini sıralar. Sık olanlar:

- `SendAnnouncement = 0`
- `SendChatIndicator = 1`
- `CheckConsistency = 2`
- `SendInput = 3`
- `SendChat = 4`
- `JoinRoom = 5`
- `KickBanPlayer = 6`
- `StartGame = 7`
- `StopGame = 8`
- `PauseResumeGame = 9`
- `SetGamePlayLimit = 10`
- `SetStadium = 11`
- `SetPlayerTeam = 12`
- `SetTeamsLock = 13`
- `SetPlayerAdmin = 14`
- `AutoTeams = 15`
- `Ping = 17`
- ...

Tam liste `index.d.ts` içindeki `OperationType` enum definition'unda (yaklaşık satır 15-205). Custom event tipleri için 1000+ değer aralığını kendine ayırmak iyi bir disiplindir.

## onBeforeOperationReceived, düşük seviye filtre

Tüm gelen event'leri yorumlamak veya engellemek için:

```javascript
room.onBeforeOperationReceived = (type, msg, globalFrameNo, clientFrameNo) => {
  if (type === OperationType.SendChat) {
    if (msg.msg && msg.msg.includes("yasaklı")) {
      return false; // mesajı reddet
    }
  }
};
```

Bu, `onBeforePlayerChat`'in altındaki katmandır. Daha güçlü ama daha tehlikeli, sistem event'lerini yanlışlıkla reddetmek odanı bozar. Chat moderasyon için `onBeforePlayerChat` yeterli; bu API başka bir şey filtrelemen gerektiğinde devreye girer.

## Pratik tavsiye

Normal bot yazıyorsan `EventFactory`'yi unutabilirsin. `room.startGame()` çağrısı her zaman doğru seçim.

İhtiyaç doğacağı an: replay analizi/düzenleme, custom client protokolü, derinlemesine debug, ya da node-haxball'a ait olmayan bir Haxball-uyumlu yazılım üretmek.

Detaylı event tipleri ve alanları [referans/eventfactory.md](../referans/eventfactory.md) içinde.
