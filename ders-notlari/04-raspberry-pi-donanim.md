# Raspberry Pi 5 — Donanım

## 1. Pi nedir, bu projede ne yapıyor

Kredi kartı boyutunda, ARM işlemcili küçük bir Linux bilgisayar. Bu projede **makine başına
bir tane** duracak ve üç iş yapacak (brief Karar #2):

1. PLC'yi Modbus ile okumak (USB-RS485 adaptörü üzerinden)
2. Operatör monitörünü sürmek (tam ekran Chromium)
3. Andon butonunu ve ışık kulesini GPIO'dan yönetmek

**Neden Pi:** monitörü süren bir cihaz zaten gerekliydi. Veri toplamanın marjinal maliyeti
makine başına **10 €'luk bir USB-RS485 adaptörü**ne indi (brief §3.2).

**Pi bir PLC değildir.** Gerçek zamanlı garanti vermez, emniyet fonksiyonu üstlenemez,
titreşim ve sıcaklık toleransı endüstriyel kartlardan düşüktür. Bu yüzden emniyet devrelerine
hiç dokunmuyoruz ve makine PLC'siz kalmıyor — Pi sadece *izliyor*.

## 2. Pi 5'in Pi 4'ten farkları

| Konu | Fark | Sonucu |
|---|---|---|
| İşlemci | Belirgin daha hızlı | Chromium kiosk rahat çalışır |
| Güç | Daha yüksek akım ister (resmi 27 W adaptör) | Telefon şarjı yetmez |
| Isı | Daha sıcak çalışır | Aktif soğutma pratikte zorunlu |
| GPIO | **RP1** adlı yardımcı yonga üzerinden geçer | Eski `RPi.GPIO` kütüphanesi çalışmaz |
| HDMI | İki adet **micro-HDMI** | Normal HDMI kablosu takılmaz |
| Güç düğmesi | Var | Nazik kapatma için |
| RTC | Pil bağlantısı var ama pil opsiyonel | Kapalıyken zamanı unutur |

## 3. Elimizdeki donanım

| Parça | Seçim | Neden |
|---|---|---|
| Kart | Raspberry Pi 5, **2 GB** × 2 | Yük Chromium'dan geliyor; ölçüldü, 2 GB rahat (brief §4.2) |
| Depolama | Endurance microSD 32 GB × 2 | Sürekli yazmaya dayanıklı tip. Normal kart sahada ölür |
| Güç | Resmi 27 W USB-C adaptör | Pi 5 yüksek akım ister |
| Soğutma | Resmi Active Cooler | Pi 5 sıcak çalışır |
| Ekran kablosu | micro-HDMI → HDMI | Pi 5'te normal boy HDMI yok |
| Seri | USB-RS485 adaptör × 2 | PLC'nin COM2 portuna bağlanacak; ikisi bench'te birbiriyle test edilecek |

## 4. Fiziksel kurulum sırası

1. **Active Cooler'ı Pi'ye güç vermeden önce tak.** Sonradan takmak termal pedi bozar. İki
   plastik pim deliklere hizalanır, eşit bastırılır; fan kablosu USB portlarına yakın 4 pinli
   **FAN** konnektörüne.
2. SD kartı tak.
3. micro-HDMI kabloyu **USB-C güç girişine en yakın** HDMI portuna tak.
4. Klavye, fare, ağ kablosu.
5. **En son güç.**

İlk açılış birkaç dakika sürer ve kart kendini bir kez yeniden başlatabilir. Normal.

**Kapatma:** `sudo poweroff` ile kapat, sonra fişi çek. Doğrudan fiş çekmek dosya sistemini
bozabilir — üretimde bu riski read-only root kaldıracak, ama bench'te henüz açık değil.

## 5. Portlar ve konnektörler

| Konnektör | Not |
|---|---|
| USB-C | Sadece güç girişi |
| micro-HDMI ×2 | Güç girişine yakın olan birincil ekran |
| USB-A ×4 | 2× USB 3.0 (mavi), 2× USB 2.0. RS485 adaptörü ve klavye buraya |
| Ethernet | Gigabit. **Üretimde bu kullanılacak, Wi-Fi değil** |
| GPIO | 40 pinli sıra |
| FAN | Active Cooler konnektörü |
| PCIe | Hızlı SSD için (bu projede kullanılmıyor) |
| CAM/DISP | Kamera ve ekran şeritleri (kullanılmıyor) |
| microSD | Kartın altında |

**Neden Wi-Fi değil Ethernet:** kablosuz bağlantı fabrika ortamında kararsızdır ve gecikme
öngörülemez. Zaten monitör için de kablo çekiliyor; aynı güzergâh (brief §3.1).

## 6. GPIO

GPIO = genel amaçlı giriş/çıkış pinleri. **PLC'deki `X` girişleri ve `Y` çıkışlarının
karşılığı**, ama çok daha hassas.

### Pin sırası

40 pinlik sırada üç tür pin var:
- **Güç:** 3,3 V ve 5 V çıkışları
- **GND:** toprak (8 adet). Her devrede en az bir GND kullanılır
- **GPIO:** programlanabilir pinler

**İki numaralandırma sistemi var ve karıştırılırsa yanlış pine bağlarsın:**

| Sistem | Ne sayar | Örnek |
|---|---|---|
| **BCM** | Yongadaki GPIO numarası | `GPIO17` |
| **BOARD** | Konnektördeki fiziksel bacak sırası | pin 11 |

`gpiozero` **BCM** kullanır. `Button(17)` yazdığında kastedilen GPIO17'dir, 17. bacak değil.
Pin haritası için `pinout` komutunu Pi'de çalıştırabilirsin.

### Elektriksel kurallar — pazarlıksız

| Konu | Kural |
|---|---|
| Gerilim | **3,3 V.** 5 V bile riskli, 24 V kartı anında yakar |
| 24 V saha sinyali | **Asla doğrudan bağlanmaz.** Her zaman optokuplör veya röle kartı üzerinden (brief §5.4, §13.4) |
| Çıkış akımı | Pin başına birkaç mA. Işık kulesi doğrudan sürülmez — röle kartı gerekir |
| Ortak GND | Sinyal kaynağıyla Pi'nin GND'si ortak olmalı, yoksa okuma kararsız olur |
| Pi 5 farkı | GPIO, RP1 üzerinden geçer. `RPi.GPIO` çalışmaz; **`gpiozero`** kullanılır |

**Bench'te buton doğrudan GPIO'ya bağlanabilir** çünkü butonun kendi gerilimi yoktur: bir
bacağı GPIO pinine, diğeri GND'ye. Direnç gerekmez — Pi'nin içinde pull-up direnci var ve
`gpiozero` onu yazılımdan açar.

**Sahada bu geçerli değildir.** Oradaki her sinyal 24 V'tur ve araya optokuplör girer.

### Pull-up / pull-down nedir

Bir giriş pini hiçbir yere bağlı değilse gerilimi belirsizdir ve rastgele okur ("floating").
Pull-up direnci pini varsayılan olarak 3,3 V'a çeker; butona basınca GND'ye kısa devre olur
ve pin 0 okur. Yani **basılı = 0, serbest = 1**. Ters mantık gibi gelir, alışılır.

### Diğer arabirimler (şimdilik kullanılmıyor)

| Arabirim | Ne için | Bu projede |
|---|---|---|
| UART | Seri haberleşme | Kullanılmıyor — USB-RS485 adaptörü tercih edildi |
| I2C | Sensör, RTC, ekran | İleride sıcaklık sensörü gerekirse |
| SPI | Hızlı çevre birimi | Gerekmiyor |
| 1-Wire | DS18B20 sıcaklık sensörü | Pano içi sıcaklık ölçümü için düşünülebilir |

`raspi-config` → Interface Options altından açılırlar.

## 7. Isı ve throttling

```bash
vcgencmd measure_temp      # anlık sıcaklık
vcgencmd get_throttled     # 0x0 = hiç kısılma olmadı
vcgencmd measure_volts     # çekirdek gerilimi
```

**Bench referansı (2026-08-27):** 43,9 °C, açık havada, idle, Active Cooler takılı.

İşlemci ısınınca kendini yavaşlatır — buna **throttling** denir ve `get_throttled` bayrağı
yanar. Asıl kabul kriteri sıcaklık rakamı değil, **`get_throttled` = 0** olmasıdır
(brief §9.1 F).

**Neden ciddi:** İzmir yazında kapalı bir elektrik panosunun içi 50–60 °C'yi bulabilir. Pi
oradan hava alacak. Filo sipariş edilmeden önce **gerçek pano içi sıcaklık ölçülecek**
(brief §4.4). Sıcak çıkan konumlarda fansız endüstriyel x86 mini PC alternatifi var —
yazılım aynı, sadece GPIO için USB çözüm gerekir.

## 8. Güç

Düşük gerilim, Pi'da tuhaf ve teşhisi **en zor** arıza sebebidir: rastgele donmalar, USB
cihazların kaybolması, SD kart hataları, açılışta takılma. Hepsi "bozuk kart" gibi görünür
ama sebep beslemedir.

- Resmi adaptör kullan, telefon şarjı deneme
- Uzun/ince USB-C kablo kullanma
- Ekranda sarı şimşek simgesi = düşük gerilim uyarısı
- Üretimde hedef: 16 Pi'nin de ana pano UPS'inden bir dalgalanmayı atlatması (brief §4.4)

## 9. RTC (gerçek zaman saati)

Pi 5'te pil bağlantısı var ama pil opsiyoneldir; pil yoksa kart **kapalıyken zamanı unutur**
ve açılışta 1970'ten başlar.

Bu projede kritik: güç kesintisinden sonra açılan bir Pi, NTP ile senkron olana kadar yanlış
zaman damgası üretir. Kural (brief §4.2): **chrony senkron olduğunu bildirmeden telemetri
gönderilmez**; gönderilenler `ts_synced: false` ile işaretlenir.

## 10. SD kart ömrü

SD kartlar sınırlı sayıda yazmaya dayanır ve sürekli yazma altında **aylar içinde** ölebilir.
Üç önlem:

1. **Endurance sınıfı kart** (SanDisk Max Endurance / Samsung PRO Endurance)
2. **Read-only root** — üretimde kök dosya sistemi salt okunur, yazmalar RAM'e gider
3. **Swap kontrolü** — ölçtük: bu imajda swap **zram** üzerinde, yani RAM içinde, SD'ye
   yazmıyor

**Kart bozulursa ne olur:** Pi açılmaz veya tuhaf davranır. Çözüm çekmecedeki önceden
image'lanmış yedek kartı takmaktır — 20 dakikada değişim (brief §3.6). Bu yüzden **18 ünite**
alınacak: 16 + 2 yedek.

## 11. İki bench ünitesi

| Ünite | Hostname | Rol |
|---|---|---|
| Pi #1 | `andon-bench` | Kirli geliştirme. Kur, boz, dene |
| Pi #2 | `andon-pilot` | Temiz. Sadece kanıtlanmış şeyler kurulur, elle müdahale edilmez |

Ayrım kasıtlı: "7 gün el değmeden çalıştı" diyebilmek için gerçekten el değmemiş olması
gerekiyor (brief §9.1 F). Ayrıca iki ünite Desktop ve Lite imajlarını **aynı anda, aynı
koşullarda** ölçmeyi sağlar.

## 12. İş güvenliği — pazarlıksız (brief §13.4)

- **Emniyet devrelerine dokunulmaz.** Acil stop, emniyet rölesi, ışık bariyeri, kapı kilidi:
  ne okumak ne yazmak için bağlantı yapılır.
- **Pano içi çalışma** enerji kesilerek ve LOTO (etiketle-kilitle) prosedürüyle yapılır.
  Elektrik yetkinliği ve izni olmayan hiçbir iş tek başına yapılmaz.
- **Kule lambası tap'i** sadece optokuplörle, akım çekmeyecek ve lamba davranışını
  değiştirmeyecek şekilde bağlanır.
- **Makineye kalıcı bir ekleme** (sensör, buton kutusu, ekran kolu) risk değerlendirmesini
  etkileyebilir; CE beyanı olan makinelerde İSG uzmanına sor.
- **Ekran ve buton konumu** operatörün hareket alanını, görüş hattını veya kaçış yolunu
  engellememeli.
- **ESD:** karta çıplak elle dokunmadan önce topraklı bir metale dokun. Statik elektrik
  yongayı sessizce öldürebilir.
