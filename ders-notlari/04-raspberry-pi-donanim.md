# Raspberry Pi 5 — Donanım

## Pi nedir, bu projede ne yapıyor

Kredi kartı boyutunda, ARM işlemcili küçük bir Linux bilgisayar. Bu projede **makine başına
bir tane** duracak ve üç iş yapacak (brief Karar #2):

1. PLC'yi Modbus ile okumak (USB-RS485 adaptörü üzerinden)
2. Operatör monitörünü sürmek (tam ekran Chromium)
3. Andon butonunu ve ışık kulesini GPIO'dan yönetmek

**Neden Pi:** monitörü süren bir cihaz zaten gerekliydi. Veri toplamanın marjinal maliyeti
makine başına **10 €'luk bir USB-RS485 adaptörü**ne indi.

## Elimizdeki donanım

| Parça | Seçim | Neden |
|---|---|---|
| Kart | Raspberry Pi 5, **2 GB** | Yük Chromium'dan geliyor; ölçüldü, 2 GB rahat (brief §4.2) |
| Depolama | Endurance microSD 32 GB | Sürekli yazmaya dayanıklı tip. Normal kart sahada ölür |
| Güç | Resmi 27 W USB-C adaptör | ⚠️ Pi 5 yüksek akım ister; telefon şarjıyla çalıştırma |
| Soğutma | Resmi Active Cooler | Pi 5 sıcak çalışır, aktif soğutma pratikte zorunlu |
| Ekran kablosu | micro-HDMI → HDMI | Pi 5'te normal boy HDMI yok |
| Seri | USB-RS485 adaptör | PLC'nin COM2 portuna bağlanacak |

## Fiziksel kurulum sırası

1. **Active Cooler'ı Pi'ye güç vermeden önce tak.** Sonradan takmak işlemcinin üstündeki
   termal pedi bozar. İki plastik pim deliklere hizalanır, eşit bastırılır; fanın kablosu
   USB portlarına yakın 4 pinli **FAN** konnektörüne.
2. SD kartı tak.
3. micro-HDMI kabloyu **USB-C güç girişine en yakın** HDMI portuna tak (Pi 5'te iki tane var).
4. Klavye, fare, ağ kablosu.
5. **En son güç.**

İlk açılış birkaç dakika sürer ve kart kendini bir kez yeniden başlatabilir. Normal.

## Portlar ve konnektörler

| Konnektör | Not |
|---|---|
| USB-C | Güç girişi. Veri için kullanılmaz |
| micro-HDMI ×2 | Güç girişine yakın olan birincil ekran |
| USB-A ×4 | 2 adet USB 3.0 (mavi), 2 adet USB 2.0. RS485 adaptörü ve klavye buraya |
| Ethernet | Gigabit. Üretimde bu kullanılacak, Wi-Fi değil |
| GPIO | 40 pinli sıra. Andon butonu ve ışık kulesi buraya |
| FAN | Active Cooler'ın 4 pinli konnektörü |
| microSD | Kartın altında |

## GPIO — dikkat edilecekler

GPIO = genel amaçlı giriş/çıkış pinleri. **PLC'deki `X` girişleri ve `Y` çıkışlarının
karşılığı**, ama çok daha hassas.

| Konu | Kural |
|---|---|
| Gerilim | **3,3 V.** 5 V bile riskli, 24 V kartı anında yakar |
| 24 V saha sinyali | **Asla doğrudan bağlanmaz.** Her zaman optokuplör veya röle kartı üzerinden (brief §5.4, §13.4) |
| Çıkış akımı | Çok düşük. Işık kulesi doğrudan sürülmez, röle kartı gerekir |
| Pi 5 farkı | GPIO artık RP1 adlı yardımcı yonga üzerinden geçiyor. Eski `RPi.GPIO` kütüphanesi çalışmaz; `gpiozero` kullanılır |

**Bench'te buton doğrudan GPIO'ya bağlanabilir** çünkü butonun kendi gerilimi yoktur: bir
bacağı GPIO pinine, diğeri GND'ye. Direnç gerekmez — Pi'nin içinde pull-up direnci var ve
`gpiozero` onu yazılımdan açıyor.

**Sahada bu geçerli değildir.** Oradaki her sinyal 24 V'tur ve araya optokuplör girer.

## Isı

Pi 5, Pi 4'ten belirgin şekilde sıcak çalışır.

```bash
vcgencmd measure_temp      # anlık sıcaklık
vcgencmd get_throttled     # 0x0 = hiç kısılma olmadı
```

**Bench referansı (2026-08-27):** 43,9 °C, açık havada, idle, Active Cooler takılı.

Asıl kriter sıcaklık rakamı değil, **`get_throttled` = 0** olmasıdır. İşlemci ısınınca
kendini yavaşlatır (throttling) ve bu bayrak yanar. İzmir yazında kapalı bir pano içinde
bu ciddi bir risk — filo sipariş edilmeden önce pano içi sıcaklık ölçülecek (brief §4.4).

## Güç

Düşük gerilim, Pi'da tuhaf ve teşhisi en zor arıza sebebidir: rastgele donmalar, USB
cihazların kaybolması, SD kart hataları. Hepsi "bozuk kart" gibi görünür ama sebep beslemedir.

- Resmi adaptör kullan, telefon şarjı deneme
- Uzun/ince USB-C kablo kullanma
- Üretimde hedef: 16 Pi'nin de ana pano UPS'inden bir dalgalanmayı atlatması

## RTC (gerçek zaman saati) yok

Pi 5'te pil bağlantısı opsiyoneldir; pil takılı değilse kart **kapalıyken zamanı unutur.**

Bu projede önemli: güç kesintisinden sonra açılan bir Pi, NTP ile senkron olana kadar
yanlış zaman damgası üretir. Kural (brief §4.2): **chrony senkron olduğunu bildirmeden
telemetri gönderilmez**; gönderilenler `ts_synced: false` ile işaretlenir.

## SD kart ömrü

SD kartlar sınırlı sayıda yazmaya dayanır. İki önlem:

1. **Endurance sınıfı kart** (SanDisk Max Endurance / Samsung PRO Endurance)
2. **Read-only root** — üretimde kök dosya sistemi salt okunur yapılacak, tüm yazmalar RAM'e
   gidecek (yazılım notunda detay)

Ayrıca ölçtük: bu imajda swap **zram** üzerinde, yani RAM içinde. SD karta yazmıyor.

## İki bench ünitesi

| Ünite | Hostname | Rol |
|---|---|---|
| Pi #1 | `andon-bench` | Kirli geliştirme. Kur, boz, dene |
| Pi #2 | `andon-pilot` | Temiz. Sadece kanıtlanmış şeyler kurulur, elle müdahale edilmez |

Ayrım kasıtlı: "7 gün el değmeden çalıştı" diyebilmek için gerçekten el değmemiş olması
gerekiyor (brief §9.1 F).

## İş güvenliği — pazarlıksız

- **Emniyet devrelerine dokunulmaz.** Acil stop, emniyet rölesi, ışık bariyeri, kapı kilidi:
  ne okumak ne yazmak için bağlantı yapılır.
- **Pano içi çalışma** enerji kesilerek ve LOTO (etiketle-kilitle) prosedürüyle yapılır.
- **Kule lambası tap'i** bile sadece optokuplörle, akım çekmeyecek ve lamba davranışını
  değiştirmeyecek şekilde bağlanır.
