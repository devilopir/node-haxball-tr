# Proxy, deploy ve IP limiti

Botu kendi makinende yazmak başka, bir sunucuda 7/24 ayakta tutmak başka. Bu sayfa üretim için bilinmesi gereken sınırları ve pratikleri toplar.

## IP başına 2 oda limiti

Haxball backend'i aynı public IP'den en fazla **iki public oda**nın açılmasına izin verir. Üçüncü oda kurma denemesi başarısız olur. `showInRoomList: false` private oda da sayılır mı sorusunun cevabı: hayır, sayılmaz, listeden gizlenmiş oda backend tarafında "yok" gibi davranır, limit dışında.

Mantıklı çözümler:

**Birden fazla VPS / sunucu**: Her sunucu kendi IP'sini taşır, ikişer oda.

**HTTPS proxy**: Botun trafiğini bir proxy üzerinden geçirirsen, Haxball backend'e proxy'nin IP'si görünür. Her proxy başına 2 oda kurma izni.

## Proxy kullanımı

```javascript
const { HttpsProxyAgent } = require("https-proxy-agent");

const proxyAgent = new HttpsProxyAgent("http://proxy-user:proxy-pass@proxy.example.com:8080");

Room.create({...}, {
  ...,
  proxyAgent: proxyAgent,
});
```

`https-proxy-agent` paketi `node-haxball`'ın devDependencies'inde, ana dependency değil. Manuel kurman gerekir:

```sh
npm install https-proxy-agent
```

Proxy'nin Haxball backend ile WebRTC trafiğini destekleyebilmesi gerekir. UDP forward etmeyen proxy'ler oda kursa bile WebRTC el sıkışmasında çuvallar. Datacenter proxy'leri genelde çalışır, residential proxy'ler daha pahalı ama bazı senaryolarda gerekir.

Bedava proxy listeleriyle uğraşma, bağlantı kararsızlığı bot'un zamanın çoğunu reconnect'e harcamasına neden olur.

## VPS seçimi

Haxball bot'u CPU yoğun değildir. 1 GB RAM, 1 vCPU küçük bir VPS rahatlıkla 16 kişilik bir odayı kaldırır. Asıl önemli şey **gecikme**.

Oyuncuların çoğu Türkiye'deyse İstanbul/Frankfurt'taki bir sunucu. Avrupa karması ise Frankfurt veya Amsterdam. Latency 50 ms'in üstüne çıkarsa oyun "lag"li hisseder; 100 ms üstü kabul edilemez.

Hetzner, Contabo, DigitalOcean, Vultr, bot için kullanılan tipik isimler. Bedava tier'lar (Oracle Free, AWS Free) genelde kotalar yüzünden uzun süreli host'a uygun değil.

## Process management

Bot'un crash etmeden ya da deploy sırasında kapansa otomatik geri başlamasını istersin. Üç yaygın çözüm:

**systemd** (Linux native):

```ini
# /etc/systemd/system/haxbot.service
[Unit]
Description=Haxball Bot
After=network.target

[Service]
Type=simple
User=botuser
WorkingDirectory=/home/botuser/bot
ExecStart=/usr/bin/node bot.js
Restart=always
RestartSec=5
StandardOutput=append:/var/log/haxbot.log
StandardError=append:/var/log/haxbot.log

[Install]
WantedBy=multi-user.target
```

```sh
sudo systemctl enable haxbot
sudo systemctl start haxbot
sudo journalctl -fu haxbot  # log'u izle
```

**pm2** (Node ecosystem):

```sh
npm install -g pm2
pm2 start bot.js --name haxbot
pm2 logs haxbot
pm2 startup  # boot'ta başlat
pm2 save
```

pm2 daha çok feature sunuyor (zero-downtime restart, log rotation, web UI). systemd daha düşük seviye ama Linux standart.

**Docker**: Bot'u container'a paketleyip docker-compose veya kubernetes ile yönet. Birden fazla oda işletiyorsan ölçekleme daha kolay. Tek oda için overkill.

## Token rotation

Token expire olursa oda düşer. Otomatize çözüm zor, recaptcha'yı sunucudan çözmek imkansıza yakındır. Pratik yaklaşımlar:

**Manuel rotation**: Yeni token alıp `room.token = newToken` set et. Discord komutu, Telegram bot, web arayüzü, token'ı sana ulaştıracak bir yol kur, bot bunu içeri alsın.

