# Modbus ve Seri Haberleşme — Linux Tarafından Bakış

Bu konunun otomasyon tarafını zaten biliyorsun. Bu not **Linux ve Python tarafında** neyin
nasıl göründüğünü anlatıyor.

## 1. Fiziksel katman: RS-485

- **Fark sinyalli (differential)**, iki telli: A ve B. Gürültüye dayanıklı, uzun mesafe
- **Çok noktalı (multidrop):** aynı hatta birden fazla cihaz olabilir
- **Yarı çift yönlü (half-duplex):** aynı anda ya konuşulur ya dinlenir
- Uzun hatlarda uçlara **120 Ω sonlandırma direnci** gerekir

**Bu projede hat çok kısa:** Pi pano içinde, PLC'nin yanında, ~1 m kablo (brief §3.1). Yani
sonlandırma ve gürültü sorunları büyük ölçüde yok. Her Pi kendi makinesine bire bir bağlı —
segment paylaşımı yok.

**USB-RS485 adaptörü** Linux'ta bir seri port olarak görünür: `/dev/ttyUSB0`.

## 2. Seri port parametreleri

İki taraf **birebir aynı** olmalı, yoksa çöp okursun:

| Parametre | Bizim seçim | Not |
|---|---|---|
| Baud rate | 19200 veya 38400 | Hız. Yüksek = hızlı ama gürültüye hassas |
| Data bits | 8 | |
| Parity | N (yok) | |
| Stop bits | 1 | |

Kısaca **8-N-1**. Brief §5.3'ün kararı: boş COM2 portunda ayarı sen belirlediğin için
**her makinede aynı yapılandırma** kullanılacak — mevcut ASCII/RTU karmaşası böylece
problem olmaktan çıkıyor.

## 3. Modbus RTU vs TCP vs ASCII

| Tip | Taşıma | Bu projede |
|---|---|---|
| **Modbus RTU** | Seri (RS-485), ikili (binary) çerçeve | Grup 1–2: Delta DVP-SS2 |
| **Modbus TCP** | Ethernet, port 502 | Grup 3: AS218TX |
| **Modbus ASCII** | Seri, metin çerçeve. Daha yavaş | Kullanmayacağız |

Protokol mantığı üçünde de aynı; sadece paketleme farklı. `pymodbus` üçünü de yapar —
S7-1500 üzerinden gitseydik ASCII cihazlara erişemeyecektik (brief §3.2).

## 4. Master / slave ve adresleme

- **Master** soru sorar, **slave** cevap verir. Bizim collector master'dır.
- Her slave'in bir **istasyon adresi** vardır (1–247).
- **Kural (brief §5.3):** slave adresi hat içinde 1'den başlayarak sırayla verilir. Bu adres
  `machine_id` ile eşleşmek **zorunda değildir** — `machine_id` global ve anlamsızdır,
  slave adresi yerel bir haberleşme detayıdır. Etikete ikisini de yaz.

## 5. Register tipleri ve adresler

| Tip | Ne | Fonksiyon kodu |
|---|---|---|
| Coil | Tek bit, okunur/yazılır | 01 / 05 |
| Discrete input | Tek bit, sadece okunur | 02 |
| Input register | 16 bit, sadece okunur | 04 |
| **Holding register** | 16 bit, okunur/yazılır | **03 / 06** |

Bizim okuyacağımız D register'ları **holding register**'dır.

### Delta DVP adres tabanları — **[DOĞRULANDI]**

**Kaynak:** `vendor-docs/ES-EX-SS-SA-SX-SC-EH-SS2-SX2-SA2-ES2-SV-SV2_PLC_MODBUS_ADRESLERI.xls`
(2026-09-02). Ayrıca bench'te `0x1000`'den cevap alınarak deneysel olarak da doğrulandı.

| Cihaz | Excel'deki ilk adres | 0-tabanlı taban | Register tipi |
|---|---|---|---|
| S0 | 1 | `0x0000` | coil |
| X0 | 11025 | `0x0400` | discrete input |
| Y0 | 1281 | `0x0500` | coil |
| T0 (word) | 41537 | `0x0600` | holding |
| M0 | 2049 | `0x0800` | coil |
| C0 (word) | 43585 | `0x0E00` | holding |
| D0 | 44097 | `0x1000` | holding |

