# Andon & OEE — Çalışma Günlüğü

Bu dosya, projede fiilen ne yapıldığının tarihli kaydıdır.
Mimari kararlar ve gerekçeleri `andon-proje-brifi.md` içinde; öğrenme aşamaları
`andon-ogrenme-yolu.md` içinde. Burası "ne yaptım, ne çıktı" defteri.

Her oturum bir başlık: yapılanlar, ölçümler, kararlar, açık kalanlar, sıradaki adım.

---

## 2026-08-27 — Bench kurulumu, ilk ölçümler, kimlik şeması

### Yapılanlar

- Raspberry Pi 5 (2 GB) imajlandı: Raspberry Pi OS 64-bit **with desktop**,
  hostname `andon-bench`, kullanıcı `andon`, SSH parola ile açık.
- Laptop'tan SSH bağlantısı kuruldu — öğrenme yolu **Aşama 1 tamamlandı**.
- İlk ölçümler alındı (aşağıda), brief §4.2 ve §4.3'e işlendi.
- `machines.yaml` şablonu yazıldı (M001, hiçbir değer doğrulanmamış).

### Ölçümler

| Ölçüm | Değer | Ortam |
|---|---|---|
| `free -m` used / available | 450 / 1555 MB | Desktop imaj, idle, Chromium kapalı, masaüstü oturumu açık |
| `vcgencmd measure_temp` | 43,9 °C | Active Cooler, açık hava, oda sıcaklığı |
| `swapon --show` | `/dev/zram0`, 2 GB, kullanım 0 B | RAM içinde sıkıştırılmış; SD karta yazmıyor |

**Yorum:** Desktop imajın idle maliyeti, brief §4.6'da hafızadan tahmin edilen
500–700 MB'ın altında çıktı. Karar #6 (Pi 5 / 2 GB) lehine veri — ama kapatmıyor.
Asıl test §9.1 F: Chromium kiosk açık, en büyük çizim yüklü, 7 gün boyunca.

### Kararlar

- **Raspberry Pi Connect kullanılmayacak** — brief Karar #25.
  Dışarıya kalıcı bağlantı açıyor, doğrudan bağlanamadığında vendor relay'i üzerinden
  geçiyor, erişimi şahsi bir hesaba bağlıyor. LAN içinde SSH zaten var.
  Not: Lite imajında shell-only varyantı varsayılan kurulu geliyor, golden image'da
  kaldırılacak.
- **Kimlik şeması değişti** — brief Karar #26.
  Eski: `M01`–`M16`, tek hat varsayımı. Yeni: `machine_id` anlamsız ve kalıcı (`M001`),
  `asset_code` (`MAK25-30-511`) ayrı alan, `site`/`area`/`line` zamanla değişen özellikler.
  Sebep: çok fabrika/çok hat gerçeği, bazı makinelerin envanter kodunun olmaması,
  kodların düzeltilebilmesi, makinelerin hat değiştirebilmesi.
  Yerleşim geçmişi ayrı tabloda: `machine_assignment` (brief §12.5).
- **MQTT topic'ten hat çıkarıldı:** `andon/makine/<machine_id>/...`
- **IP: statik değil, router'da MAC'e DHCP rezervasyonu.**
- **Git: hat başına repo yok.** Kod repo'ları hattan bağımsız; her hat `andon-ansible`
  altında bir envanter klasörü ekler.

### Açık kalanlar

- Pi #2'nin RAM'i teyit edilecek (2 GB mı?) — paralel imaj ölçümü buna bağlı
- Kodsuz makineler için envanter kodu talebi (Faz 0)
- `machines.yaml` içindeki hiçbir register adresi doğrulanmadı — `map_verified: false`

---

## 2026-08-29 — Repo kurulumu, geliştirme akışının değişmesi, IT onayı

### Yapılanlar

- PC üzerinde `Desktop/andon` klasörü Cowork'e bağlandı; `andon-collector` ve
  `andon-notes` repo'ları burada oluşturuldu ve ilk commit'leri atıldı.