**Restart-based rotation**: Bot crash olduğunda restart loop yeni token bekler, env var değiştir, restart. systemd `Restart=always` + Discord bildirim yeterli.

**Token havuzu**: Birkaç token önden hazır tut, biri expire olunca diğerine geç. Birden fazla oda işletiyorsan tasarrufu mantıklı.

## Logging

Bot 7/24 çalışacaksa loglara hakim olmalısın:

```javascript
const fs = require("fs");
const path = require("path");

const LOG_FILE = "./logs/bot.log";
fs.mkdirSync(path.dirname(LOG_FILE), { recursive: true });

function log(level, message) {
  const ts = new Date().toISOString();
  const line = `[${ts}] [${level}] ${message}\n`;
  fs.appendFileSync(LOG_FILE, line);
  console.log(line.trim());
}

room.onPlayerJoin = (p) => log("info", `katıldı: ${p.name} (${p.auth})`);
room.onPlayerLeave = (p) => log("info", `ayrıldı: ${p.name}`);
room.onTeamGoal = (teamId) => log("info", `gol: takım ${teamId}`);
room.onClose = (reason) => log("error", `oda kapandı: ${reason}`);
```

`fs.appendFileSync` her satırda diske yazar, yoğun oda için yavaş olabilir. `winston` veya `pino` gibi async logger paketlerine geçmek daha iyi. Log rotation için `logrotate` (Linux) veya logger'ın kendi rotation feature'ları.

## Monitoring

Bot ayakta mı, kaç kişi var, son gol ne zaman atılmış, bunları dışarıdan görmek için:

**HTTP endpoint**: Bot içine basit bir HTTP server göm:

```javascript
const http = require("http");

http.createServer((req, res) => {
  if (req.url === "/health") {
    res.end(JSON.stringify({
      players: room?.players.length ?? 0,
      gameActive: !!room?.gameState,
      uptime: process.uptime(),
    }));
  }
}).listen(8080);
```

Bir uptime monitoring service (UptimeRobot, Pingdom) bu endpoint'i dakikada bir çağırırsa düştüğünde sana haber verir.

**Metrics**: Prometheus veya StatsD ile metric göndermek için `prom-client` paketi. Aşırı için aşırı; tek bot için HTTP endpoint yeterli.

## Güvenlik

**Token sızıntısı**: Token'ı asla logda, repo'da, ekran görüntüsünde yayma. Birinin eline geçerse senin odanı çalabilir.

**Auth key sızıntısı**: `auth.key` dosyasını gitignore'a koy. Bot'un kalıcı kimliği değerli, yıldız, lig kayıtları buna bağlı.

**Admin yetkisi**: Bot'un admin verme komutu varsa kötüye kullanılmasına engel. Tek bir auth (kendi auth'un) "süper admin", diğerleri ondan yetki alır.

**Brute force**: Açık komut sistemine spam atılırsa bot lag yapar. Rate limit ekle:

```javascript
const cmdCooldowns = new Map();

room.onPlayerChat = (id, message) => {
  if (!message.startsWith("!")) return;
  const last = cmdCooldowns.get(id) ?? 0;
  if (Date.now() - last < 1000) return false; // 1s cooldown
  cmdCooldowns.set(id, Date.now());
  // komut işle...
};
```

## Yedekleme

Veri kaybı senaryoları:

- Disk crash: VPS sağlayıcının snapshot'larına güven, haftalık manuel yedek al
- Process crash: SQLite/JSON tutarlılığını koru, transaction kullan
- Yanlış komut: "tüm istatistikleri sıfırla" gibi komutlara double-confirmation
- Bot kodunda bug: git ile geçmiş, bug'lı versiyona dönebilirsin

Hostlamanın disk yedekleme feature'larını kullan; ucuzdur, bir gün gerekir.

## Maliyet

Tek oda işleten Türk bot tipik maliyeti aylık ~5-10 EUR (orta segment VPS). İki oda işletiyorsan farklı IP gerekir (proxy ~3-5 EUR/ay veya ikinci VPS), toplam ~15 EUR.

Lig sistemine, replay arşivine, web paneline ihtiyaç oluştukça maliyet artar, ama bu sayfanın kapsamı dışındadır.
