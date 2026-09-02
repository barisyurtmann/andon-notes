# Komut Sözlüğü

Tek satırlık karşılıklar. Detay için ilgili ders notuna bak.
⭐ = bu projede fiilen kullandık.

## Linux — gezinme ve okuma

| Komut | Ne yapar |
|---|---|
| ⭐ `pwd` | Neredeyim |
| ⭐ `ls` / `ls -l` / `ls -la` | Listele / detaylı / gizliler dahil |
| ⭐ `cd` | Klasör değiştir (`cd ~`, `cd ..`, `cd -`) |
| ⭐ `cat DOSYA` | Tamamını ekrana bas |
| `less DOSYA` | Kaydırarak oku (`q` ile çık) |
| `head -20` / `tail -20` | İlk / son 20 satır |
| `tail -f DOSYA` | Son satırları canlı izle |
| `wc -l` | Satır say |
| `grep "kelime" DOSYA` | Dosya içinde ara |
| `find . -name "*.yaml"` | Dosya ara |
| `du -sh KLASÖR` | Klasör ne kadar yer kaplıyor |
| `echo METİN` | Metni ekrana yaz (`>>` ile dosyaya ekler, `$DEGISKEN` içeriğini gösterir) |

## Linux — değiştirme

| Komut | Ne yapar |
|---|---|
| ⭐ `nano DOSYA` | Metin düzenle (`Ctrl+O` kaydet, `Ctrl+X` çık) |
| ⭐ `mkdir -p a/b` | Klasör oluştur |
| ⭐ `cp` / `cp -r` | Kopyala / klasör kopyala |
| `mv` | Taşı veya yeniden adlandır |
| `rm` / `rm -r` | Sil — **geri dönüşü yok** |
| `touch` | Boş dosya oluştur |
| `chmod +x` | Çalıştırılabilir yap |
| `chown` | Sahip değiştir |

## Linux — sistem

| Komut | Ne yapar |
|---|---|
| ⭐ `sudo` | Yönetici yetkisiyle çalıştır |
| ⭐ `apt update` / `apt install` | Paket listesi tazele / kur |
| ⭐ `free -m` | RAM durumu (**`available` sütununa bak**) |
| `df -h` | Disk doluluğu |
| ⭐ `hostname` / `whoami` | Makine adı / kullanıcı |
| ⭐ `which KOMUT` | Kurulu mu, nerede |
| `uptime` / `uname -a` | Ne zamandır açık / çekirdek bilgisi |
| `uptime -s` | Açılış anının tam zamanı |
| `timedatectl` | Saat, dilim, NTP senkron durumu |
| `chronyc tracking` | NTP senkron detayları |
| `dpkg -l` / `apt show PAKET` | Kurulu paketler / paket bilgisi |
| `ps aux` / `top` | Süreçler / canlı izleme |
| `kill PID` | Süreci durdur |
| ⭐ `swapon --show` | Swap nerede: zram mı, dosya mı |
| `sudo reboot` / `sudo poweroff` | Yeniden başlat / kapat |

## Linux — borular ve yönlendirme

| İfade | Ne yapar |
|---|---|
| `\|` | Bir komutun çıktısını diğerine ver |
| `>` / `>>` | Dosyaya yaz (üzerine) / sonuna ekle |
| `2>` | Hata çıktısını yönlendir |

## Ağ

| Komut | Ne yapar |
|---|---|
| ⭐ `ip a` | IP adresleri |
| `ip link` | Arabirimler ve MAC adresleri |
| `ip route` | Gateway ve yönlendirme |
| `ping ADRES` | Karşı taraf yaşıyor mu |
| `ss -tulpn` | Açık portlar ve onları tutan programlar |
| `curl -I URL` | HTTP erişimi test et |
| `nc -zv ADRES PORT` | Belirli porta erişim var mı |
| `nmcli device status` | Ağ bağlantısı durumu |

## SSH

| Komut | Ne yapar |
|---|---|
| ⭐ `ssh KULLANICI@IP` | Uzak makinede oturum aç |
| ⭐ `ssh -T git@github.com` | Sadece kimlik doğrula |
| ⭐ `ssh-keygen -t ed25519 -C "etiket"` | Anahtar çifti üret |
| ⭐ `cat ~/.ssh/id_ed25519.pub` | Açık anahtarı göster |
| `ssh-copy-id KULLANICI@IP` | Açık anahtarı karşıya kur |
| `ssh-keygen -R IP` | Eski parmak izi kaydını sil |
| `scp DOSYA KULLANICI@IP:/yol/` | Dosya kopyala |
| `rsync -av KLASÖR/ KULLANICI@IP:/yol/` | Klasör senkronize et |

## Git

