# Raspberry Pi 5 — Yazılım

## 1. İşletim sistemi

**Raspberry Pi OS** = Debian'ın ARM için derlenmiş hali + Pi'ye özel araçlar
(`raspi-config`, `vcgencmd`, GPIO sürücüleri).

| | Desktop | Lite |
|---|---|---|
| Masaüstü arayüzü | Var | Yok, sadece komut satırı |
| Idle RAM | **450 MB** (ölçüldü) | ~150 MB |
| Kullanım | Öğrenme / bench | Üretim adayı |

**Bench kararı: Desktop.** Linux'a sıfırdan başlarken headless Lite gereksiz yavaşlatır.
**Filo kararı: AÇIK** — 7 günlük Chromium testinden sonra (brief §4.6).

⚠️ Lite imajında **Raspberry Pi Connect'in shell-only varyantı varsayılan kuruludur**;
golden image'da kaldırılacak (brief Karar #25).

**64-bit seç.** 32-bit sürüm eski kartlar için var; Pi 5'te anlamı yok.

## 2. Kart imajlama — Raspberry Pi Imager

`raspberrypi.com/software`. Üç seçim: Device (Raspberry Pi 5), OS (Raspberry Pi OS 64-bit,
"with desktop"), Storage (hedef kart).

### "Edit Settings" ekranı — en kritik adım

"Next" sonrası çıkan **"apply OS customisation settings?"** sorusunda **Edit Settings**.
Buradaki ayarlar karta yazılır, Pi ilk açılışta hazır gelir:

- **Hostname** — makinenin ağdaki adı (`andon-bench`)
- **Username / password** — bizde `andon`
- **Wi-Fi** — ağ adı, parola, ülke `TR`
- **Locale** — saat dilimi `Europe/Istanbul`, klavye `tr`
- **Services → Enable SSH** — işaretlenmezse uzaktan bağlanamazsın

Bu ekran olmadan kart açılır ama SSH kapalı gelir ve monitör+klavye şart olur.

**Imager ne yapıyor:** indirdiği imaj dosyasını karta bire bir yazıyor ve sonra bu ayarları
kartın açılış bölümüne ekliyor. Kartta iki bölüm oluşur: küçük FAT (boot, Windows'tan da
görünür) ve büyük ext4 (kök dosya sistemi).

## 3. İlk açılışta yapılacaklar

```bash
sudo apt update && sudo apt upgrade -y     # güncelle
hostname                                    # doğru mu
ip a                                        # IP adresi
free -m                                     # RAM
vcgencmd measure_temp                       # sıcaklık
swapon --show                               # swap nerede
```

Sonra laptop'tan `ssh andon@IP` ile bağlan ve monitörü bırak. Hedef: **GUI Pi'de değil,
laptop'ta.**

## 4. Pi'ye özel komutlar

Bunlar sadece Raspberry Pi OS'ta vardır:

| Komut | Ne yapar |
|---|---|
| `sudo raspi-config` | Menülü ayar aracı |
| `vcgencmd measure_temp` | İşlemci sıcaklığı |
| `vcgencmd get_throttled` | Kısılma bayrağı. `0x0` = sorun yok |
| `vcgencmd measure_volts` | Çekirdek gerilimi |
| `pinout` | GPIO pin haritasını ekrana çizer |
| `rpi-eeprom-update` | Kartın firmware'ini günceller |

`raspi-config` içinde bizi ilgilendirenler:
- **System Options** → hostname, açılışta masaüstü/konsol
- **Interface Options** → SSH, VNC, seri port, I2C, SPI
- **Performance** → **Overlay File System**
- **Localisation Options** → saat dilimi, klavye

## 5. Açılış dosyaları

Pi, PC gibi BIOS/UEFI ile açılmaz. SD karttaki firmware doğrudan okunur:

| Dosya | Ne için |
|---|---|
| `/boot/firmware/config.txt` | Donanım ayarları: ekran çözünürlüğü, fan eğrisi, arabirimler |
| `/boot/firmware/cmdline.txt` | Çekirdeğe verilen açılış parametreleri (tek satır) |

