# Linux Temelleri

## 1. Linux nedir

- **Linux** = çekirdek (kernel). Donanımı yöneten katman. Tek başına kullanılmaz.
- **Dağıtım (distro)** = çekirdek + komutlar + paket yöneticisi + (varsa) masaüstü.
  Debian, Ubuntu, Raspberry Pi OS — hepsi birer dağıtım.
- **Raspberry Pi OS** = Debian'ın ARM için derlenmiş hali + Pi'ye özel eklentiler.

**Pratik sonuç:** internette çözüm ararken "Debian" veya "Ubuntu" için yazılmış cevaplar
Pi'de de çalışır — GPIO, `raspi-config` veya `vcgencmd` geçmiyorsa.

**Neden Windows değil (bu projede):** GPIO yok, read-only root yok, `systemd` yok, Ansible
ile filo yönetimi yok, 2 GB'a Chromium'la sığmıyor (brief Karar #15).

## 2. Komut satırının mantığı

```
komut  seçenekler  hedef
ls     -l          /etc
```
→ "listele, uzun formatta, `/etc` klasörünü"

- Kısa seçenek: `-l`, birleştirilebilir → `ls -la` = `ls -l -a`
- Uzun seçenek: `--help`, `--version`
- Yardım: `komut --help` (hızlı) veya `man komut` (detaylı, `q` ile çık)

**Otomasyon karşılığı:** TIA Portal'da bir işi menüden tıklayarak da yaparsın, ne yaptığını
bilerek doğrudan da. Linux'ta terminal ikincisidir — burada tıklanacak menü çoğu zaman yoktur.

**İki hayat kurtaran alışkanlık:**
- **`Tab`** — dosya/komut adını tamamlar. Hem hızlı hem yazım hatası yapmazsın.
- **Yukarı ok** — önceki komutları getirir. `history` tüm listeyi verir.

## 3. Dosya sistemi

Tek bir ağaç. `C:\` `D:\` yok. Kök `/`.

| Yol | Ne var |
|---|---|
| `/home/andon` | Senin dosyaların. Kısayolu `~` |
| `/etc` | Sistem ve servis ayar dosyaları |
| `/var/log` | Loglar (biz `journalctl` kullanıyoruz) |
| `/dev` | Donanım aygıtları. USB-RS485 burada `/dev/ttyUSB0` |
| `/usr/bin`, `/bin` | Kurulu programlar |
| `/tmp` | Geçici dosyalar, reboot'ta silinir |
| `/mnt`, `/media` | Takılan diskler, USB bellekler |
| `/proc`, `/sys` | Gerçek dosya değil — çekirdeğin canlı durumu. `cat /proc/cpuinfo` |

**Yol yazımı:**
- `.` = bulunduğum klasör, `..` = bir üst, `~` = ev klasörüm
- `/` ile başlayan = kökten itibaren **mutlak** yol
- `/` ile başlamayan = bulunduğum yerden **göreli** yol

**Gizli dosya:** adı `.` ile başlar, normal `ls`'te görünmez (`.gitignore`, `.ssh/`, `.git/`).
Görmek için `ls -a`.

**Uzantı zorunlu değil.** Linux'ta `.txt` `.sh` gibi uzantılar sadece insan içindir; dosyanın
ne olduğunu içeriği ve izinleri belirler.

## 4. Temel komutlar

### Gezinme ve okuma

| Komut | Ne yapar |
|---|---|
| `pwd` | Neredeyim |
| `ls` / `ls -l` / `ls -la` | Listele / detaylı / gizliler dahil |
| `cd KLASÖR` | Klasör değiştir (`cd ~`, `cd ..`, `cd -` önceki klasör) |
| `cat DOSYA` | Tamamını ekrana bas |
| `less DOSYA` | Kaydırarak oku, `q` ile çık |
| `head -20 DOSYA` | İlk 20 satır |
| `tail -20 DOSYA` | Son 20 satır |
| `tail -f DOSYA` | Son satırları **canlı izle** (log takibi) |
| `wc -l DOSYA` | Satır say |
| `find . -name "*.yaml"` | Dosya ara |
| `grep "kelime" DOSYA` | Dosya içinde ara |

### Değiştirme

| Komut | Ne yapar |
|---|---|
| `nano DOSYA` | Metin düzenleyici. Kaydet `Ctrl+O`+Enter, çık `Ctrl+X` |
| `mkdir -p a/b/c` | Klasör oluştur (ara klasörler dahil) |
| `cp KAYNAK HEDEF` | Kopyala. Klasör için `cp -r` |
| `mv KAYNAK HEDEF` | Taşı **veya** yeniden adlandır — ikisi aynı komut |
| `rm DOSYA` | Sil. Klasör için `rm -r`. **Çöp kutusu yok** |
| `touch DOSYA` | Boş dosya oluştur |

⚠️ `rm -rf /` gibi komutları kopyalayıp yapıştırma. `rm`'in geri dönüşü yoktur.

### Joker karakterler (wildcard)

| İfade | Anlamı | Örnek |
|---|---|---|
| `*` | Herhangi bir karakter dizisi | `rm *.bak` |
| `?` | Tek karakter | `ls dosya?.txt` |
| `{a,b}` | Listeden biri | `cp x.{yaml,yaml.bak}` |

## 5. Borular ve yönlendirme

Linux'un en güçlü fikri: **küçük programları birbirine bağlamak.**

| İfade | Ne yapar | Örnek |
|---|---|---|
| `\|` | Bir komutun çıktısını diğerine ver | `ls -l \| grep yaml` |
| `>` | Çıktıyı dosyaya yaz (**üzerine yazar**) | `ls > liste.txt` |
| `>>` | Çıktıyı dosyanın **sonuna ekle** | `echo "not" >> gunluk.txt` |
| `2>` | Sadece hata çıktısını yönlendir | `komut 2> hata.log` |
| `<` | Dosyayı komuta girdi olarak ver | `sort < liste.txt` |

Örnek: "logda 'error' geçen son 20 satır"
```bash
journalctl -u andon-collector | grep -i error | tail -20
```

### `echo` — metni ekrana veya dosyaya yazdır

`echo` verdiğin metni olduğu gibi geri yazar. Tek başına işe yaramaz gibi görünür; asıl
değeri **yönlendirme ve değişkenlerle** birlikte kullanılmasındadır.

```bash
echo "merhaba"                      # ekrana yazar
echo $HOME                          # değişkenin içeriğini gösterir → /home/andon
echo "not" >> gunluk.txt            # dosyanın SONUNA ekler
echo "yeni icerik" > dosya.txt      # dosyanın ÜZERİNE yazar (eskisi gider)
echo -n "satır sonu yok"            # sonuna yeni satır koymaz
```

Nerelerde kullanılır:
- **Değişken kontrolü:** `echo $PATH` ile ortam değişkenine bakmak
- **Tek satırlık dosya yazmak:** ayar dosyasına hızlıca bir satır eklemek
- **Script içinde ilerleme mesajı:** "şu adım bitti" diye yazdırmak
- **Bir komutun ne göndereceğini önce görmek:** `echo rm *.bak` yazıp çıktıya bakarsan,
  `rm` gerçekten neyi silecekmiş görürsün. **Tehlikeli komutlardan önce iyi bir alışkanlık**

**Otomasyon karşılığı:** HMI'ya mesaj basmak gibi. Program akışını görünür kılar.

⚠️ `>` ile `>>` farkı: `>` dosyayı **sıfırlar**, `>>` **ekler**. Karıştırmak bir dosyanın
içeriğini silmenin en hızlı yoludur.

## 6. `sudo` ve yetkiler

`sudo` = "bu komutu yönetici (root) yetkisiyle çalıştır".

**Kural: `sudo` sadece ev klasörünün dışına çıkarken.**

| Komut | `sudo` | Neden |
|---|---|---|
| `sudo apt install git` | Evet | `/usr/bin`'e kuruyor, sistem geneli |
| `sudo nano /etc/...` | Evet | `/etc` sistem klasörü |
| `nano ~/dosya.txt` | Hayır | Kendi klasörün |
| `git commit` | Hayır | Kendi dosyaların |

⚠️ **Asla `sudo git` yazma.** Dosyaların sahibi `root` olur, sonra kendi dosyanı
düzenleyemezsin ve sebebini bulmak yarım saatini alır.

⚠️ `sudo su` veya `sudo -i` ile root kabuğunda kalmayı alışkanlık yapma. Her komutun başına
`sudo` yazmak yavaş değil, kasıtlı bir yavaşlıktır.

## 7. İzinler ve sahiplik

`ls -l` çıktısı:

```
-rw-r--r-- 1 andon andon 1234 Aug 27 10:00 machines.yaml
│└┬┘└┬┘└┬┘   └─┬─┘ └─┬─┘
│ │  │  │      │     └── grup
│ │  │  │      └──────── sahip
│ │  │  └─────────────── herkes: oku
│ │  └────────────────── grup: oku
│ └───────────────────── sahip: oku + yaz
└─────────────────────── tip: - dosya, d klasör, l kısayol
```

| İzin | Harf | Dosyada | Klasörde |
|---|---|---|---|
| Okuma | `r` | İçeriği okuyabilir | İçindekileri listeleyebilir |
| Yazma | `w` | Değiştirebilir | Dosya ekleyip silebilir |
| Çalıştırma | `x` | Program olarak çalıştırabilir | İçine girebilir (`cd`) |

```bash
chmod +x script.py           # çalıştırılabilir yap
chmod 644 dosya              # sahip rw, diğerleri r
sudo chown andon:andon dosya # sahibi ve grubu değiştir
```

Sayısal gösterim: `r=4, w=2, x=1`. `644` = sahip 6 (4+2 = rw), grup 4 (r), herkes 4 (r).

## 8. Kullanıcılar

- Her işlem bir kullanıcı adına çalışır. Bizde `andon`.
- `root` = sınırsız yetkili kullanıcı. Doğrudan giriş **kapalı** olacak (brief §13.1).
- `whoami` kim olduğumu, `id` hangi gruplarda olduğumu söyler.
- Servisler de bir kullanıcı adına çalışır — `systemd` dosyasındaki `User=andon` satırı budur.

**Neden önemli:** collector'ın seri porta erişebilmesi için kullanıcının `dialout` grubunda
olması gerekir. GPIO için `gpio` grubu. Bunlar sırası gelince yapılacak.

## 9. Süreçler (process)

Çalışan her program bir **süreçtir** ve bir numarası (PID) vardır.

| Komut | Ne yapar |
|---|---|
| `ps aux` | Çalışan tüm süreçler |
| `ps aux \| grep python` | Sadece python süreçleri |
| `top` / `htop` | Canlı süreç ve kaynak izleme (`q` ile çık) |
| `kill PID` | Sürece "kapan" sinyali gönder |
| `kill -9 PID` | Zorla öldür (son çare) |
| `Ctrl+C` | Öndeki komutu durdur |
| `komut &` | Arka planda çalıştır |

**Bu projede:** collector'ın gerçekten çalışıp çalışmadığına `systemctl status` ile bakacağız,
`ps` ile değil. Ama bir şey takıldığında `top` ilk bakılacak yerdir.

## 10. Servisler ve loglar — giriş

Detayı `08-systemd-servisler.md` içinde. Bilmen gereken iki komut:

```bash
systemctl status ssh              # servis çalışıyor mu
journalctl -u ssh -f              # canlı log
```

`systemd` Linux'ta açılışta her şeyi başlatan programdır; bizim collector, Andon servisi ve
ingest servisi de birer `systemd` servisi olacak.

## 11. Paket ve paket yönetimi

### "Paket" ne demek

**Paket**, kurulmaya hazır bir yazılımın kutulanmış hali. İçinde şunlar var:

- Programın kendisi (çalıştırılabilir dosyalar, kütüphaneler)
- Nereye kurulacağı bilgisi
- **Bağımlılık listesi:** "bu program çalışmak için şu paketlere de ihtiyaç duyar"
- Kurulum/kaldırma sırasında çalışacak küçük scriptler
- Sürüm numarası

Debian'da paket dosyasının uzantısı `.deb`. Ama tek tek `.deb` indirmezsin — **depo
(repository)** denen merkezî sunuculardan `apt` ile çekersin.

**Otomasyon karşılığı:** vendor'un verdiği hazır kütüphane bloğu. İçini yazmazsın, projene
eklersin, çalışır. Farkı: `apt` bir bloğu eklerken onun ihtiyaç duyduğu diğer blokları da
kendisi bulup ekler.

### Neden tek tek indirip kurmak yerine paket

| Elle indirip kurmak | Paket yöneticisi |
|---|---|
| Bağımlılıkları sen bulursun | `apt` otomatik çözer |
| Güncellemeyi sen takip edersin | `apt upgrade` hepsini birden |
| Ne kurduğunu unutursun | `dpkg -l` listeler |
| Kaldırırken artık dosya kalır | `apt remove` temiz kaldırır |
| Kaynağı belirsiz | Depo imzalı, doğrulanmış |

Bu, filo yönetiminin de temeli: 18 Pi'de aynı paketlerin aynı sürümü olacak (brief §3.6).

### Komutlar

```bash
sudo apt update                    # paket listesini tazele (kurulum yapmaz)
sudo apt upgrade                   # kurulu paketleri güncelle
sudo apt install -y git python3-pip
sudo apt remove PAKET
apt search kelime
```

```bash
apt list --installed | wc -l       # kaç paket kurulu
dpkg -l | grep python3             # belirli bir paket kurulu mu
apt show git                       # paket hakkında bilgi
```

⚠️ **`update` ile `upgrade` farklıdır.** `update` sadece "depoda neler var" listesini
tazeler, hiçbir şey kurmaz. `upgrade` asıl güncellemeyi yapar. Bu yüzden hep birlikte
yazılır: `sudo apt update && sudo apt upgrade`.

### Python paketleri ayrı bir dünya

`apt` sistem paketlerini yönetir; Python kütüphaneleri için **`pip`** vardır ve depoları
farklıdır (PyPI). İkisi çakışabilir — bu yüzden yeni Debian sürümleri sistem Python'ına
doğrudan `pip install` yapılmasını engeller. Doğrusu **sanal ortam** (`venv`) kullanmaktır
(bkz. `09-python-servis-temelleri.md`).

**Üretimde:** `apt` elle çalıştırılmayacak, Ansible playbook'u yapacak. Sebep: 18 node'un
aynı sürümde kalması (brief §3.6).

## 12. Disk, bellek, sistem durumu

```bash
free -m        # RAM
df -h          # disk doluluğu
du -sh KLASÖR  # bir klasör ne kadar yer kaplıyor
uptime         # ne zamandır açık, yük ortalaması
uname -a       # çekirdek ve mimari bilgisi
```

### `free -m` nasıl okunur

```
        total   used   free   shared  buff/cache   available
Mem:     2006    450    678       52         992        1555
```

**`free` sütununa değil `available` sütununa bak.** `buff/cache`, çekirdeğin boş RAM'i disk
önbelleği olarak kullanmasıdır; bir program bellek isteyince anında bırakılır. "RAM'im
dolmuş" paniği hep buradan çıkar. `available` = bir programın şu an gerçekten alabileceği.

## 13. Ortam değişkenleri

Kabuğun taşıdığı ayarlar. `echo $HOME`, `echo $PATH` ile bakılır.

- `$PATH` = komutların aranacağı klasör listesi. `which git` bunun içinde arar.
- `export DEGISKEN=deger` = bu kabuk oturumu için tanımla.
- Kalıcı olması için `~/.bashrc` dosyasına yazılır.

**Bu projede:** parolalar ve token'lar `.env` dosyasında ortam değişkeni olarak tutulacak ve
**asla git'e girmeyecek** (brief §13.1).

## 14. Hata mesajlarını okumak

Linux'ta hata mesajları genelde tam olarak sorunu söyler. En sık görecekklerin:

| Mesaj | Anlamı | Çözüm |
|---|---|---|
| `command not found` | Program kurulu değil veya yazım hatası | `which`, `apt install` |
| `Permission denied` | Yetki yok | `sudo` gerekebilir, ya da izinlere bak |
| `No such file or directory` | Yol yanlış | `pwd`, `ls` ile kontrol et |
| `Operation not permitted` | Yetki var ama sistem izin vermiyor | Genelde koruma mekanizması |
| `Address already in use` | O portu başka bir program tutuyor | `ss -tulpn` ile bak |

**Kural:** hata mesajını özetleme, tamamını kopyala. Okumayı öğrenmek öğrenmenin yarısıdır.

## 15. Kaynak

- *The Linux Command Line* — William Shotts, `linuxcommand.org/tlcl.php`, ücretsiz PDF.
  **Bölüm 1–11 yeter.**
- `linuxjourney.com` — "Grasshopper" bölümü.
- Video izlemek öğrenmek gibi hissettirir, değildir. Komutu kendin yazmadan öğrenmiyorsun.
