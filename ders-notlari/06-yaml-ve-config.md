# YAML ve Config Dosyaları

## 1. YAML nedir

Bir programın çalışırken okuyacağı **ayarları** tutan metin dosyası formatı. Kodun kendisi
değil; koda "hangi makine, hangi port, hangi adres" diye söyleyen liste.

**Otomasyon karşılığı:** ladder programın sabittir — setpoint'leri, zamanlayıcı değerlerini,
istasyon adreslerini HMI'dan veya bir parametre tablosundan verirsin. Programı her makine
için baştan yazmazsın, tabloyu değiştirirsin. YAML o parametre tablosunun karşılığı.

**Neden JSON veya XML değil:** JSON'da yorum satırı yazamazsın ve bu projede
`# TODO: VERIFY` işaretleri hayati. XML gereksiz gürültülü. Ayrıca Docker, Ansible ve
Grafana zaten YAML kullanıyor — bir kez öğrenip dört yerde kullanacaksın.

## 2. Neden config koddan ayrılır

Brief §5.1'in temel kararı: **makine ekleme maliyeti bir config satırı olmalı, kod
değişikliği değil.**

Alternatifi: her makinenin adresi Python kodunun içine yazılır. Sonucu ya 16 ayrı kod dosyası
(yani 16 ayrı proje), ya da içinde 16 `if` bulunan tek dosya. İkincisinde 9 numaralı
makinenin adresini düzeltmek için kodu değiştirmen, test etmen ve 16 Pi'ye yeniden dağıtman
gerekir.

İkinci sebep: **kod her Pi'de aynıdır ve git'te sürüm kontrollüdür; config her Pi'de
farklıdır.** İkisi birbirine karışırsa Ansible ile filo yönetimi çalışmaz.

## 3. Sözdizimi

### Temel kurallar

| Kural | Örnek | Not |
|---|---|---|
| `anahtar: değer` | `slave: 1` | İki noktadan **sonra boşluk şart** |
| Girinti = iç içelik | `map:` altındakiler 2 boşluk içeride | **TAB TUŞU YASAK** |
| `- ` = liste elemanı | `- id: M001` | 16 makine = 16 tane `- id:` bloğu |
| `#` sonrası yorum | `slave: 1   # makine no` | Program okumaz |
| `{ }` `[ ]` kısa yazım | `{ addr: 0x112C, words: 2 }` | Uzun yazımla aynı şey |
| `null` | `asset_code: null` | "Değer yok" — bilerek boş |
| `---` | Dosya başında | Belge ayırıcı; birden fazla belge olabilir |

**En sık hata, farkla:** editör TAB atar, YAML
`found character '\t' that cannot start any token` der. `nano` bu dosyada TAB atmaz ama
VS Code'da çalışırken dikkat — editörü "TAB yerine boşluk" olarak ayarla.

**`null` neden boş bırakmaktan iyi:** boş bırakınca "unutuldu mu, bilerek mi boş" ayrımı
kaybolur. `null` yazmak "biliyorum, henüz değeri yok" demektir.

### Veri tipleri

```yaml
sayi: 1                 # int
ondalik: 1.5            # float
hex: 0x112C             # onaltılık int (Modbus adresleri böyle yazılır)
metin: M001             # str
tirnakli: "1"           # str — tırnak varsa metindir
dogru: true             # bool  (yes/on da kabul edilir, ama true/false yaz)
bos: null               # None  (~ de aynı anlama gelir)
```

⚠️ **Klasik tuzak:** tırnaksız `on`, `off`, `yes`, `no`, `Y`, `N` **mantıksal** olarak okunur.
Norveç ülke kodu `NO` tırnaksız yazılırsa `false` olur. Şüphedeysen tırnak kullan.

### Liste ve sözlük

```yaml
# sözlük (anahtar/değer)
makine:
  id: M001
  slave: 1

# liste
portlar:
  - 1883
  - 502

# liste elemanları sözlük olabilir — bizim machines.yaml böyle
machines:
  - id: M001
    slave: 1
  - id: M002
    slave: 2
```

### Çok satırlı metin

```yaml
aciklama: |            # satır sonları KORUNUR
  Birinci satır
  İkinci satır

ozet: >                # satır sonları BOŞLUĞA çevrilir (tek paragraf olur)
  Bu uzun bir cümle
  ama tek satır sayılır
```

