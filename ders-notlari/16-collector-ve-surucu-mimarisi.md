# Collector ve sürücü mimarisi

**Ne zaman okunur:** Faz 1 adım E ve sonrası. Kod `andon-collector` repo'sunda.

Bu not, PLC'den veri okuyan servisin nasıl kurulduğunu ve neden öyle kurulduğunu anlatır.
Modbus'ın kendisi `13-modbus-ve-seri-haberlesme.md`'de, Python temelleri
`09-python-servis-temelleri.md`'de, systemd `08-systemd-servisler.md`'de.

## 1. Collector ne yapar

Tek cümleyle: **`machines.yaml`'ı okur, her makineye kendi ayarıyla bağlanır, saniyede bir
sayaçları okur, ne kadar arttığını hesaplar ve sonucu bir yere yazar.**

```
machines.yaml ──▶ config.py ──▶ worker (makine başına 1 thread)
                                   │
                                   ├─ driver.read()      ham register'lar
                                   ├─ counters.update()  rollover-güvenli fark
                                   └─ sink.emit()        journal / JSONL / (ileride MQTT)
```

## 2. Sürücü (driver) mimarisi — Karar #22

**Problem:** hatta en az üç farklı PLC ailesi var ve ileride başka markalar da gelecek
(brief §2). Her biri için ayrı bir program yazmak, altı ay sonra bakımı imkânsız hale gelen
üç ayrı proje demektir.

**Çözüm:** ortak bir çekirdek + protokol başına ince bir sürücü. Sürücü üç şey bilir:

```python
connect()   # bağlantıyı aç, True/False dön, asla hata fırlatma
read()      # {alan_adı: ham_sayı} dön
close()     # bağlantıyı bırak
```

Bunun üstündeki her şey — zamanlama, fark hesabı, loglama, çıktı — hangi protokolün
konuşulduğunu **bilmez.**

**Otomasyon karşılığı:** ladder'da yazdığın bir fonksiyon bloğu gibi düşün. Blok "iki analog
girişin ortalamasını al" işini yapar; girişin 4-20 mA mı yoksa 0-10 V mu olduğu bloğun
umurunda değildir, o dönüşüm girişte halledilir. Sürücü, o giriş dönüşümüdür.

**Pratik sonucu:** yeni bir PLC ailesi eklemek = `drivers/` klasörüne bir dosya + kayıt
tablosuna bir satır. Ana servise dokunulmaz. Yeni bir **makine** eklemek ise sadece
`machines.yaml`'a bir blok — kod hiç değişmez.

Şu an iki sürücü var:

| Sürücü | Kim kullanır |
|---|---|
| `modbus_rtu` | Grup 1–2: RS-485 üzerinden Delta DVP-SS2 |
| `modbus_tcp` | Grup 3: AS218TX (Ethernet varsa) **ve** simülatör testleri |

