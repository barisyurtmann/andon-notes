# SQL ve TimescaleDB

**Bu, projenin en yüksek getirili öğrenme kalemi** (brief §7.2). OEE hesabının tamamı SQL
window fonksiyonlarıdır. Buraya para ve zaman harcamaya değer.

## 1. Veritabanı neden

Telemetri bir yerde saklanmalı ve **sorgulanabilir** olmalı. MQTT taşır, saklamaz.
Excel bu hacimde (günde 1,4 M satır) çöker.

**Seçim: PostgreSQL + TimescaleDB eklentisi.** Timescale ayrı bir ürün değil, Postgres'in
zaman serisi eklentisidir — yani önce Postgres öğrenirsin, Timescale üstüne gelir.

## 2. Tablo, satır, sütun

```sql
CREATE TABLE telemetry (
    msg_id      TEXT PRIMARY KEY,
    machine_id  TEXT NOT NULL,
    ts          TIMESTAMPTZ NOT NULL,
    received_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    ts_synced   BOOLEAN NOT NULL,
    parts_total BIGINT,
    scrap_total BIGINT,
    state       SMALLINT NOT NULL,
    fault_code  INTEGER,
    cycle_ms    INTEGER
);
```

| Kavram | Anlamı |
|---|---|
| `PRIMARY KEY` | Benzersiz kimlik. Aynı `msg_id` iki kez giremez |
| `NOT NULL` | Bu alan boş olamaz |
| `TIMESTAMPTZ` | Zaman dilimi bilgisi taşıyan zaman. **Hep bunu kullan** |
| `BIGINT` | Büyük tamsayı — 32-bit sayaç rollover'ı için gerekli |

