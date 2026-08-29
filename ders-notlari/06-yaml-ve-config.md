# YAML ve Config Dosyaları

## YAML nedir

Bir programın çalışırken okuyacağı **ayarları** tutan metin dosyası formatı. Program kodun
kendisi değil; koda "hangi makine, hangi port, hangi adres" diye söyleyen liste.

**Otomasyon karşılığı:** ladder programın sabittir — setpoint'leri, zamanlayıcı değerlerini,
istasyon adreslerini HMI'dan veya bir parametre tablosundan verirsin. Programı her makine
için baştan yazmazsın, tabloyu değiştirirsin. YAML o parametre tablosunun Linux karşılığı.

**Neden JSON veya XML değil:** JSON'da yorum satırı yazamazsın ve bu projede `# TODO: VERIFY`
işaretleri hayati. XML gereksiz gürültülü. Ayrıca Docker, Ansible ve Grafana zaten YAML
kullanıyor — bir kez öğrenip dört yerde kullanacaksın.

## Neden config koddan ayrılır

Brief §5.1'in temel kararı: **makine ekleme maliyeti bir config satırı olmalı, kod
değişikliği değil.**

Alternatifi şu olurdu: her makinenin adresi Python kodunun içine yazılır. Sonucu ya 16 ayrı
kod dosyası (yani 16 ayrı proje), ya da içinde 16 `if` bulunan tek dosya. İkincisinde
9 numaralı makinenin adresini düzeltmek için kodu değiştirmen, test etmen ve 16 Pi'ye
yeniden dağıtman gerekir.

İkinci sebep: **kod her Pi'de aynıdır ve git'te sürüm kontrollüdür; config her Pi'de
farklıdır.** İkisi birbirine karışırsa Ansible ile filo yönetimi çalışmaz.

## Sözdizimi — altı kural

| Kural | Örnek | Not |
|---|---|---|
| `anahtar: değer` | `slave: 1` | İki noktadan **sonra boşluk şart** |
| Girinti = iç içelik | `map:` altındakiler 2 boşluk içeride | **TAB TUŞU YASAK** |
| `- ` = liste elemanı | `- id: M001` | 16 makine = 16 tane `- id:` bloğu |
| `#` sonrası yorum | `slave: 1   # makine no` | Program okumaz, insan okur |
| `{ }` kısa yazım | `{ addr: 0x112C, words: 2 }` | Uzun yazımla aynı şey |
| `null` | `asset_code: null` | "Değer yok" — bilerek boş |

**En sık hata, farkla:** editör TAB atar, YAML
`found character '\t' that cannot start any token` der. `nano` bu dosyada TAB atmaz ama
VS Code'da çalışırken dikkat.

**`null` neden boş bırakmaktan iyi:** boş bırakınca "unutuldu mu, bilerek mi boş" ayrımı
kaybolur. `null` yazmak "biliyorum, henüz değeri yok" demektir.

## Yapı örneği

```yaml
machines:                    # kök: "makineler" listesi
  - id: M001                 # liste elemanı 1
    driver: modbus_rtu       # bu makinenin özellikleri
    slave: 1
    map:                     # iç içe blok
      parts_total:
        addr: 0x112C
        words: 2
  - id: M002                 # liste elemanı 2
    driver: modbus_tcp
```

Girinti seviyeleri: `machines` kökte, `- id` 2 boşluk, `map` altındakiler 6 boşluk.
Girinti YAML'da süs değil, **anlamın kendisidir.**

## Sayı biçimleri

- `1` → tamsayı
- `0x112C` → onaltılık (hex) tamsayı. Modbus adresleri böyle yazılır, okunması kolay olsun diye
- `1.5` → ondalık
- `"1"` → tırnak içindeyse metin
- `true` / `false` → mantıksal

## Bizim config dosyamız: `machines.yaml`

Yeri: `andon-collector/machines.yaml` (PC'deki repo).

İçindeki alanların anlamı:

| Alan | Ne | Değişir mi |
|---|---|---|
| `id` | Anlamsız, kalıcı iç kimlik (`M001`) | **Asla** |
| `asset_code` | Fabrikanın envanter kodu (`MAK25-30-511`) | Düzeltilebilir, boş olabilir |
| `display_name` | Ekranlarda görünen ad | Değişebilir |
| `site` / `area` / `line` | Fabrika / bölüm / hat | **Değişir** — geçmişi veritabanında |
| `driver` | Hangi protokol sürücüsü yüklenecek | Makineye göre |
| `port` / `host` / `slave` | Nereye bağlanılacak | Kuruluma göre |
| `map` | Kanonik alan → makinedeki adres eşlemesi | Makineye göre |
| `map_verified` | Adresler gerçek donanımda doğrulandı mı | `false` → `true` |

**Dondurulan şey adresler değil, alan isimleridir.** `parts_total` ve `state` her makinede
aynı isimle durur; `addr` her makinede farklıdır. Bir makinede "3" arıza, başkasında "3" tip
değişimi olursa tüm OEE hesabı çöp olur (brief §5.1).

## Şablon dosyada `UNKNOWN` ve `# TODO: VERIFY`

Doğrulanmamış hiçbir değer kesin gibi yazılmaz. Bilinmeyenler `null` + `UNKNOWN` yorumuyla,
şüpheli adresler `# TODO: VERIFY` ile işaretlenir, ve dosyanın başında
`map_verified: false` durur.

Sebep brief §5.5: **doğrulanmamış bir adrese üretim PLC'sinde yazmak bu projedeki en pahalı
hatadır.** Yer tutucu bir değer, altı ay sonra doğrulanmış veri gibi görünür.

## Nerede daha karşılaşacağız

| Dosya | Ne zaman |
|---|---|
| `docker-compose.yml` | Sunucuda Mosquitto + TimescaleDB kurarken |
| Ansible playbook / inventory | Faz 2, filo yönetimi |
| Grafana provisioning | Faz 5, dashboard'ları elle tıklamadan kurmak |

Sözdizimi hepsinde aynı; sadece anahtar isimleri değişir.