Bu dosyalar Windows'tan da görünür (kartın FAT bölümü) — Pi açılmıyorsa buradan müdahale
edilebilir. Bu, "kart bozuldu" sanılan durumların bir kısmının kurtarma yoludur.

## 6. Overlay File System (read-only root)

**Ne yapar:** kök dosya sistemini salt okunur yapar; tüm yazmalar RAM'de tutulan geçici bir
katmana gider ve her reboot'ta silinir.

**Neden bu projede kritik (brief §4.3):**

| Problem | Overlay FS'in cevabı |
|---|---|
| SD kart yazma aşınması | Yazma neredeyse sıfıra iner |
| Ani güç kesintisinde bozulma | Kimsenin yazmadığı dosya sistemi bozulmaz |
| Üniteler birbirinden farklılaşıyor | Her reboot bilinen-iyi duruma döner |

**Bedeli — planlanmış sonuçlar:**
- **Loglar reboot'ta kaybolur** → merkezî log toplama (Loki veya rsyslog) **zorunlu** hale
  gelir. 16 node'da bunu zaten istersin.
- **Güncelleme için overlay geçici kapatılmalı** → bir Ansible playbook. Zorunlu kılınan
  disiplin bir özelliktir: "9 numaralı makinede dokümansız elle düzenleme" imkânsızlaşır.
- **Store-and-forward kuyruğu kalıcı yer ister** → küçük bir USB bellek (veya SD'de ayrı
  yazılabilir bölüm) read-write mount edilir. Read-only root'un tek kasıtlı istisnası budur.

**Şimdilik AÇMA** — geliştirmeyi engeller. Faz 2'de golden image kurulurken açılacak.
`sudo raspi-config` → Performance → Overlay File System.

## 7. Swap ve zram

Swap = RAM dolduğunda diske taşınan alan. SD karta yazarsa aşınma yaratır.

```bash
swapon --show
```

Bizim imajda çıktı `/dev/zram0` → swap **zram**: RAM içinde sıkıştırılmış bir alan, SD'ye
hiç yazmaz. Sorun yok.

⚠️ Çıktıda `/var/swap` gibi bir **dosya yolu** görünürse o swap SD karta yazıyordur ve golden
image'da kapatılmalıdır (`sudo dphys-swapfile swapoff` + servisi devre dışı bırak).

## 8. Servisler ve otomatik başlatma

Pi'de kalıcı çalışacak her şey `systemd` servisi olacak (detay: `08-systemd-servisler.md`).

```bash
systemctl status andon-collector
sudo systemctl enable andon-collector    # açılışta başlasın
journalctl -u andon-collector -f         # canlı log
```

**Neden `rc.local` veya `crontab @reboot` değil:** `systemd` çökme sonrası yeniden başlatma,
bağımlılık sırası (ağ hazır olmadan başlama) ve düzgün loglama sağlar. Diğerleri sağlamaz.

## 9. Kiosk modu (Faz 1–2)

Üretimde Pi açılınca masaüstü değil, **tam ekran Chromium** gelecek: operatör ekranı.

Planlanan sonuçlar:
- Chromium kiosk modunda haftalar içinde bellek sızdırır → `systemd` timer ile **gecelik
  reboot**. Read-only root reboot'u zaten bedava yaptığı için ucuz sigorta (brief §4.2)
- Operatör kiosk'tan çıkıp ayarlara ulaşamamalı. Lite imajı bu konuda Desktop'tan daha sıkı —
  filo imajı kararının en güçlü argümanı bu
- Monitörün uyku modu ve ekran koruyucu kapatılacak; OLED değil LCD tercih (burn-in)
- Ekranda gösterilecek teknik doküman **sunucuda WebP'ye render edilmiş** olacak, PDF değil —
  2 GB RAM kararı buna bağlı (brief §6.3)

## 10. Python ve GPIO

Raspberry Pi OS'ta Python 3 kurulu gelir. Paket kurma:

```bash
sudo apt install python3-pip
pip install --break-system-packages paket     # ya da sanal ortam kullan
python3 -m venv .venv && source .venv/bin/activate
```