- `machines.yaml` PC'ye taşındı — artık tek doğru kopyası burada.
- Bu çalışma günlüğü başlatıldı.
- `ders-notlari/` klasörü açıldı: Linux temelleri, SSH, git, Raspberry Pi donanım ve
  yazılım, YAML, komut sözlüğü. Kural: buraya bir konu **fiilen kullanıldıktan sonra**
  girer; dosyalar her oturumda büyür.
- **Ders notlarının kapsamı genişletildi:** sadece kullandığımız komutlar değil, her
  konunun **temeli** baştan atıldı. 16 dosya: Linux, SSH, git, Pi donanım/yazılım, YAML,
  ağ, systemd, Python, MQTT, Docker, SQL/TimescaleDB, Modbus/RS-485, Ansible, Grafana,
  komut sözlüğü. Her dosyanın başında "ne zaman lazım" yazıyor; ileri fazların notları
  hazır bekliyor.

### Kararlar

- **IT onayı alındı: fabrika ağında Linux cihazlara izin var.**
  Bu, brief Karar #1'in geri dönüş şartını ortadan kaldırıyor — telemetri S7-1500
  üzerinden geçmeyecek, ~1.800–2.800 € comm modülü ve ~1 aylık SCL geliştirme
  masaya gelmeyecek. §11 madde 3 kapandı.
- **Geliştirme akışı değişti: kod PC'de yazılır, GitHub'a push edilir, Pi `git pull`
  ile alır.** Eski plan (Pi üzerinde geliştir) tek Pi varken geçerliydi.
  GPIO ve seri port testleri donanım orada olduğu için Pi'de kalmaya devam ediyor;
  hızlı iterasyon için VS Code Remote-SSH kullanılacak.
- **Üretimdeki Pi'lerin repo'ya yazma yetkisi olmayacak** — sadece okuma (deploy key).
  Faz 2'de yerini Ansible alacak.
- **Repo'lar private.** Şahsi GitHub hesabı geçici çözüm; kalıcı olarak firma
  organizasyonuna taşınacak (brief §15 bus factor).

### Teknik notlar

- Cowork'ün bağlı klasörde çalıştırdığı kabuktan GitHub'a **HTTPS erişimi var**
  (`curl https://github.com` → 200), ama **SSH portu (22) erişilemiyor**.
  Sonuç: `git add` / `git commit` otomatik yapılabiliyor, `git push` PC'den
  (VS Code veya Git Bash) yapılacak.

### Açık kalanlar

- GitHub'da `andon-collector` ve `andon-notes` private repo'ları açılacak, `remote`
  eklenecek, ilk push yapılacak
- Pi için read-only deploy key üretilecek
- Öğrenme yolu Aşama 2 (kabuk alıştırması) hâlâ bekliyor

### Bekleyen ölçüm — RTC penceresi

**Soru:** açılıştan chrony senkronuna kaç saniye geçiyor? Cevap, filoya RTC pili alınıp
alınmayacağını belirleyecek.

**Yöntem (iki adımda, tek blok halinde yapıştırılamaz):**
1. `sudo reboot`
2. 30–60 sn bekle, tekrar SSH ile bağlan, sonra:
```bash
uptime -s
timedatectl
journalctl -b -u chrony | grep -i "selected\|synchron"
```

**Yorum kuralı:** pencere < ~15 sn ise pil gerekmiyor (o sırada makine de kapalı, ayrıca
sunucu `received_at` damgası var). Uzunsa veya read-only root'ta saat çok geriye düşüyorsa
pil almaya değer.

**Bağlı [DOĞRULA]:** overlay FS açıkken `fake-hwclock` saati diske yazamıyor — davranışı
Faz 2'de golden image kurulurken test edilecek.

**Öğrenilen ders:** `sudo reboot` SSH oturumunu keser; altındaki komutlar çalışmaz.
Reboot içeren adımlar bundan sonra iki parça halinde verilecek.

