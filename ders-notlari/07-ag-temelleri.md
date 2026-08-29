# Ağ Temelleri

Bu konu her yerde karşımıza çıkıyor: Pi'ye SSH, MQTT bağlantısı, IT ile VLAN konuşması,
DHCP rezervasyonu. Temeli bir kez oturtalım.

## 1. IP adresi

Ağdaki her cihazın adresi. `192.168.0.166` gibi dört sayı (IPv4).

```
inet 192.168.0.166/24 brd 192.168.0.255
     └──────┬──────┘└┬┘     └────┬────┘
         adres      maske     broadcast
```

| Parça | Anlamı |
|---|---|
| `192.168.0.166` | Cihazın adresi |
| `/24` | **Subnet mask** = `255.255.255.0`. "İlk üç hane ağı, son hane cihazı belirtir" |
| `brd 192.168.0.255` | Broadcast: ağdaki herkese aynı anda gönderim adresi |

**Aynı ağdaki cihazlar doğrudan konuşur.** `192.168.0.x` içindeki iki cihaz birbirini
görür; başka bir ağa gitmek için **gateway** (router) gerekir.

**Özel (private) adres aralıkları** — internette kullanılmaz, sadece iç ağlarda:
- `10.0.0.0 – 10.255.255.255`
- `172.16.0.0 – 172.31.255.255`
- `192.168.0.0 – 192.168.255.255`

`127.0.0.1` = **loopback**, cihazın kendisi. "localhost" budur.

## 2. DHCP ve statik IP

| | Statik IP (cihazda) | DHCP rezervasyonu (router'da) |
|---|---|---|
| Ayar nerede | Her cihazın içinde ayrı ayrı | Tek yerde, router'daki tabloda |
| Ağ planı değişirse | 16 cihaza tek tek girersin | Tabloyu düzeltirsin |
| Çakışma riski | Var | Yok |
| Yedek Pi takarken | IP'yi elle ayarlaman gerekir | MAC'i tabloya yaz, biter |
| IT ile ilişki | Onların bilmediği elle atanmış adresler | Kayıt onların sisteminde |

**Bu projenin kararı: DHCP rezervasyonu** (brief §12.4). Ayarın 16 farklı kutunun içinde
saklı olması tek kişilik ekipte bakım kâbusudur.

**DHCP nasıl çalışır:** cihaz açılınca "bana bir adres verin" diye bağırır, DHCP sunucusu
(genelde router) havuzdan bir adres verir. **Rezervasyon** = "bu MAC adresine hep şu IP'yi
ver" kaydı.

## 3. MAC adresi

Ağ kartının donanımsal kimliği: `dc:a6:32:xx:xx:xx`. Fabrikada yazılır, değişmez.

```bash
ip link          # MAC adreslerini gösterir
```

DHCP rezervasyonu MAC'e yapılır. Ayrıca brief §15'teki envanter tablosunda her Pi'nin MAC'i
yazılı olacak.

## 4. Port

Bir IP adresinde çalışan farklı servisleri ayıran numara. "Adres bina, port daire".

| Port | Servis | Bu projede |
|---|---|---|
| 22 | SSH | Pi'ye bağlanma |
| 80 / 443 | HTTP / HTTPS | Web app, Grafana |
| 123 | NTP | Saat senkronizasyonu |
| 502 | Modbus TCP | AS218TX'ten okuma |
| 1883 | MQTT | **Pi → sunucu telemetri** |
| 8883 | MQTT over TLS | Şifreli MQTT (IT isterse) |
| 3000 | Grafana | Dashboard |
| 5432 | PostgreSQL / TimescaleDB | Veritabanı |

```bash
ss -tulpn        # bu makinede hangi port açık, hangi program tutuyor
```

**Brief §3.6 taahhüdü:** Pi'lar tam olarak iki şeyle konuşur — MQTT broker'ı (1883) ve NTP.
Başka port yok, internet yok, peer-to-peer yok.

## 5. DNS

İsimden adrese çeviri: `github.com` → `140.82.x.x`. İç ağda genelde router yapar.

```bash
ping github.com          # hem DNS'i hem erişimi test eder
```

`Temporary failure in name resolution` hatası = DNS çalışmıyor veya o ağdan dışarı çıkış yok.

## 6. Gateway ve yönlendirme

**Gateway** = kendi ağının dışına çıkarken kullandığın kapı (router). `ip route` ile görülür.

Bir paket hedefe giderken birden fazla router'dan geçer; her geçiş bir "hop"tur.
`traceroute` ile izlenebilir.

## 7. VLAN ve segmentasyon

**VLAN** = fiziksel olarak aynı switch üzerinde, mantıksal olarak ayrılmış ağ.

Bu projede (brief §3.6):
- **Üretim VLAN'ı** — 16 Pi + sunucu. İnternet yok
- **Ofis ağı** — normal PC'ler, internet var
- Aralarında **firewall** kuralı

**Neden:** üretim ağındaki bir cihaz ele geçirilirse ofis ağına atlayamasın; ofisten gelen
gereksiz trafik üretimi yormasın. IT'nin ilk soracağı şeylerden biridir.

**Not:** geliştirme PC'sinin GitHub'a erişimi ofis ağı üzerindendir ve bu taahhütle
çelişmez — üretim VLAN'ı yine kapalı kalır.

## 8. Faydalı komutlar

| Komut | Ne yapar |
|---|---|
| `ip a` | IP adresleri |
| `ip link` | Arabirimler ve MAC adresleri |
| `ip route` | Yönlendirme tablosu, gateway |
| `ping ADRES` | Karşı taraf yaşıyor mu |
| `ss -tulpn` | Açık portlar ve onları tutan programlar |
| `nmcli device status` | Bağlantı durumu (Bookworm'da ağ yöneticisi) |
| `curl -I https://site` | HTTP erişimi test et |
| `nc -zv ADRES PORT` | Belirli bir porta erişilebiliyor mu |

## 9. Bu projede ağın taşıdığı yük

```
Pi ──MQTT/1883──▶ Sunucu (Mosquitto → ingest → TimescaleDB)
Pi ──NTP/123───▶ Sunucu
Pi ◀─monitör─── aynı Ethernet kablosu üzerinden HDMI değil; monitör Pi'ye doğrudan bağlı
```

- **Topoloji:** yıldız, yönetilebilir switch (brief §3.1)
- **Ethernet, Wi-Fi değil** — fabrikada kablosuz kararsızdır
- **Ağ kopsa ne olur:** Pi telemetriyi yerel kuyruğa yazar, bağlantı gelince gönderir
  (store-and-forward). Kopya kayıt oluşmaması için idempotency anahtarı kullanılır
  (brief §12.2)
- **İnternet kopsa ne olur:** Andon çalışmaya devam eder. Sadece Telegram bildirimi gitmez —
  o bir kolaylık katmanıdır, çalışma şartı değil (brief §7.1)