Yeni Debian sürümleri sistem Python'ına doğrudan `pip install` yapılmasını engeller (paket
yöneticisiyle çakışmasın diye). Doğru yol **sanal ortam** (`venv`) veya `apt` paketleri.

**GPIO kütüphanesi:** Pi 5'te GPIO, RP1 üzerinden geçer; **eski `RPi.GPIO` çalışmaz.**
Kullanılacak: **`gpiozero`** (arka planda `lgpio`).

```python
from gpiozero import Button
button = Button(17, pull_up=True, bounce_time=0.05)
button.wait_for_press()
print("basildi")
```

- `17` = **BCM** numarası (fiziksel bacak numarası değil)
- `pull_up=True` = pinin içindeki pull-up direncini aç
- `bounce_time=0.05` = **debounce**, mekanik butonun zıplamasını yok sayma süresi.
  PLC'de rung'a koyduğun filtre zamanının Python karşılığı

## 11. Aygıt isimleri ve udev

USB-RS485 adaptörü `/dev/ttyUSB0` olarak görünür. **Bu isim sabit değildir** — iki adaptör
varsa veya reboot olursa `ttyUSB1` olabilir ve collector yanlış porta bağlanır.

Çözüm: **udev kuralı** yazıp adaptörün seri numarasına sabit bir isim bağlamak:
`/dev/andon-rs485`.

Bakma komutları:
```bash
ls -l /dev/ttyUSB*
dmesg | tail -20                     # az önce takılan cihaz ne olarak tanındı
udevadm info -a -n /dev/ttyUSB0      # kuralı yazmak için gereken bilgiler
```

Bu, brief §9.1 adım B'nin çıktısı — sırası gelince yazacağız.

**Seri porta erişim izni:** kullanıcının `dialout` grubunda olması gerekir:
```bash
sudo usermod -aG dialout andon      # sonra oturumu kapat/aç
```

## 12. Ağ ayarları

Raspberry Pi OS Bookworm'da ağ yönetimi **NetworkManager** ile yapılır:

```bash
nmcli device status          # arabirimler ve durumları
nmcli connection show        # tanımlı bağlantılar
ip a                         # IP adresleri
```

**Bu projede statik IP yapılandırılmayacak** — router'da MAC adresine **DHCP rezervasyonu**
kullanılacak (brief §12.4). Ayar 16 kutunun içinde değil tek bir tabloda dursun.

## 13. Zaman senkronizasyonu

```bash
timedatectl                  # saat, dilim, senkron durumu
chronyc tracking             # NTP detayları
```

Pi'de RTC pili olmadığı için zaman NTP'den gelir. **Kural (brief §4.2): chrony senkron
olduğunu bildirmeden telemetri publish edilmez.** Saat kayması tüm veri setini değersiz kılar.

Üretimde NTP kaynağı **yerel sunucu** olacak, internet değil (brief §7.1).

## 14. Yedekleme ve golden image

- **Golden image** = kurulmuş, ayarlanmış, test edilmiş bir kartın birebir kopyası. 18 ünite
  bundan yazılacak.
- Kart kopyalama: Windows'ta Win32DiskImager / Raspberry Pi Imager, Linux'ta `dd`.
- **Kural:** üretime giden hiçbir ünite elle kurulmaz. Tek imaj + Ansible (brief §3.6).
- Yedek üniteler çekmecede önceden image'lanmış bekler; arıza halinde 20 dakikada değişim.

## 15. Bu projede Pi'de ne çalışacak

| Servis | İş |
|---|---|
| `andon-collector` | PLC'yi 1 Hz poll et, MQTT'ye gönder |
| `andon-button` | GPIO butonunu dinle, Andon çağrısı yay |
| Chromium kiosk | Operatör ekranı |
| chrony | Saat senkronizasyonu (yerel sunucudan NTP) |
| (log ajanı) | Merkezî loga gönder — read-only root nedeniyle zorunlu |

İlk üçü `systemd` servisi olacak: açılışta otomatik başlar, çökerse geri gelir, logu
`journalctl` ile okunur.
