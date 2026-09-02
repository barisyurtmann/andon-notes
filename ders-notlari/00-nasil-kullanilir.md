# Ders Notları — nasıl kullanılır

Bu klasör, projede ihtiyaç duyulan kavramların ve komutların sade notudur.

## Kuralları

1. **Önce temel, sonra ayrıntı.** Her konunun temeli baştan atılır; projede yeni bir şey
   kullandıkça ilgili dosyanın devamına eklenir. Amaç referans kitabı değil, altı ay sonra
   "bu neydi ya" dediğinde bir dakikada hatırlamak.
2. **Sade dil.** Karmaşık anlatım öğretmez, korkutur.
3. **Mümkünse otomasyon karşılığıyla.** PLC / ladder / pano dünyasından bir karşılığı varsa
   yazılır. Yeni bir kavramı bildiğin bir şeye bağlamak, tanımını okumaktan hızlıdır.
4. **Neden'i de yazar.** Bir kural varsa gerekçesi de yanında durur; brief'in ilgili maddesi
   parantez içinde referans verilir.
5. **Her şeyi şimdi okuman gerekmiyor.** Bazı dosyalar ileriki fazlar için hazır duruyor;
   başında "ne zaman lazım" yazar.

## Okuma sırası

| Ne zaman | Hangi dosyalar |
|---|---|
| **Şimdi** (Faz 0–1 başı) | 01 Linux, 02 SSH, 03 Git, 04 Pi donanım, 05 Pi yazılım, 07 Ağ |
| **Aşama 3'te** | 08 systemd, 09 Python |
| **Faz 1 ilerleyince** | 06 YAML, 10 MQTT, 11 Docker, 13 Modbus, **16 Collector** |
| **Faz 1 sonu / Faz 2** | 12 SQL & TimescaleDB, 14 Ansible |
| **Faz 5** | 15 Grafana |
| **Her zaman** | 99 Komut sözlüğü |

## Dosyalar

| Dosya | İçerik |
|---|---|
| `01-linux-temelleri.md` | Kabuk, dosya sistemi, komutlar, borular, izinler, süreçler, `sudo` |
| `02-ssh-ve-uzak-erisim.md` | SSH, parmak izi, anahtar çiftleri, `scp`, config dosyası |
| `03-git.md` | Git vs GitHub, commit/branch/remote, geri alma, `.gitignore` |
| `04-raspberry-pi-donanim.md` | Kart, güç, soğutma, portlar, GPIO, ısı, SD ömrü, iş güvenliği |
| `05-raspberry-pi-yazilim.md` | Pi OS, Imager, `raspi-config`, overlay FS, kiosk, udev |
| `06-yaml-ve-config.md` | YAML sözdizimi, tipler, config'i koddan ayırma |
| `07-ag-temelleri.md` | IP, subnet, DHCP, MAC, port, DNS, VLAN |
| `08-systemd-servisler.md` | Servis dosyası, `systemctl`, `journalctl`, timer |
| `09-python-servis-temelleri.md` | Servis yazmak için gereken kadar Python |
| `10-mqtt.md` | Topic, QoS, retained, LWT, mesaj şeması, idempotency |
| `11-docker.md` | İmaj, konteyner, volume, `docker compose` |
| `12-sql-ve-timescaledb.md` | SQL temelleri, window fonksiyonları, hypertable, aggregate |
| `13-modbus-ve-seri-haberlesme.md` | RS-485, Modbus RTU/TCP, 32-bit tuzakları, `pymodbus` |
| `14-ansible-ve-filo-yonetimi.md` | Inventory, playbook, idempotency, Vault |
| `15-grafana-ve-gosterim.md` | Dashboard, kiosk, provisioning, Grafana vs Power BI |
| `16-collector-ve-surucu-mimarisi.md` | Sürücü deseni, rollover matematiği, mesaj şeması, simülatörle test |
| `99-komut-sozlugu.md` | Tüm komutlar, tek satırlık karşılıklarıyla |

## Nerede ne var

| Ne arıyorum | Nereye bakarım |
|---|---|
| "Bu komut ne işe yarıyordu?" | Bu klasör, veya `99-komut-sozlugu.md` |
| "Bunu neden böyle yaptık?" | `andon-proje-brifi.md` (Claude Project) |
| "Hangi gün ne yaptım, ne ölçtüm?" | `../CALISMA-GUNLUGU.md` |
| "Sırada ne var?" | `andon-ogrenme-yolu.md` (Claude Project) |
