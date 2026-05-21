# Oyun akışı event'leri

Bir maçın yaşam döngüsü birkaç event üzerinden takip edilir. Bu sayfa hangi event'in ne zaman tetiklendiğini, parametrelerinin ne anlama geldiğini ve hangi sıra ile geldiğini netleştiriyor.

## Oda yaşam döngüsü

```
[bağlantı kurulur]
  ↓
onBeforeRoomLink(link)         ← addon zinciri için
  ↓
onAfterRoomLink(link)          ← oda artık public listede aktif
  ↓
[oyuncular gelir-gider, maçlar başlar-biter]
  ↓
onClose(reason)                ← bağlantı koptu / kapatıldı
```

Pratikte event handler atamak için **`onAfterRoomLink`** veya doğrudan `onOpen` içinde yapılır. `onBeforeRoomLink` ağırlıklı olarak addon yazarken kullanılır.

## Maç yaşam döngüsü

```
room.startGame() veya admin "Start" butonu
  ↓
onBeforeGameStart(byId) → onGameStart(byId) → onAfterGameStart(byId)
  ↓
onGameTick()                   ← saniyede 60 kez, maç süresince
  ↓
onPlayerBallKick(playerId)     ← top vurulduğunda
  ↓
onTeamGoal(teamId, goalId, goal, ballDiscId, ballDisc)
  ↓
onPositionsReset()             ← gol sonrası mevkiler resetlenir, maç devam eder
  ↓
[skor veya süre limitine ulaşıldı]
  ↓
onGameEnd(winningTeamId)       ← kazanan takım id'si, beraberlik 0
  ↓
[oyun otomatik durur, maç state'i sıfırlanır]
  ↓
onGameStop(byId)               ← oyun fiilen durur
```

İki kritik nokta:

`onGameEnd` ile `onGameStop` aynı şey değildir. `onGameEnd`, skor/süre limitine ulaşılıp maç doğal bittiğinde gelir; `onGameStop` ise oyunun fiilen durduğu, oyuncuların eski state'lerine döndüğü andır. Skor limiti dolmadan admin `stopGame()` çağırırsa `onGameEnd` gelmez, sadece `onGameStop` gelir.

`onPositionsReset`, her gol sonrası ve maç başında tetiklenir. Maç başında onun çağırıldığını bilmek bazen sürpriz olur.

## Tüm "oyuncu" event'leri

| Event | Parametre listesi | Ne zaman |
|---|---|---|
| `onPlayerJoin(player)` | full Player obj | Oyuncu odaya girer |
| `onPlayerLeave(player, reason, isBanned, byId)` | full Player obj | Çıktı/atıldı. `reason: null` ise oyuncu kendi ayrıldı; `reason: string` ise kicklendi/banlandı ve string verilen sebeptir. `isBanned: true` ban, `false` kick'tir. `byId` eylemi tetikleyen oyuncunun id'si (host için 0) |
| `onPlayerChat(id, message)` | int id | Mesaj attı |
| `onPlayerAdminChange(id, isAdmin, byId)` | int id | Admin'lendi/alındı |
| `onPlayerTeamChange(id, teamId, byId)` | int id | Takım değiştirildi |
| `onPlayerAvatarChange(id, value)` | int id | Avatar değişti (kullanıcı tarafından) |
| `onPlayerInputChange(id, value)` | int id | Tuş state'i değişti. AFK detection için kullanılabilir ama frekansı yüksektir (saniyede çok kez tetiklenir) |
| `onPlayerBallKick(playerId)` | int id | Topa vurdu |
| `onPlayerSyncChange(playerId, value)` | int id | Sync durumu değişti |

**Önemli ayrım**: `onPlayerJoin` ve `onPlayerLeave` callback'leri full `Player` objesi alır; diğer "oyuncu olayları" `(id, ...)` formatında int id alır. İhtiyaç olursa `room.getPlayer(id)` ile çevrilir.