| Komut | Ne yapar |
|---|---|
| ⭐ `git init` | Klasörü repo yap |
| ⭐ `git config user.name/email` | Commit kimliği |
| ⭐ `git status` | Ne değişti |
| ⭐ `git add .` | Staging'e al |
| ⭐ `git commit -m "..."` | Kaydet |
| ⭐ `git log --oneline` | Geçmiş |
| `git diff` | Değişiklikleri göster |
| ⭐ `git branch -M main` | Ana dal adı |
| ⭐ `git remote add origin URL` | Uzak repo bağla |
| ⭐ `git push -u origin main` | İlk gönderim |
| `git pull` / `git clone` | Al / ilk kez indir |
| `git restore DOSYA` | Değişikliği geri al |
| `git revert COMMIT` | Bir commit'i geri alan yeni commit |
| `git blame DOSYA` | Satırları kim ne zaman yazmış |
| `git tag v1.0` | Sürüm etiketi |

## Raspberry Pi

| Komut | Ne yapar |
|---|---|
| ⭐ `sudo raspi-config` | Menülü ayar aracı |
| ⭐ `vcgencmd measure_temp` | İşlemci sıcaklığı |
| `vcgencmd get_throttled` | Kısılma bayrağı (`0x0` = temiz) |
| `vcgencmd measure_volts` | Çekirdek gerilimi |
| `pinout` | GPIO pin haritası |
| `dmesg \| tail` | Az önce takılan cihaz ne olarak tanındı |
| `udevadm info -a -n /dev/ttyUSB0` | udev kuralı için cihaz bilgisi |
| `sudo usermod -aG dialout andon` | Seri port izni ver (`-a` = ekle; yazmazsan grupların silinir) |
| `groups` | Hangi gruplardayım (yeni oturumda güncellenir) |
| `ls -l /dev/ttyUSB*` | Seri adaptörler görünüyor mu |

## systemd

| Komut | Ne yapar |
|---|---|
| `sudo systemctl daemon-reload` | Servis dosyası değiştiyse **önce bu** |
| `sudo systemctl start/stop/restart AD` | Başlat / durdur / yeniden başlat |
| `sudo systemctl enable/disable AD` | Açılışta başlasın / başlamasın |
| ⭐ `systemctl status AD` | Durum ve son loglar |
| `journalctl -u AD -f` | Canlı log |
| `journalctl -u AD -n 50` | Son 50 satır |
| `systemctl list-timers` | Zamanlanmış görevler |

## Docker

| Komut | Ne yapar |
|---|---|
| `docker compose up -d` | Yığını başlat (arka planda) |
| `docker compose down` | Durdur ve kaldır |
| `docker compose logs -f` | Logları izle |
| `docker ps` | Çalışan konteynerler |
| `docker exec -it AD bash` | Konteynerin içine gir |
| `docker volume ls` | Kalıcı veri alanları |

## MQTT

| Komut | Ne yapar |
|---|---|
| `mosquitto_sub -h SUNUCU -t "topic/#" -v` | Dinle |
| `mosquitto_pub -h SUNUCU -t "topic" -m "mesaj"` | Gönder |

## Python

| Komut | Ne yapar |
|---|---|
| `python3 dosya.py` | Çalıştır |
| `python3 -m venv .venv` | Sanal ortam oluştur |
| `source .venv/bin/activate` | Sanal ortamı etkinleştir (`source` = scripti kendi kabuğunda çalıştır) |
| `deactivate` | Sanal ortamdan çık |
| `pip install "paket[ek]==sürüm"` | Kur; `[ek]` opsiyonel bağımlılık, `==` sürüm sabitleme |
| `pip install -r requirements.txt` | Bağımlılıkları kur |
| `python3 -c "import yaml; yaml.safe_load(open('x.yaml'))"` | YAML geçerli mi |

## Ansible

| Komut | Ne yapar |
|---|---|
| `ansible-playbook -i inventory playbooks/site.yml` | Playbook çalıştır |
| `... --check` | Deneme; değişiklik yapmaz |
| `ansible -i inventory grup -m ping` | Hepsi ayakta mı |
| `ansible-vault encrypt DOSYA` | Secret şifrele |

## Klavye kısayolları

| Tuş | Nerede | Ne yapar |
|---|---|---|
| `Ctrl+C` | Terminal | Çalışan komutu durdur |
| `Ctrl+D` | Terminal | Oturumu kapat |
| `Ctrl+O` / `Ctrl+X` | nano | Kaydet / çık |
| `Ctrl+K` / `Ctrl+U` | nano | Satırı kes / yapıştır |
| `Ctrl+W` | nano | Ara |
| `q` | less / man | Çık |
| Yukarı ok | Terminal | Önceki komut |
| ⭐ `Tab` | Terminal | Dosya/komut adını tamamla |
