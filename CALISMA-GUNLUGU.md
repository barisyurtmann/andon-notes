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

### Sıradaki adım

Öğrenme yolu **Aşama 2** — kabuk komutları ve git akışı.