## "Player vs id" tasarımı

Bu çift tipli yaklaşım kasıtlıdır:

- `onPlayerLeave` çağırıldığı an oyuncu zaten `room.players` listesinden çıkmıştır, id verilseydi `getPlayer(id)` `null` dönerdi. Bu yüzden bu callback'lere full obje verilir, callback yaşadığı sürece Player verisi hâlâ erişilebilirdir.
- `onPlayerChat(id, message)` gibi sık çağırılan event'lerde her seferinde Player objesi geçirmek gereksiz allocate yapar. id geçilir, ihtiyaç olursa lookup yapılır.

Pratikte chat handler'da Player objesine ihtiyacın olduğunda `room.getPlayer(id)` yeterli, performansa etki etmez.

## Veto edilebilir event'ler

`onBeforeX` handler'larından `false` döndürerek bazı event'leri iptal edebilirsin. Onaylanmış veto destekleyen event'ler:

- `onBeforePlayerChat` → mesaj yayınlanmaz (chat moderasyon, küfür filtresi)
- `onBeforeOperationReceived` → herhangi bir client operasyonunu reddet (düşük seviyeli filtre)
- `onBeforePlayerJoin` → bazı sürümlerde join'i engelliyor, default backend'de tutarsız davranabilir; ban listesi için `kickPlayer(id, reason, true)` daha güvenilir

Veto edilemez "haber verme" event'leri (`onAfterPlayerJoin`, `onGameTick` vb.) `false` döndürse de bir şey değişmez.

## onGameTick uyarısı

`onGameTick`, saniyede 60 kez çağırılır. Bu callback'in içinde:

- Disk I/O yapma, her tick için stat dosyası yazmak bir saatte 216.000 yazım demek.
- `JSON.parse` veya regex gibi pahalı işlemler yapma.
- `room.players` üzerinde her tick `.filter` çağırma alışkanlığı tek başına sorun değil ama "her tick'te 5 farklı filter" bir noktada birikir.

Tick başına yapılacak hesap varsa state'i sayaçlarla yönet, her N tick'te bir gerçek iş yap:

```javascript
let tickCount = 0;
room.onGameTick = () => {
  tickCount++;
  if (tickCount % 60 === 0) {
    // saniyede bir kez yapılacak iş
  }
};
```

Pratikte botların çoğu `onGameTick`'i hiç kullanmaz; pozisyon takibi gerektiren özel mantıklar (top sahibi tespiti, AFK detection ile pozisyon) için devreye girer.

## Oyuncu aktivitesi ve AFK

`onPlayerInputChange(playerId, value)`, oyuncunun tuş state'i her değiştiğinde tetiklenir (hareket veya ateş). AFK detection için tipik kullanım: her oyuncu için son input zamanını sakla, belirli süre dolduysa kickle. Spec'te oturmak aktivite sayılmaz çünkü input gönderilmez. Detaylı uygulama [tutoriallar/03-afk-kick-botu.md](../tutoriallar/03-afk-kick-botu.md) içinde.

`onPlayerInputChange` frekansı yüksektir, oyuncu hareket halinde saniyede 10+ kez tetiklenebilir. Sayaç tutarken her tick'te disk yazımı yapma; bellekte tut, periyodik kontrol et. AFK kapsamını genişletmek istersen `onPlayerBallKick` ve `onPlayerChat`'i de aktivite kaynağı olarak ekleyebilirsin.

## Maç hiç başlamadan oyuncu giriyorsa

`room.startGame()` çağırılmadığı sürece oyun başlamaz, `onGameStart` tetiklenmez, `onGameTick` çalışmaz. Admin tarafından "Start" butonuna basılması beklenir veya bot otomatik başlatır.

Maç süresince oyuncu gelirse `onPlayerJoin` normal tetiklenir; yeni gelen spec'te başlar, takıma alınması için ya admin manuel atar ya da bot logic'i karar verir.
