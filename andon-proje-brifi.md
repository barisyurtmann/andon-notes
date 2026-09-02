# Andon & OEE Sistemi — Proje Brifi

**Durum:** Tasarım mutabık, pilot öncesi. **IT onayı alındı (2026-08-29).**
**Sahip:** Junior otomasyon mühendisi (tek kişi; hem otomasyon hem veri mühendisliği rolü)
**Son güncelleme:** 2026-09-02
**Sürüm:** 2.7 (TR) — v2.6'ya göre değişen: §5.1 (32-bit word sırası **doğrulandı**), §5.2
(DVP adres tabanları vendor tablosuyla **doğrulandı**), §5.3 ve Karar #5 (COM2 fabrika
ayarında bırakılıyor), §9.1 E, §11 madde 4–5, §14.2, yeni Karar #28

> **Sahibin yetkinlik profili — anlatım dili bunun üzerine kurulmalı.**
> PLC, ladder, Modbus, RS-485, elektrik panosu ve saha tarafı **biliniyor**.
> Linux, Raspberry Pi, Docker, MQTT, TimescaleDB, Ansible, Grafana ve genel olarak veri
> mühendisliği **bilinmiyor; bu proje ile birlikte öğreniliyor.** Bu dokümanda geçen bir
> terimin yazılı olması, sahibinin onu bildiği anlamına gelmez.
> Öğrenme planı §7.2'de, uygulaması `andon-ogrenme-yolu.md` dosyasında.