### Anchor ve alias — tekrarı azaltma

```yaml
ortak: &varsayilan
  driver: modbus_rtu
  poll_hz: 1

machines:
  - <<: *varsayilan       # yukarıdaki bloğu buraya kopyala
    id: M001
    slave: 1
  - <<: *varsayilan
    id: M002
    slave: 2
```

`&isim` tanımlar, `*isim` kullanır, `<<:` birleştirir. 16 makinede tekrarı azaltır — ama
okunabilirliği düşürür. **Şimdilik kullanmayacağız**; dosya karmaşıklaşırsa düşünürüz.

## 4. Bizim config dosyamız: `machines.yaml`

Yeri: `andon-collector/machines.yaml` (PC'deki repo, tek doğru kopya).

| Alan | Ne | Değişir mi |
|---|---|---|
| `id` | Anlamsız, kalıcı iç kimlik (`M001`) | **Asla** |
| `asset_code` | Fabrikanın envanter kodu (`MAK25-30-511`) | Düzeltilebilir, boş olabilir |
| `display_name` | Ekranlarda görünen ad | Değişebilir |
| `site` / `area` / `line` | Fabrika / bölüm / hat | **Değişir** — geçmişi veritabanında |
| `driver` | Hangi protokol sürücüsü yüklenecek | Makineye göre |
| `port` / `host` / `slave` | Nereye bağlanılacak | Kuruluma göre |
| `poll_hz` | Saniyede kaç okuma | Genelde 1 |
| `map` | Kanonik alan → makinedeki adres eşlemesi | Makineye göre |
| `map_verified` | Adresler gerçek donanımda doğrulandı mı | `false` → `true` |

**Dondurulan şey adresler değil, alan isimleridir.** `parts_total` ve `state` her makinede
aynı isimle durur; `addr` her makinede farklıdır. Bir makinede "3" arıza, başkasında "3" tip
değişimi olursa tüm OEE hesabı çöp olur (brief §5.1).

**Sürücü mimarisi:** `driver: modbus_rtu` / `modbus_tcp` / `gpio_sensor` — yeni bir PLC
ailesi eklemek = yeni bir sürücü dosyası + config satırı. Ana servise dokunulmaz
(brief Karar #22).

## 5. `UNKNOWN`, `null` ve `# TODO: VERIFY`

Doğrulanmamış hiçbir değer kesin gibi yazılmaz:
- Bilinmeyenler → `null` + `# UNKNOWN` yorumu
- Şüpheli adresler → `# TODO: VERIFY`
- Dosyanın başında → `map_verified: false`

**Sebep brief §5.5:** doğrulanmamış bir adrese üretim PLC'sinde yazmak bu projedeki en
pahalı hatadır. Yer tutucu bir değer, altı ay sonra doğrulanmış veri gibi görünür.

## 6. Python'dan okuma

```python
import yaml

with open("machines.yaml", "r", encoding="utf-8") as f:
    cfg = yaml.safe_load(f)

for m in cfg["machines"]:
    print(m["id"], m["driver"])
```

**`safe_load` kullan, `load` değil** — `load` dosyadaki kodu çalıştırabilir, güvenlik açığıdır.

Sonuç bir `dict`; `cfg["machines"]` bir liste, her elemanı bir `dict`.

## 7. Doğrulama

Yazdığın YAML geçerli mi, hızlı kontrol:

```bash
python3 -c "import yaml,sys; yaml.safe_load(open('machines.yaml')); print('ok')"
```

Hata varsa satır ve sütun numarasıyla söyler. VS Code'da YAML eklentisi de anında uyarır.

## 8. Nerede daha karşılaşacağız

| Dosya | Ne zaman |
|---|---|
| `docker-compose.yml` | Sunucuda Mosquitto + TimescaleDB (brief §9.1 D) |
| Ansible playbook / inventory | Faz 2, filo yönetimi |
| Grafana provisioning | Faz 5, dashboard'ları elle tıklamadan kurmak |
| `systemd` unit dosyaları | YAML değil, INI formatı — benzer ama farklı |

Sözdizimi ilk üçünde aynı; sadece anahtar isimleri değişir.