Yani `D0` = 0x1000, `D23` = 0x1017, `D300` = 0x112C.

**COM2 yapılandırma register'ları** (aynı kaynak, `SS2/SX2` sütununda listeli):

| Register | Excel | 0-tabanlı | Ne |
|---|---|---|---|
| `D1120` | 45217 | `0x1460` | protokol / baud / format |
| `D1121` | 45218 | `0x1461` | istasyon adresi |
| `M1120` | 3169 | `0x0C60` coil | ayarı koru |
| `M1143` | 3192 | `0x0C77` coil | RTU / ASCII seçimi |

⚠️ Bu tablo **sadece Delta DVP** ailesi içindir. Başka bir marka geldiğinde hiçbir işe
yaramaz; o ailenin kendi dokümanı gerekir.

### Off-by-one tuzağı — gösterim farkı

Modbus fonksiyon kodları adresleri **0-tabanlı** kullanır; dokümanlar ve mühendislik
araçları genelde **1-tabanlı** `4xxxx` gösterimini kullanır. `pymodbus` 0-tabanlı ister.

| Register tipi | Doküman gösterimi | Protokol adresi |
|---|---|---|
| Word (D, T-word, C-word) | `4xxxx` | değer − **40001** |
| Coil (Y, M, S) | `0xxxx` | değer − **1** |
| Discrete input (X, T-bit, C-bit) | `1xxxx` | değer − **10001** |

Örnek: `D23` → Excel 44120 → `44120 − 40001 = 4119 = 0x1017`.

**Kural:** bench'te bilinen bir D register'ı okuyup **beklenen değeri gördüğünü**
doğrulamadan hiçbir adresi kesin kabul etme.

### Vendor tablosunu programatik okurken: hex sütununa değil Modbus sütununa bak

Bu Excel'de hex sütunu **özel sayı biçimiyle** gösteriliyor: hücre `17` sayısını tutuyor,
biçim dizesi (`\1000`) başına "10" ekleyip ekranda **1017** gösteriyor.

| Satır | Hücredeki ham değer | Biçim | Ekranda |
|---|---|---|---|
| D23 | `17` | `\1000` | 1017 |
| D256 | `0` | `\1\100` | 1100 |
| D297 | `29` | `\1\100` | 1129 |

Dosya doğru; ham değeri okuyan her araç yanılır. **`MODBUS ADRESLERİ` sütunu düz sayıdır,
biçim hilesi yoktur** — otomatik çıkarımda hep onu kullan.

*(Bu, 2026-09-02'de yaşandı: ham değer okundu, "dosya hatalı" sanıldı, sahibin itirazıyla
düzeltildi.)*

## 6. 32-bit değerler — iki tuzak

Sayaçlar 32-bit, register'lar 16-bit. Yani bir sayaç **iki register**tir.

### Tuzak 1: word sırası

Delta'da 32-bit değer `D300` (düşük word) ve `D301` (yüksek word) olarak tutulur. Modbus
üzerinden okurken sıranın doğru olduğunu **bench'te bilinen bir değerle** doğrula.

**Yanlış almak sessiz ve çok pahalı bir hatadır:** sayaç normal görünür, sadece rollover
anında saçmalar ve o zaman kimse sebebini anlamaz.

**Test:** PLC'ye `0x00010002` gibi asimetrik bir değer yaz, okuduğunu karşılaştır.

### Tuzak 2: atomik okuma

İki word'ü **ayrı** Modbus isteklerinde okursan, tam iki okuma arasında sayaç artarsa
tutarsız değer alırsın (düşük word yeni, yüksek word eski).

**Çözüm: tek istekte oku.**
```python
rr = client.read_holding_registers(address=0x112C, count=2, slave=1)
```
Collector kodunda bu bir yorum satırıyla işaretlenecek.

### Rollover

32-bit sayaç `4294967295`'ten `0`'a döner. Fark hesabı negatif çıkarsa `+4294967296`
eklenir — bu düzeltme **sunucuda**, SQL'de yapılır (bkz. `12-sql-ve-timescaledb.md`).

## 7. Serbest akan sayaç kuralı

**Sayaçlar PLC'den asla sıfırlanmaz** (brief Karar #3). Farkları sunucuda hesaplarız.

