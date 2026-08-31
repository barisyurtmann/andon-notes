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
ver" kaydı. Yani cihaz yine DHCP ile soruyor, sadece cevap hep aynı geliyor.

⚠️ **Rezervasyon o VLAN'ın DHCP sunucusunda durur.** Pi'yi başka bir VLAN'daki porta
takarsan rezervasyon geçerli olmaz; o ağın havuzundan rastgele bir adres alır.

### IP adresi cihazdan değil, ağdan gelir

Bir cihaz "ben şu IP olayım" diye karar vermez. Bir ağa katılır, o ağın **DHCP sunucusu**
ona o ağa ait bir adres verir. Doğru soru "Pi ne alır" değil, **"Pi hangi ağa katıldı"**.

**Wi-Fi:** bir SSID aslında **bir VLAN'a açılan kablosuz kapıdır.** IT bir SSID'yi istediği
VLAN'a bağlayabilir. SSID ofis VLAN'ına bağlıysa cihaz ofis adresi alır; üretim VLAN'ına
bağlıysa üretim adresi. Aynı SSID adı, farklı eşleme, farklı sonuç.

**Ethernet:** belirleyici olan **switch portunun hangi VLAN'a atandığıdır.** Aynı kablo,
aynı cihaz, farklı port → farklı ağ, farklı IP.

**Bu projede:** üretimdeki Pi'ler **Ethernet** kullanır (brief §3.1) — fabrikada kablosuz
kararsızdır ve gecikme öngörülemez; ayrıca monitör için zaten aynı güzergâhtan kablo gidiyor.

### ⚠️ Bir Pi iki ağa birden bağlı olmamalı

Bir Pi hem ofis Wi-Fi'ına (internetli) hem üretim Ethernet'ine bağlıysa, **o Pi iki ağı
köprülemiş olur** ve "üretim ağında internet yok" taahhüdü (brief §3.6) fiilen çöker.

**Kural: üretime giden golden image'da Wi-Fi kapalı olacak.** Bench ünitesinin Wi-Fi'da
olması sorun değil — orası geliştirme makinesi.

### Pratikte hangi ortamda ne olacak

| Ortam | Bağlantı | IP nasıl gelir |
|---|---|---|
| Bench (Faz 0–1) | Ofis Wi-Fi | Ofis DHCP'sinden |
| Bench (Aşama 4 sonrası) | Aynı | Ofis DHCP'sinde MAC rezervasyonu → sabit |
| Üretim (16 Pi) | Üretim VLAN'ı, Ethernet, **Wi-Fi kapalı** | Üretim DHCP'sinde MAC rezervasyonu |

### Sorun giderme: IP yok veya `169.254.x.x`

`169.254.x.x` = "kimse bana adres vermedi, kendime uydurdum" (link-local). Anlamı **DHCP
cevap vermedi.** Sebepleri: port yanlış VLAN'da, kablo takılı değil, o VLAN'da DHCP sunucusu
yok, ya da havuz dolu.

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

### "İnternete erişimi yok" ≠ "ona erişilemez"

Bu ikisi **ayrı kurallardır ve karıştırılır:**

| Yön | Ne demek | Kararımız |
|---|---|---|
| Pi → internet (dışarı) | Pi'nin kendi başına dışarı çıkması | **Kapalı** |
| PC → Pi (içeri, port 22) | Senin ona SSH ile bağlanman | **Ayrıca açılmalı** |

Pi'nin internete çıkamaması, sana kapalı olduğu anlamına gelmez. Ama otomatik de açık
değildir: farklı VLAN'lar varsayılan olarak birbirini görmez.

### Yönetim erişimi — iki seçenek

**A. Doğrudan kural:** ofisteki belirli bir IP'den üretim VLAN'ına, sadece TCP 22.
Dar ve denetlenebilir.

**B. Atlama sunucusu (jump host) — tercih edilen:**

```
PC (ofis) ──SSH/22──▶ Sunucu (üretim VLAN) ──SSH/22──▶ 16 Pi
```

- IT'nin yazacağı kural tek hedef, tek port → kabul etmesi kolay
- Tüm yönetim erişimi tek noktadan geçer, log tek yerde
- **Ansible zaten sunucuda çalışacak** (Faz 2); o makine nasılsa tüm Pi'lere erişebilir
  olmalı. Yeni bir yol açmıyoruz, var olanı kullanıyoruz
- 1 makineden 16'ya çıkarken yeni kural gerekmez

Kullanımı `~/.ssh/config` ile şeffaflaşır — bkz. `02-ssh-ve-uzak-erisim.md`, ProxyJump.

### Firewall kural listesi (IT'ye götürülecek)

| Kural | Yön | Port | Not |
|---|---|---|---|
| Üretim VLAN → internet | — | — | **Kapalı** (brief §3.6) |
| Üretim VLAN → ofis ağı | — | — | **Kapalı** |
| Ofis (belirli IP) → sunucu | içeri | 22 | Yönetim erişimi |
| Ofis → sunucu | içeri | 3000, 80/443 | Grafana, web app |
| Pi'ler → sunucu | içeri | 1883, 123 | Aynı VLAN'daysa kural gerekmez |
| Pi ↔ Pi | — | — | **Kapalı.** Peer-to-peer yok |

