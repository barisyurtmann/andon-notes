# Python — Servis Yazmak İçin Gereken Kadarı

Amaç Python öğrenmek değil, **sürekli çalışan, çökmeyen, logunu düzgün tutan bir servis
yazmak.** Bu projede yazacağın her şey aşağıdaki yapılardan oluşuyor.

**SCL/ST yazabiliyorsan yarısı zaten tanıdık gelecek.**

## 1. Temel yapılar

### Değişkenler ve tipler

```python
sayac = 0                    # int (tamsayı)
sicaklik = 43.9              # float (ondalık)
makine = "M001"              # str (metin)
calisiyor = True             # bool
```

Tip belirtmene gerek yok, Python kendisi anlar. Ama tipler önemlidir: `"1" + 1` hata verir.

### Koleksiyonlar

```python
liste = ["M001", "M002", "M003"]        # list — sıralı, tekrar edebilir
liste.append("M004")
print(liste[0])                          # M001  (sayma 0'dan başlar)

sozluk = {"id": "M001", "slave": 1}     # dict — anahtar/değer çiftleri
print(sozluk["slave"])                   # 1
```

**`dict` bu projede en çok kullanacağın yapı:** YAML config okunduğunda bir `dict` olur,
MQTT mesajı `dict` olarak hazırlanıp JSON'a çevrilir.

### Koşul ve döngü

```python
if state == 2:
    print("calisiyor")
elif state == 3:
    print("ariza")
else:
    print("diger")

for makine in liste:
    print(makine)

while True:              # sonsuz döngü — servisin kalbi
    oku()
    time.sleep(1)        # 1 Hz poll (brief 6.1)
```

⚠️ **Python'da girinti sözdizimin parçasıdır.** Süslü parantez yok; blokları boşluk belirler.
YAML'daki kuralın aynısı: **TAB değil boşluk**, tutarlı olsun (4 boşluk standarttır).

### Fonksiyon

```python
def register_oku(adres, adet=1):
    """Bir Modbus register'ı okur."""
    ...
    return deger

sonuc = register_oku(0x112C, adet=2)
```

`adet=1` = varsayılan değer; çağırırken vermezsen 1 kullanılır.

## 2. Hata yakalama — servisin çökmemesi için

```python
try:
    deger = plc_oku()
except ConnectionError as e:
    log.warning("PLC cevap vermedi: %s", e)
    deger = None
except Exception as e:
    log.exception("Beklenmeyen hata")
finally:
    baglanti_kapat()
```

**Bu projede kritik:** PLC bir poll'a cevap vermezse servis çökmemeli, o turu atlayıp devam
etmeli. Kabloyu birinin çekmesi normal bir olaydır, felaket değil.

**Kural:** `except Exception` ile her şeyi yutup sessiz kalma — **mutlaka logla.** Sessizce
yutulan hata, üç ay sonra "veri neden eksik" sorusuna dönüşür.

## 3. Logging — `print` değil

```python
import logging

log = logging.getLogger("collector")
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(name)s %(message)s",
)

log.debug("ayrintili bilgi")      # normalde gösterilmez
log.info("M001 baglandi")         # normal akış
log.warning("poll gecikti")       # dikkat
log.error("PLC cevap vermiyor")   # hata, ama servis ayakta
log.exception("beklenmeyen")      # hata + tam iz (traceback)
```

| Neden `print` değil | Açıklama |
|---|---|
| Seviye | `INFO`/`ERROR` ayrımı yapabilirsin; üretimde `DEBUG` kapatılır |
| Zaman damgası | Otomatik |
| `journalctl` uyumu | `systemd` servisinde çıktı doğrudan log sistemine düşer |
| Kaynak | Hangi modülün yazdığı belli olur |

**Kural:** servis kodunda `print` yok, `logging` var.

## 4. YAML config okuma

```python
import yaml

with open("machines.yaml", "r", encoding="utf-8") as f:
    cfg = yaml.safe_load(f)

for m in cfg["machines"]:
    print(m["id"], m["driver"])
```

- **`safe_load` kullan**, `load` değil — `load` dosyadaki kodu çalıştırabilir, güvenlik açığı
- `with open(...)` = dosyayı aç, işi bitince otomatik kapat
- Sonuç bir `dict`: `cfg["machines"]` bir liste, her elemanı bir `dict`

**Neden config ayrı:** brief §5.1 — makine ekleme maliyeti bir config satırı olmalı, kod
değişikliği değil.