**Neden:** sıfırlanan bir sayaçta bir poll kaçarsa parçalar **sessizce kaybolur**. Serbest
akan sayaçta iki saat ölü kalan bir Pi döndüğünde sayacı okur, son bildiği değerle
karşılaştırır ve fark bozulmamıştır. Sistem kendini iyileştirir.

Sıfırlanan sayacı olan makineler config'de `resetting: true` ile işaretlenir — bu bir
istisnadır, kural değil.

## 8. Python tarafı — `pymodbus`

```python
from pymodbus.client import ModbusSerialClient

client = ModbusSerialClient(
    port="/dev/andon-rs485",   # udev takma adı
    baudrate=19200,
    bytesize=8, parity="N", stopbits=1,
    timeout=1,
)
client.connect()

rr = client.read_holding_registers(address=0x112C, count=2, slave=1)
if rr.isError():
    log.warning("read failed")
else:
    lo, hi = rr.registers          # word sirasi DOGRULANACAK
    parts_total = (hi << 16) | lo
```

`(hi << 16) | lo` = yüksek word'ü 16 bit sola kaydır, düşük word'le birleştir → 32-bit sayı.

### Simülatör — donanımsız geliştirme

`pymodbus` bir **slave simülatörü** de içerir. §5.1 register haritasını taklit eden bir
simülatör yazarsan (sayaç artıran, durum değiştiren, arıza kodu üreten), collector'ı, ingest
servisini, veritabanı şemasını ve OEE SQL'ini **hiç PLC'ye dokunmadan** geliştirebilirsin
(brief §14.1).

Ayrıca sadece simülatörle yapılabilecek testler: rollover, ağ kesintisi, geç gelen veri.
Gerçek PLC'de sayacı 0xFFFFFFF0'a getiremezsin.

## 9. Linux tarafı — seri port

```bash
ls -l /dev/ttyUSB*                    # adaptör görünüyor mu
dmesg | tail -20                      # az önce takılan cihaz ne olarak tanındı
sudo usermod -aG dialout andon        # kullanıcıya seri port izni (sonra yeniden giriş)
```

**udev kuralı zorunlu:** `/dev/ttyUSB0` reboot'lar arasında sabit **değildir**. Adaptörün
seri numarasına sabit bir isim bağlanır: `/dev/andon-rs485` (brief §9.1 adım B).

## 10. En pahalı hata — yazmadan önce oku

**Sıra disiplini (brief §5.5):**

1. COM2'yi yapılandır
2. Adaptörü bağla
3. **Mevcut bir D register'ı OKU** ve beklediğin değeri gördüğünü doğrula
4. *Ancak ondan sonra* sayaç rungları yükle

**Doğrulanmamış bir adrese üretim PLC'sinde yazmak, bu projedeki en pahalı hatadır.**

Ayrıca DVP'ye download **STOP modu** gerektirir — yani 13 makinede planlı duruş. Bu teknik
olmaktan çok bir planlama problemi ve gecikmenin en olası kaynağıdır (brief §5.5).

## 11. PLC'ye hiç dokunulamayan makineler

Grup 4 (program yok, vendor desteği yok) için makineyi değil, **davranışını** okuyoruz
(brief §2):

- **Kule lambası tap'i** — optokuplörle Pi GPIO'ya. Çalışıyor/uyarı/arıza durumu, ~5 €
- **Fotoelektrik/endüktif sensör** — çıkış oluğunda parça sayımı, ~30 €
- **Akım trafosu (CT) klempi** — motor beslemesinde çalışıyor/boşta ayrımı

**Bu tekniği her makine için yedekte tut** — PLC değişikliği politik veya pratik olarak
imkânsız çıkan her makinede aynı yol geçerli.

⚠️ Kule lambası tap'i **emniyet devresine müdahale etmez**: optokuplör lamba hattına paralel
bağlanır, akım çekmez, arıza durumunda lamba davranışını değiştirmez (brief §13.4).

---

## Word sırası (word order) — ölçüldü, artık tahmin değil

### Problem nedir

Modbus register'ları **16 bit**tir. Yani tek bir register en fazla 65535 tutabilir. Bir üretim
sayacı bunu bir vardiyada aşar. Çözüm: iki ardışık register kullanıp 32 bit yapmak.

