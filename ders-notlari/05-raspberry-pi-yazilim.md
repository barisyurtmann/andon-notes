# Raspberry Pi 5 — Yazılım

## İşletim sistemi

**Raspberry Pi OS** = Debian'ın ARM için derlenmiş hali + Pi'ye özel araçlar
(`raspi-config`, `vcgencmd`, GPIO sürücüleri).

İki varyant:

| | Desktop | Lite |
|---|---|---|
| Masaüstü arayüzü | Var | Yok, sadece komut satırı |
| Idle RAM | **450 MB** (ölçüldü) | ~150 MB |
| Kullanım | Öğrenme / bench | Üretim adayı |

**Bench kararı: Desktop** — Linux'a sıfırdan başlarken headless Lite gereksiz yavaşlatır.
**Filo kararı: AÇIK** — 7 günlük Chromium testinden sonra verilecek (brief §4.6).

⚠️ Lite imajında **Raspberry Pi Connect'in shell-only varyantı varsayılan kuruludur**;
golden image'da kaldırılacak (brief Karar #25).

## Kart imajlama — Raspberry Pi Imager

`raspberrypi.com/software` adresinden indirilir. Üç seçim:

| Kutu | Bizim seçim |
|---|---|
| Device | Raspberry Pi 5 |
| Operating System | Raspberry Pi OS (64-bit), "with desktop" |
| Storage | Hedef SD kart |

### "Edit Settings" ekranı — en kritik adım

"Next" sonrası çıkan **"apply OS customisation settings?"** sorusunda **Edit Settings**
seçilir. Buradaki ayarlar karta yazılır, yani Pi ilk açılışta hazır gelir:

- **Hostname** — makinenin ağdaki adı (`andon-bench`)
- **Username / password** — bizde `andon`
- **Wi-Fi** — ağ adı, parola, ülke `TR`
- **Locale** — saat dilimi `Europe/Istanbul`, klavye `tr`
- **Services → Enable SSH** — bu işaretlenmezse uzaktan bağlanamazsın

Bu ekran olmadan kart açılır ama SSH kapalı gelir ve monitör+klavye şart olur.

## Pi'ye özel komutlar

Bunlar sadece Raspberry Pi OS'ta vardır, normal Debian'da yoktur:

| Komut | Ne yapar |
|---|---|
| `sudo raspi-config` | Menülü ayar aracı: arabirimler, overlay FS, açılış seçenekleri |
| `vcgencmd measure_temp` | İşlemci sıcaklığı |
| `vcgencmd get_throttled` | Kısılma (throttling) bayrağı. `0x0` = sorun yok |
| `vcgencmd measure_volts` | Çekirdek gerilimi |

`raspi-config` içinde bizi ilgilendiren yerler:
- **Interface Options** → SSH, VNC, seri port
- **Performance** → **Overlay File System** (aşağıda)
- **System Options** → hostname, açılışta masaüstü/konsol seçimi

## Açılış dosyaları

Pi, PC gibi BIOS/UEFI ile açılmaz. SD karttaki firmware doğrudan okunur:

| Dosya | Ne için |
|---|---|
| `/boot/firmware/config.txt` | Donanım ayarları: ekran, fan, arabirimler |
| `/boot/firmware/cmdline.txt` | Çekirdeğe verilen açılış parametreleri |

Bu dosyalar Windows'tan da görünür (SD kartın FAT bölümü) — Pi açılmıyorsa buradan
müdahale edilebilir.

## Overlay File System (read-only root)

**Ne yapar:** kök dosya sistemini salt okunur yapar; tüm yazmalar RAM'de tutulan geçici bir
katmana gider ve her reboot'ta silinir.

**Neden bu projede kritik (brief §4.3):**

| Problem | Overlay FS'in cevabı |
|---|---|
| SD kart yazma aşınması | Yazma neredeyse sıfıra iner |
| Ani güç kesintisinde bozulma | Kimsenin yazmadığı dosya sistemi bozulmaz |
| Üniteler birbirinden farklılaşıyor | Her reboot bilinen-iyi duruma döner |

**Bedeli:** loglar reboot'ta kaybolur → merkezî log toplama zorunlu hale gelir. Ayrıca
güncelleme yapmak için overlay geçici olarak kapatılmalı. Bu bir kusur değil, dayatılan
disiplin: "9 numaralı makinede dokümansız elle düzenleme" imkânsızlaşır.

**Şimdilik AÇMA** — geliştirmeyi engeller. Faz 2'de golden image kurulurken açılacak.

Açılışı: `sudo raspi-config` → Performance → Overlay File System.

## Swap ve zram

Swap = RAM dolduğunda diske taşınan alan. SD karta yazarsa aşınma yaratır.

```bash
swapon --show
```

Bizim imajda çıktı `/dev/zram0` — yani swap **zram**'dir: RAM içinde sıkıştırılmış bir alan,
SD karta hiç yazmaz. Sorun yok, kapatmaya gerek yok.

⚠️ Çıktıda `/var/swap` gibi bir **dosya yolu** görünürse o swap SD karta yazıyor demektir ve
golden image'da kapatılmalıdır.

## Kiosk modu (ileride)

Üretimde Pi açılınca masaüstü değil, **tam ekran Chromium** gelecek: operatör ekranı.
Sonuçları:

- Chromium kiosk modunda haftalar içinde bellek sızdırır → `systemd` timer ile **gecelik
  reboot** planlanacak. Read-only root reboot'u zaten bedava yaptığı için ucuz sigorta.
- Operatörün kiosk'tan çıkıp ayarlara ulaşamaması gerekiyor. Lite imajı bu konuda Desktop'tan
  daha sıkı — filo imajı kararının en güçlü argümanı bu.
- Monitörün uyku modu ve ekran koruyucu kapatılacak.

## GPIO kütüphanesi

Pi 5'te GPIO, RP1 yardımcı yongası üzerinden geçer. **Eski `RPi.GPIO` çalışmaz.**

Kullanılacak: **`gpiozero`** (arka planda `lgpio`).

```python
from gpiozero import Button
button = Button(17, pull_up=True, bounce_time=0.05)
button.wait_for_press()
print("basildi")
```

- `17` = GPIO pin numarası (fiziksel bacak numarası değil)
- `pull_up=True` = pinin içindeki pull-up direncini aç
- `bounce_time=0.05` = **debounce**, mekanik butonun zıplamasını yok sayma süresi.
  PLC'de rung'a koyduğun filtre zamanının Python karşılığı

## Aygıt isimleri ve udev

USB-RS485 adaptörü `/dev/ttyUSB0` olarak görünür. **Ama bu isim sabit değildir** — iki
adaptör varsa veya reboot olursa `ttyUSB1` olabilir ve collector yanlış porta bağlanır.

Çözüm: **udev kuralı** yazıp adaptörün seri numarasına sabit bir isim bağlamak:
`/dev/andon-rs485`. Bu, §9.1 adım B'nin çıktısı — sırası gelince yazacağız.

## Bu projede Pi'de ne çalışacak

| Servis | İş |
|---|---|
| `andon-collector` | PLC'yi 1 Hz poll et, MQTT'ye gönder |
| `andon-button` | GPIO butonunu dinle, Andon çağrısı yay |
| Chromium kiosk | Operatör ekranı |
| chrony | Saat senkronizasyonu (sunucudan NTP) |

Üçü de `systemd` servisi olacak: açılışta otomatik başlar, çökerse geri gelir, logu
`journalctl` ile okunur. **Bu kalıp bir sonraki notun konusu.**