## 5. JSON — MQTT mesajı için

```python
import json

mesaj = {
    "machine_id": "M001",
    "ts": "2026-08-29T09:14:03Z",
    "counters": {"total": 1048576},
    "state": 2,
}
metin = json.dumps(mesaj)        # dict → JSON metni (göndermek için)
geri = json.loads(metin)         # JSON metni → dict (almak için)
```

JSON ile `dict` neredeyse aynı şeydir; JSON ağda taşınan metin hali.

## 6. Zaman

```python
import time
from datetime import datetime, timezone

time.sleep(1)                                  # 1 saniye bekle
simdi = datetime.now(timezone.utc)             # UTC zaman
metin = simdi.isoformat().replace("+00:00","Z")
```

**Kural (brief Karar #19): her şey UTC saklanır.** Gösterim katmanı `Europe/Istanbul`'a
çevirir. Türkiye'de DST yok ama kural yine geçerli — vardiya sınırı hesapları ve geç gelen
veri bunu gerektirir.

## 7. Servis iskeleti — bu projede her şey bu kalıpta

```python
#!/usr/bin/env python3
"""Andon collector - reads one machine and publishes to MQTT."""

import logging
import time
import yaml

log = logging.getLogger("collector")


def load_config(path="machines.yaml"):
    with open(path, "r", encoding="utf-8") as f:
        return yaml.safe_load(f)


def poll_once(machine):
    """Read one machine. Returns a dict, or None on failure."""
    try:
        ...
        return {"machine_id": machine["id"], "state": 2}
    except Exception:
        log.exception("poll failed for %s", machine["id"])
        return None


def main():
    logging.basicConfig(level=logging.INFO,
                        format="%(asctime)s %(levelname)s %(message)s")
    cfg = load_config()
    log.info("started with %d machine(s)", len(cfg["machines"]))

    while True:                          # servisin kalbi
        for machine in cfg["machines"]:
            data = poll_once(machine)
            if data:
                publish(data)            # MQTT
        time.sleep(1)                    # 1 Hz


if __name__ == "__main__":
    main()
```

`if __name__ == "__main__":` = "bu dosya doğrudan çalıştırıldıysa `main()`'i çağır".
Başka bir dosyadan içe aktarılırsa çalışmaz. Standart kalıp, ezberle yeter.

## 8. Kütüphane kurma

```bash
sudo apt install python3-pip
python3 -m venv .venv                 # sanal ortam oluştur
source .venv/bin/activate             # etkinleştir (istem başında (.venv) görünür)
pip install pymodbus paho-mqtt pyyaml
deactivate                            # çık
```

**Sanal ortam (venv) nedir:** projeye özel, sistemden ayrı bir Python paket klasörü. İçine
kurduğun kütüphaneler sadece o projede görünür.

**Neden:** iki proje aynı kütüphanenin farklı sürümlerini isterse çakışır. Ayrıca sistem
Python'ına paket kurmak `apt`'ın yönettiği dosyaları bozabilir — yeni Debian sürümleri bu
yüzden doğrudan `pip install` yapmayı engeller.

**`source` ne yapıyor:** normalde bir script çalıştırdığında **yeni bir kabuk** açılır, script
biter, o kabuk kapanır — yaptığı değişiklikler senin kabuğunu etkilemez. `source` scripti
*senin* kabuğunda çalıştırır; böylece `activate` senin `PATH`'ini değiştirebilir ve `python3`
yazdığında `.venv` içindeki Python çalışır.

Çalıştığını istem satırından anlarsın: `(.venv) andon@andon-bench:~ $`
Çıkmak için `deactivate`. **Her yeni SSH oturumunda tekrar `source` yapılır** — kabuk
kapanınca ayar gider.

**`pip install "paket[ek]==sürüm"` çözümlemesi:**

| Parça | Anlamı |
|---|---|
| `pip` | Python'un paket yöneticisi (`apt`'ın Python karşılığı) |
| `pymodbus` | Kütüphane adı |
| `[serial]` | **Opsiyonel ek** — seri port desteği ayrı bir bağımlılık (`pyserial`); köşeli parantez "onu da kur" demek |
| `==3.6.9` | **Sürüm sabitleme** |
| Tırnak | Köşeli parantezi kabuk kendi işareti sanar; tırnak onu düz metin yapar |

