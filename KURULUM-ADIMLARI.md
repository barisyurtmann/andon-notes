# Kurulum Adımları — sıfırdan çalışır sisteme

**Bu dosya nedir:** bir Raspberry Pi'yi kutudan çıkarıp üretime hazır hale getirmenin
**tekrar edilebilir prosedürü.** Tarih yok, tartışma yok, karar gerekçesi yok — sadece
"ne yapılır, nasıl doğrulanır".

**Neden ayrı bir dosya:**

| Dosya | Ne için | Nasıl okunur |
|---|---|---|
| `CALISMA-GUNLUGU.md` | Hangi gün ne oldu, ne ölçüldü, ne karar verildi | Tarih sırasıyla, anlatı |
| **`KURULUM-ADIMLARI.md`** | **Sıfırdan bir üniteyi kurmanın prosedürü** | Yukarıdan aşağı, sırayla |
| `andon-proje-brifi.md` | Neden böyle tasarlandı | Referans |
| `ders-notlari/` | Komutlar ve kavramlar ne demek | Sözlük |

**Bu dosya iki şeye dönüşecek:** golden image tarifi (brief §4.6) ve runbook'un çekirdeği
(brief §15). O yüzden her adımda **doğrulama kriteri** var — "yaptım" değil, "şunu gördüm".

**İşaretler:** ✅ tamamlandı · ⬜ bekliyor · ⏸ bilerek ertelendi

---

## A. Donanım hazırlığı ✅

1. **Active Cooler'ı karta güç vermeden önce tak.** Sonradan takmak termal pedi bozar.
   İki plastik pim deliklere hizalanır, eşit bastırılır; fan kablosu USB portlarına yakın
   4 pinli **FAN** konnektörüne.
2. SD kartları kalemle **A** ve **B** diye etiketle. A: Desktop/öğrenme. B: yedekte.
3. Bağlantı sırası: kart → micro-HDMI (**USB-C girişine en yakın** port) → monitör, klavye,
   fare → **en son güç.**

**Doğrulama:** kart açılıyor, yeşil LED yanıp sönüyor, ekranda görüntü var.

---

## B. Kart imajlama ✅

Raspberry Pi Imager (`raspberrypi.com/software`).

| Kutu | Seçim |
|---|---|
| Device | Raspberry Pi 5 |
| OS | Raspberry Pi OS (64-bit), **"with desktop"** |
| Storage | Kart A |

**"Edit Settings" ekranı — atlanamaz:**

- Hostname: `andon-bench`
- Username: `andon` + parola (not et)
- Wi-Fi: ağ + parola, ülke `TR`
- Locale: `Europe/Istanbul`, klavye `tr`
- Services: **Enable SSH** + password authentication