Planlanan: `gpio_sensor` (Grup 4 — okunabilir PLC'si olmayan makineler, kule lambası + sensör).

## 3. Rollover matematiği — bu notun en önemli bölümü

### Problem

DVP sayaçları **serbest akan totalizer**'dır: sayar, hiç sıfırlanmaz (brief §5.1, Karar #3).
32-bit bir sayaç 4.294.967.295'e ulaşınca **0'a döner.** Buna rollover (veya wrap) denir.

Düz çıkarma bu anda çöker:

```
önceki  = 4294967290
şimdiki = 5
fark    = 5 - 4294967290 = -4294967285      ← saçma
```

### Çözüm: modüler aritmetik

```python
fark = (şimdiki - önceki) % 2**32
```

```
(5 - 4294967290) % 4294967296 = 11          ← doğru
```

`%` (modulo) operatörü bölmeden kalanı verir. 32-bit bir sayaç zaten `2**32`'ye göre
kalan aritmetiğiyle çalıştığı için, aynı işlemi Python'da yapmak rollover'ı **bedavaya**
çözer. Ek bir `if` gerekmez.

### Ama aynı formül tehlikeli bir şeyi de sessizce yapar

Sayaç **sıfırlanırsa** (operatör resetledi, PLC programı değişti, enerji gitti):

```
önceki  = 1000
şimdiki = 0
fark    = (0 - 1000) % 2**32 = 4294966296   ← dört milyar parça!
```

Matematik açısından bu bir rollover'dan **ayırt edilemez.** İkisi de "sayı küçüldü"
görünür. Formül tek başına doğruyu bulamaz.

### Ayıran şey: makullük

Bir makine bir saniyede dört milyar parça üretemez. Ayrım buradan gelir:

```python
tavan = max_delta_per_second * (geçen_süre + 1)
if fark <= tavan:
    # gerçek üretim (rollover dahil) - güven
else:
    # imkânsız - bir şeyler ters
```

`max_delta_per_second` `machines.yaml`'da makine bazlıdır, varsayılanı 1000/saniye. Amacı
çevrim süresini denetlemek **değil** — reset'i ve bozuk çerçeveyi yakalamak. O yüzden bilerek
cömert.

Tavan aşılırsa collector iki durumu ayırır:

| Durum | Ne raporlanır |
|---|---|
| Sayı küçüldü, rollover'la açıklanamayacak kadar | `counter_reset_suspected` |
| Sayı büyüdü ama imkânsız kadar | `implausible_jump` |

### Neden fark `0` değil de `null` yazılıyor

Anomali durumunda collector **sayı uydurmaz.** `delta = null` yazar ve sebebini yanına koyar.

`0` yazmak "bu saniyede hiç parça üretilmedi" demektir — bu bir **iddiadır** ve yanlış
olabilir. `null` ise "bu farkı hesaplayamadım" der. Brief §14.2 sayım doğruluğunu en üste
koyuyor; **cevap vermemek, yanlış cevap vermekten iyidir.**

Ham sayaç değeri her durumda yazılır. Kanonik olan odur (brief §5.1: "ham değerleri sakla,
hesaplanmışları değil"); buradaki fark **türetilmiş** bir alandır.

### Servis yeniden başlarsa

Önceki değer bellekte tutulur. Servis yeniden başlayınca (veya gecelik reboot'ta, brief §4.2)
ilk okumanın karşılaştıracağı bir şey yoktur → `first_sample`, fark yok.

**Sistemden parça kaybolmaz:** ham totalizer yine yayınlanır, sunucu farkı kendi hesaplar
(brief §4.3). Sadece collector'ın kendi yerel farkında bir örneklik boşluk olur.

### Kesinti sırasında sayaç ilerlerse

Hiçbir şey yapmaya gerek yok. Tavan geçen süreyle ölçeklendiği için, 16 saniyelik bir
kesintiden sonra gelen 95 parçalık fark **doğru** kabul edilir. Bu, brief §4.3'teki
"Pi döndüğünde sayacı okur, son bildiği değerle karşılaştırır ve fark bozulmamıştır"
ilkesinin koddaki karşılığıdır.

### 32-bit'i iki register'dan birleştirmek

PLC'de 32-bit sayaç iki ardışık 16-bit register'da durur (D300 + D301). Bench'te ölçüldü
(2026-09-02): **Delta DVP `lo_hi`'dır** — ilk register düşük word.

```python
değer = (reg[1] << 16) | reg[0]      # lo_hi
```

`<< 16` = 65536 ile çarp (yüksek word'ün yeri). `|` = iki yarımı birleştir.

**İki register TEK istekte okunur.** Ayrı ayrı okursan, aradaki milisaniyede sayaç artarsa
yarısı eski yarısı yeni bir değer alırsın — 65535'e kadar sapan, ama tamamen normal görünen
bir sayı. Bu bir hız optimizasyonu değil, **doğruluk şartıdır.**

## 4. Makine başına bir thread

Pilot tasarımda her makinenin kendi Pi'si ve kendi RS-485 hattı var (brief §3.1), yani tek
döngü de yeterdi. Thread'ler iki başka sebep için var:

1. Devreye alma sırasında bir Pi geçici olarak iki makineye bakabilir.
2. **Cevap vermeyen bir makine diğerini bekletmemeli.** Seri port zaman aşımı 1 saniyedir;
   ortak döngüde bu, diğer her makineden çalınan 1 saniyedir.

`asyncio` değil `threading`: pymodbus'ın senkron istemcisi sıkıcı ve kanıtlanmış yoldur,
ve kendi thread'inde bloklayan bir okumayı gece 02:00'de anlamak bir olay döngüsünden kolaydır.

## 5. `time.monotonic()` — duvar saati değil

Döngü periyodu `time.monotonic()` ile ölçülür, `datetime.now()` ile değil. İki sebep:

1. **Kayma (drift).** Okuma 40 ms sürdü ve sonra 1 sn uyudun → çevrim 1,04 sn. Ayda yaklaşık
   bir saat kayar. Doğrusu: bir sonraki tick'in zamanını sabitleyip ona kadar uyumak.
2. **Saat sıçraması.** Pi 5'te RTC yok (brief §4.2). Açılıştan bir süre sonra chrony saati
   **düzeltir** ve saat ileri veya geri sıçrar. `monotonic()` bundan etkilenmez — sadece
   ileri akar, hiç sıçramaz.

Aynı sebeple: chrony senkron olduğunu bildirmeden üretilen kayıtlar `ts_synced: false` ile
işaretlenir. Veri atılmaz, ama sunucu bunu bilir.

## 6. Mesaj şeması ve idempotency

Her okuma brief §12.2'deki mesaja dönüşür:

```json
{
  "msg_id": "M001-8f2a1c-000000015",
  "machine_id": "M001",
  "boot_id": "8f2a1c",
  "seq": 15,
  "ts": "2026-09-02T11:08:39.847Z",
  "ts_synced": true,
  "counters": { "total": 0 },
  "state": 2,
  "derived": { "total_delta": 1, "total_wrapped": true }
}
```

**`msg_id` = `machine_id` + `boot_id` + `seq`.** Bu üçlü niye:

- `seq` her okumada bir artar — ama servis yeniden başlayınca 1'e döner.
- `boot_id` her serviste yeniden üretilen rastgele bir dizedir. Yani `seq` sıfırlansa bile
  `msg_id` **asla tekrar etmez.**
- Veritabanında `UNIQUE(msg_id)` + `INSERT ... ON CONFLICT DO NOTHING` (Karar #18).

**Neye yarıyor:** ağ kesildiğinde kuyruğa biriken mesajlar bağlantı gelince tekrar
gönderilir. Bir kısmı zaten gitmiş olabilir. `msg_id` sayesinde ikinci kez yazılmaz —
kopya kayıt oluşmaz. Buna **idempotency** denir: aynı işlemi iki kez yapmak, bir kez
yapmakla aynı sonucu verir.

`derived` bloğu brief'te yoktu, sonradan eklendi. Bu serbesttir (§5.1: "yeni alan eklemek
serbest"); **yasak olan mevcut bir alanın anlamını değiştirmektir.**

## 7. Sink — çıktının takıldığı yer

`sink.py` bugün iki şey yapar: journal'a okunur bir satır yazar, istenirse dönen bir JSONL
dosyasına da yazar.

**MQTT buraya eklenecek.** Aynı üç metodu (`emit`, `close`) uygulayan bir sınıf daha olacak,
üstündeki hiçbir şey değişmeyecek. Broker henüz kurulu olmadığı için (brief §9.1 adım D)
collector şimdilik sunucusuz çalışıyor — ve bu iyi: poll, fark ve rollover mantığı
Mosquitto var olmadan kanıtlandı.

> **JSONL uyarısı.** Filo Pi'sinde kök dosya sistemi salt okunurdur ve her yazma RAM
> overlay'e gider (brief §4.3). 1 Hz'de bu günde ~20 MB eder; 2 GB'lık bir Pi'de birkaç
> günde belleği doldurur. O yüzden JSONL varsayılan olarak **kapalıdır** ve açıldığında
> boyut sınırıyla döner. Bench ve kabul testi aracıdır (brief §9.1 F'deki vardiya sayımı
> karşılaştırması için lazım), üretim veri yolu değil.

## 8. Simülatörle test — `tools/dvp_sim.py`

Üç test gerçek donanımda yapılamaz:

| Test | Neden gerçek PLC'de yapılamaz | Simülatörde |
|---|---|---|
| 32-bit rollover | Sayacı 0xFFFFFFF0'a getiremezsin; doğal yoldan gelmesi 4,29 milyar parça sürer | `--start 0xFFFFFFF0` |
| Sayaç sıfırlanması | Çalışan bir makinenin sayacını bilerek bozmak demek | `--reset-after 8` |
| PLC'nin susması | Fişi çekmek işe yarar ama tekrarlanabilir değil ve üretimde yapılamaz | simülatörü öldür |

Simülatör Modbus **TCP** konuşur, RS-485 değil. Uygulama katmanı birebir aynıdır — aynı
fonksiyon kodu, aynı register'lar, aynı word sırası — o yüzden sürücünün üstündeki her şey
gerçekte olacağı gibi test edilir. Sadece sürücü farklıdır. Sürücü mimarisinin işe yaradığı
yer tam burasıdır.

Simülatör config'i `machines.sim.yaml`'da, ayrı dosyada durur. **`machines.yaml` asla sahte
bir makine taşımaz** — üretimde koşan config ile masaüstünde koşan config aynı dosyayı
paylaşırsa, bir gün simülatör saha ekranında görünür.

## 9. Sık karşılaşılacak hatalar

| Belirti | Muhtemel sebep |
|---|---|
| `config error: ... is not a canonical field` | `machines.yaml`'da alan adı yanlış yazılmış (`part_total`). Alan isimleri dondurulmuştur (brief §5.1) |
| `ASCII framing normally uses 7 data bits` | `framer: ascii` ile `bytesize: 8` birlikte yazılmış. Bu kombinasyon hatta sessizlik üretir — teşhisi en zor arıza |
| `port did not open` | Adaptör takılı değil, kullanıcı `dialout` grubunda değil, ya da portu başka bir program tutuyor |
| `map_verified is false` uyarısı | Normal ve kasıtlı. Bu makinede adresler gerçek donanımla doğrulanana kadar susmaz |
| `first_sample` anomalisi | Servis yeni başladı. İlk okumanın karşılaştıracağı değer yok. Bir sonraki okumadan itibaren normale döner |
| `counter_reset_suspected` | Sayaç geriye gitti. Ya PLC'de reset var, ya adres yanlış, ya word sırası bu aile için farklı |

## 10. İlgili notlar

- `13-modbus-ve-seri-haberlesme.md` — Modbus, RS-485, adres aritmetiği
- `09-python-servis-temelleri.md` — Python temelleri
- `08-systemd-servisler.md` — servisi açılışta başlatmak
- `06-yaml-ve-config.md` — YAML sözdizimi
- `10-mqtt.md` — sink'e eklenecek katman