**Kural (brief Karar #19): her şey UTC saklanır**, gösterim katmanı `Europe/Istanbul`'a çevirir.

## 3. Temel sorgular

```sql
SELECT * FROM telemetry LIMIT 10;

SELECT machine_id, ts, parts_total
FROM telemetry
WHERE machine_id = 'M001'
  AND ts >= now() - interval '1 hour'
ORDER BY ts DESC;

SELECT machine_id, count(*) AS satir
FROM telemetry
GROUP BY machine_id
ORDER BY satir DESC;
```

| Anahtar kelime | Ne yapar |
|---|---|
| `SELECT` | Hangi sütunlar |
| `FROM` | Hangi tablo |
| `WHERE` | Filtre |
| `GROUP BY` | Grupla (ve `count`, `sum`, `avg` gibi toplama fonksiyonları kullan) |
| `ORDER BY` | Sırala (`DESC` = tersten) |
| `LIMIT` | İlk N satır |
| `JOIN` | İki tabloyu birleştir |

## 4. Idempotent yazma

```sql
INSERT INTO telemetry (msg_id, machine_id, ts, state)
VALUES ('M001-8f2a1c-148213', 'M001', '2026-08-25T09:14:03Z', 2)
ON CONFLICT (msg_id) DO NOTHING;
```

`ON CONFLICT ... DO NOTHING` = "bu `msg_id` zaten varsa hiçbir şey yapma, hata da verme".

**Bu tek satır, brief Karar #18'in tamamıdır.** Store-and-forward kuyruğu yeniden bağlanınca
mesajları tekrar gönderir; QoS 1 de kopya üretebilir. Kopya kayıt oluşmamasının garantisi bu.

## 5. Window fonksiyonları — OEE'nin kalbi

Normal toplama (`GROUP BY`) satırları ezer. **Window fonksiyonu satırları korur ve her
satıra komşularına bakarak bir değer ekler.** Sayaç farkı almanın tek doğru yolu budur.

### `LAG()` — bir önceki satır

```sql
SELECT
    machine_id,
    ts,
    parts_total,
    parts_total - LAG(parts_total) OVER (
        PARTITION BY machine_id ORDER BY ts
    ) AS uretilen
FROM telemetry
WHERE machine_id = 'M001'
ORDER BY ts;
```

- `OVER (...)` = pencereyi tanımlar
- `PARTITION BY machine_id` = her makine kendi içinde hesaplansın (makineler karışmasın)
- `ORDER BY ts` = pencerede sıra zamana göre

Sonuç: her satırda "bir önceki ölçümden bu yana kaç parça üretildi".

**Neden serbest akan sayaç:** sayaç PLC'den sıfırlanmıyorsa (brief Karar #3), bir poll
kaçsa bile fark bozulmaz — iki saat ölü kalan Pi döndüğünde farkı doğru hesaplar. Sıfırlanan
sayaçta kaçan poll = sessizce kaybolan parçalar.

**32-bit rollover:** sayaç `4294967295`'ten `0`'a dönerse fark negatif çıkar. SQL'de
düzeltilir:

```sql
CASE WHEN fark < 0 THEN fark + 4294967296 ELSE fark END
```

### Gaps-and-islands — durumda geçen süre

"Makine ne kadar süre `state = 3` (arıza) kaldı?" sorusu. Yöntem: durum **değiştiği** anları
işaretle, aynı durumun ardışık satırlarını tek "ada" olarak grupla, adanın başı ile sonu
arasındaki süreyi al.

```sql
WITH degisim AS (
  SELECT machine_id, ts, state,
         CASE WHEN state IS DISTINCT FROM
              LAG(state) OVER (PARTITION BY machine_id ORDER BY ts)
              THEN 1 ELSE 0 END AS yeni_ada
  FROM telemetry
),
ada AS (
  SELECT *, SUM(yeni_ada) OVER (PARTITION BY machine_id ORDER BY ts) AS ada_no
  FROM degisim
)
SELECT machine_id, state,
       MIN(ts) AS baslangic,
       MAX(ts) AS bitis,
       MAX(ts) - MIN(ts) AS sure
FROM ada
GROUP BY machine_id, state, ada_no
ORDER BY baslangic;
```

`WITH ... AS (...)` = **CTE (common table expression)**: sorguyu adımlara bölmenin yolu.
Uzun sorguları okunabilir yapar.

**Availability, duruş Pareto'su ve MTTR hesaplarının tamamı bu desenle çıkar.**

## 6. TimescaleDB'nin eklediği

### Hypertable

```sql
SELECT create_hypertable('telemetry', 'ts');
```

Tabloyu zamana göre otomatik parçalara böler (chunk). Sen normal tablo gibi sorgularsın;
Timescale sadece ilgili zaman aralığındaki parçalara bakar → çok daha hızlı.

### Sıkıştırma

```sql
ALTER TABLE telemetry SET (timescaledb.compress);
SELECT add_compression_policy('telemetry', INTERVAL '7 days');
```

7 günden eski veriyi otomatik sıkıştırır. Brief §6.2'deki hesap: 75 GB/yıl ham →
**5–7 GB/yıl sıkıştırılmış.**

### Continuous aggregate

```sql
CREATE MATERIALIZED VIEW oee_saatlik
WITH (timescaledb.continuous) AS
SELECT machine_id,
       time_bucket('1 hour', ts) AS saat,
       max(parts_total) - min(parts_total) AS uretim
FROM telemetry
GROUP BY machine_id, saat;
```

Önceden hesaplanmış özet; **yeni veri geldikçe kendini günceller.** Dashboard'lar ham
telemetriyi değil bunu okur.

**Katı kısıt (brief §6.4): Power BI asla ham telemetriye bağlanmaz**, sadece bu aggregate
katmanını okur. İki sebep: sorgu performansı ve **tanım kayması** — OEE mantığı DAX'ta
yeniden yazılırsa iki farklı OEE sayısı çıkar. Tanım SQL'de, tek yerde yaşar.

### Retention

```sql
SELECT add_retention_policy('telemetry', INTERVAL '18 months');
```

Eski ham veriyi otomatik siler. Aggregate'ler kalır.

## 7. Katmanlı model

```
raw (ham telemetri)  →  cleaned (temizlenmiş)  →  aggregated (OEE özetleri)
```

Buna **medallion pattern** denir; bir ürün değil bir desendir ve düz SQL ile uygulanır
(brief §7.3).

**Kural: ham değerleri sakla, hesaplanmışları değil.** OEE tanımı sonradan değişirse geçmiş
yeniden türetilebilmeli.

## 8. Master veri ve yerleşim geçmişi

Telemetri sadece `machine_id` taşır. Makinenin ne olduğu ve nerede olduğu ayrı tablolarda:

```sql
CREATE TABLE machine (
    machine_id   TEXT PRIMARY KEY,
    asset_code   TEXT,
    display_name TEXT,
    plc_type     TEXT,
    created_at   TIMESTAMPTZ,
    retired_at   TIMESTAMPTZ
);

CREATE TABLE machine_assignment (
    machine_id TEXT REFERENCES machine(machine_id),
    site       TEXT,
    area       TEXT,
    line       TEXT,
    valid_from TIMESTAMPTZ NOT NULL,
    valid_to   TIMESTAMPTZ
);
```

**Neden atama tarihli:** makine hat değiştirebilir. Hat bilgisi `machine` tablosunda tek
sütun olsaydı, makine taşındığı gün **geçmişteki tüm raporlar sessizce değişirdi.**
"Geçen ay 4 INC hattının OEE'si neydi" sorusu o ayki gerçek makine listesiyle
cevaplanmalı (brief §12.5).

Veri mühendisliğinde bu desenin adı **slowly changing dimension**.

## 9. Roller ve yetkiler

Brief §13.1: collector/ingest sadece `INSERT`; Grafana sadece `SELECT`; Power BI sadece
aggregate şemaya `SELECT`. Hiçbiri superuser değil.

```sql
CREATE ROLE ingest LOGIN PASSWORD '...';
GRANT INSERT ON telemetry TO ingest;
```

## 10. Nasıl öğrenilir

- **`pgexercises.com`** — tarayıcıda, sorulu cevaplı. Window fonksiyonları bölümüne kadar git
- **PostgreSQL resmi dokümanı** — window fonksiyonları bölümü iyi yazılmış
- **En etkili yöntem:** `pymodbus` simülatörüyle sahte veri üret, OEE sorgusunu yaz, sonucu
  **elle hesapla ve karşılaştır** (brief §14.1). Gerçek veriyle bunu yapamazsın

**Süre:** 3–4 hafta, yayılarak. Faz 1 ile iç içe öğrenilir.