### Aşama 2 — tamam

Kabuk alıştırması Pi üzerinde yapıldı: `pwd`/`ls -la`/`cd`/`mkdir`, `nano` ile YAML yazma,
`cat`/`cp`/`mv`/`rm`, `echo` ile `>` ve `>>` farkı, `free -m`/`df -h`/`systemctl status`.
Çıktı paylaşılmadı (monitör başında çalışıldı), sahibin beyanına göre kavrandı.

**Not:** Aşama 3'ten itibaren laptop'tan SSH ile çalışmak pratik olacak — çıktıyı kopyalamak
ve kod yapıştırmak için. Üretim disiplini de zaten bu (GUI Pi'de değil laptop'ta).

### Açılan konu — yönetim erişimi (ağ)

IT üretim VLAN'ını internetsiz ayrı bir alana (ör. `192.168.5.x`) alırsa, **ofisten Pi'lere
SSH erişimi otomatik olmaz** — ayrı bir firewall kuralı gerekir. "Pi internete çıkamaz" ile
"Pi'ye erişilemez" farklı şeyler.

**Karar önerisi: atlama sunucusu (jump host).** Ofisten sadece sunucuya SSH açılır, Pi'lere
sunucu üzerinden geçilir (`ProxyJump`). Gerekçe: IT'nin yazacağı kural tek hedef/tek port
olur, tüm yönetim erişimi tek noktadan geçip loglanır, ve Ansible zaten sunucuda çalışacağı
için o makine nasılsa tüm Pi'lere erişebilir olmalı — yeni bir yol açmıyoruz.

**Açık soru:** sunucu hangi VLAN'da olacak? Üretim VLAN'ında olması hem MQTT kurallarını
gereksiz kılıyor hem jump host rolünü doğal hale getiriyor.

**Ek kural (yeni):** üretime giden golden image'da **Wi-Fi kapalı olacak.** Bir Pi hem ofis
Wi-Fi'ına hem üretim Ethernet'ine bağlıysa iki ağı köprüler ve "üretim ağında internet yok"
taahhüdü çöker. Bench ünitesi için geçerli değil.

**Netleşen:** IP adresi cihazdan değil ağdan gelir. Wi-Fi'da SSID→VLAN eşlemesi, Ethernet'te
switch portunun VLAN'ı belirleyicidir. Bizim seçimimiz Ethernet + MAC'e DHCP rezervasyonu:
Pi yine DHCP ile sorar, cevap hep aynı gelir, ayar tek tabloda durur.

**Switch hakkında netleşen:** switch IP dağıtmaz. Yönetilmeyen switch bulunduğu ağı
çoğaltır — uplink'i üretim VLAN'ındaysa ona takılan her cihaz `192.168.5.x` alır.
Yönetilebilir switch'in IP'si sadece kendi yönetim arayüzü içindir.

