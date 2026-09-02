# andon-notes

Andon & OEE projesinin çalışma kaydı ve ders notları.

| Yol | Ne için |
|---|---|
| `CALISMA-GUNLUGU.md` | Tarihli iş kaydı: ne yapıldı, ne ölçüldü, ne karar verildi |
| `ders-notlari/` | Kavram ve komut notları. Temel baştan atıldı; kullandıkça büyür |

Mimari kararların gerekçeleri `andon-proje-brifi.md` içinde, öğrenme aşamaları
`andon-ogrenme-yolu.md` içinde (ikisi de Claude Project'te; her yeni sohbet oradan okuyor).

**İş bölümü:** notları Claude yazar ve commit'ler, sahibi okur ve push'lar.

## Ders notları

| Dosya | İçerik | Ne zaman okunur |
|---|---|---|
| `00-nasil-kullanilir.md` | Klasörün kuralları, okuma sırası, dizin | İlk |
| `01-linux-temelleri.md` | Kabuk, dosya sistemi, komutlar, izinler, süreçler | Şimdi |
| `02-ssh-ve-uzak-erisim.md` | SSH, parmak izi, anahtar çiftleri | Şimdi |
| `03-git.md` | Git vs GitHub, commit/branch/remote, `.gitignore` | Şimdi |
| `04-raspberry-pi-donanim.md` | Kart, güç, GPIO, ısı, SD ömrü, iş güvenliği | Şimdi |
| `05-raspberry-pi-yazilim.md` | Pi OS, Imager, `raspi-config`, overlay FS, kiosk | Şimdi |
| `06-yaml-ve-config.md` | YAML sözdizimi, config'i koddan ayırma | Faz 1 |
| `07-ag-temelleri.md` | IP, subnet, DHCP, MAC, port, VLAN | Şimdi |
| `08-systemd-servisler.md` | Servis dosyası, `systemctl`, `journalctl`, timer | Aşama 3 |
| `09-python-servis-temelleri.md` | Servis yazmak için gereken kadar Python | Aşama 3 |
| `10-mqtt.md` | Topic, QoS, retained, LWT, idempotency | Faz 1 |
| `11-docker.md` | İmaj, konteyner, volume, compose | Faz 1 adım D |
| `12-sql-ve-timescaledb.md` | SQL, window fonksiyonları, hypertable | Faz 1 sonu |
| `13-modbus-ve-seri-haberlesme.md` | RS-485, Modbus, 32-bit tuzakları, `pymodbus` | Faz 1 adım E |
| `14-ansible-ve-filo-yonetimi.md` | Inventory, playbook, Vault | Faz 2 |
| `15-grafana-ve-gosterim.md` | Dashboard, kiosk, Grafana vs Power BI | Faz 5 |
| `16-collector-ve-surucu-mimarisi.md` | Sürücü deseni, rollover, mesaj şeması, simülatör | Faz 1 adım E |
| `99-komut-sozlugu.md` | Tüm komutlar, tek satırlık karşılıklarıyla | Her zaman |
