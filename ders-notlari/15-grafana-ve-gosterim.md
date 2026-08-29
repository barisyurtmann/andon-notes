# Grafana ve Gösterim Katmanı

**Ne zaman lazım: Faz 1 sonu (tek panel) ve Faz 5 (asıl iş).** En kolay konu, en sona bırakıldı.

## 1. Grafana nedir

Veritabanına bağlanıp grafik ve gösterge çizen açık kaynak dashboard aracı. Kendi verisi
yoktur — TimescaleDB'ye SQL sorar, sonucu çizer.

## 2. İki farklı iş, iki farklı araç

Brief §6.4'ün ayrımı — **birini ikisi için kullanma:**

| Katman | Araç | Neden |
|---|---|---|
| Makine ekranı (operatör) | **Özel web app** | Doküman görüntüleyici + Andon butonları; Grafana bunu yapamaz |
| Andon panosu (tavan ekranı) | **Grafana kiosk** | Yerel, ücretsiz, sağlam kiosk modu, saniye seviyesi yenileme |
| Mühendislik / hata ayıklama | **Grafana** | Ham veriye hızlı bakış |
| Yönetim raporlama, aylık analiz | **Power BI** | Keşifsel dilimleme, yönetimin konuştuğu dil |

### Saha ekranlarında neden Power BI değil

- **Gerçekten canlı değil.** Import modda zamanlanmış yenileme; DirectQuery sadece yaklaşır
  ve veritabanını yorar
- **Kiosk modu kırılgan** — oturum düşer, login ekranı gelir ve üç gün kimse fark etmez
- **Ekran başına lisans** çözülmemiş bir soru
- **Bulut barındırmalı** — bir internet kesintisi Andon panosunu öldürür, bu da §7.1 ile
  çelişir

### Katı kısıt

**Power BI asla ham telemetri tablolarına bağlanmaz**, sadece aggregate katmanını okur.
İki sebep:
1. Sorgu performansı — günde 1,4 M satırda Power BI yavaşlar ve canlı sistemi yorar
2. **Tanım kayması** — OEE mantığı DAX'ta yeniden yazılırsa Grafana/SQL tanımından ayrışır
   ve ortaya **iki farklı OEE sayısı** çıkar. Tanım SQL'de, tek yerde yaşar; iki araç da
   onu tüketir

## 3. Kavramlar

| Terim | Ne demek |
|---|---|
| **Data source** | Veri kaynağı bağlantısı (bizde TimescaleDB) |
| **Dashboard** | Bir sayfa dolusu panel |
| **Panel** | Tek bir grafik, tablo veya gösterge |
| **Query** | Panelin çalıştırdığı SQL |
| **Variable** | Dashboard üstündeki açılır liste (makine seç, hat seç) |
| **Alert** | Bir eşik aşılınca bildirim |
| **Kiosk mode** | Menüsüz, tam ekran gösterim |
| **Provisioning** | Dashboard ve data source'ları YAML dosyasından otomatik kurma |

## 4. Bu projede kurulacaklar

### Faz 1 — tek panel

"M001 son 24 saat: üretim sayacı ve durum." Fazlası değil. Amaç veri akışının çalıştığını
görmek.

### Faz 5 — asıl dashboard'lar

| Dashboard | İçerik |
|---|---|
| **Andon panosu** (tavan ekranı) | Aktif çağrılar, hangi makine, kaç dakikadır bekliyor. Büyük yazı, uzaktan okunur |
| **Filo sağlığı** | 16 yeşil/kırmızı nokta — hangi Pi ayakta (MQTT LWT'den) |
| **OEE görünümü** | Availability / Performance / Quality, makine ve vardiya bazında |
| **Duruş Pareto'su** | Sebep kodlarına göre sıralı çubuk grafik — yönetimin ilk isteyeceği şey |
| **Mühendislik** | Ham telemetri, hata ayıklama için |

## 5. Kiosk modu

Tavan ekranındaki Pi tam ekran Chromium ile Grafana'yı açacak:

```
http://sunucu:3000/d/DASHBOARD_ID?kiosk&refresh=5s
```

- `&kiosk` = menü ve kenar çubukları gizlenir
- `&refresh=5s` = 5 saniyede bir yenile
- Anonim erişim veya sabit bir görüntüleyici hesabı gerekir — ekran login soramaz

## 6. Provisioning — elle tıklama yok

Dashboard'ları arayüzden kurup orada bırakmak, bus factor açısından "kafada tutmak"la aynı
şeydir. Doğrusu: dashboard tanımlarını **JSON olarak git'e koymak** ve Grafana'yı YAML
provisioning dosyalarıyla kurmak.

```yaml
# grafana/provisioning/datasources/timescale.yml
apiVersion: 1
datasources:
  - name: TimescaleDB
    type: postgres
    url: timescaledb:5432
    database: andon
    user: grafana
    secureJsonData:
      password: ${GRAFANA_DB_PASSWORD}
    jsonData:
      sslmode: disable
```

Böylece sunucu ölürse `docker compose up` her şeyi geri getirir — dashboard'lar dahil
(brief §8.2, §15).

## 7. Yetki

Brief §13.1: **Grafana veritabanına sadece `SELECT` yetkisiyle bağlanır.** Superuser değil.
Bir dashboard sorgusunun yanlışlıkla veri silmesi mümkün olmamalı.

## 8. Operatör güveni — ekran tasarımı kuralları

Brief §13.3, teknik bir konu kadar önemli:

- Makine ekranında operatör **kendi verisini görebilsin** (vardiya üretimi, hedef, kendi
  Andon çağrılarının cevap süresi)
- Sebep kodu ekranında **"Bilinmiyor" ve "Diğer" her zaman olsun** — zorlanan cevap uydurma
  cevaptır
- **Hiçbir ekranda operatör isimleri sıralanmış bir "performans" listesi olmasın**
- Ekranlarda makinenin **kendi adı ve kodu** yazsın (`display_name` / `asset_code`), iç
  kimlik (`M001`) değil

**Neden burada yazıyor:** projeyi öldürmesi en olası şey teknik bir arıza değil, operatörün
sistemi gözetleme olarak algılamasıdır (brief §8.3). Dashboard tasarımı bunun en görünür
yeridir.

## 9. Öğrenme

Deneyerek öğrenilir, 2–3 gün. Önce SQL'i bil — Grafana sadece SQL'in sonucunu çiziyor.
Panelde takıldığın yer genelde Grafana değil, sorgudur.