**İki somut bulgu:**
1. **16 port yetmiyor.** 16 Pi + 1 sunucu + 1 uplink = 18 port → **24 portlu alınacak.**
2. **Model isimleri yanıltıcı:** TL-SG1016 yönetilmeyen, TL-SG1016**DE** Easy Smart (VLAN'lı).
   Brief §3.1 yönetilebilir diyor; gerekçe port bazlı teşhis (gece 02:00'de "hangi port
   link görmüyor" sorusu), portu uzaktan kapatabilme, VLAN esnekliği, IT'nin envanteri.

**IT'ye sorulacak yeni soru:** üretim VLAN'ında **DHCP sunucusu olacak mı?** Tam izole bir
VLAN'da router bacağı yoksa DHCP de yoktur; o zaman ya statik IP'ye dönülür ya da DHCP kendi
sunucumuzda (`dnsmasq`) çalışır. Bu, brief §12.4'teki rezervasyon kararını yeniden açar.

**PoE açık:** brief §4.4'teki PoE seçeneği seçilirse switch de PoE'li olmalı. Karar verilmedi.

**Kritik netleşme:** laptop'a elle `192.168.5.x` yazmak onu üretim ağına sokmaz —
belirleyici olan **portun VLAN'ı**. Port doğruysa IP ister DHCP'den gelir ister elle yazılır;
port yanlışsa hiçbir ayar işe yaramaz.

**Bulunan kısayol — izole ada testi:** laptop + switch + Pi, uplink hiçbir yere takılı değil.
VLAN/DHCP/IT gerekmeden tüm zincir (kablolama, switch, SSH) kanıtlanabilir. Elle statik IP
ile veya hiç ayar yapmadan mDNS ile (`ssh andon@andon-bench.local`). IT portu hazırlayınca
sadece uplink takılacak. **Bu test IT'yi beklemeden yapılabilir.**

**Zamanlama:** bu konuşma **kablolamadan önce** yapılacak. Erişim hiç açılmazsa alternatif
panonun yanında laptop'la çalışmak — 16 makinede ciddi yavaşlama.

**Netleşen:** ayrı bir Wi-Fi ağına gerek yok — konu kablosuz ağ değil yönlendirme.
Farklı IP alanına erişmek, internete erişmekle aynı mekanizma (gateway üzerinden).

**İlk istenecek: Yol A — masadaki ethernet portunun üretim VLAN'ına alınması.** PC Wi-Fi'dan
internete, ethernet'ten üretim ağına bağlı olur. IT hiçbir firewall kuralı yazmaz, VLAN
izolasyonu bozulmaz. Şart: o porttan **default gateway verilmemeli**, yoksa internet kesilir.
Reddedilirse Yol B (ofisten sunucuya tek kural + ProxyJump).

Notlara işlendi: `07-ag-temelleri.md` (yönlendirme mantığı, iki yol, Windows kontrol
komutları, **IT'ye gönderilecek hazır metin**), `02-ssh-ve-uzak-erisim.md` (ProxyJump).

### Plan değişikliği — systemd ertelendi, gerçek veriye geçildi

Sahibin itirazı: "gerçekten veri çekip çekmediğimizi denemeden servis dosyası yapmak ne
kazandırıyor?" **Haklı.** Projenin gerçek bilinmeyenleri register haritası, word sırası ve
adaptörün çalışması; `systemd`'nin bir scripti ayakta tutması bilinmeyen değil. Brief §9.1'in
sırası da aynı fikirde: adım B (RS-485 kanıtı) `systemd`'den önce geliyor.

**Karar:** `heartbeat.py` yazıldı ve elle çalıştırıldı (döngü + `logging` + `try/except` +
`signal` iskeleti öğrenildi, collector aynı iskeleti kullanacak). **Servise dönüştürme,
collector'ın ilk hali hazır olduğunda yapılacak** — ayakta kalması gereken gerçek bir iş
olduğunda. İptal değil, erteleme: read-only root açıldığında elle çalıştırma zaten imkânsız
olacak (brief §4.3).

**Yeni sıra: brief §9.1 adım B.** Elde **yedek bir DVP-SS2 var**, bu yüzden simülatör yerine
gerçek PLC ile gidiliyor. Sıra "önce oku, sonra yaz" (brief §5.5):
1. USB-RS485 ↔ DVP COM2 kablolaması (A/B etiketleri ters olabilir — çalışmazsa ilk deneme)
2. `scan_dvp.py` — **hiçbir şey yazmadan** hangi framer/baud/parity kombinasyonunun cevap
   verdiğini tara
3. Cevap gelirse: WPLSoft'tan D300=2, D301=1 girilip `0x112C`'den 2 register okunacak →
   `[2, 1]` görülürse **hem 0x1000 adres tabanı hem lo_hi word sırası kanıtlanır**
   (brief §5.1 ve §5.2'deki iki [DOĞRULA] birden kapanır)
4. Cevap gelmezse: COM2 slave olarak yapılandırılacak (D1120/D1121/M1120/M1143 —
   **manuelden teyit edilecek**)

**Kullanılan PLC bench'te, hiçbir makineye bağlı değil.**

### Modbus adres tablosu doğrulandı (2026-09-02)

**Tarama sonucu:** DVP-SS2 COM2, **hiçbir ayar değiştirilmeden**, fabrika ayarıyla cevap
veriyor: **ASCII, 9600, 7-E-1, slave 1.**

**Bunun büyük sonucu:** brief §5.3 / Karar #5 "COM2 her yerde RTU 19200'e standardize edilecek"
diyordu. Zaten hepsi fabrika ayarındaysa, standardize etmek için 13 üretim PLC'sine yazmanın
karşılığı sadece biraz verimlilik; bedeli 13 makinede duruş. 1 Hz'de birkaç register okumak
ASCII 9600'de ~30–50 ms sürüyor — bol bol yeterli. **Öneri: ayarı olduğu gibi bırakmak.**
⚠️ Bir bench PLC'nin fabrika ayarında olması sahadaki 13 makineyi kanıtlamaz — Faz 0'da her
makinede `scan_dvp.py` çalıştırılıp yetenek matrisine yazılacak (makinelere hiçbir şey
yazmadan yapılabilir).

**Vendor adres tablosu geldi:** `vendor-docs/ES-EX-...-MODBUS_ADRESLERI.xls`. Tabanlar
doğrulandı ve hafızadan yazılan tabloyla **birebir tutuyor**: X=0x0400, Y=0x0500, T=0x0600,
M=0x0800, C=0x0E00, D=0x1000. Ayrıca D1120=0x1460, D1121=0x1461, M1120=0x0C60 coil,
M1143=0x0C77 coil — dördü de SS2'de destekli.

**Yaşanan hata ve düzeltmesi:** Excel'in hex sütununu programatik okurken ham değerler
(`17`, `0`) görülüp "dosya hatalı" sanıldı. Sahibin itirazı üzerine incelendi: hücrelerde
**özel sayı biçimi** var (`\1000` → 17'yi "1017" gösteriyor). **Dosya doğru, yorum yanlıştı.**
Kalan kural: otomatik çıkarımda `MODBUS ADRESLERİ` sütunu kullanılacak, hex sütunu değil.

**Hâlâ açık:** word sırası. WPLSoft'tan D300=2, D301=1 girilip 0x112C'den 2 register
okunacak; `[2, 1]` beklenen sonuç.

**Gelmedi:** AS serisi adres dosyası (Grup 3 / AS218TX için gerekecek).

### Sıradaki adım

Öğrenme yolu **Aşama 3** — Python scriptinden `systemd` servisine. Projenin yazılım
iskeleti burada başlıyor.

---

## Word sırası testi hazırlandı (2026-09-02)

`andon-collector/tools/read_test.py` yazıldı. Sadece okuyan bir betik: `0x112C`
(= D300) adresinden **tek istekte 2 register** okur, sonucu iki farklı varsayımla
(`lo_hi` ve `hi_lo`) yorumlar ve hangisinin doğru olduğunu ekrana yazar.

**Neden tek istek:** iki register ayrı ayrı okunursa aradaki milisaniyede sayaç
artabilir. O zaman düşük word yeni, yüksek word eski olur ve ortaya gerçekte hiç
var olmamış bir sayı çıkar. 32-bit sayaçlarda bu, sessizce yanlış OEE üreten
klasik hatadır.

**Test yöntemi:** WPLSoft device monitor'den elle `D300 = 2`, `D301 = 1` girilir.
Okuma `[2, 1]` gelirse ilk register düşük word'dür (`lo_hi`); `[1, 2]` gelirse
tersidir (`hi_lo`). Değerler küçük ve birbirinden farklı seçildi ki karışmasın.

**Bu test kapatacak [DOĞRULA] maddeleri:** brief §5.1 (word sırası), §5.2 (adres tabanı).

Sonuç henüz alınmadı.

---

## Word sırası doğrulandı — 2026-09-02

**Ham çıktı (Pi `andon-bench`, bench DVP-SS2):**

```
Baglandi: /dev/ttyUSB0 9600 7E1 ASCII, slave=1
Okunuyor: adres 0x112C (4396) = D300, 2 register
ham registerlar : [2, 1]
  lo_hi yorumu  : 65538
  hi_lo yorumu  : 131073
  >>> SONUC: word sirasi = lo_hi  (ilk register dusuk word)
Port kapatildi.
```

WPLSoft'tan `D300 = 2`, `D301 = 1` girilmişti.

### Bu tek okuma üç şeyi birden kapattı

1. **Word sırası = `lo_hi`.** İlk register düşük word. 32-bit değer =
   `(reg[1] << 16) | reg[0]`. Brief §5.1'deki `[DOĞRULA]` kapandı.
2. **Adres tabanı doğru.** `0x112C`'den okunan değerler gerçekten D300/D301 idi — yani
   D tabanı `0x1000` sadece vendor tablosunda değil sahada da doğru. Brief §5.2 `[DOĞRULANDI]`.
3. **Dönüşüm aritmetiği doğru.** `D000 = 44097 → D300 = 44397 → 44397 − 40001 = 4396 = 0x112C`
   zinciri işliyor. Artık her D register'ının adresini hesapla ile bulabiliriz.

**Neden 2 ve 1 seçildi:** küçük, birbirinden farklı ve ikisi de tek başına anlamlı değil.
Simetrik bir değer (örneğin `1, 1`) hiçbir şey kanıtlamazdı; büyük değerler ekranda karışırdı.

**Bu sonuç DVP ailesine ait.** AS serisi veya başka bir marka geldiğinde word sırası o aile
için ayrıca ölçülecek. `machines.yaml`'daki `order` alanı tam bu yüzden makine bazlı.

### Karar #5 iptal, Karar #28 yürürlükte

Brief §5.3 "COM2'yi her yerde RTU 19200 8-N-1'e standardize et" diyordu. Tarama sonucu bunu
geçersiz kıldı: fabrika ayarı (ASCII 9600 7-E-1) hiçbir PLC değişikliği olmadan cevap veriyor.

Hesap:

| | RTU'ya standardize | Fabrika ayarında bırak |
|---|---|---|
| Hız kazancı | 1 Hz poll'de ölçülemez (~30–50 ms vs 1000 ms bütçe) | — |
| Maliyet | 13 planlı duruş penceresi | 0 |
| Risk | Doğrulanmamış `D1120`/`M1143` bitlerine yazma → haberleşme kaybı | 0 |
| Kod tarafı | Config'de 4 satır az | Config'de 4 satır çok |

Collector zaten çok protokollü (Karar #22) — yani "karmaşayı kaldırmak" aslında karmaşayı
PLC'den config'e taşımaktan ibaretti. Dört satır için 13 duruş alınmaz.

**İstisna:** bir makinede fabrika ayarı çalışmazsa veya tek RS-485 üzerine birden fazla PLC
(multi-drop) bağlanacaksa, sadece o makinelerde ayar değiştirilir.

### Güncellenen dosyalar

- `andon-proje-brifi.md` → **v2.7** (Claude Project + bu repo'da çalışma kopyası)
- `andon-collector/machines.yaml` → `serial:` bloğu, `slave: 1`, `addr: 0x112C`, `order: lo_hi`
- `KURULUM-ADIMLARI.md` → H.3 ✅, H.4 sonuç

### Hâlâ açık

- **`state` hangi D register'ında olacak?** Henüz kararlaştırılmadı. `parts_total` doğrulandı,
  `state` `TODO: VERIFY` olarak duruyor.
- AS serisi Modbus adres tablosu elde yok (Grup 3 / AS218TX için gerekli).

---

## GitHub'a ilk gerçek push — 2026-09-02

Brief §11 madde 15'in ilk yarısı kapandı. `andon-collector` ve `andon-notes` artık GitHub'da.

| Repo | Push edilen | İçerik |
|---|---|---|
| `andon-collector` | 3 commit | `machines.yaml` (+49/−16), `tools/read_test.py` (151 satır), `tools/scan_dvp.py` (168 satır) |
| `andon-notes` | 17 commit | 21 dosya, 5953 satır — günlük, `KURULUM-ADIMLARI.md`, 16 ders notu, brief v2.7 çalışma kopyası |

Doğrulama: `git status -sb` her iki repo'da `## main...origin/main` — `ahead` yok.
`git log --oneline -1 origin/main` uzak dalların gerçekten `03d14d2` ve `09c6149`'a
ilerlediğini gösterdi.

### Push öncesi denetim — üç kontrol

**1. Secret taraması: temiz.** `password|token|api_key|PRIVATE KEY` kalıpları tarandı.
Çıkan tek şey `ders-notlari/11-docker.md` ve `15-grafana-ve-gosterim.md` içindeki
`${DB_PASSWORD}` / `${GRAFANA_DB_PASSWORD}` — bunlar placeholder, gerçek değer değil.
Brief §13.1 ihlali yok.

**2. `.gitignore` boşluğu bulundu ve kapatıldı.** `andon-notes/.gitignore` sadece üç satırdı
(`*.bak`, `~$*`, `.DS_Store`); collector'daki secret bloğu orada yoktu. Şu an notes'ta secret
tutulmuyor ama ileride `.env` veya Vault parolası bu klasöre düşerse sessizce commit'lenirdi.
İki dosya artık aynı secret bloğunu taşıyor.

**3. `andon-collector`'da artık bir `master` dalı çıktı.** `main` ile ortak atası yoktu
(orphan). İçeriği `main`'in ilk commit'i `61c3503` ile birebir aynıydı — repo'nun hem yerelde
hem GitHub'da başlatılmasından kalan ikinci başlangıç. `git branch -D master` ile silindi,
kaybolan benzersiz içerik yok. Teşhis ve genel kural: `ders-notlari/03-git.md` §12.

### Push'u neden Claude çalıştıramadı

GitHub kimlik bilgisi **Windows Credential Manager'da**, kullanıcı hesabına bağlı duruyor.
Claude'un eriştiği kabuk klasöre ulaşabiliyor ama o kasaya ulaşamıyor:

```
fatal: could not read Username for 'https://github.com': terminal prompts disabled
```

Bu bir arıza değil, tasarımın kendisi. **İş bölümü buradan çıkıyor:** dosya düzenleme ve
commit her yerden yapılabilir, `push` PC'den yapılır. Aynı ayrım Pi'ler için de geçerli
olacak — onlar deploy key ile sadece `pull` edecek, hiç push etmeyecek (brief §4.7).

### Yeni ders notu içeriği

`ders-notlari/03-git.md` dört bölüm büyüdü (191 → 304 satır):

- §10 `origin/main` bir yerel işaretçidir; `ahead`/`behind` okuma, `origin/main..main` farkı
- §11 Kimlik doğrulama yöntemleri: Credential Manager, PAT, SSH key, deploy key
- §12 Orphan branch: teşhis (`git merge-base`), güvenli silme, tekrar oluşmaması için kural
- §13 Push öncesi kontrol listesi (secret taraması dahil) ve push sonrası doğrulama

### Hâlâ açık

- **`vendor-docs` git'te değil.** Brief §15 madde 3 "vendor dokümanları git'te" diyor;
  klasör şu an sadece PC'nin diskinde. İki dosya var: DVP adres tablosu (`.xls`, 3,1 MB) ve
  AS200/AS300 adres tablosu (`.pdf`, 313 KB). Kaybolursa §5.2'nin kaynağı kaybolur.
- **Pi için read-only deploy key** üretilmedi (brief §11 madde 15'in ikinci yarısı).
- **`state` register adresi** hâlâ `TODO: VERIFY` (brief §11 madde 16).