**Sürüm sabitlemek neden önemli:** `pymodbus`'ın API'si sürümler arasında değişti — fonksiyon
isimleri ve import yolları farklı. Sabitlemezsen bugün çalışan kod altı ay sonra kurduğunda
çalışmayabilir.

`requirements.txt` ile bağımlılıklar sabitlenir:
```
pymodbus==3.6.9
paho-mqtt==2.1.0
PyYAML==6.0.2
```
→ `pip install -r requirements.txt`

**Sürüm sabitlemek önemli:** altı ay sonra kurduğunda aynı sürümler gelsin, davranış
değişmesin.

## 9. Bu projede kullanılacak kütüphaneler

| Kütüphane | Ne için |
|---|---|
| `pymodbus` | Modbus RTU/TCP okuma; ayrıca **slave simülatörü** içerir (brief §14.1) |
| `paho-mqtt` | MQTT publish/subscribe |
| `PyYAML` | Config okuma |
| `gpiozero` | GPIO — buton ve röle |
| `python-snap7` | S7-1500'den motor kodu okuma (sunucuda) |
| `psycopg` | TimescaleDB'ye yazma (ingest servisi) |

## 10. Öğrenmene gerek olmayanlar

Sınıflar (başta), `async`/`await`, tip sistemi, dekoratörler, Flask/Django/FastAPI
(web app'e gelene kadar), unit test framework'leri (basit `assert` yeter).

Bu ölçekte fonksiyon + `dict` + `try/except` + `logging` ile her şey yazılır. Karmaşıklığı
gerçekten ihtiyaç duyunca ekle.

**Kaynak:** *Automate the Boring Stuff with Python*, `automatetheboringstuff.com`,
**Bölüm 1–6 yeter**, online ücretsiz.


## venv — projenin kendi Python'u

Python'da kütüphaneler (`pymodbus` gibi) bilgisayara kurulur. Sorun: iki proje aynı
kütüphanenin farklı sürümünü isterse çakışırlar.

**venv** = o projeye ait, kendi kütüphanelerini taşıyan ayrı bir Python kopyası.

```
~/.venv/
├── bin/python        ← projenin kendi Python'u
├── bin/pip           ← projenin kendi kurulum aracı
└── lib/...           ← pymodbus burada, sistemde değil
```

**Pano karşılığı:** her makinenin kendi panosu olması. Ortak pano da olurdu, ama biri
diğerini bozar.

### `activate` ne yapıyor

```bash
source ~/.venv/bin/activate
```

Kabuğa şunu der: *"bundan sonra `python3` yazınca sistemdekini değil, buradakini kullan."*
Bir kısayol tanımlar, başka bir şey yapmaz.

`source .venv/bin/activate` **bulunduğun klasörde** arar. venv başka yerdeyse çalışmaz —
2026-09-02'de bench Pi'sinde tam olarak bu oldu: venv `~/.venv` içindeydi ama komut
`~/andon-lab` içinden çalıştırılıyordu.

### Bu projedeki kural: `activate` kullanma

Kısayol yerine tam adresi yaz:

| Yol | Sonuç |
|---|---|
| `source ~/.venv/bin/activate` sonra `python3 collector.py` | Çalışır, ama "aktif miydi" şüphesi kalır |
| `~/.venv/bin/python collector.py` | Aynı şey, şüphe yok |

**Neden ikincisi:**

1. venv'in nerede olduğu önemsizleşir.
2. Yanlış Python'la çalışma ihtimali sıfır.
3. **systemd zaten bunu yapar.** Servisin kabuğu yoktur, `activate` çalıştıramaz:

```
ExecStart=/opt/andon/.venv/bin/python3 /opt/andon/andon-collector/collector.py
```

Yani `activate`'i öğrenmek yerine doğrudan yolu yazmayı alışkanlık edinmek, servis
yazarken zaten yapman gereken şeyi şimdiden yapmak demektir.

### Kütüphane kurmak

```bash
~/.venv/bin/pip install "pymodbus[serial]==3.6.9"
~/.venv/bin/pip install -r requirements.txt     # dosyadaki hepsini kur
~/.venv/bin/pip list                            # kurulu olanları göster
```

`==3.6.9` sürümü **sabitler**. Brief §6: pymodbus 3.x içinde API kırıldı; sürüm
sabitlenmeden filoya çıkılmaz. `requirements.txt` bu sabitlemenin yazılı hali.
