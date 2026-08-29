# Ders Notları — nasıl kullanılır

Bu klasör, projede **fiilen karşılaştığım** kavramların ve komutların notudur.

## Kuralları

1. **Önce görürüz, sonra yazarız.** Buraya bir komut ancak projede gerçekten kullandıktan
   sonra girer. Kullanmadığım bir şeyi ezberlemek zaman kaybı.
2. **Sade dil.** Amaç referans kitabı değil, altı ay sonra "bu neydi ya" dediğimde bir
   dakikada hatırlamak.
3. **Mümkünse otomasyon karşılığıyla.** PLC / ladder / pano dünyasından bir karşılığı varsa
   yazılır. Yeni bir kavramı bildiğim bir şeye bağlamak, tanımını okumaktan hızlıdır.
4. **Dosyalar büyür.** Her oturumda yeni gördüğüm komutlar ilgili dosyaya eklenir; baştan
   eksiksiz olmaları beklenmez.

## Dosyalar

| Dosya | İçerik |
|---|---|
| `01-linux-temelleri.md` | Kabuk, dosya sistemi, temel komutlar, `sudo` mantığı |
| `02-ssh-ve-uzak-erisim.md` | Uzaktan bağlanma, parmak izi, SSH key'leri |
| `03-git.md` | Git nedir, GitHub'dan farkı, kullandığımız komutlar |
| `04-raspberry-pi-donanim.md` | Kart, güç, soğutma, portlar, GPIO, elektriksel kurallar |
| `05-raspberry-pi-yazilim.md` | Pi OS, Imager, `raspi-config`, `vcgencmd`, overlay FS |
| `06-yaml-ve-config.md` | YAML sözdizimi ve config'i koddan ayırma fikri |
| `99-komut-sozlugu.md` | Şimdiye kadar geçen tüm komutlar, tek satırlık karşılıklarıyla |

## İleride eklenecekler

Sırası geldikçe: `systemd` servisleri, Python servis kalıbı, MQTT, Docker, SQL ve window
fonksiyonları, Modbus/RS-485, Ansible.

## Nerede ne var

| Ne arıyorum | Nereye bakarım |
|---|---|
| "Bu komut ne işe yarıyordu?" | Bu klasör |
| "Bunu neden böyle yaptık?" | `andon-proje-brifi.md` (Claude Project) |
| "Hangi gün ne yaptım, ne ölçtüm?" | `../CALISMA-GUNLUGU.md` |
| "Sırada ne var?" | `andon-ogrenme-yolu.md` (Claude Project) |