⬜ **Raspberry Pi Connect aktifleştirilmeyecek** (brief Karar #25). Teklif edilirse geç.

**Doğrulama:** kart yazıldı ve doğrulandı.

---

## C. İlk açılış ve temel ölçümler ✅

```bash
hostname                 # andon-bench
whoami                   # andon
ip a                     # IP adresi
free -m                  # RAM
vcgencmd measure_temp    # sıcaklık
swapon --show            # swap nerede
```

**Doğrulama ve kayıt (ölçümler çalışma günlüğüne yazılır):**

| Ölçüm | Bench sonucu |
|---|---|
| `free -m` used / available | 450 / 1555 MB |
| Sıcaklık (idle) | 43,9 °C |
| Swap | `/dev/zram0` → SD'ye yazmıyor |

---

## D. Laptop'tan SSH ✅

```
ssh andon@<IP>
```

İlk bağlantıda parmak izi sorulur → tam olarak **`yes`**.
Parola yazarken ekranda hiçbir şey görünmez, normaldir.

**Doğrulama:** laptop'tan bağlanıp `free -m` çalıştırabiliyorsun.
**Bu noktada klavye, fare ve monitör Pi'den çıkarılır.**

---

## E. Doğrudan ethernet bağlantısı (bench) ✅

Laptop ↔ Pi doğrudan kablo veya switch üzerinden, DHCP olmadan.

Pi tarafı:
```bash
sudo nmcli con add type ethernet ifname eth0 con-name lab-static ip4 192.168.5.11/24
sudo nmcli con up lab-static
ip a
```

Windows tarafı: Ayarlar → Ağ → Ethernet → IP ataması → El ile → IPv4:
IP `192.168.5.50`, maske `255.255.255.0`, **ağ geçidi ve DNS boş.**

**Doğrulama:** `ping 192.168.5.11` ve `ssh andon@192.168.5.11` çalışıyor.

⚠️ Gateway boş bırakılmazsa Windows internet trafiğini bu kablodan göndermeye çalışır.

---

## F. Python çalışma ortamı ✅

```bash
sudo usermod -aG dialout andon     # seri port izni
sudo apt install -y python3-venv
mkdir -p ~/andon-lab && cd ~/andon-lab
python3 -m venv .venv
source .venv/bin/activate
pip install "pymodbus[serial]==3.6.9"
```

⚠️ `usermod` sonrası **SSH oturumunu kapat ve tekrar bağlan** — grup değişikliği yeni
oturumda geçerli olur.

**Doğrulama:**
```bash
groups                              # listede 'dialout' var
python3 -c "import pymodbus; print(pymodbus.__version__)"
```

---

## G. `systemd` servis kalıbı ⏸ **ertelendi**

`heartbeat.py` yazıldı ve elle çalıştırıldı; döngü + `logging` + `try/except` + `signal`
iskeleti öğrenildi.

**Servise dönüştürme, collector'ın ilk hali hazır olduğunda yapılacak** — ayakta kalması
gereken gerçek bir iş olduğunda. İptal değil, erteleme: read-only root açıldığında elle
çalıştırma zaten imkânsız olacak.

---

## H. Modbus: DVP-SS2 ile ilk okuma ⏸ **devam ediyor**

⚠️ **Kullanılan PLC bench'te olmalı, hiçbir makineye bağlı olmamalı.**

**H.1 — Kablolama**

```
USB-RS485        DVP-SS2 COM2
      A ───────  D+  ("+")
      B ───────  D-  ("−")
    GND ───────  SG  (varsa)
```
⚠️ A/B etiketleri üreticiye göre ters olabilir. Cevap alamazsan **ilk deneme: A ve B'yi
yer değiştir.**

**H.2 — Adaptör görünüyor mu**
```bash
ls -l /dev/ttyUSB*
dmesg | tail -20
```

**H.3 — Tarama (hiçbir şey yazmaz)** ✅
```bash
cd ~/andon-lab && source .venv/bin/activate
python3 scan_dvp.py
```
Script kaynağı: `andon-collector/tools/scan_dvp.py`

**Doğrulama:** en az bir satırda `CEVAP VAR`. Ayarları kaydet.

**[ÖLÇÜLDÜ] 2026-09-02 — sonuç:**

| Parametre | Değer |
|---|---|
| Framer | **ASCII** |
| Baud | **9600** |
| Veri / eslik / stop | **7-E-1** |
| Slave (istasyon) | **1** |

Bunlar DVP-SS2 COM2 **fabrika varsayılanı**. PLC'ye hiçbir yazma yapılmadan
okuma mümkün — yani Bölüm I (RTU'ya çevirme) zorunlu değil. Bkz. aşağıdaki not.

**H.4 — Adres tabanı ve word sırası doğrulaması**

WPLSoft device monitor'den elle gir: `D300 = 2`, `D301 = 1`, `D302 = 5`.
Sonra `0x112C` adresinden **tek istekte 2 register** oku.

Script: `andon-collector/tools/read_test.py` (sadece okur, hiçbir yazma içermez)

```bash
cd ~/andon-lab && source .venv/bin/activate
python3 read_test.py
```

Adres hesabı (vendor tablosundan, [DOĞRULANDI]):
`D000 = 44097` → `D300 = 44097 + 300 = 44397` → protokol 0-tabanlı `44397 - 40001 = 4396 = 0x112C`

⚠️ İki register **ayrı ayrı değil, tek istekte** okunur. Ayrı okunursa aradaki
milisaniyede sayac artabilir → yarısı eski yarısı yeni bozuk bir değer çıkar.

| Okunan | Anlamı |
|---|---|
| `[2, 1]` | Adres tabanı 0x1000 **ve** word sırası lo_hi doğrulandı |
| `[1, 2]` | Word sırası hi_lo |
| Başka bir şey | Adres tabanı yanlış — manuelden teyit et |

**Kapanacak [DOĞRULA] maddeleri:** brief §5.1 (word sırası), §5.2 (adres tabanı).

---

## I. COM2'yi Modbus RTU'ya standardize et ⬜ **muhtemelen gereksiz — karar bekliyor**

⚠️ **2026-09-02 sonrası durum:** H.3 fabrika ayarıyla (ASCII 9600 7-E-1) cevap verdi.
1 Hz poll'de ASCII 9600'ün maliyeti ~30–50 ms — fazlasıyla yeterli. 13 üretim PLC'sine
yazma yapmak, kazandırdığından fazla risk taşıyor. Brief §5.3 / Karar #5 gözden geçirilecek.

Sadece H.3'te cevap alınamazsa veya ayarları değiştirmek gerekirse.

Hedef: **RTU, 19200, 8-N-1** (brief §5.3).

⚠️ İlgili register'lar `D1120` (protokol/baud/format), `D1121` (istasyon adresi),
`M1120` (ayarı koru), `M1143` (RTU/ASCII) olarak **hafızadan** yazıldı —
**DVP-SS2 manuelinden teyit edilecek.**

⚠️ **Üretim PLC'sinde bu adım öncesinde mutlaka H.4 (okuma doğrulaması) yapılır.**
Doğrulanmamış adrese yazmak bu projedeki en pahalı hatadır.

---

## J. udev kuralı: sabit port adı ⬜

`/dev/ttyUSB0` reboot'lar arasında sabit değildir. Adaptörün seri numarasına sabit isim
bağlanır: `/dev/andon-rs485`.

```bash
udevadm info -a -n /dev/ttyUSB0 | head -40
```

**Doğrulama:** reboot sonrası `ls -l /dev/andon-rs485` aynı adaptörü gösteriyor.

---

## K. SSH key, parola girişinin kapatılması, DHCP rezervasyonu ⬜

Sıra önemli — ters yapılırsa Pi'ye erişim kaybedilir:

1. Laptop'ta key üret (`ssh-keygen -t ed25519`)
2. Public key'i Pi'ye kopyala (`ssh-copy-id`)
3. **Key ile girişi test et**
4. *Ancak sonra* `/etc/ssh/sshd_config` içinde `PasswordAuthentication no`, `PermitRootLogin no`
5. `sudo systemctl restart ssh`

Aynı oturumda: router'da MAC'e **DHCP rezervasyonu**, ve Pi için GitHub **read-only deploy key**.

**Doğrulama:** parolasız giriyor, parola ile giriş reddediliyor, reboot sonrası IP değişmiyor,
Pi `git pull` yapabiliyor ama `git push` yapamıyor.

**Aynı reboot'ta yapılacak ölçüm:** açılıştan chrony senkronuna geçen süre (RTC pili kararı).
```bash
uptime -s
timedatectl
journalctl -b -u chrony | grep -i "selected\|synchron"
```

---

## L. GPIO: Andon butonu ⬜

Buton bir bacağı **GPIO17**, diğeri **GND**. Direnç yok (dahili pull-up).

```python
from gpiozero import Button
button = Button(17, pull_up=True, bounce_time=0.05)
button.wait_for_press()
```

⚠️ **Sahada 24 V asla doğrudan GPIO'ya bağlanmaz** — optokuplör veya röle kartı zorunlu.

**Doğrulama:** butona basınca ekranda yazı çıkıyor.

---

## M. Collector servisi ⬜

`machines.yaml` okuyan, Modbus'tan veri çeken, MQTT'ye gönderen servis.
**`systemd` servisine dönüşme anı burası** (G adımı burada tamamlanır).

**Doğrulama:** reboot sonrası kendiliğinden kalkıyor; çökünce `Restart=always` geri getiriyor;
`journalctl -u andon-collector -f` düzgün log gösteriyor.

---

## N. Sunucu tarafı ⬜

Docker: Mosquitto + TimescaleDB (`docker-compose.yml`).

**Doğrulama:** Pi'den gönderilen mesaj veritabanında görünüyor; aynı mesaj iki kez
gönderildiğinde tek satır oluyor (idempotency).

---

## O. Üretim sertleştirmesi ⬜

Golden image'a girmeden önce:

- [ ] Wi-Fi kapalı (üretim ünitelerinde)
- [ ] Raspberry Pi Connect kurulu değil / devre dışı
- [ ] VNC kapalı
- [ ] `swapon --show` → zram (dosya ise kapat)
- [ ] Overlay FS (read-only root) açık
- [ ] Chromium kiosk + gecelik reboot timer
- [ ] Merkezî log gönderimi
- [ ] Monitör uyku modu ve ekran koruyucu kapalı
- [ ] Fiziksel etiket: `machine_id`, `asset_code`, IP, MAC, Modbus slave adresi

---

## P. Golden image ve çoğaltma ⬜

Kurulmuş, test edilmiş kartın birebir kopyası çıkarılır; 18 üniteye yazılır.
Makineye özel ayarlar (hostname, `machine_id`, config) **Ansible** ile verilir.

**Kural:** üretime giden hiçbir ünite elle kurulmaz.

---

## Kabul kriterleri (brief §9.1 F) ⬜

Filo siparişi bunlara bağlı. Testler Pi #2 (`andon-pilot`) üzerinde koşar.

- [ ] 7 gün kesintisiz çalışma, elle müdahale olmadan
- [ ] Chromium RSS 7 gün boyunca 1,2 GB'ı aşmadı
- [ ] Ağ kablosu 30 dk çekildi → veri kaybı yok, kopya kayıt yok
- [ ] Güç 10 kez ani kesildi → her seferinde temiz açılış
- [ ] Vardiya sonu sistem sayımı elle sayımla birebir tuttu (3 vardiya üst üste)
- [ ] Andon butonu → bildirim gecikmesi < 5 sn
- [ ] `vcgencmd get_throttled` = 0, en sıcak günde pano içinde
- [ ] §4.6 filo imajı kararı ve Karar #6 (RAM) yazılı olarak kapatıldı