```
D300 = 0x0002   (16 bit)
D301 = 0x0001   (16 bit)
                 ────────
birleşince     = 32 bit sayı
```

Ama **hangisi düşük yarım?** İki ihtimal var ve ikisi de yaygın:

| Varsayım | Formül | Sonuç (D300=2, D301=1) |
|---|---|---|
| `lo_hi` — ilk register düşük | `(reg[1] << 16) \| reg[0]` | 65538 |
| `hi_lo` — ilk register yüksek | `(reg[0] << 16) \| reg[1]` | 131073 |

İkisi arasında **65536 kat** fark var. Ve kötü haber: yanlış seçersen sistem *çalışıyor gibi
görünür*. Sayaç artar, grafik yükselir, kimse fark etmez — ta ki sayaç 65536'yı geçip
rollover olana kadar. O anda üretim rakamı saçmalar ve kimse neden olduğunu bilemez.

Bu yüzden bu, tahmin edilecek değil **ölçülecek** bir şeydir.

### `<<` ve `|` ne demek

İki bit operatörü:

**`x << 16`** — sola kaydırma (left shift). Sayının bitlerini 16 basamak sola iter.
Etkisi: `x × 65536`. Yani sayıyı "yüksek yarıya" yerleştirir.

```
      1  =  0000 0000 0000 0001
1 << 16  =  0000 0000 0000 0001 0000 0000 0000 0000   (= 65536)
```

**`a | b`** — bit-OR. İki sayıyı bit bit birleştirir; herhangi birinde 1 varsa sonuç 1.
Yüksek yarım ile düşük yarım çakışmadığı için burada "yan yana yapıştır" anlamına gelir.

```
0000...0001 0000 0000 0000 0000   (65536)
0000...0000 0000 0000 0000 0010   (2)
─────────────────────────────── OR
0000...0001 0000 0000 0000 0010   (65538)
```

PLC tarafında karşılığı: `DMOV` gibi 32-bit komutların iki register'ı nasıl birleştirdiği.
Aynı iş, farklı dil.

### Ölçüm — 2026-09-02, bench DVP-SS2

**Yöntem:** WPLSoft device monitor'den elle `D300 = 2`, `D301 = 1` girildi. Sonra `0x112C`
adresinden **tek istekte** 2 register okundu.

**Sonuç:** `[2, 1]` → **`lo_hi`**. İlk register düşük word.

Değerler neden 2 ve 1: küçük, birbirinden farklı, karışma ihtimali yok. Simetrik bir değer
(`1, 1`) hiçbir şey kanıtlamazdı.

**Bu sonuç Delta DVP ailesine aittir.** Başka bir marka geldiğinde aynı test o marka için
tekrarlanır. `machines.yaml`'daki `order` alanı bu yüzden makine bazlıdır — evrensel bir
sabit değildir.

### Neden tek istekte okunur

```python
# DOĞRU - atomik
rr = client.read_holding_registers(address=0x112C, count=2, slave=1)

# YANLIŞ - iki ayrı istek
lo = client.read_holding_registers(address=0x112C, count=1, slave=1)
hi = client.read_holding_registers(address=0x112D, count=1, slave=1)
```

Yanlış versiyonda iki istek arasında birkaç milisaniye geçer. Sayaç tam o anda 65535'ten
65536'ya geçerse: düşük word yeni (0), yüksek word eski (0) okunur → sonuç 0. Üretim sayacı
bir anda sıfırlanmış görünür.

Bu, yılda birkaç kez olan ve log'a hiçbir hata yazmayan türden bir hatadır. Tek istek bunu
protokol seviyesinde imkânsız kılar: PLC iki register'ı aynı tarama anında paketler.

### Rollover'a dayanıklı fark hesabı

Sayaç 32-bit maksimumu (4.294.967.295) aşınca başa döner. Fark hesabı bunu ele almalı:

```python
delta = (yeni - eski) & 0xFFFFFFFF
```

`& 0xFFFFFFFF` = "sadece alttaki 32 biti al". Negatif çıkan farkı otomatik olarak doğru
pozitif değere çevirir. Çıplak `yeni - eski` yazarsan rollover anında büyük negatif bir sayı
alırsın ve o vardiyanın üretimi eksi görünür.