> **Bu doküman nasıl kullanılır.** Bu, projenin yaşayan referansıdır. Claude Project
> knowledge'ta tutulur; böylece her yeni sohbet tam bağlamla başlar. Bir karar değiştiğinde
> veya açık bir soru cevaplandığında burası güncellenir.
>
> **Çalışma kopyası (2026-09-02'den itibaren):** aynı dosya `andon-notes/andon-proje-brifi.md`
> yolunda git'te de tutulur (§15 madde 3). **İkisi her güncellemede birlikte yazılır.**
> Çelişki görülürse Claude Project'teki sürüm esastır; git kopyası yedek ve düzenleme aracıdır.
>
> **İşaretler:**
> - **[DOĞRULA]** — teyit edilmemiş. Gerçek donanım veya vendor dokümanı ile doğrulanmadan
>   üzerine bina kurulmayacak.
> - **[DOĞRULANDI]** — gerçek donanımda veya vendor dokümanında teyit edildi; kaynak ve tarih
>   yazılı.
> - **[YENİ]** — v2.0'da eklenen bölüm; orijinal brifte yoktu.
> - **[ÖNERİ]** — karar verilmemiş, tartışmaya açık öneri. Karar günlüğüne henüz girmedi.
> - **[ÖLÇÜLDÜ]** — bench'te gerçek donanımda ölçüldü; tarih ve rakam yazılı.

---

## 1. Amaç

16 makinelik bir montaj hattı için iki işlevi kapsayan bir Andon ve OEE izleme sistemi kurmak.
**Bu hat pilottur; sistem birden fazla hatta, bölüme ve fabrikaya yayılacak şekilde
tasarlanır (§2).**

**A. Makine izleme**
- Her makinede canlı parametreleri gösteren bir ekran (verimlilik, belirli bir andan itibaren
  üretim, makine durumu)
- Tüm verinin geçmişe dönük sorgu ve analiz için veritabanına yazılması
- Merkezî olarak bir motor kodu seçildiğinde ilgili teknik dokümanın her makinede otomatik
  görüntülenmesi

**B. Andon çağrıları**
- Operatörün bakım, kalite, malzeme vb. çağırması için fiziksel butonlar
- İlgili sorumlunun bilgilendirilmesi ve gelmesi
- Her çağrının kaydı: açıldı → onaylandı → çözüldü, zaman damgalarıyla

**Değer önerisi B'dir, A değil.** Bir lamba ve bir korna 50 €. Bu sistemi haklı çıkaran şey
tepki süresi verisidir: MTTA, MTTR, makine/vardiya başına çağrı sayısı ve çağrı sebeplerinin
Pareto'su. B'yi erken ve görünür şekilde inşa et — operatör gönlünü ve yönetim desteğini
A için kazandıran şey odur.

---

## 2. Makine envanteri

| Grup | Adet | PLC | Haberleşme durumu | Yaklaşım | Risk |
|---|---|---|---|---|---|
| 1 | 8 | Delta DVP-SS2 | İki COM portu da boş | Pi COM2'yi Modbus ile poll eder | Düşük |
| 2 | 5 | Delta DVP-SS2 | Delta HMI bağlı | **[DOĞRULA]** HMI büyük olasılıkla COM1'de (RS-232). Öyleyse COM2 boş — Grup 1 ile aynı, HMI'ya dokunulmaz | Düşük |
| 3 | 1 | Delta AS218TX | HMI bağlı | **[DOĞRULA]** AS200 serisinde dahili Ethernet olmalı → doğrudan Modbus TCP, PLC değişikliği yok | Çok düşük |
| 4 | 2 | Bilinmiyor (Çin menşeli) | Program yok, vendor desteği yok | Harici algılama — PLC'yi okumaya çalışma | Orta |

**Mevcut protokol karmaşası:** çoğunlukla Modbus, kısmen ASCII, kısmen RTU.
**Çözüm (revize 2026-09-02):** karmaşa PLC'ye yazarak *ortadan kaldırılmıyor*, config'e
taşınıyor. Her makinenin seri ayarı `machines.yaml`'da yazılı olur; collector her makineye
kendi ayarıyla bağlanır. Gerekçe §5.3.

> **[YENİ] Tesis gerçeği: tek hat değil, çok hat ve çok fabrika.**
> Firmada birden fazla fabrika, birden fazla bölüm ve ürün ölçüsüne göre adlandırılmış hatlar
> var (`4INC`, `3INC`, `2_5INC` gibi). **Yayılma bazen hat bazlı bile olmayacak — tek tek
> makine bazlı ilerlenebilecek.** Mimarî sonuçları:
>
> 1. **Sistem makine merkezlidir, hat merkezli değil.** Her makine kendi Pi'si ile bağımsız
>    devreye alınabilir. Bir hattın tamamının hazır olması hiçbir şeyin ön şartı değildir.
> 2. **Hat, kimliğin parçası olamaz** — makineler hat değiştirebilir. Yerleşim bilgisi bir
>    *özelliktir* ve zamana bağlıdır (§12.5).
> 3. **İkinci hat için kod kopyalanmaz.** Kod repo'ları hattan bağımsızdır; her hat sadece
>    `andon-ansible` içinde bir envanter klasörü ekler (§12.4).
> 4. Bu doküman boyunca geçen "16 makine" ifadesi **pilot hattı** anlatır, sistemin sınırını
>    değil.

> **[YENİ] PLC parkı tek tip değil ve tek tip kalmayacak.** Yukarıdaki tablo bugünün
> fotoğrafıdır. Hatta hâlihazırda en az üç farklı PLC ailesi var ve ileride başka marka/model
> makineler de bu sisteme bağlanabilir. Bunun mimarî sonuçları:
>
> 1. **Hiçbir tasarım Delta DVP-SS2'ye özel olamaz.** DVP'ye ait her detay (özel register'lar,
>    adres base'leri, ladder komutları) bu dokümanda **"Grup 1–2 için geçerli örnek"** olarak
>    okunmalı, genel kural olarak değil.
> 2. **Collector sürücü (driver) mimarisiyle yazılır.** Ortak bir çekirdek + protokol başına
>    ince bir sürücü: `modbus_rtu`, `modbus_tcp`, `gpio_sensor`, ileride `s7`, `ethernet_ip`,
>    `ascii_custom`. Yeni bir PLC ailesi eklemek = yeni bir sürücü dosyası + config satırı;
>    ana servise dokunulmaz.
> 3. **Makine ekleme maliyeti config satırı olmalı**, kod değişikliği değil. Bu, §5.1'deki
>    veri modeli sözleşmesinin asıl gerekçesidir.

### Grup 4 stratejisi: PLC'yi değil, makineyi oku

- **Kule lambası tap'i** — kırmızı/sarı/yeşil kule lambaları üzerinden optokuplör ile Pi
  GPIO'ya. PLC bilgisi olmadan makine durumu (çalışıyor / uyarı / arıza) verir. ~5 € parça.
  Legacy OEE retrofit'te standart uygulama.
- **Fotoelektrik veya endüktif sensör** çıkış oluğu / itme noktasında → parça sayımı. ~30 €.
- **Akım trafosu (CT) klempi** motor beslemesinde → çalışıyor/boşta ayrımı, kullanılabilir
  kule lambası yoksa.
- *Opsiyonel:* HMI seri hattına pasif, sadece-dinleme tap'i takıp HMI'nin ne poll ettiğini
  parse etmek. Sensörlerden sonra denenir; daha az güvenilir.

**Bu tekniği her makine için yedekte tut** — sadece Grup 4 için değil. PLC değişikliği
politik veya pratik olarak imkânsız çıkan her makinede aynı yol geçerlidir.

> **[YENİ] Kule lambası tap'inde iş güvenliği kuralı:** Bu bağlantı **emniyet devresine
> hiçbir şekilde müdahale etmez.** Optokuplör lamba hattına paralel bağlanır, akım çekmez
> ve arıza durumunda makinenin lamba davranışını değiştirmez. Emniyet rölesi, acil stop
> veya güvenlik PLC'si sinyallerine kesinlikle dokunulmaz. Detay: §13.4.

### Her şeyden bağımsız, hemen yapılacak

**Bu hafta 16 PLC programının tamamını git'e yedekle.** En az bir makinede diskteki dosyanın
PLC'nin içindekiyle uyuşmadığını göreceksin. Bunu şimdi keşfet, dördüncü ayda değil.

---

## 3. Mimari

### 3.1 Seçilen tasarım

```
╔════════════════════════════════════════════════════════════════════════╗
║  MAKİNE KATMANI (×16)                                                  ║
╠════════════════════════════════════════════════════════════════════════╣
║  8× DVP-SS2      5× DVP-SS2+HMI    1× AS218TX     2× bilinmeyen        ║
║  COM1: boş       COM1: HMI ✓       Ethernet       program yok          ║
║  COM2: ──┐       COM2: ──┐          ──┐           kule lambası         ║
║   RS-485 │  1 m   RS-485 │            │ TCP       + sensör ──┐         ║
║  ════════╪═══════════════╪════════════╪══════════════════════╪═════════║
║          ▼               ▼            │                      ▼         ║
║  ┌────────────────────────────────┐   │   ┌───────────────────────┐    ║
║  │  RASPBERRY PI 5 (makine başına)│   └──▶│ doğrudan switch'e     │    ║
║  │  ├─ USB-RS485 adaptör (10 €)   │       └───────────────────────┘    ║
║  │  ├─ operatör MONİTÖRÜ (kiosk)  │                                    ║
║  │  ├─ GPIO: andon butonu + kule  │                                    ║
║  │  └─ STORE-AND-FORWARD KUYRUĞU  │                                    ║
║  │     (kalıcı, reboot'a dayanır) │                                    ║
║  └───────────────┬────────────────┘                                    ║
╚══════════════════╪═════════════════════════════════════════════════════╝
                   │  Ethernet (yıldız topoloji, yönetilebilir switch)
                   │  ── aynı kablo Pi + monitör ──
                   │  MQTT doğrudan sunucuya — ara broker yok
                   ▼
╔════════════════════════════════════════════════════════════════════════╗
║  ANA PANO                                                              ║
╠════════════════════════════════════════════════════════════════════════╣
║   ┌──────────────────────┐          ┌───────────────────────────┐      ║
║   │  S7-1500 (1511)      │◀────────▶│  KTP1200 Basic            │      ║
║   │  • sipariş/motor kodu│  S7comm  │  • motor kodu seçimi      │      ║
║   │  • hat seviyesi durum│          │  • hat komutları          │      ║
║   │  • VERİ BORUSU DEĞİL │          │  (operatör login DEĞİL)   │      ║
║   └──────────┬───────────┘          └───────────────────────────┘      ║
║              │ snap7 / OPC UA — SUNUCUDAKİ servis tarafından okunur    ║
╚══════════════╪═════════════════════════════════════════════════════════╝
               ▼
╔════════════════════════════════════════════════════════════════════════╗
║  SUNUCU  (yerinde, UPS'li, Docker)                                     ║
╠════════════════════════════════════════════════════════════════════════╣
║   ┌───────────┐  ┌────────────┐  ┌──────────────┐  ┌────────────────┐  ║
║   │ Mosquitto │─▶│ Ingest svc │─▶│ TimescaleDB  │  │ Doküman deposu │  ║
║   │ MQTT      │  │ MQTT→DB    │  │ ├ raw        │  │ (motor koduna  │  ║
║   │ broker    │  │ idempotent │  │ ├ cleaned    │  │  göre PDF,     │  ║
║   └───────────┘  └────────────┘  │ ├ OEE agg    │  │  versiyonlu;   │  ║
║   ┌───────────┐  ┌────────────┐  │ └ andon evt  │  │  WebP §6.3)    │  ║
║   │ NTP       │  │ S7 reader  │  └──────┬───────┘  └───────┬────────┘  ║
║   │ (hat için)│  │ motor kodu │         │                  │           ║
║   └───────────┘  └────────────┘         │                  │           ║
║   ┌───────────┐  ┌──────────────────────▼──────────────────▼────────┐  ║
║   │ Bildirim  │◀─│ Web app                                          │  ║
║   │ Telegram  │  │ • makine UI + doküman görüntüleyici              │  ║
║   └─────┬─────┘  │ • FİLO SAĞLIK SAYFASI (16 yeşil/kırmızı nokta)   │  ║
║         │        └──────────────────┬───────────────────────────────┘  ║
║         │        ┌──────────────────▼───────────┐   ──▶ 16 makine      ║
║         │        │ Grafana — CANLI / OPERASYONEL│       monitörüne     ║
║         │        │ • Andon panosu (büyük ekran) │                      ║
║         │        │ • mühendislik görünümleri    │                      ║
║         │        └──────────────────────────────┘                      ║
║         │        ┌──────────────────────────────┐                      ║
║         │        │ Power BI — GERİYE DÖNÜK      │ sadece AGGREGATE     ║
║         │        │ • yönetim raporlaması        │ tablolar (§6.4)      ║
║         │        └──────────────────────────────┘                      ║
╚═════════╪══════════════════════════════════════════════════════════════╝
          ▼
  Bakım / Kalite telefonları     Andon panosu (tavan)     Yönetim
```

**Telemetri yukarı akar:** PLC → Pi → MQTT → sunucu broker → veritabanı.
**Bağlam aşağı akar:** HMI'da motor kodu seçilir → S7 → sunucudaki reader → MQTT → Pi →
monitör doğru dokümanı gösterir.

### 3.2 Telemetri neden S7-1500 üzerinden geçmiyor

Bu uzun uzun tartışıldı. Karar: S7 veri yolunun dışında kalır.

**Belirleyici argüman:** S7 sunucunun yerine geçmiyor — sunucunun **önüne** ekleniyor. Yine
sunucuya, veritabanına ve collector'a ihtiyacın var. S7 seri bağlı bir bileşen ekler;
hiçbir şeyi kaldırmaz. Seri bağlı bileşenler arıza olasılığını azaltmaz, çarpar.

**Gerçek envanterle maliyet karşılaştırması:**

| | S7-1500 konsantratör | Pi doğrudan |
|---|---|---|
| Seri arabirim donanımı | 3–4 × CM PtP RS422/485 HF @ ~500–700 € → **1.800–2.800 €** | 16 × USB-RS485 → **160 €** |
| Kablolama | Hat boyunca RS-485 çekimi + işçilik | Pano içinde 1 m |
| Grup 4 makineleri | DI modülü + aynı sensörler yine gerekir | GPIO, zaten var |
| ASCII cihazlar | **[DOĞRULA]** S7-1500'de Siemens Modbus PtP komutlarının sadece RTU olduğu düşünülüyor → ASCII makineler erişilemez | pymodbus ikisini de yapar — **ve bu 2026-09-02'de sahada belirleyici çıktı (§5.3)** |
| Geliştirme | `Modbus_Comm_Load` + `Modbus_Master` + istasyon sıralaması, SCL'de ~3–4 hafta | Python servis + YAML config, ~1 hafta |
| Makine eklemek | TIA'da düzenle + 16 makineye hizmet veren CPU'ya download | Config dosyasında altı satır |
| Bir makinenin arızası | 4–5 makineyle aynı segmenti paylaşır | Sadece o makineyi etkiler |
| Yönetilecek 16 Linux node | **Yine gerekli — monitörler için** | Aynı 16 |
| Sunucu + veritabanı | **Yine gerekli** | Aynı |

**Kilit farkındalık:** 16 hesaplama cihazı zaten tasarımda var, çünkü her makinenin monitörünü
bir şeyin sürmesi gerekiyor. Dolayısıyla tüm veri toplama katmanının marjinal maliyeti makine
başına **10 €'luk bir USB-RS485 adaptörü**. S7 yolu Linux filosundan kurtarmıyor — filonun
**üstüne** bir PLC, ~2.500 € comm modülü ve bir aylık SCL ekliyor.

**Çok hat argümanı da aynı yöne bakıyor (§2):** S7 konsantratör modeli her yeni hatta yeni bir
PLC ve yeni comm modülleri demek olurdu. Pi modelinde yeni hat = makine başına 10 € adaptör +
bir config klasörü.

**Bu kararı ne tersine çevirirdi:** IT veya tesis politikası fabrika ağında Linux cihazı
yasaklasaydı. **2026-08-29: IT onay verdi, bu şart ortadan kalktı (§3.6). Karar #1 artık
koşulsuz.**

### 3.3 S7-1500'ün gerçek bir görevi var

- KTP1200 Basic'i sürer
- Üretim siparişi ve motor kodu seçimi
- Hat seviyesi durum ve komutlar
- Güncel motor kodunu yayınlar (optimize edilmemiş bir DB üzerinden snap7 ile, ya da lisans
  varsa OPC UA), böylece monitörler hangi dokümanı göstereceğini bilir. **snap7 reader artık
  sunucuda çalışıyor**, ara bir cihazda değil.

Sadece telemetri borusu değil.

### 3.4 "Tek bir yere bakayım" ihtiyacı

Meşru bir operasyonel içgüdü, ama bunu bir PLC ile karşılamak yanlış. Yerine: her node bir
heartbeat yayınlar ve **sunucudaki web app** 16 yeşil/kırmızı noktalı bir durum sayfası
barındırır. Bu, S7 online görünümünden kesinlikle daha iyidir çünkü *hangi* makinenin
sustuğunu söyler.

> **[YENİ] Uygulama detayı:** Bunu heartbeat sayacı yerine **MQTT Last Will and Testament
> (LWT)** ile yap. Her Pi bağlanırken `.../heartbeat` topic'ine retained `online` mesajı
> yazar ve LWT olarak retained `offline` tanımlar. Bağlantı koparsa broker `offline`'ı
> kendisi yayınlar — sunucuda zaman aşımı mantığı yazmana gerek kalmaz. Detay §12.1.

### 3.5 KTP1200 Basic — bilinen kısıt

Basic paneller script desteklemez, bağlantı sayısı kısıtlıdır, alarm ve veri kaydı zayıftır.
16 makine boyunca operatör ataması yapmak Basic panelde çok zahmetli ekran işi olur.

**Karar:** operatör login'i merkezî HMI'da değil, makine monitöründe olur. Her istasyonda bir
RFID kart okuyucu (USB HID, ~15 €) daha doğaldır — operatör zaten makinenin başındadır — ve
sana bir Andon çağrısını sadece *ne zaman* değil *kimin* onayladığını verir.

> **[YENİ] KVKK uyarısı:** Operatör kimliği kişisel veridir. Badge login eklemek bu projeyi
> KVKK kapsamına sokar. §13.2'yi devreye almadan badge okuyucu satın alma.

### 3.6 IT onayı — **ALINDI (2026-08-29)**

> **Sonuç: IT fabrika ağında Linux cihazlara izin verdi.** Sahibin ifadesiyle kısıt
> konulmadı. Bu, projedeki en büyük mimari riski kapatıyor:
> - **Karar #1 koşulsuz hale geldi** — telemetri Pi'lar üzerinden gider, S7-1500 veri yolunun
>   dışında kalır. ~1.800–2.800 € comm modülü ve ~1 aylık SCL geliştirme masaya gelmeyecek.
> - **§11 madde 3 kapandı.**
>
> **Bu, aşağıdaki disiplinleri gevşetmez.** İzin alınmış olması, taahhütlerin geçersiz olduğu
> anlamına gelmez — envanter, tek golden image, ayrı VLAN, SSH key, MQTT kimlik doğrulama ve
> merkezî log yine yapılacak. Sebep IT değil, **sistemin bir kişi tarafından ayakta
> tutulabilmesi** (§15). İzin verilmiş bir kaosun bedelini yine sen ödersin.
>
> **Hâlâ netleştirilecek:** izin "fabrika ağında Linux cihaz" düzeyinde alındı. Üretim
> VLAN'ının internete çıkışı ayrı bir konudur ve §7.1 gereği **kapalı kalmalıdır** — Pi'lar
> sadece MQTT broker'ı (1883) ve NTP ile konuşur. Geliştirme makinesinin (PC) GitHub'a
> erişimi ofis ağı üzerindendir ve bu taahhütle çelişmez.

Aşağıdaki tablo, IT ile konuşmanın çerçevesi olarak yazılmıştı. **Artık ikna aracı değil,
uygulanacak kontrol listesidir.**

| Endişe | Verilen / verilecek cevap |
|---|---|
| Sahiplik / shadow IT | Yazılı envanter: MAC, IP, seri no, konum, image sürümü. Sahip: adıyla sen ve departmanın |
| Yamalama | Ansible + tek golden image. Tüm node'lar tek komutla, aynı image'dan, hiçbir yerde elle düzenleme olmadan |
| Ağ hijyeni | Ayrı üretim VLAN'ı, ofis ağından firewall ile ayrılmış |
| Trafik kapsamı | Pi'lar tam olarak iki şeyle konuşur: sunucunun MQTT broker'ı (1883) ve NTP. İnternet yok, ofis ağı yok, peer-to-peer yok |
| Verinin sahayı terk etmesi | Yok. Tamamen yerinde (§7.1) |
| Kurtarma | Çekmecede önceden image'lanmış yedek üniteler; 20 dakikada değişim. Read-only root bozulma senaryosunu daraltır |
| Erişim kontrolü | Sadece SSH key, parola ile giriş kapalı, root login kapalı, tek golden image, gereksiz servis yok |
| Kimlik doğrulama | MQTT'de her Pi için ayrı kullanıcı/parola + ACL. Ağ segmentasyonu tek savunma katmanı değil (§13.1) |
| Log ve denetim | Merkezî log (Loki/rsyslog), en az 90 gün |
| Uzaktan erişim | Vendor bulut tüneli yok — Raspberry Pi Connect kullanılmıyor (Karar #25). Erişim sadece üretim VLAN'ı içinden SSH key ile |
| Büyüme | Aynı model diğer hatlara/fabrikalara taşınacak (§2). Her yeni node aynı golden image ve aynı envanter disiplinine girer |

---

## 4. Donanım

### 4.1 Makine başına (×16)

| Kalem | Spesifikasyon | Yaklaşık maliyet |
|---|---|---|
| Hesaplama | **Raspberry Pi 5, 2 GB** | bkz. §4.2 |
| Depolama | Endurance sınıfı microSD 32 GB (SanDisk Max Endurance / Samsung PRO Endurance) | 13 € |
| Seri | USB-RS485 adaptör | 10 € |
| Kuyruk depolaması | Store-and-forward kuyruğu için read-write mount edilmiş küçük USB bellek | 5 € |
| Kasa / soğutma | **Sahibin kararı — ayrıca ele alınıyor** | — |
| Güç | **Sahibin kararı — ayrıca ele alınıyor.** §4.4'teki filo seviyesi kısıta dikkat | — |
| Ekran | Monitör + micro-HDMI kablo | değişken |
| Andon | Buton + ışık kulesi (GPIO/röle kartı üzerinden) | 20 € |

**18 adet al** — 16 + çekmecede önceden image'lanmış 2 yedek.

**Filo tekdüzeliği pazarlık konusu değildir.** Hangi kart seçilirse seçilsin, 18'i de aynı
olacak. Karışık filo, birim başına yapılan her tasarruftan daha fazlasını destek süresinde
geri alır. **Bu kural hatlar arasında da geçerlidir:** 2. hattın Pi'ları da aynı model olmalı,
aksi halde tek golden image iddiası biter.

### 4.2 Boyutlandırma gerekçesi — **REVİZE**

Modbus polling önemsiz bir iş; bir mikrodenetleyici bile yapar. **Tüm donanım ihtiyacı,
teknik çizimlerle birlikte kiosk modunda Chromium çalıştırmaktan geliyor.**

**Orijinal 4 GB boyutlandırması geçersizdir.** Tarayıcının PDF'i istemci tarafında render
edeceğini varsayıyordu; bu +300–800 MB'lık bir sıçrama ve ~1,8 GB'lık en kötü durum
üretiyordu. §6.3 kararı — dokümanları **sunucu tarafında** WebP'ye render etmek — bu sıçramayı
tamamen ortadan kaldırır.

Revize bütçe:

| | RAM |
|---|---|
| Raspberry Pi OS Lite (64-bit) | ~150 MB |
| X server + minimal WM | ~100 MB |
| Chromium, tek sekme, dashboard + WebP | 400–700 MB |
| **Gerçekçi toplam** | **~1 GB** |

2 GB kabaca 1 GB headroom bırakır. Bu rahat, sınırda değil.

> **[ÖLÇÜLDÜ] Bench ilk ölçümü — 2026-08-27, `andon-bench`, Pi 5 2 GB, Desktop imaj,
> yerel masaüstü oturumu açık, Chromium kapalı, idle:**
>
> | Ölçüm | Değer |
> |---|---|
> | `free -m` → total | 2006 MB |
> | `free -m` → used | **450 MB** |
> | `free -m` → available | **1555 MB** |
> | `swapon --show` | `/dev/zram0`, 2 GB, kullanım 0 B (bkz. §4.3) |
> | `vcgencmd measure_temp` | **43,9 °C** (Active Cooler, açık hava, oda sıcaklığı) |
>
> **Yorumu:** Desktop imajın idle maliyeti §4.6'da hafızadan tahmin edilen 500–700 MB'ın
> altında çıktı. Chromium için kalan bütçe ~1,5 GB. **Bu, imaj kararını kapatmaz** — §9.1
> F'deki asıl test Chromium kiosk açıkken, en büyük çizim yüklüyken ve 7 gün boyunca
> yapılacak. Bellek sızıntısı idle ölçümünde görünmez.

**Bu bütçe Lite imajı varsayar.** §4.6 filo imajı kararı açık; tam masaüstü imajı bu tabloya
idle'da ölçülen ~450 MB'ı ekler. Karar #6 bu yüzden §4.6 kapanana kadar koşulludur.

**Seçilen: Raspberry Pi 5, 2 GB.** Türkiye fiyatlarıyla 4 GB versiyon 2 GB'ın kabaca iki katı;
18 ünitede bu, iş yükünün artık ihtiyaç duymadığı headroom için gerçek para demek.

**Bu karar koşulludur. İki şeyin geçerli kalmasına bağlı:**

1. **§6.3 kalacak.** Dokümanlar sunucu tarafında WebP'ye render edilecek.
2. **Render genişliği 3840 px'de kalacak.** Belleği kurtaran format değil, *decode edilmiş*
   bitmap: `genişlik × yükseklik × 4 bayt`. 1920 px ≈ 10 MB; 3840 px ≈ 41 MB; 7000 px'lik bir
   A1 çizim ≈ 140 MB. Asıl emniyet mekanizması bu sınırdır.

**Neden Pi 5, Pi 4 değil:**
- Pi 3 geliştirme aşaması planlanmıyor
- Pi 5 sahada kiosk / overlay-FS / Ansible ekosisteminin olgunlaşacağı kadar zaman geçirdi
- **[DOĞRULA]** Pi 5'in resmi üretim devamlılığı taahhüdünün Pi 4'ten uzun olduğu
  düşünülüyor. Resmi Raspberry Pi longevity tablosundan teyit et.

**Planlanacak Pi 5 sonucu:** Pi 5'te GPIO, RP1 southbridge üzerinden geçer. `RPi.GPIO`
çalışmaz. `gpiozero` kullan (`lgpio` backend'i ile). **[DOĞRULA]** — ilk butonu bağlarken
teyit et.

**Filo alımından önce doğrulama:** pilot ünitede gerçek kiosk uygulamasını en büyük gerçek
çizim yüklü halde çalıştır, bir hafta `free -m` ve Chromium RSS'i izle. **Bu ölçüm, filoya
gidecek imaj üzerinde yapılmalıdır (§4.6).**

> **[YENİ] İki bench ünitesi var (2026-08-27).** Elde **iki Pi 5** bulunuyor. Rolleri ayrıldı:
> - **Pi #1 — `andon-bench`:** kirli geliştirme ünitesi. Kurulur, bozulur, denenir.
> - **Pi #2 — `andon-pilot`:** temiz ünite. Sadece çalıştığı kanıtlanmış şeyler kurulur, elle
>   müdahale edilmez. §9.1 F'deki **7 gün kesintisiz** testi burada koşar ve golden image
>   adayı budur.
>
> Kazanç: bir ünite yedi gün el değmeden koşarken diğerinde geliştirme sürebilir; ayrıca
> Desktop ve Lite imajları **aynı anda, aynı koşullarda** ölçülebilir — §4.6 kararı bir hafta
> erken kapanabilir. **Şart:** Pi #2 de 2 GB olmalı. **[DOĞRULA — Pi #2'nin RAM'i teyit
> edilecek.]**

**2 GB ile zorunlu:** Chromium kiosk modunda haftalar içinde bellek sızdırır. `systemd` timer
ile gecelik reboot planla. Read-only root reboot'u zaten bedava yaptığı için bu ucuz sigorta.

> **[YENİ] Pi 5 ısı uyarısı.** Pi 5, Pi 4'ten belirgin şekilde daha sıcak çalışır ve pratikte
> aktif soğutma ister. İzmir yazında kapalı bir elektrik panosunun içinde bu kritik:
> 1. Pano içi sıcaklığı **ölçmeden** filo sipariş etme.
> 2. Sıcak çıkan konumlarda §4.5'teki fansız endüstriyel x86 mini PC alternatifi ciddi
>    değerlendirilmeli. Yazılım aynı; sadece GPIO için USB çözüm gerekir.
>
> **Referans taban [ÖLÇÜLDÜ]:** 43,9 °C, açık havada, idle. Kabul kriteri sıcaklık rakamı
> değil, `vcgencmd get_throttled` = 0 olmasıdır (§9.1 F).

> **[YENİ] Pi 5'te RTC yok (pil bağlantısı opsiyonel).** Kural: **chrony senkron olduğunu
> bildirmeden telemetri publish edilmez.** Servis başlangıcında `chronyc tracking` kontrolü
> yap, senkron değilse veriyi kuyruğa al ama zaman damgasını `unsynced` bayrağıyla işaretle.
> Detay §12.2.

### 4.3 Güvenilirlik yaklaşımı

**Read-only root dosya sistemi** (`raspi-config` → Performance → Overlay File System). Root
salt okunur; tüm yazmalar RAM overlay'e gider.

- Karta yazma neredeyse sıfıra iner — aşınma önemsizleşir
- **Güç kesintisi bir olay olmaktan çıkar**
- Her reboot bilinen-iyi duruma döner; yaramazlık yapan bir Pi'ya saha ziyareti değil, güç
  çevrimi uygularsın

**Bu yüzden makine başına SSD gerekmiyor.** SSD yazma aşınmasını çözer ama güç kesintisi
bozulmasına hiçbir şey yapmaz. Read-only root ikisini de bedavaya çözer.

> **[ÖLÇÜLDÜ] Swap, SD karta yazmıyor — 2026-08-27.** `swapon --show` çıktısı `/dev/zram0`,
> 2 GB. Swap **zram**'dir: RAM içinde sıkıştırılmış bir alan, SD karta hiç yazmaz. Read-only
> root ile çelişmez, filo imajında kapatılması gerekmez. **Golden image kurulurken tekrar
> kontrol et:** çıktıda bir dosya yolu görünüyorsa swap kapatılacak.

**Planlanacak sonuçlar:**
- Reboot'ta loglar kaybolur → **merkezî log toplama zorunlu** (Loki veya rsyslog).
- Güncelleme overlay'i geçici olarak kapatmayı gerektirir → bir Ansible playbook. Zorunlu
  kılınan disiplin bir özelliktir: 9 numaralı makinede dokümansız elle düzenleme olamaz.
- **Store-and-forward kuyruğu kalıcı depolama ister.** RAM overlay'deki kuyruk reboot'ta
  kaybolur; küçük bir USB belleği read-write mount edip kuyruğu oraya koy. Read-only root'un
  tek kasıtlı istisnası budur.

**Tamponu kaybetmek neden yine de tolere edilebilir:** DVP sayaçları hiç sıfırlanmayan
serbest-akan totalizer'lardır. İki saat ölü kalan bir Pi döndüğünde sayacı okur, son bildiği
değerle karşılaştırır ve fark bozulmamıştır — **sayımdan hiç parça kaybolmaz.**

> **[ÖNERİ] USB bellek yerine SD kartta ayrı yazılabilir bölüm.** Makine başına bir USB bellek,
> sahada 16 adet çıkabilecek/kırılabilecek parça demek. Alternatif: aynı endurance SD kartta
> kuyruğa ayrılmış küçük bir ext4 bölümü, root yine read-only.
> **Karar verilmedi.** Pilotta ikisini de dene, birini seç ve Karar #7'yi güncelle.

### 4.4 Güç ve ortam

**Soğutma ve güç kaynağı seçimi sahibin sorumluluğunda.** Mimarîyi etkilediği için iki filo
seviyesi kısıt burada kalıyor:

- **[DOĞRULA] Filoya karar vermeden önce pano sıcaklıklarını ölç.** İzmir yaz ortamında ayakta
  kalmalı. Sıcak konumlarda fansız endüstriyel kutu kabul edilebilir — yazılım aynı.
- **UPS stratejisi.** Hedef: 16 Pi'nin de ana pano UPS'inden bir güç dalgalanmasını
  atlatması. PoE kullanılırsa switch'leri UPS'e almak bunu tek üniteyle sağlar.
- Düşük gerilim, Pi'da tuhaf ve teşhisi zor davranışların bir numaralı sebebidir.

### 4.5 Fiyatlandırmaya değer alternatif

Yenilenmiş bir mini PC (Dell OptiPlex Micro, Lenovo ThinkCentre Tiny, HP EliteDesk Mini) ikinci
elde çoğunlukla i5, 8 GB RAM ve SSD ile 60–120 € arasında. x86, standart Debian, SD kart derdi
yok, daha iyi termal. **Handikap:** GPIO yok — USB üzerinden 5 €'luk bir Arduino Nano veya USB
röle/giriş kartı ile çözülür.

**Hangisi seçilirse seçilsin, tek tipte standardize ol.**

### 4.6 İşletim sistemi — Linux kesin, imaj seçimi AÇIK

**Sadece Linux.** Pi üzerinde Windows değerlendirildi ve reddedildi (Karar #15): GPIO yok,
2 GB'a Chromium ile sığmaz, read-only root yok, Ansible/Docker/systemd yok, üretim örneği yok.
Windows'un bu projedeki tek meşru yeri geliştirme PC'si.

#### Bench / geliştirme ünitesi — karar verildi

**Raspberry Pi OS with desktop, 64-bit.** Sahibin Linux deneyimi yok; headless Lite ile
başlamak öğrenmeyi gereksiz yavaşlatır. **Kuruldu ve çalışıyor — 2026-08-27.**

#### Filo golden image (16+2 ünite) — **[AÇIK]**

**Karar kriteri:** §9.1 F'deki bellek ve kararlılık testleri **filoya gidecek imaj üzerinde**
çalıştırılır. Desktop imajla test geçerse filo Desktop kullanır; geçmezse Lite'a düşülür.
Ölçüm karar verir. **İki Pi sayesinde iki imaj paralel ölçülebilir (§4.2).**

**Lite lehine argümanlar:**

1. **Bellek — [ÖLÇÜLDÜ], argüman zayıfladı.** Tahmin 500–700 MB idi; ölçüm **450 MB used /
   1555 MB available**. Bu argüman tek başına Lite'ı gerektirmiyor.
2. **Operatör kilidi.** Tam masaüstünde operatör kiosk'tan çıkıp dosya yöneticisine
   ulaşabilir. 16 istasyonda bu yaşanır. **Artık Lite lehine en güçlü argüman budur.**
3. **Yama ve denetim yüzeyi.** §3.6'daki "minimal servis, tek golden image" taahhüdünü
   zayıflatır.

**Lite'ın kendi getirdiği iş:** Lite imajında **Raspberry Pi Connect'in shell-only varyantı
varsayılan kuruludur**; Karar #25 gereği golden image'da kaldırılacak.

**Karar tarihi: filo siparişinden önce.** Sipariş sonrası geri dönüş, 18 ünitenin yeniden
imajlanması demektir. **Karar #6 (2 GB) bu kapanana kadar koşulludur.**

### 4.7 Geliştirme akışı — **REVİZE (2026-08-29, Karar #27)**

**Kod ve config PC'de yazılır, GitHub'a push edilir, Pi `git pull` ile alır.**

```
PC (yazma, commit, push) ──▶ GitHub (private) ──▶ Pi: git pull (read-only deploy key)
                                                  Faz 2'den itibaren: Ansible
```

Eski plan "geliştirme doğrudan Pi üzerinde" idi ve tek Pi varken doğruydu. İki Pi ve bir PC
varken geçersiz: repo'nun kalıcı yeri geliştirme makinesidir, Pi çalışan kopyayı taşır.

Sonuçları:
- **Üretimdeki Pi'lerde git bulunmaz, elle düzenleme yapılmaz.** Read-only root açıldığında
  bu zaten teknik olarak zorunlu olur (§4.3).
- **Pi'ler repo'ya yazamaz.** GitHub'da her Pi için **read-only deploy key** (§12.4).
- **GPIO ve seri port testleri Pi'de kalır** — donanım orada. Pi'ye dosya taşımanın yolu
  `scp` veya `git pull`; kalıcı olan her şey PC'deki repo'ya döner.
- Repo'lar **private**. Şahsi GitHub hesabı geçici; kalıcı olarak firma organizasyonuna
  taşınacak (§15 bus factor).

**Yerel çalışma klasörü (2026-09-02 itibarıyla kurulu):**

```
C:\Users\byurtman\Desktop\andon\
├─ andon-collector\    kod + config (machines.yaml) + tools\
├─ andon-notes\        CALISMA-GUNLUGU.md, KURULUM-ADIMLARI.md,
│                      ders-notlari\, andon-proje-brifi.md (çalışma kopyası)
└─ vendor-docs\        Delta Modbus adres tabloları (.xls) — §5.2'nin kaynağı
```

> **[YENİ] Donanım gelmeden yazılabilecek kısım:** `pymodbus` bir **Modbus slave simülatörü**
> de içerir. DVP register haritasını (§5.1) taklit eden bir simülatör yazıp collector'ı, ingest
> servisini ve veritabanı şemasını hiç PLC'ye dokunmadan geliştirebilirsin. Detay §14.1.

---

## 5. PLC tarafı tasarım

### 5.1 Veri modeli sözleşmesi

**Register adresleri makine bazlı belirlenir.** Makine parkı karışık (§2), her makineden aynı
bilgiler alınamayacak ve bazılarında PLC'ye hiç dokunulamayacak.

**Dondurulan şey adres değil, alan isimleri ve anlamlarıdır:**

#### Ortak veri modeli (kanonik alanlar) — bu tablo değişmez

| Alan | Tip | Anlam | Zorunlu mu |
|---|---|---|---|
| `machine_id` | metin | **Anlamsız, kalıcı iç kimlik** (`M001`…). Fabrika kodu değildir — §12.4 | Evet |
| `ts` | zaman (UTC) | Ölçüm anı | Evet |
| `parts_total` | 32-bit tamsayı | Toplam üretim sayacı | Hayır* |
| `scrap_total` | 32-bit tamsayı | Toplam hurda sayacı | Hayır |
| `state` | tamsayı | 0=kapalı, 1=boşta, 2=çalışıyor, 3=arıza, 4=tip değişimi | Evet |
| `fault_code` | tamsayı | Aktif arıza kodu | Hayır |
| `cycle_ms` | tamsayı | Son çevrim süresi (ms) | Hayır |
| `andon_bits` | tamsayı | Andon çağrı bitleri (PLC'den geliyorsa) | Hayır |

\* `parts_total` yoksa o makinede OEE hesaplanamaz; §2'deki harici sensör yolu devreye girer.

**`state` enum'ı kesinlikle değişmez.** Makinenin kendi durum kodları farklıysa çeviri
**collector'da** yapılır, veritabanında değil.

**Telemetride sadece `machine_id` taşınır.** Fabrika kodu, adı, hattı, bölümü telemetri
mesajına konmaz — bunlar master veridir ve zamanla değişir (§12.5).

#### Adresler config'e taşınır

Her makine için eşleme YAML'da yaşar. Güncel dosya: `andon-collector/machines.yaml`.

```yaml
machines:
  - id: M001
    asset_code: MAK25-30-511     # fabrikanın envanter kodu; boş olabilir
    display_name: "4 INC Pres 2"
    site: FAB1
    area: MONTAJ
    line: 4INC                   # yerleşim — değişebilir, geçmişi §12.5'te
    driver: modbus_rtu
    port: /dev/andon-rs485
    serial:                      # makine bazlı — §5.3, Karar #28
      framer: ascii
      baud: 9600
      bytesize: 7
      parity: E
      stopbits: 1
    slave: 1
    map_verified: false          # doğrulanana kadar false
    map:
      parts_total: { addr: 0x112C, words: 2, order: lo_hi }   # order [DOĞRULANDI] 2026-09-02
      state:       { addr: 0x1130, words: 1 }                 # TODO: VERIFY
```

Yeni makine eklemek = bu dosyaya bir blok yazmak. Kod değişmez.

#### Yetenek matrisi (capability matrix) — Faz 0 çıktısı

| Makine | asset_code | parts_total | scrap | state | fault | cycle | Kaynak | Not |
|---|---|---|---|---|---|---|---|---|
| M001 | ? | ? | ? | ? | ? | ? | DVP COM2 | Faz 0'da doldurulacak |

**Faz 0'da her makinede `andon-collector/tools/scan_dvp.py` çalıştırılır** (sadece okur) ve
bulunan seri ayarı bu matrise yazılır.

#### Değişmeyen kurallar

- **Sayaçlar serbest akan totalizer olmalı.** PLC'den asla sıfırlanmaz; farkları sunucuda
  hesapla, 32-bit rollover'ı ele al. (Karar #3.)
- **Sıfırlanan sayacı olan makineler config'de `resetting: true` ile işaretlenir.**
- **Ham değerleri sakla, hesaplanmışları değil.**
- **Yeni alan eklemek serbest; mevcut alanın anlamını değiştirmek yasak.** Anlam değişecekse
  yeni isim ver (`state_v2`).

> **Neden bu bölüm "en önemli eser":** otomasyon yarısı ile veri yarısının sahibi aynı kişi
> olduğu için sözleşmeyi esnetmeye karşı doğal bir denetim yok. Adresler serbest bırakıldı;
> **anlamlar serbest bırakılırsa 16 makinelik bir sistem değil, 16 ayrı proje çıkar.**

> **[DOĞRULANDI] 32-bit word sırası = `lo_hi` — 2026-09-02, bench DVP-SS2.**
>
> **Yöntem:** WPLSoft device monitor'den elle `D300 = 2`, `D301 = 1` girildi. `0x112C`
> adresinden tek istekte 2 register okundu.
> **Sonuç:** `[2, 1]` → **ilk register düşük word (low word)**, ikinci register yüksek word.
> 32-bit değer = `(reg[1] << 16) | reg[0]`.
>
> Betik: `andon-collector/tools/read_test.py`. Ham çıktı `andon-notes/CALISMA-GUNLUGU.md`'de.
>
> **Bu test aynı anda §5.2'yi de doğruladı:** `0x112C`'den okunan değerler gerçekten
> D300/D301 idi, yani D tabanı `0x1000` sahada teyit edildi.
>
> **Uyarı — bu sonuç DVP ailesine aittir, evrensel değildir.** AS serisi, S7 veya başka bir
> marka eklendiğinde word sırası **o aile için ayrıca ölçülür.** `machines.yaml`'daki `order`
> alanı tam bu yüzden makine bazlıdır.

> **Atomik okuma sorunu.** 32-bit sayacın iki word'ünü ayrı isteklerde okursan rollover anına
> denk gelip tutarsız değer alabilirsin. **Tek istekte oku**
> (`read_holding_registers(addr, count=2)`). `read_test.py` ve collector bunu böyle yapar.

> **[ÖNERİ] D308'e program sürümü koy.** Her PLC programına git tag'iyle eşleşen bir sürüm
> numarası yazan bir sabit ekle. Collector okur, farklıysa uyarır. Böylece "diskteki dosya
> PLC'dekiyle uyuşmuyor" problemi sessizce oluşmaz. Maliyeti tek bir MOV rungu.

### 5.2 DVP Modbus adres eşlemesi — *sadece Delta DVP için geçerli*

**[DOĞRULANDI — 2026-09-02.** Kaynak: `vendor-docs/ES-EX-SS-SA-SX-SC-EH-SS2-SX2-SA2-ES2-SV-SV2_PLC_MODBUS_ADRESLERI.xls`,
Delta'nın resmi adres tablosu. Ayrıca bench'te D300 okumasıyla sahada teyit edildi.]

| Cihaz | Tip | Modbus base (0-tabanlı) |
|---|---|---|
| S adım röleleri | coil | 0x0000 |
| X girişler | discrete input | 0x0400 |
| Y çıkışlar | coil | 0x0500 |
| T zamanlayıcılar | coil / register | 0x0600 |
| M röleler | coil | 0x0800 |
| C sayaçlar | coil / register | 0x0E00 |
| D register | holding register | 0x1000 |

Yani `D0` = 0x1000, `D300` = 0x112C.

#### Gösterim dönüşüm kuralları

Vendor tablosu 1-tabanlı Modbus gösterimi (`4xxxx`) kullanır; `pymodbus` 0-tabanlı ister.

| Gösterim | Dönüşüm | Örnek |
|---|---|---|
| Word (`4xxxx`) | `adres − 40001` | D23 → 44120 → **4119** = 0x1017 |
| Coil (`0xxxx`) | `adres − 1` | — |
| Discrete input (`1xxxx`) | `adres − 10001` | — |

**D300 hesabı (doğrulanmış örnek):** D000 = 44097 → D300 = 44097 + 300 = 44397 →
44397 − 40001 = 4396 = **0x112C**.

#### COM2 yapılandırma register'ları (vendor tablosundan)

| Cihaz | Tip | Adres | İşlev |
|---|---|---|---|
| `D1120` | holding register | 0x1460 | COM2 haberleşme protokolü / baud / format |
| `D1121` | holding register | 0x1461 | İstasyon (slave) adresi |
| `M1120` | coil | 0x0C60 | COM2 ayarını koru |
| `M1143` | coil | 0x0C77 | RTU / ASCII seçimi |

Adresler tablodan alındı; **bit anlamları** (hangi bit hangi baud'u seçer) hâlâ manuelden
teyit edilecek — ama §5.3 kararı gereği bunlara yazma planı yok.

> **Off-by-one tuzağı.** Yukarıdaki dönüşüm kuralları uygulanmadan hiçbir adres kullanılmaz.
> Bench'te bilinen bir D register'ı okuyup **beklenen değeri gördüğünü** doğrulamadan hiçbir
> adresi kesin kabul etme. Bu disiplin 2026-09-02'de uygulandı ve tabanı doğruladı.

> **Excel okuma tuzağı — 2026-09-02'de yaşandı.** Vendor dosyasındaki "HEXADECİMAL ADRES"
> sütunu **özel sayı biçimi (custom number format)** kullanır: hücrede ham `17` durur ama
> ekranda `1017` görünür. Dosyayı programla okursan ham değeri alırsın ve yanlış sonuç
> çıkarırsın. **Kural: programatik çıkarımda "MODBUS ADRESLERİ" sütununu kullan** — o düz
> sayıdır, biçimlendirme yoktur. Detay: `ders-notlari/13-modbus-ve-seri-haberlesme.md`.

### 5.3 COM2 seri ayarı — **REVİZE (2026-09-02, Karar #28)**

**Karar: COM2 fabrika ayarında bırakılır. Ayar PLC'ye yazılmaz, config'e yazılır.**

**[ÖLÇÜLDÜ] 2026-09-02, bench DVP-SS2, `scan_dvp.py`:**

| Parametre | Değer |
|---|---|
| Framer | **ASCII** |
| Baud | **9600** |
| Veri / eşlik / stop | **7-E-1** |
| Slave | **1** |

Bunlar DVP-SS2 COM2'nin **fabrika varsayılanıdır** — yani hiçbir PLC değişikliği yapmadan
okunabiliyor.

**Eski karar (v2.6'ya kadar):** "COM2 her yerde Modbus RTU 19200/38400 8-N-1'e standardize
edilir." Gerekçesi ASCII/RTU karmaşasını ortadan kaldırmaktı.

**Neden değişti:**

| Argüman | Değerlendirme |
|---|---|
| Hız | 1 Hz poll'de ASCII 9600'ün bir okuma turu ~30–50 ms. Bütçe 1000 ms. **Hız gerekçesi yok.** |
| Karmaşayı kaldırma | Karmaşa kaldırılmıyor, sadece yeri değişiyor. Grup 3 (AS218TX, Modbus TCP) ve Grup 4 (sensör) zaten farklı. Collector nasılsa çok protokollü (Karar #22) |
| Maliyet | 13 üretim PLC'sine yazma = 13 planlı duruş penceresi. §5.5'e göre bu projenin **en olası gecikme sebebi** |
| Risk | `D1120`/`M1143` bit anlamları doğrulanmadı. Yanlış yazarsan PLC'nin haberleşmesini komple kaybedersin ve geri almak için makinenin başına gitmen gerekir |
| Kazanç | Config'de dört satır tasarruf |

**Kazanç config'de dört satır; bedeli 13 duruş ve bir haberleşme kaybı riski.** Ödünleşim
açık.

**Uygulama:** her makinenin seri ayarı `machines.yaml`'da `serial:` bloğunda tutulur (§5.1).
Faz 0'da her makinede `scan_dvp.py` çalıştırılır ve bulunan ayar oraya yazılır.

**Bu kararı ne tersine çevirirdi:** bir makinede fabrika ayarı çalışmıyorsa, ya da bir hatta
tek RS-485 üzerinde birden fazla PLC multi-drop bağlanacaksa — o zaman **sadece o makinelerde**
ayar değiştirilir, filo genelinde değil.

> **İstasyon adresi planı.** Modbus slave adresi **hat içinde** anlamlıdır. Fabrika varsayılanı
> her PLC'de 1'dir; her makinenin kendi RS-485 hattı ve kendi Pi'si olduğu için **çakışma
> yoktur ve değiştirmeye gerek yoktur.** Multi-drop yapılacak bir hat çıkarsa orada sırayla
> numaralandır ve `machines.yaml`'daki `slave` alanına yaz. Slave adresi `machine_id` ile
> eşleşmek zorunda değildir. Etikete ikisini de yaz.

### 5.4 Andon butonları

**Butonları Raspberry Pi GPIO'ya bağla** (`gpiozero`). Bedava, PLC değişikliği yok. Fiziksel
ışık kulesini de aynı GPIO'dan bir röle kartı üzerinden sür.

Bu önemli bir sadeleşme: **projenin tüm Andon yarısı, tek bir üretim PLC programına
dokunmadan kurulabilir.** Önce onu yap.

> **Buton donanımı detayları:**
> - `gpiozero.Button(pin, pull_up=True, bounce_time=0.05)` — debounce yazılımda.
> - Sahadan gelen her kabloyu **optokuplör veya röle kartı** üzerinden geçir; 24 V'u doğrudan
>   3.3 V GPIO'ya bağlama.
> - Aynı butondan 30 sn içinde gelen ikinci çağrı yeni kayıt açmaz. 3 sn basılı tutmak =
>   çağrıyı iptal et. İptaller de loglanır (§12.3).

### 5.5 Duruş uyarısı

DVP'ye download STOP modu gerektirir. Sayaç rungları eklemek **13 makinede planlı duruş**
demektir. Bu teknik olmaktan çok bir planlama problemi ve gecikmenin en olası kaynağı.

**2026-09-02 iyileşmesi:** Karar #28 sayesinde *seri ayarı için* duruş gerekmiyor. Duruş
ihtiyacı artık sadece **sayaç rungu eklenecek makineler** için geçerli — ve mevcut C sayaçları
kullanılabilirse o da düşebilir.

Alternatif: mevcut C sayaçlarını oku ve sıfırlamaları sunucuda ele al, ya da Grup 4 harici
sensörlerini kullan.

**Sıra disiplini: yazmadan önce oku.** Önce mevcut bir D register'ı oku ve adres haritasını
teyit et. *Ancak ondan sonra* sayaç rungları yükle. Doğrulanmamış bir adrese üretim PLC'sinde
yazmak, bu projedeki en pahalı hatadır.

---

## 6. Yazılım yığını

| Katman | Seçim | Not |
|---|---|---|
| Collector | Python + `pymodbus`, YAML config, **sürücü mimarisi** | Yeni makine = config satırı; yeni PLC ailesi = yeni sürücü dosyası |
| GPIO | `gpiozero` (`lgpio` üzerinden) | Pi 5'te zorunlu |
| Yerel tampon | Pi üzerinde store-and-forward kuyruğu | Replay'de idempotent olmalı |
| S7 okuma | `python-snap7`, **sunucuda çalışır** | OPC UA daha temiz ama lisans gerektirir |
| Taşıma | MQTT (Mosquitto) + JSON | Sparkplug B şimdilik atlanıyor |
| Broker | **Sunucu, Docker içinde** | Karar #13 |
| Veritabanı | **TimescaleDB** | Telemetri için hypertable, master için ilişkisel tablo |
| Canlı dashboard | **Grafana** | §6.4 |
| Geriye dönük analiz | **Power BI** | Sadece aggregate. §6.4 |
| Makine UI | Küçük özel web app | Grafana doküman gösteremez |
| Bildirim | Telegram bot + fiziksel ışık kuleleri | Karar #17 |
| Filo yönetimi | Ansible + tek golden image | Pazarlıksız |
| Konteyner | Docker + docker-compose | Sunucu tarafı |
| Log | Loki veya merkezî rsyslog | Read-only root nedeniyle zorunlu |
| Zaman | **Sunucudan NTP** | İlk gün kur. Saat kayması tüm veriyi değersiz kılar |

**Pinlenen sürüm:** `pymodbus==3.6.9`. 3.x içinde API kırılmaları oldu; sürüm sabitlenmeden
filoya çıkılmaz.

### 6.1 Poll ve saklama oranları

**1 Hz** poll et. Durum bilgisini değişimde, sürekli değerleri düşük çözünürlükte sakla.

### 6.2 Ölçek gerçeklik kontrolü

16 makine × ~20 değer × 1 Hz ≈ **günde 1,4 M satır**, ~75 GB/yıl sıkıştırılmamış →
**TimescaleDB ile 5–7 GB/yıl.** Hat sayısı arttıkça doğrusal büyür ve yine sorun değildir:
100 makine bile yılda ~35–45 GB.

### 6.3 Doküman render optimizasyonu — taşıyıcı unsur

**Dokümanları sunucu tarafında WebP'ye render et.** Bu:
- En büyük bellek sıçramasını kaldırır
- Görüntülemeyi anlık yapar
- 16 istasyonda tutarlı render sağlar
- Dokunmatiğe uygun zoom/pan yazmana izin verir

**2 GB donanım kararı buna bağlı (§4.2).** Katı mimari gereksinim.

Uygulama: `pdftoppm` (poppler) → PNG → Pillow → WebP. Sayfa başına iki varyant:

| Varyant | Genişlik | Decoded boyut |
|---|---|---|
| `screen` | 1920 px | ~10 MB |
| `zoom` | 3840 px | ~41 MB |

**3840 px'i bellek bütçesini yeniden yapmadan aşma.** Çıktı yolu
`/<motor_kodu>/<versiyon>/pNNN_<varyant>.webp` — §8.2'deki versiyonlama gereksinimini de
karşılar.

### 6.4 Grafana vs Power BI

| Katman | Araç | Neden |
|---|---|---|
| Makine ekranı | Özel web app | Doküman + Andon butonları |
| Andon panosu | **Grafana kiosk** | Yerel, ücretsiz, sağlam kiosk, saniye seviyesi yenileme |
| Mühendislik | **Grafana** | Ham veriye hızlı bakış |
| Yönetim raporlama | **Power BI** | Keşifsel dilimleme, yönetimin konuştuğu dil |

**Saha ekranlarında neden Power BI değil:** gerçekten canlı değil, kiosk modu kırılgan, ekran
başına lisans sorusu var, bulut barındırmalı — internet kesintisi Andon panosunu öldürür.

**Katı kısıt: Power BI asla ham telemetriye bağlanmaz.** Sadece aggregate katmanı okur.
Sebep: sorgu performansı ve **tanım kayması** — OEE mantığı DAX'ta yeniden yazılırsa iki
farklı OEE sayısı çıkar. Tanım SQL'de, tek yerde yaşar.

**Zamanlama:** Power BI Faz 5. Faz 1'in ihtiyacı tek bir Grafana paneli.

---

## 7. Veri mühendisliği kapsamı

### 7.1 Açıkça gerekmeyen araçlar

**Snowflake, Databricks, Spark, Kafka, Airflow, AWS.** Bunlar 100–1000× daha büyük veri
setlerini hedefler.

**Bulut burada bir erişilebilirlik riskidir:** internet giderse Andon butonu yine bakımı
çağırabilmelidir. Her şeyi yerinde tut.

**İki istisna:**
- **dbt.** 6. aydan sonra, OEE mantığı karmaşıklaşınca. Onunla başlama.
- **Power BI.** Sadece geriye dönük yönetim raporlaması, aggregate tablolar üzerinde.

> **Telegram da buluta değiyor ve bu bir çelişki.** Çözüm katmanlamak:
>
> | Katman | Mekanizma | İnternet gerekir mi |
> |---|---|---|
> | 1 (birincil) | Makinedeki fiziksel ışık kulesi + korna | Hayır |
> | 2 | Tavandaki Andon panosu (Grafana kiosk) | Hayır |
> | 3 | Yerel ağ bildirimi (web app / PWA) | Hayır |
> | 4 | Telegram | **Evet** |
>
> Kural: **Telegram bir kolaylık katmanıdır, çalışma şartı değil.** Üretim verisinin Telegram
> üzerinden dışarı çıkması bir veri politikası konusudur.

### 7.2 Öğrenme planı

**Gerçek boşluk Pi değil, Linux'tur.** Öğrenilecekler, öncelik sırasıyla:

| Sıra | Konu | Neden lazım | Süre |
|---|---|---|---|
| 1 | **Linux temelleri** — SSH, dosya sistemi, izinler, `systemctl`, `journalctl`, `apt`, `nano` | Her şey bunun üstünde | 1–2 hafta |
| 2 | **Python servis zanaatı** — fonksiyon, `try/except`, `logging`, YAML, döngü + `sleep` | Collector, Andon servisi, ingest | 1–2 hafta |
| 3 | **Git** | Kod, config ve PLC yedekleri | 2–3 gün |
| 4 | **SQL, window fonksiyonları** — `LAG()`, gaps-and-islands | **OEE hesabının tamamı budur.** En yüksek getirili kalem | 3–4 hafta |
| 5 | **Docker + compose** | Sunucudaki her şey konteynerde | 1 hafta |
| 6 | **MQTT** | Pi ↔ sunucu haberleşmesi | 2–3 gün |
| 7 | **TimescaleDB** | hypertable, continuous aggregate, sıkıştırma | 1 hafta |
| 8 | **Ansible** | Faz 2'ye kadar ertelenebilir | 1 hafta |
| 9 | **Grafana** | En sona | 2–3 gün |

1–3 Faz 0'da, 4–7 Faz 1 ile iç içe, 8–9 sonra.

**Öğrenmeye gerek olmayanlar:** Kubernetes, Kafka, Spark, Airflow, dbt (6. aya kadar), bulut
sertifikaları, CI/CD, ileri Python.

**Öğrenme yöntemi:** her konuyu **bu projedeki gerçek bir görev üzerinden** öğren.

> **Somut kontrol listesi (kurs yerine):**
> 1. İmajlama + SSH key ile bağlanma, parola girişi kapalı
> 2. `nano` / `apt` / `journalctl` akıcılığı
> 3. **Bir Python scriptini `systemd` servisine dönüştürme**
> 4. `gpiozero` ile buton okuma
> 5. USB-RS485 için udev kuralı
> 6. Git temel akışı
>
> **Madde 3 tek başına, piyasadaki Pi kurslarının tamamından daha değerlidir.**
>
> **Uygulama planı:** `andon-ogrenme-yolu.md`. Aşamalar, ölçümler ve çalışma kuralları orada.
> Ders notları: `andon-notes/ders-notlari/` (16 dosya).
>
> **Elenen kurs:** Udemy "Raspberry Pi For Beginners". Örtüşme ~%20; `systemd`, udev, overlay
> FS, Docker, MQTT ve SQL hiç yok. **Karar #23.**

### 7.3 Her aracı aşan kavramlar

- **Idempotency** — yeniden işleme asla kayıt çoğaltmamalı
- **Geç gelen veri** — altı saatlik tampon bir anda boşalır; pipeline bunu doğru ele almalı
- **Şema evrimi**
- **Doğal anahtar vs vekil anahtar (surrogate key)** — kimlik değişebilen bir iş bilgisine
  bağlanmaz. §12.4'ün tamamı bunun uygulanmasıdır
- **Yavaş değişen boyut (slowly changing dimension)** — "o tarihte hangi hattaydı" sorusu
  cevaplanabilmeli (§12.5)
- **Katmanlı model** — raw → cleaned → aggregated

Daha büyük ölçekte zorlananlar Databricks bilmeyenler değildir; idempotency'yi
içselleştirmemiş olanlardır.

---

## 8. Açık sorular ve boşluklar

### 8.1 Hâlâ gereken tanımlar — anlamlı OEE'yi bloke ediyor

- [ ] **Vardiya takvimi ve planlı duruş.** Availability = çalışma süresi ÷ *planlı* üretim
      süresi. Bu tablo olmadan availability anlamsızdır.
- [ ] **Motor kodu × makine başına ideal çevrim süresi.** Bu olmadan Performance ölçülemez.
- [ ] **Hurda kaydı.** Kimse girmiyorsa Quality kalıcı %100 olur ve OEE kurgudur.
- [ ] **Duruş sebep kodları.** "4 dakika durdu — sebep seç." Bu olmadan Pareto yoktur.
- [ ] **OEE'nin resmi tanımı yazılı olarak yönetimle mutabık.** Planlı bakım availability'yi
      düşürür mü, setup duruş mu sayılır, ısınma fireleri hurdaya girer mi.
- [ ] **OEE hangi seviyede raporlanacak** — makine, hat, ürün? 16 makinenin ortalaması hat
      OEE'si **değildir**. **Hat bazlı raporlama §12.5'teki atama tablosunu zorunlu kılar.**
- [ ] **Veri saklama politikası.** Ham telemetri 12–18 ay (öneri), aggregate süresiz.
- [ ] **Sayım doğrulaması.** İlk 2 hafta her vardiya sonu sistem vs elle sayım.
- [ ] **Mevcut durumun baseline'ı.** Sistem devreye girmeden önce 1 hafta Andon cevap
      sürelerini **elle** ölç. Bu ölçüm projenin politik sermayesidir.

### 8.2 Operasyonel boşluklar

- [ ] **Andon eskalasyon kuralları.**
- [ ] **Nöbet listesi.** Bildirim şu an vardiyada olana gitmeli.
- [ ] **Yedekleme ve geri yükleme — test edilmiş.** Test edilmemiş yedek, yedek değildir.
- [ ] **Ana panoda UPS.**
- [ ] **Ağ segmentasyonu.** Makine VLAN'ı, ofis ağına firewall (§3.6).
- [ ] **Sunucu tek arıza noktası.** Azaltıcılar: UPS, test edilmiş geri yükleme, Pi tarafında
      tamponlama.
- [ ] **Sunucu kaç tane olacak?** Çok fabrikaya yayılınca her fabrikanın kendi sunucusu mu?
      §7.1 **fabrika başına yerel sunucu** yönünde baskı yapıyor. Faz 3'ten önce karara bağla.
- [ ] **Pi arıza davranışı.** Operatör Andon butonunu kaybeder. Kablolu yedek yol veya
      fiziksel korna düşün.
- [ ] **Doküman versiyonlama ve denetim.**
- [ ] **Değişiklik yönetimi.** 13 üretim PLC programını değiştirmek — kim onaylar, geri dönüş?
- [ ] **MTTA ve MTTR'ın tam tanımı.** İkisini de ayrı sakla, raporda hangisi olduğunu yaz.
- [ ] **Çağrıyı kim kapatır?** Öneri: bakımcı kendi badge'iyle makinede kapatır.
- [ ] **Yanlış/kazara çağrı politikası.** İptaller istatistiğe girer mi?
- [ ] **Sunucu donanım yedeği.** Geri yükleme yılda en az bir kez gerçekten denensin.
- [ ] **Ekran yönetimi.** Uyku ve ekran koruyucu kapalı; OLED değil LCD.

### 8.3 Projeyi öldürmesi en olası boşluk

**Operatör kabulü.** 16 operatör bunu gözetleme olarak algılarsa sebep kodlarını oyunlaştırır
ve butonları görmezden gelir — bunu hiçbir mimari kurtarmaz.

**Karşı önlem:** ilk günden itibaren *onlar için* apaçık faydalı olsun.

> 1. **Kurmadan önce konuş.** Kıdemli 2–3 operatöre buton konumunu ve sebep kodu listesini
>    sordur.
> 2. **Yazılı taahhüt: bireysel performans değerlendirmesinde kullanılmaz.** Alamıyorsan badge
>    login'i ertele.
> 3. **Veriyi onlara da göster.**
> 4. **İlk hafta hızlı bir kazanım seç.**
> 5. **Sebep kodu listesini kısa tut.** 8–10 seçenek.
> 6. **Ekranlarda makinenin kendi adı ve kodu yazsın** (`display_name` / `asset_code`), iç
>    kimlik (`M001`) değil.

---

## 9. Fazlı plan

Gerçekçi olarak, bu tek işin değilse **5–7 ay**; tam zamanlı ~4 ay. *(Pilot hat için; sonraki
hatlar çok daha hızlı — kod ve golden image hazır olacak.)*

| Faz | Süre | İçerik |
|---|---|---|
| **0. Tanımlar & envanter** | 2–3 hafta | §8.1 tanımları. **[DOĞRULA]** maddeleri. PLC programlarını git'e yedekle. Pano sıcaklıkları. HMI port atamaları. ~~IT'ye sor~~ **tamamlandı** |
| **1. Pilot, tek makine** | 3–4 hafta | §9.1. Çirkin olması kabul edilebilir |
| **2. Sağlamlaştır & 5 makineye çoğalt** | 3–4 hafta | Golden image, Ansible, merkezî log, heartbeat |
| **3. Kalan makineler + S7/HMI** | 4–6 hafta | Grup 2–4. Motor kodu yayını, doküman gösterimi |
| **4. Andon iş akışı** | 3–4 hafta | Eskalasyon, nöbet listesi, bildirimler, tepki süresi raporlaması |
| **5. Dashboard & cila** | sürekli | Grafana OEE görünümleri, duruş Pareto'su. **Power BI burada başlar** |

**6–8. haftadaki pilot, projeyi bitirmeni sağlayacak politik sermayeyi satın alan şeydir.**

**Hızlı güven kazancı:** AS218TX'te Ethernet varsa, hiçbir PLC değişikliği olmadan bir saat
içinde okunabilir.

> **Faz 0'a eklenecekler:**
> - Mevcut Andon tepki sürelerinin elle baseline ölçümü (§8.1)
> - Kıdemli operatörlerle tasarım görüşmesi (§8.3)
> - İK/hukuk ile KVKK ön görüşmesi, badge login yapılacaksa (§13.2)
> - Üretim planlama ile sayaç rungu eklenecek makinelerin duruş takvimi (§5.5)
> - **Yetenek matrisinin doldurulması (§5.1)**
> - **Her makinede `scan_dvp.py` çalıştırılması** (sadece okur) ve bulunan seri ayarının
>   `machines.yaml`'a yazılması (§5.3, Karar #28)
> - **Makine envanter listesi:** `asset_code`, `display_name`, `site`, `area`, `line` —
>   kodsuz makineler için kod talebi (§11 madde 13)
> - **Linux/Pi temel öğrenme bloğu (§7.2):** takibi `andon-ogrenme-yolu.md`.
>   **Başladı: 2026-08-27, Aşama 1 tamam.**

### 9.1 Faz 1 görev sırası

Her adım bir öncekinin kanıtlanmış olmasına bağlıdır ve E'den önce hiçbir şey üretime dokunmaz.

**A. Bench kurulumu.** Raspberry Pi OS **with desktop**, 64-bit. §7.2'nin 1–3 kalemleri ve
altı maddelik kontrol listesi burada tamamlanır. SSH key kur, parola girişini kapat.
**Overlay FS'i henüz açma** — geliştirmeyi engeller; Faz 2'de golden image kurarken aç.

> **Kod nerede yazılır:** PC'de, `andon-collector` repo'sunda (§4.7, Karar #27). Pi bu repo'yu
> `git pull` veya `scp` ile alır. Pi üzerinde kalıcı düzenleme yapılmaz.

> **IP adresleme:** statik IP değil, **router'da MAC'e DHCP rezervasyonu** (§12.4).

*Not:* **B–E adımlarının hiçbiri imaj seçimine bağlı değildir.**

**B. USB-RS485 adaptörünü bench'te kanıtla.** **Bir udev kuralı yaz**: adaptörün seri
numarasına sabit isim (`/dev/andon-rs485`) bağla — `/dev/ttyUSB0` reboot'lar arasında sabit
değildir.

**C. Önce Andon yarısını kur.** Bir buton, bir röle, `gpiozero`. Buton → MQTT → Telegram.
~50 satır. PLC erişimi gerektirmez. Pratikte Karar #12.

**D. Sunucu tarafını ayağa kaldır.** Docker: Mosquitto + TimescaleDB. PC'de veya Pi #2'de.

**E. Ancak şimdi makineye git.** Bir Grup 1 makinesi seç. Adaptörü bağla → `scan_dvp.py` ile
seri ayarını bul (**yazma yok**) → **mevcut bir D register'ı oku** → adres haritasını doğrula →
`machines.yaml`'a yaz → *ondan sonra* gerekiyorsa sayaç rungları ekle (§5.5).

> **[DOĞRULANDI] Bench'te E adımının provası yapıldı — 2026-09-02.** `scan_dvp.py` seri ayarını
> buldu (ASCII 9600 7-E-1, slave 1), `read_test.py` adres tabanını ve word sırasını doğruladı.
> Üretim makinesinde aynı sıra izlenecek.

> **F. Pilot kabul kriterleri — bunlar sağlanmadan Faz 2'ye geçme.** Filo siparişi buna bağlı.
> **Testler Pi #2 (`andon-pilot`) üzerinde koşar (§4.2).**
> - [ ] 7 gün kesintisiz çalışma, elle müdahale olmadan
> - [ ] Chromium RSS 7 gün boyunca 1,2 GB'ı aşmadı (en büyük çizim açıkken)
> - [ ] Ağ kablosu 30 dk çekildi → **hiç veri kaybı ve hiç kopya kayıt yok**
> - [ ] Güç 10 kez ani kesildi → her seferinde temiz açılış
> - [ ] Vardiya sonu sistem sayımı ile elle sayım **birebir** tuttu (3 vardiya üst üste)
> - [ ] Andon butonu → Telegram gecikmesi < 5 sn
> - [ ] `vcgencmd get_throttled` = 0, en sıcak günde pano içinde
> - [ ] **Bellek testleri filoya gidecek imaj üzerinde çalıştırıldı**
> - [ ] Golden image'da `swapon --show` kontrol edildi (§4.3)
> - [ ] Golden image'da Raspberry Pi Connect kurulu değil / devre dışı (Karar #25)
> - [ ] Pi'de git yok veya sadece read-only deploy key ile pull yapıyor (Karar #27)
> - [ ] **§4.6 filo imajı kararı ve Karar #6 (RAM) yazılı olarak kapatıldı**

---

## 10. Karar günlüğü

| # | Karar | Gerekçe | Durum |
|---|---|---|---|
| 1 | Telemetri S7-1500 üzerinden **geçmez** | S7 sunucunun yerine değil önüne geçiyor; dayanıklılık kazancı olmadan 1.800–2.800 € ve ~1 ay ekliyor. Çok hat senaryosunda maliyet her hatta tekrarlanırdı | **Kesin — 2026-08-29 itibarıyla koşulsuz.** IT onayı geldi, geri dönüş şartı ortadan kalktı (§3.6) |
| 2 | Makine başına bir Pi: toplama + gösterim + Andon GPIO | Monitörler için hesaplama cihazı zaten gerekli; veri toplama makine başına 10 €'ya iniyor | Kesin |
| 3 | Serbest akan 32-bit sayaçlar, asla sıfırlanmaz | Kesintiler karşısında kendi kendini iyileştirir; parça kaybolmaz | Kesin |
| 4 | Register adresleri makine bazlı; dondurulan şey ortak veri modeli | PLC parkı karışık. Adres eşlemesi YAML'a taşındı; alan isimleri ve `state` enum'ı dondurulmuş kalır (§5.1) | **Revize** |
| 5 | ~~COM2 her yerde Modbus RTU'ya standardize~~ | **Karar #28 ile değiştirildi (2026-09-02).** Fabrika ayarı zaten çalışıyor; 13 PLC'ye yazmanın bedeli kazancından büyük | **Geçersiz — bkz. #28** |
| 6 | **Raspberry Pi 5, 2 GB** | §6.3 WebP kararı 4 GB gerekçesini geçersiz kıldı. 2026-08-27 ölçümü destekliyor: idle 450 MB used / 1555 MB available. Kesinleşmesi §9.1 F testine ve §4.6'ya bağlı | **Revize — koşullu; ölçüm lehte** |
| 7 | Read-only root + endurance SD, makine başına SSD yok | SSD güç kesintisi bozulmasını çözmez; read-only ikisini de bedavaya çözer. Swap zram üzerinde, SD'ye yazmıyor (§4.3) | **Revize** |
| 8 | Andon butonları Pi GPIO'da, PLC DI'da değil | DVP-SS2 I/O dar; Andon yarısını PLC işinden bağımsızlaştırır | **Revize** |
| 9 | Operatör login makinede, KTP1200'de değil | Basic panel kısıtları; operatör zaten makinenin başında | Kesin |
| 10 | Sadece yerinde; bulut veri ambarı yok | Ölçek gerektirmiyor; internet kesintisi Andon'u durdurmamalı | Kesin |
| 11 | S7 sipariş/motor kodu/hat durumu rolünü korur | Donanımın doğru kullanımı | Kesin |
| 12 | Andon yarısı OEE yarısından önce | Operatör kabulü bir numaralı risk | Kesin |
| 13 | **IOT2050 kaldırıldı** | Dört görevinin de daha iyi bir yeri var. Ödünleşim: sunucu daha sert bir tek arıza noktası (§8.2) | Kesin |
| 14 | Canlı için Grafana, geriye dönük raporlama için Power BI | Farklı işler. Power BI sadece aggregate okur; OEE tanımı SQL'de tek yerde kalır. Faz 5 | Kesin |
| 15 | Sadece Linux; Pi üzerinde Windows reddedildi | GPIO yok, 2 GB'a sığmaz, read-only root yok, Ansible/Docker/systemd yok | Kesin |
| 16 | Geliştirme Pi 5 üzerinde; Pi 3 prototip aşaması yok | Bir taşıma adımını kaldırır | **Karar #27 ile kısmen değişti** |
| 17 | Bildirim katmanlı: fiziksel ışık kulesi ve yerel bildirim birincil, Telegram ikincil | Telegram internet gerektirir; §7.1 ile çelişir | **Yeni — onay bekliyor** |
| 18 | Idempotency anahtarı: `machine_id + boot_id + seq`; UNIQUE + ON CONFLICT DO NOTHING | Store-and-forward replay'i kopya üretmemeli (§12.2) | **Yeni — onay bekliyor** |
| 19 | Tüm zaman damgaları UTC (`timestamptz`); gösterimde Europe/Istanbul | Vardiya sınırı hesapları, geç gelen veri, olası mevzuat değişikliği | **Yeni — onay bekliyor** |
| 20 | MQTT kimlik doğrulama zorunlu: Pi başına kullanıcı/parola + topic ACL | VLAN tek savunma katmanı olamaz (§13.1) | **Yeni — onay bekliyor** |
| 21 | Sistem bireysel performans değerlendirmesinde kullanılmaz — yazılı taahhüt | §8.3 ve §13.2'nin ortak gereği | **Yeni — onay bekliyor** |
| 22 | Collector PLC/protokol bağımsız, sürücü mimarisiyle yazılır | Makine parkı karışık ve büyüyebilir (§2) | **Yeni** |
| 23 | "Raspberry Pi eğitimi" alınmaz; Linux + SQL öğrenilir | Piyasadaki Pi kursları hobi odaklı (§7.2) | **Yeni** |
| 24 | Bench Desktop imajla; filo golden image seçimi AÇIK | §9.1 F testi karar verir. **Karar tarihi: filo siparişinden önce** | **Açık — Faz 1 sonunda** |
| 25 | Raspberry Pi Connect kullanılmaz; uzaktan erişim sadece VLAN içi SSH key ile | Dışarıya kalıcı bağlantı açar, vendor relay'i üzerinden geçebilir, erişimi şahsi hesaba bağlar. LAN'da SSH zaten var. Lite imajda varsayılan kurulu gelir; golden image'da kaldırılacak | **Kesin** |
| 26 | **Kimlik şeması: `machine_id` anlamsız ve kalıcı; fabrika kodu, ad, hat, bölüm birer özellik** | Tesis gerçeği: çok fabrika, çok bölüm, makine bazlı yayılma (§2). Hat kimliğe konamaz — makineler hat değiştirir. Fabrika kodu da kimlik olamaz: bazı makinelerin kodu yok ve kodlar düzeltilebilir; kimlik değişirse geçmiş kopar. `machine_id` = `M001`; `asset_code`, `display_name`, `site`, `area`, `line` ayrı alanlar; yerleşim tarihli atama tablosunda (§12.5). MQTT topic'inden hat çıkarıldı. **Ekranlarda `asset_code`/`display_name` yazar** | **Yeni — onay bekliyor** |
| 27 | **Geliştirme PC merkezli: kod PC'de yazılır → GitHub (private) → Pi `git pull` ile alır** | *Revizyon (Karar #16'nın "Pi üzerinde geliştir" kısmı):* Tek Pi varken doğruydu; iki Pi ve bir PC varken repo'nun yeri geliştirme makinesidir. Üretimdeki Pi'lerde git bulunmaz ve elle düzenleme yapılmaz — read-only root bunu zaten zorunlu kılacak. Pi'ler için **read-only deploy key**; Faz 2'de yerini Ansible alır. GPIO/seri testleri Pi'de kalır (donanım orada). Repo'lar private; şahsi hesap geçici, firma organizasyonuna taşınacak (§4.7, §12.4, §15) | **Yeni — uygulandı 2026-08-29** |
| 28 | **[YENİ] COM2 fabrika ayarında bırakılır; seri ayarı PLC'ye değil `machines.yaml`'a yazılır** | *Karar #5'in yerine geçer.* 2026-09-02 bench ölçümü: DVP-SS2 COM2 fabrika varsayılanı (ASCII 9600 7-E-1, slave 1) hiçbir PLC değişikliği olmadan cevap veriyor. 1 Hz poll'de ASCII 9600'ün maliyeti ~30–50 ms; bütçe 1000 ms — hız gerekçesi yok. Standardizasyonun kazancı config'de dört satır; bedeli 13 planlı duruş (§5.5, projenin en olası gecikme sebebi) ve doğrulanmamış `D1120`/`M1143` bit anlamlarına yazma riski. Collector zaten çok protokollü (Karar #22), yani karmaşa kaldırılmıyor sadece yeri değişiyor. **İstisna:** bir makinede fabrika ayarı çalışmazsa veya multi-drop RS-485 gerekirse, sadece o makinelerde ayar değiştirilir | **Yeni — uygulandı 2026-09-02** |

---

## 11. Acil sonraki adımlar

1. Pilot hattın 16 PLC programının tamamını git'e yedekle — **bu hafta**
2. Hattı yürü ve makine başına kaydet: PLC modeli, HMI'nin hangi COM portunda olduğu, diğer
   portta ne olduğu, baud/format, pano sıcaklığı. **`scan_dvp.py` seri ayar kısmını otomatik
   yapar**
3. ~~IT'ye fabrika ağında Linux izni sor~~ — **KAPANDI 2026-08-29: izin verildi (§3.6)**
4. ~~DVP-SS2 COM2 yapılandırma register'larını manuelden doğrula~~ — **önceliği düştü.**
   Karar #28 gereği bu register'lara yazma planı yok. Adresleri vendor tablosundan alındı
   (§5.2); bit anlamları sadece istisnai bir makinede gerekirse doğrulanacak
5. ~~DVP-SS2 Modbus cihaz adres base'lerini manuelden doğrula~~ — **KAPANDI 2026-09-02:**
   vendor adres tablosu + bench okuması ile doğrulandı (§5.2)
6. AS218TX'te dahili Ethernet olup olmadığını teyit et. **AS serisi Modbus adres tablosu
   henüz elde yok — Delta'dan temin et**
7. §8.1 tanım görüşmelerini başlat — en uzun süren ve en çok şeyi bloke eden bunlar
8. ~~Pilot bench donanımını sipariş et~~ — **tamamlandı (2026-08-27), iki Pi elde**
9. Mevcut Andon tepki sürelerinin **elle baseline ölçümü**ne bu hafta başla (§8.1)
10. Badge login yapılacaksa İK/hukuk ile KVKK ön görüşmesi (§13.2)
11. Üretim planlama ile sayaç rungu eklenecek makinelerin duruş penceresini aç (§5.5)
12. Kıdemli operatörlerle sebep kodu listesi ve buton konumu üzerine konuş (§8.3)
13. **Kodsuz makineler için envanter kodu talep et.** `machine_id` anlamsız olduğu için sistem
    kodsuz da çalışır, ama `asset_code` boş kalan makine ekranlarda bakımın tanımadığı bir
    isimle görünür. Faz 0 içinde çözülmeli
14. **Pi #2'nin RAM'ini teyit et** (2 GB mı?) — §4.2'deki paralel imaj ölçümü buna bağlı
15. **GitHub'a ilk push'u yap** — `andon-collector` ve `andon-notes` private repo'ları açıldı
    ve bağlandı; commit'ler henüz push edilmedi. Pi için read-only deploy key üret (§4.7, §12.4)
16. **`state` register adresini belirle ve doğrula.** `parts_total` (D300, 0x112C) doğrulandı;
    `state` için hangi D register'ının kullanılacağı henüz kararlaştırılmadı (§5.1 YAML'da
    `TODO: VERIFY`)

---

## 12. [YENİ] Veri sözleşmeleri

§5.1'deki register haritası PLC ile Pi arasındaki sözleşmedir; bu bölüm Pi ile sunucu
arasındaki sözleşmedir.

### 12.1 MQTT topic şeması

```
andon/makine/<machine_id>/telemetry     QoS 1, retained=false
andon/makine/<machine_id>/state         QoS 1, retained=true
andon/makine/<machine_id>/andon         QoS 1, retained=false
andon/makine/<machine_id>/heartbeat     QoS 1, retained=true  (LWT: "offline")
andon/sunucu/motor_kodu                 QoS 1, retained=true  (aşağı yön)
andon/sunucu/komut/<machine_id>         QoS 1, retained=false (aşağı yön)
```

**Eski şema `fabrika/hat1/makine/...` idi ve yanlıştı:** hat topic'e gömülüyse bir makine hat
değiştirdiğinde topic de değişir, geçmiş kopar, abonelikler bozulur. Hat, bölüm ve fabrika
bilgisi veritabanında yaşar (§12.5).

Kurallar:
- **QoS 1 her yerde.** Kopya üretebilir — bunu idempotency ile çözüyoruz (§12.2).
- **`clean_session = False`** ve sabit `client_id = machine_id`.
- **Retained sadece durum bilgisinde.** Telemetride retained kullanma.
- **LWT ile filo sağlığı.**
- **Topic'te sadece ASCII, boşluksuz, `+` ve `#` içermeyen değerler.**

### 12.2 Mesaj şeması ve idempotency

```json
{
  "msg_id":    "M007-8f2a1c-000148213",
  "machine_id":"M007",
  "boot_id":   "8f2a1c",
  "seq":       148213,
  "ts":        "2026-08-25T09:14:03.000Z",
  "ts_synced": true,
  "counters":  { "total": 1048576, "scrap": 312 },
  "state":     2,
  "fault":     0,
  "cycle_ms":  4120
}
```

- **`msg_id` = `machine_id + boot_id + seq`.** DB'de `UNIQUE(msg_id)`, ingest'te
  `INSERT ... ON CONFLICT DO NOTHING`. **Karar #18.**
- **`ts` Pi'nin ürettiği zaman, UTC.** Sunucu ayrıca `received_at` yazar; ikisini de sakla.
- **`ts_synced`** — chrony senkron değilken üretilen kayıtlar işaretlenir.
- **Her şey UTC, `timestamptz`.** Gösterim katmanı `Europe/Istanbul`'a çevirir. **Karar #19.**
- **Mesajda hat/bölüm/fabrika alanı yoktur.**

### 12.3 Andon olay durum makinesi

```
  ACIK ──onaylandi──▶ ONAYLANDI ──cozuldu──▶ COZULDU
    │                     │
    ├──iptal──▶ IPTAL     └──eskalasyon──▶ ESKALE ──▶ (ONAYLANDI'ya döner)
    │
    └──N dk onay yok──▶ ESKALE
```

Zaman damgaları: `acilis_ts`, `onay_ts`, `cozum_ts`, `iptal_ts`, `eskalasyon_ts[]`.
Kimlikler: `acan_operator`, `onaylayan_kisi`, `kapatan_kisi`.

- **MTTA** = `onay_ts − acilis_ts`
- **MTTR (dar)** = `cozum_ts − onay_ts`
- **MTTR (geniş)** = `cozum_ts − acilis_ts` ← operatörün gerçekten yaşadığı süre

**İkisini de hesapla ve raporda hangisi olduğunu yaz.**

**[ÖNERİ]** Onay ikiye ayrılabilir: Telegram'dan onay = "yola çıktım"; makinede badge okutma =
"geldim". Çok daha dürüst bir MTTA verir.

### 12.4 İsimlendirme ve kimlik şeması

**İlke: kimlik anlamsızdır, anlam bir özelliktir.**

| Nesne | Format | Örnek | Kural |
|---|---|---|---|
| **`machine_id`** | `M001`–`M999` | `M007` | **Anlamsız, kalıcı, asla yeniden kullanılmaz.** Fabrika, hat, bölüm değişse de aynı kalır |
| `asset_code` | Fabrikanın formatı | `MAK25-30-511` | Envanter kodu. **Boş olabilir**, sonradan doldurulur, düzeltilebilir |
| `display_name` | Serbest metin | `4 INC Pres 2` | Ekranlarda görünen ad |
| `site` / `area` / `line` | Kısa kod | `FAB1` / `MONTAJ` / `4INC` | **Zamanla değişir** → geçmişi §12.5'te |
| Hostname | `andon-m007` | | `machine_id` ile eşleşir |
| Modbus slave adresi | 1–247 | 1 | Fabrika varsayılanı 1; her makinenin kendi hattı olduğu için değiştirilmez (§5.3) |
| IP | DHCP rezervasyonu, MAC'e sabitli | | Tek yerden yönetilir, yedek Pi takarken IP elle ayarlanmaz |
| Doküman yolu | `/<motor_kodu>/<versiyon>/pNNN_<varyant>.webp` | | §6.3 |

**Gösterim kuralı:** operatör, bakımcı ve yönetim hiçbir ekranda `M007` görmez. Ekranlarda
`asset_code` ve `display_name` yazar.

**Git repo yapısı — hat sayısından bağımsız:**

```
andon-collector/          kod + config + tools/. Hat/fabrika kavramı yok
andon-notes/              CALISMA-GUNLUGU.md, KURULUM-ADIMLARI.md, ders-notlari/,
                          andon-proje-brifi.md (çalışma kopyası)
vendor-docs/              Delta Modbus adres tabloları — §5.2'nin kaynağı
andon-webapp/             kod (ileride)
andon-ansible/            dağıtım + envanter (Faz 2)
  ├─ playbooks/           ortak
  ├─ inventory/
  │   ├─ hat-4inc/        machines.yaml, IP/MAC listesi
  │   └─ hat-3inc/        ileride: sadece yeni klasör
  └─ group_vars/
plc-programs/
  ├─ hat-4inc/
  └─ hat-3inc/
```

**Yeni hat için kod repo'su kopyalanmaz — sadece yeni bir envanter klasörü açılır.** Kodu
fork'lamak bir hafta sonra iki farklı yazılıma ve iki katına çıkmış bakım yüküne yol açar.

**Repo konumu ve erişim (Karar #27):**
- **Kalıcı kopya:** PC'de `C:\Users\byurtman\Desktop\andon\`, oradan GitHub'a (private)
- **Pi'ler:** sadece okuma. GitHub'da her Pi için **read-only deploy key**; push yetkisi yok
- **Faz 2'den itibaren** dağıtımı Ansible yapar, Pi'ler git'e hiç dokunmaz

**Fiziksel etiketleme:** her Pi'nin üzerinde `machine_id`, `asset_code`, IP, MAC ve Modbus
slave adresi yazan bir etiket olsun.

**Bench üniteleri:** Pi #1 `andon-bench` (kirli geliştirme), Pi #2 `andon-pilot` (temiz, kabul
testi). Kullanıcı: `andon`.

### 12.5 Makine master verisi ve yerleşim geçmişi

Telemetri tablosu sadece `machine_id` taşır. Makinenin *ne olduğu* ve *nerede olduğu* ayrı
ilişkisel tablolarda durur:

```
machine(machine_id PK, asset_code, display_name, plc_type, created_at, retired_at)

machine_assignment(machine_id FK, site, area, line, valid_from, valid_to)
```

**Neden atama ayrı ve tarihli:** bir makine 4 INC'ten 3 INC'e alındığında yeni satır açılır,
eskisi `valid_to` ile kapanır. *"Geçen ay 4 INC hattının OEE'si neydi"* sorusu **o ayki gerçek
makine listesiyle** cevaplanır.

Hat bilgisi tek sütun olsaydı, makine taşındığı gün geçmişteki tüm raporlar sessizce
değişirdi — kimse fark etmez, sonra sayılar tutmaz ve sistemin güvenilirliği tartışmaya
açılır. **Sonradan eklemesi zor, şimdi eklemesi bir tablo.**

**Kurallar:**
- Aynı makine için çakışan tarih aralığı olamaz (`valid_to` boş olan en fazla bir satır)
- `machine_id` asla değişmez; `asset_code` düzeltilebilir ve düzeltme geçmişi bozmaz
- Hat bazlı her rapor, ölçüm zamanına göre atama tablosuyla birleştirilir

---

## 13. [YENİ] Güvenlik, hukuk ve iş güvenliği

### 13.1 Bilgi güvenliği

- **MQTT:** her Pi için ayrı kullanıcı/parola, topic ACL. Anonim erişim kapalı. **Karar #20.**
  Parolalar Ansible Vault'ta.
- **TLS:** üretim VLAN'ında zorunlu değil ama Mosquitto'da açmak ucuz.
- **SSH:** sadece key, parola girişi ve root login kapalı.
- **Uzaktan erişim: vendor bulut tüneli kullanılmaz — Raspberry Pi Connect dahil.** Dışarıya
  kalıcı bağlantı açar ve gerektiğinde Raspberry Pi Ltd'nin relay sunucusu üzerinden geçer;
  §3.6 taahhüdünü ve §7.1'i ihlal eder, erişimi şahsi hesaba bağlayarak §15 bus factor
  disiplinini bozar. Tek yol: üretim VLAN'ı içinden SSH key. **Karar #25.**
  **Not:** Lite imajında shell-only varyantı varsayılan kuruludur; golden image'da kaldırılacak.
- **VNC:** yerel ağda kaldığı sürece politika ihlali değil, ama filo golden image'ında
  **kapalı** olacak — "gereksiz servis yok" taahhüdü ve daha önemlisi masaüstünden elle
  düzeltme yapmanın o üniteyi filodan farklılaştırması (§4.3). Bench'te geçici açılabilir.
- **Git erişimi:** üretimdeki Pi'ler repo'ya yazamaz — read-only deploy key (Karar #27).
  Repo'lar private.
- **Veritabanı rolleri:** collector/ingest sadece `INSERT`; Grafana sadece `SELECT`; Power BI
  sadece aggregate şemaya `SELECT`. Hiçbiri superuser değil.
- **Web app:** operatör ekranı kiosk'ta açık kalır; yönetim/mühendislik görünümleri login ister.
- **Secrets asla git'te olmaz.** `.env` + Ansible Vault; `.gitignore` ilk commit'te yazıldı.
- **Yedekler şifreli**, özellikle saha dışına çıkıyorsa.

### 13.2 KVKK — operatör verisi

Badge login eklendiği anda bu proje **kişisel veri işleyen bir sistem** olur.

- [ ] **İK ve hukuk ile görüş** — badge okuyucu satın almadan önce
- [ ] **Aydınlatma metni**
- [ ] **Amaç sınırlaması** — disiplin veya performans değerlendirmesi için kullanılmaz
      (**Karar #21**)
- [ ] **Saklama süresi** — operatör kimliğine bağlı kayıtlar için 6–12 ay, sonra anonimleştirme
- [ ] **Erişim sınırlaması**
- [ ] **Varsa sendika / işçi temsilcisi bilgilendirmesi**

**Tasarım prensibi:** raporların varsayılanı **makine bazlı** olsun, kişi bazlı değil.

### 13.3 Operatör güveni — teknik karşılıkları

- Makine ekranında operatör **kendi verisini görebilsin**
- Sebep kodu ekranında "Bilinmiyor" ve "Diğer" **her zaman olsun**
- Hiçbir ekranda operatör isimleri sıralanmış bir "performans" listesi olmasın
- Ekranlarda makinenin kendi adı/kodu yazsın, iç kimlik değil (§12.4)

### 13.4 İş güvenliği ve makine güvenliği

- **Emniyet devrelerine dokunulmaz.** Acil stop, emniyet rölesi, ışık bariyeri, kapı kilidi
  sinyallerine ne okuma ne yazma amaçlı bağlantı yapılır.
- **Pano içi çalışma** enerji kesilerek ve LOTO prosedürüyle yapılır.
- **24 V saha sinyalleri 3.3 V GPIO'ya doğrudan bağlanmaz** — optokuplör/röle zorunlu.
- **Makineye kalıcı bir ekleme yapıldığında** risk değerlendirmesi güncellenebilir; CE beyanı
  olan makinelerde özellikle. İSG uzmanına sor.
- **Ekran ve buton konumu** hareket alanını, görüş hattını veya kaçış yolunu engellememeli.
- **PLC programı değiştirilen her makinede** ilk çalıştırma yetkili gözetiminde, makine boşta.
- **Test PLC'si bench'te olmalı, hiçbir makineye bağlı olmamalı.** 2026-09-02 Modbus testleri
  bu kurala uyularak yapıldı.

---

## 14. [YENİ] Test ve devreye alma disiplini

### 14.1 Donanımsız geliştirme

`pymodbus` bir slave simülatörü içerir. §5.1 haritasını taklit eden bir simülatör yaz. Bununla
PLC'ye hiç dokunmadan geliştirilebilecekler: collector ve YAML config yapısı; ingest servisi,
veritabanı şeması, continuous aggregate'ler; store-and-forward kuyruğu ve replay; OEE SQL'i.

### 14.2 Mutlaka yazılacak testler

| Test | Nasıl | Neyi kanıtlar | Durum |
|---|---|---|---|
| Word sırası | PLC'ye asimetrik değer (D300=2, D301=1) yaz, tek istekte oku | §5.1 word sırası | **✅ 2026-09-02 — `lo_hi`.** `tools/read_test.py` |
| Adres tabanı | Bilinen bir D register'ı oku, beklenen değeri gör | §5.2 D tabanı 0x1000 | **✅ 2026-09-02** |
| Seri ayar taraması | `tools/scan_dvp.py` — framer/baud/parity kombinasyonları | Makinenin gerçek ayarı | **✅ 2026-09-02 — ASCII 9600 7-E-1** |
| 32-bit rollover | Simülatörde sayacı 0xFFFFFFF0'dan başlat | Delta hesabı rollover'da doğru | ⬜ |
| Ağ kesintisi | Kabloyu 30 dk çek, tak | Kuyruk çalışıyor, veri kaybı yok | ⬜ |
| Replay kopyası | Aynı mesaj partisini iki kez gönder | Idempotency (§12.2) | ⬜ |
| Geç gelen veri | 6 saatlik tamponu bir anda boşalt | Aggregate'ler doğru yeniden hesaplanıyor | ⬜ |
| Ani güç kesintisi | Pi'nin fişini 10 kez çek | Read-only root gerçekten çalışıyor | ⬜ |
| Saat kayması | NTP'yi kapat, saati 10 dk kaydır | `ts_synced` bayrağı çalışıyor | ⬜ |
| Makine hat değiştirme | Atama tablosuna yeni satır aç, eski dönem raporunu tekrar çalıştır | §12.5 geçmişi bozmuyor | ⬜ |
| Sayım doğruluğu | Vardiya sonu elle sayım vs sistem | Verinin güvenilirliği — en önemlisi | ⬜ |

### 14.3 Devreye alma sırasında

- Her makinede **ilk 3 vardiya** sistem sayımı ile elle sayım karşılaştırılır
- İlk hafta günlük olarak filo sağlık sayfası kontrol edilir
- Her makine için tek sayfalık kabul formu: register haritası doğrulandı mı, sayım tuttu mu,
  buton çalışıyor mu, sıcaklık kaç, `asset_code` girildi mi

---

## 15. [YENİ] Süreklilik — bus factor 1 problemi

Bu sistemi bir kişi kuruyor ve bir kişi bakacak. O kişi ayrılırsa sistem ölmemeli.

Minimum gereken (Faz 2 sonuna kadar):

1. **Runbook** — tek dosya, Türkçe:
   - Bir makinenin ekranı donmuşsa ne yapılır? (fişi çek tak)
   - Filo sağlık sayfasında bir nokta kırmızıysa ne yapılır?
   - Yedek Pi nasıl takılır? (adım adım)
   - Sunucu açılmıyorsa kime haber verilir, hangi docker komutu çalıştırılır?
   - Andon çalışmıyorsa geçici çözüm nedir?
   - **Çekirdeği hazır:** `andon-notes/KURULUM-ADIMLARI.md` (A→P, adım adım, doğrulama
     kriterleriyle). Golden image tarifi de bu dosyadan çıkacak
2. **Mimari diyagram** — §3.1, panonun kapağına asılı A3 çıktı
3. **Her şey git'te** — collector, web app, Ansible, PLC programları, docker-compose, vendor
   dokümanları, bu doküman ve çalışma günlüğü. **Bu brief 2026-09-02'den itibaren
   `andon-notes/andon-proje-brifi.md`'de de tutuluyor**
4. **Parola kasası** — ekipten en az bir kişi daha erişebilsin
5. **Envanter tablosu** — her cihaz: `machine_id`, `asset_code`, MAC, IP, seri no, konum,
   image sürümü
6. **Yılda bir kez gerçek geri yükleme tatbikatı**
7. **Repo'lar firma organizasyonuna taşınacak.** Şahsi GitHub hesabı geçici bir başlangıçtır;
   erişim tek kişiye bağlı kalırsa madde 4 ve 5 anlamsızlaşır (Karar #27)

**Bunu bir yük olarak değil, kaldıraç olarak gör:** "sistem bana bağımlı değil" diyebilmek,
bir sonraki bütçe talebinde en güçlü argümanın olur.

---

## Ek: Claude'dan sırada istenecekler

- Pi tarafı store-and-forward collector: MQTT + kalıcı yerel kuyruk, idempotent replay
- Andon buton servisi: `gpiozero` → MQTT → bildirim (Faz 1 adım C)
- `pdf2webp.py` — §6.3'e göre sunucu tarafı doküman render
- 32-bit serbest akan sayaç bloğu için DVP ladder
- Python collector + YAML config (word sırası `lo_hi` artık sabit girdi)
- **Modbus slave simülatörü** (§14.1)
- TimescaleDB şeması: ham telemetri, `machine` ve `machine_assignment` masterları (§12.5),
  Andon olayları, OEE için continuous aggregate'ler
- Window fonksiyonlarıyla OEE SQL'i
- Pi filosu için Ansible playbook + golden image kurulumu
- Makine tarafı web app
- Andon eskalasyon mantığı ve bildirim servisi (§12.3)
- **IT sunumu için tek sayfalık özet** (§3.6 tablosuna dayalı)
- **Operatör bilgilendirme metni ve KVKK aydınlatma taslağı** (§13.2)
- **Runbook iskeleti** (§15)