### Sunucu hangi VLAN'da

- **Üretim VLAN'ında (önerilen):** Pi'lerle aynı ağda, MQTT için kural gerekmez, jump host olur
- Ayrı sunucu VLAN'ında: daha "temiz" ama Pi↔sunucu için ek kurallar gerekir

### Nasıl oluyor: yönlendirme (routing)

**Farklı bir IP alanına erişmek özel bir durum değil — internete erişmekle aynı şey.**

PC'n `192.168.0.50`, Google `142.250.x.x`. PC şöyle düşünür: *"Hedef benim ağımda mı?
Değilse gateway'e ver, o halleder."*

```
PC 192.168.0.50 ──▶ gateway 192.168.0.1 ──▶ (yönlendirir) ──▶ hedef
```

`192.168.5.11`'e erişmek de aynı mekanizma. Tek soru: **firewall izin veriyor mu.**
Yani IT'den istenen şey yeni bir ağ ya da yeni bir Wi-Fi değil, **var olan iki ağ arasında
dar bir geçiş izni** (ya da hiç geçiş gerektirmeyen Yol A, aşağıda).

### Yönetim erişimi için iki uygulama yolu

**Yol A — masaya üretim VLAN'ından kablo (önerilen ilk istek)**

Masadaki ethernet portu üretim VLAN'ına alınır. PC'de iki bağlantı olur:

| Bağlantı | Ağ | Ne için |
|---|---|---|
| Wi-Fi | Ofis, `192.168.0.x` | İnternet, GitHub |
| Ethernet | Üretim, `192.168.5.x` | Pi'ler, sunucu |

Windows ikisini aynı anda kullanır; hangi hedefin hangi arabirimden gideceğini yönlendirme
tablosu belirler. **IT hiçbir firewall kuralı yazmaz** — VLAN'lar arası geçiş yoktur, sadece
o porta VLAN atanır. İzolasyon ilkesi hiç bozulmaz.

⚠️ **Kritik detay:** o porttan **default gateway verilmemeli.** Verilirse Windows internet
trafiğini de oradan göndermeye çalışır ve internet kesilir. Üretim VLAN'ında zaten internet
yok; sadece IP ve subnet mask yeter.

**Yol B — yönlendirme + firewall kuralı**

Masa ofis VLAN'ında kalır, ofis→üretim yönünde dar bir kural yazılır. Bu durumda **mutlaka
jump host** iste: ofisten sadece sunucuya, sadece TCP 22.

| | Yol A (kablo) | Yol B (kural) |
|---|---|---|
| IT'nin yazacağı kural | **Yok** | 1 kural |
| VLAN'lar arası geçiş | Yok | Var, dar |
| Sende ne değişir | Kabloyu takarsın | `~/.ssh/config`'e ProxyJump |
| Ansible nereden | PC bağlıyken | Sunucudan |

### Windows tarafında kontrol

```powershell
ipconfig            # hangi bağlantı hangi IP almış
route print         # hangi hedef hangi arabirimden gidiyor
ping 192.168.5.11
```

Yol A'da `route print` çıktısında iki satır olmalı: `0.0.0.0` (internet) → Wi-Fi,
`192.168.5.0` → Ethernet.

### IT'ye gönderilecek metin (hazır)

> Üretim hattına 16 adet Raspberry Pi ve bir sunucu kuracağız. Bu cihazlar ayrı bir VLAN'da,
> **internet erişimi olmadan** çalışacak; sadece sunucunun MQTT portu (1883) ve NTP (123) ile
> konuşacaklar. Ofis ağına erişimleri olmayacak.
>
> Cihazları kurup bakabilmem için yönetim erişimine ihtiyacım var. Tercihim: **masamdaki
> ethernet portunun üretim VLAN'ına alınması.** Böylece VLAN'lar arasında geçiş kuralı
> yazmanıza gerek kalmaz; ben internete Wi-Fi üzerinden bağlı kalırım. Bu portta bana
> **default gateway verilmemesini** rica ediyorum.
>
> Mümkün değilse alternatif: ofis ağındaki tek bir IP'den üretim VLAN'ındaki **sadece
> sunucuya, sadece TCP 22** izni. Pi'lere sunucu üzerinden erişirim.
>
> Ayrıca Grafana ve web arayüzünü ofisten görebilmek için sunucuya TCP 3000 ve 80/443
> erişimi gerekecek. Cihaz envanterini (MAC, IP, seri no, konum) yazılı paylaşacağım; hepsi
> tek golden image'dan kurulacak ve Ansible ile yönetilecek.

### Erişim hiç açılmazsa

Panonun yanına gidip laptop'ı üretim switch'ine takarak çalışırsın. Çalışır ama her müdahale
için sahaya inmek demektir — 16 makinede ciddi yavaşlama. **Bu konuşma kablolamadan önce
yapılır.**

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
