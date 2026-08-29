# Komut Sözlüğü

Projede fiilen kullandığımız komutlar, alfabetik. Detay için ilgili ders notuna bak.

## Linux — genel

| Komut | Ne yapar | Örnek |
|---|---|---|
| `apt` | Paket kur/güncelle | `sudo apt install -y git` |
| `cat` | Dosyanın tamamını ekrana bas | `cat machines.yaml` |
| `cd` | Klasör değiştir | `cd ~/andon-collector` |
| `chmod` | İzin değiştir | `chmod +x script.py` |
| `chown` | Sahip değiştir | `sudo chown andon:andon dosya` |
| `cp` | Kopyala | `cp a.yaml a.yaml.bak` |
| `df -h` | Disk doluluğu | `df -h` |
| `find` | Dosya ara | `find . -name "*.yaml"` |
| `free -m` | RAM durumu (MB) | `free -m` |
| `hostname` | Makinenin adı | `hostname` |
| `ip a` | Ağ arabirimleri ve IP'ler | `ip a` |
| `journalctl` | Servis loglarını oku | `journalctl -u andon-button -f` |
| `less` | Dosyayı kaydırarak oku (`q` ile çık) | `less /etc/hostname` |
| `ls` | Klasör içeriği | `ls -la` |
| `mkdir` | Klasör oluştur | `mkdir -p ~/andon-collector` |
| `mv` | Taşı / yeniden adlandır | `mv eski.yaml yeni.yaml` |
| `nano` | Metin düzenleyici | `nano machines.yaml` |
| `poweroff` | Kapat | `sudo poweroff` |
| `pwd` | Neredeyim | `pwd` |
| `reboot` | Yeniden başlat | `sudo reboot` |
| `rm` | Sil (geri dönüşü yok) | `rm -r klasor` |
| `sudo` | Yönetici yetkisiyle çalıştır | `sudo apt update` |
| `swapon --show` | Swap nerede: zram mı, dosya mı | `swapon --show` |
| `systemctl` | Servis yönet | `systemctl status ssh` |
| `whoami` | Hangi kullanıcıyım | `whoami` |
| `which` | Bu komut kurulu mu, nerede | `which git` |

## SSH

| Komut | Ne yapar |
|---|---|
| `ssh KULLANICI@IP` | Uzak makinede oturum aç |
| `ssh -T git@github.com` | Sadece kimlik doğrula, oturum açma |
| `ssh-keygen -t ed25519 -C "etiket"` | Anahtar çifti üret |
| `cat ~/.ssh/id_ed25519.pub` | Açık anahtarı göster (paylaşılacak olan) |

## Git

| Komut | Ne yapar |
|---|---|
| `git init` | Bu klasörü repo yap |
| `git config user.name "..."` | Commit'lere yazılacak isim |
| `git status` | Ne değişti |
| `git add .` | Değişiklikleri staging'e al |
| `git commit -m "Mesaj"` | Kaydet |
| `git log --oneline` | Geçmişi kısa listele |
| `git diff` | Henüz staged olmayan değişiklikler |
| `git branch -M main` | Ana dalın adını `main` yap |
| `git remote add origin URL` | Uzak repo'yu bağla |
| `git push -u origin main` | İlk gönderim |
| `git push` / `git pull` | Gönder / al |

## Raspberry Pi'ye özel

| Komut | Ne yapar |
|---|---|
| `sudo raspi-config` | Menülü ayar aracı (arabirimler, overlay FS) |
| `vcgencmd measure_temp` | İşlemci sıcaklığı |
| `vcgencmd get_throttled` | Kısılma bayrağı. `0x0` = sorun yok |
| `vcgencmd measure_volts` | Çekirdek gerilimi |

## Kısayollar

| Tuş | Nerede | Ne yapar |
|---|---|---|
| `Ctrl+C` | Terminal | Çalışan komutu durdur |
| `Ctrl+D` | Terminal | Oturumu kapat (`exit` ile aynı) |
| `Ctrl+O` | nano | Kaydet (sonra Enter) |
| `Ctrl+X` | nano | Çık |
| `Ctrl+K` | nano | Satırı kes |
| `q` | less / man | Çık |
| Yukarı ok | Terminal | Önceki komutu getir |
| `Tab` | Terminal | Dosya/komut adını tamamla |

`Tab` ile tamamlama en çok zaman kazandıran alışkanlıktır: `cd ~/and` yazıp `Tab`'a bas,
gerisini kendi yazar. Yazım hatası da yapmamış olursun.

## Henüz görmediğimiz ama gelecek olanlar

`systemctl enable`, `docker compose up`, `mosquitto_pub`, `mosquitto_sub`, `psql`,
`ansible-playbook`. Sırası gelince buraya eklenecek.
