# Linux Temelleri

## Linux nedir, Raspberry Pi OS nedir

- **Linux** = çekirdek (kernel). Donanımı yöneten katman. Tek başına kullanılmaz.
- **Dağıtım (distro)** = çekirdek + komutlar + paket yöneticisi + (varsa) masaüstü.
  Debian, Ubuntu, Raspberry Pi OS — hepsi birer dağıtım.
- **Raspberry Pi OS** = Debian'ın ARM işlemci için derlenmiş hali + Pi'ye özel eklentiler.

**Pratik sonuç:** internette çözüm ararken "Debian" veya "Ubuntu" için yazılmış cevaplar
Pi'de de çalışır — GPIO, `raspi-config` veya `vcgencmd` geçmiyorsa. Debian dokümantasyonu
Pi dokümantasyonundan kat kat geniştir.

## Komut satırının mantığı

Her satır aynı kalıpta:

```
komut  seçenekler  hedef
ls     -l          /etc
```
→ "listele, uzun formatta, `/etc` klasörünü"

Seçenekler genelde `-` ile tek harf (`-l`), `--` ile uzun yazım (`--help`). Birleştirilebilir:
`ls -la` = `ls -l -a`.

**Otomasyon karşılığı:** TIA Portal'da bir işi menüden tıklayarak da yaparsın, ne yaptığını
bilerek doğrudan da. Linux'ta terminal ikincisidir — burada tıklanacak menü çoğu zaman
hiç yoktur.

## Dosya sistemi

Tek bir ağaç. `C:\` `D:\` yok. Kök `/`.

| Yol | Ne var |
|---|---|
| `/home/andon` | Senin dosyaların. Kısayolu `~` |
| `/etc` | Sistem ve servis ayar dosyaları. `systemd` servisleri buraya yazılır |
| `/var/log` | Loglar (ama biz `journalctl` kullanıyoruz) |
| `/dev` | Donanım aygıtları. USB-RS485 adaptörü burada `/dev/ttyUSB0` olarak görünür |
| `/usr/bin` | Kurulu programlar |

**Yol yazım kuralları:**
- `.` = bulunduğum klasör
- `..` = bir üst klasör
- `~` = ev klasörüm (`/home/andon`)
- `/` ile başlayan yol = kökten itibaren tam yol (mutlak)
- `/` ile başlamayan = bulunduğum yerden itibaren (göreli)

## Temel komutlar

### Gezinme ve bakma

| Komut | Ne yapar | Not |
|---|---|---|
| `pwd` | Neredeyim | "print working directory" |
| `ls` | Ne var | `ls -l` detaylı, `ls -a` gizli dosyalar dahil, `ls -la` ikisi |
| `cd KLASÖR` | Klasör değiştir | `cd ~` eve döner, `cd ..` bir üst |
| `cat DOSYA` | Dosyanın tamamını ekrana bas | Kısa dosyalar için |
| `less DOSYA` | Dosyayı kaydırarak oku | `q` ile çıkılır. Uzun dosyalar için |
| `find` | Dosya ara | `find . -name "*.yaml"` |

**Gizli dosya:** adı `.` ile başlayan dosyalar normal `ls`'te görünmez. `.gitignore`,
`.ssh/`, `.git/` böyledir. Görmek için `ls -a`.

### Düzenleme ve dosya işleri

| Komut | Ne yapar |
|---|---|
| `nano DOSYA` | Metin düzenleyici. Kaydet `Ctrl+O` + Enter, çık `Ctrl+X` |
| `mkdir KLASÖR` | Klasör oluştur. `mkdir -p a/b/c` ara klasörleri de açar |
| `cp KAYNAK HEDEF` | Kopyala. Klasör için `cp -r` |
| `mv KAYNAK HEDEF` | Taşı veya yeniden adlandır (ikisi aynı komut) |
| `rm DOSYA` | Sil. Klasör için `rm -r`. **Geri dönüşü yok, çöp kutusu yok** |

### Sistem

| Komut | Ne yapar |
|---|---|
| `free -m` | RAM durumu, MB cinsinden |
| `df -h` | Disk doluluk, okunabilir birimlerle |
| `ip a` | Ağ arabirimleri ve IP adresleri |
| `systemctl status SERVIS` | Bir servisin durumu |
| `journalctl -u SERVIS -f` | Servisin logunu canlı izle (`-f` = takip et) |
| `which KOMUT` | Bu komut kurulu mu, nerede? |
| `sudo reboot` | Yeniden başlat |
| `sudo poweroff` | Kapat |

### Paket kurma

```bash
sudo apt update              # paket listesini tazele
sudo apt install -y git      # kur (-y = "evet" sorularını geç)
```

`apt` = Debian'ın paket yöneticisi. Windows'taki "programı indir, kur"un karşılığı, ama
merkezî bir depodan ve tek komutla.

## `free -m` nasıl okunur

```
               total   used   free   shared  buff/cache   available
Mem:            2006    450    678       52         992        1555
```

**`free` sütununa değil `available` sütununa bak.** `buff/cache`, çekirdeğin boş RAM'i disk
önbelleği olarak kullanmasıdır; bir program bellek isteyince anında bırakılır. "RAM'im
dolmuş" paniği hep bu yüzden yaşanır. `available` = "bir program şu an gerçekten ne kadar
alabilir".

## `sudo` — ne zaman gerekir

`sudo` = "bu komutu yönetici yetkisiyle çalıştır".

**Kural: `sudo` sadece ev klasörünün dışına çıkarken.**

| Komut | `sudo` gerekli mi | Neden |
|---|---|---|
| `sudo apt install git` | Evet | Programı `/usr/bin`'e kuruyor, sistem geneli |
| `sudo nano /etc/...` | Evet | `/etc` sistem klasörü |
| `nano ~/dosya.txt` | Hayır | Kendi klasörün |
| `git commit` | Hayır | Kendi klasöründeki dosyalar |

⚠️ **Asla `sudo git` yazma.** Yazarsan dosyaların sahibi `root` olur, sonra normal
kullanıcıyken kendi dosyanı düzenleyemezsin ve nedenini bulmak yarım saatini alır.

## İzinler — kısa hâli

`ls -l` çıktısında satır başındaki `-rw-r--r--` gibi ifade izinleri gösterir:
okuma (r), yazma (w), çalıştırma (x); sırasıyla sahip / grup / herkes için.

`chmod +x script.py` → dosyayı çalıştırılabilir yapar.
`chown` → dosyanın sahibini değiştirir (genelde `sudo` ile).

Şimdilik bu kadarı yeter; gerektiğinde derinleşiriz.

## Kaynak

- *The Linux Command Line* — William Shotts, `linuxcommand.org/tlcl.php`, PDF ücretsiz.
  **Bölüm 1–11 yeter.**
- `linuxjourney.com` — "Grasshopper" bölümü.
