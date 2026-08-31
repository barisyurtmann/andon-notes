# SSH ve Uzaktan Erişim

## 1. SSH nedir

**S**ecure **Sh**ell. Başka bir bilgisayarda komut çalıştırmanı sağlayan şifreli bağlantı.
Varsayılan portu **22**.

**Önemli:** SSH karşı tarafa **klavye uzatır, ekran uzatmaz.** Pi'nin masaüstü kaybolmaz,
monitöründe durmaya devam eder — sen sadece metin kanalından bağlanırsın.

**Otomasyon karşılığı:** PLC'ye programlama kablosuyla bağlanıp online olmak gibi; ama kablo
yerine ağ, ve tek bir program yerine tüm makine.

**Öncesi neydi:** telnet ve FTP şifresizdi — parola ağda düz metin geçerdi. SSH bunların
şifreli yerine geçenidir. Bugün telnet kullanan bir cihaz görürsen bu bir güvenlik notudur.

## 2. Bağlanma

```bash
ssh andon@192.168.0.166
ssh andon@192.168.0.166 -p 2222     # farklı port
ssh andon@192.168.0.166 "free -m"   # bağlan, tek komutu çalıştır, çık
```

Kalıp: `ssh KULLANICI@ADRES`. Çıkmak için `exit` veya `Ctrl+D`.

### İlk bağlantıdaki parmak izi sorusu

```
The authenticity of host '192.168.0.166' can't be established.
ED25519 key fingerprint is SHA256:...
Are you sure you want to continue connecting (yes/no)?
```

Tam olarak **`yes`** yazılır — `y` kabul edilmez. SSH karşıdaki makinenin kimlik parmak izini
henüz bilmiyor, sana gösterip onaylatıyor. Onayladıktan sonra parmak izi
`~/.ssh/known_hosts` dosyasına kaydedilir ve bir daha sorulmaz.

**Bu neye yarıyor:** ağda araya girip kendini senin Pi'n gibi tanıtan bir makine olursa
parmak izi tutmaz ve SSH bağırarak uyarır. Buna "man-in-the-middle" saldırısı denir.

⚠️ **Aynı adreste bu uyarı tekrar çıkarsa dur.** İki ihtimal: SD kart yeniden imajlandı
(normal), ya da o IP'de artık başka bir makine var (normal değil).

Kart yeniden imajlandıysa eski kaydı silmek gerekir:
```bash
ssh-keygen -R 192.168.0.166
```

### Parola yazarken ekranda hiçbir şey görünmez

Yıldız yok, imleç ilerlemez. **Hata değil, kasıtlı** — parolanın uzunluğu bile görünmesin
diye. Körlemesine yaz, Enter'a bas. `sudo` da böyle davranır.

## 3. Anahtar çiftleri (SSH key)

Parolayla giriş çalışır ama üretimde kapatılacak (brief §13.1). Yerine **anahtar çifti**.

### Nasıl çalışır

İki dosya üretilir ve matematiksel olarak birbirine bağlıdır:

| Dosya | Nedir | Kural |
|---|---|---|
| `~/.ssh/id_ed25519` | **Özel anahtar** | Asla paylaşılmaz, kopyalanmaz, git'e girmez |
| `~/.ssh/id_ed25519.pub` | **Açık anahtar** | Paylaşılmak için var |

**Zihin modeli:** açık anahtar bir kilittir, özel anahtar onu açan tek anahtardır. Kilidi
istediğin kapıya takarsın (sunucu, GitHub); anahtar sende kalır. Kilidi herkes görebilir,
işe yaramaz.

Sunucu tarafında açık anahtarlar `~/.ssh/authorized_keys` dosyasında durur. Giriş sırasında
sunucu bir soru sorar, sadece özel anahtarı olan doğru cevabı üretebilir — **parola ağa hiç
çıkmaz.**

### Üretme

```bash
ssh-keygen -t ed25519 -C "andon-bench"
```

- `-t ed25519` = anahtar tipi. Modern, kısa, hızlı. Eski `rsa`'ya göre daha iyi
- `-C "..."` = yorum. Sadece hangi makinenin anahtarı olduğunu hatırlamak için
- Üç soru: dosya yolu (Enter), passphrase (Enter = boş), tekrar (Enter)

```bash
cat ~/.ssh/id_ed25519.pub    # açık anahtarı göster
```

### Passphrase

Anahtar dosyasının kendisini koruyan ikinci bir parola. Boş bırakırsan her bağlantıda parola
sorulmaz (pratik), ama anahtar dosyası çalınırsa doğrudan kullanılabilir.

- Bench makinesi, kendi repo'na erişen anahtar → boş kabul edilebilir
- Üretim sunucusu, paylaşılan ortam → passphrase konur, `ssh-agent` ile bir kez girilir

### Açık anahtarı sunucuya taşıma

```bash
ssh-copy-id andon@192.168.0.166
```

Bu komut açık anahtarı karşı tarafın `authorized_keys` dosyasına ekler. Elle de yapılabilir
ama bu komut doğru izinleri de ayarlar.

### Test etme — sıra önemli

**Önce key ile girişi test et, ancak sonra parola girişini kapat.** Ters sırada yaparsan
Pi'ye erişimini kaybedersin (kurtarma: monitör+klavye ile fiziksel giriş).

```bash
ssh -T git@github.com
```

`-T` = "terminal açma, sadece kimliğimi doğrula". GitHub şunu döner:

```
Hi KULLANICI! You've successfully authenticated, but GitHub does not provide shell access.
```

Bu **başarı** mesajıdır.

### Parola girişini kapatma

Sunucuda `/etc/ssh/sshd_config` dosyasında:

```
PasswordAuthentication no
PermitRootLogin no
```

Sonra `sudo systemctl restart ssh`. **Bu, brief §13.1'in gereğidir ve Aşama 4'te birlikte
yapılacak.**

## 4. `~/.ssh/config` — kısayol dosyası

Uzun adresleri her seferinde yazmamak için:

```
Host bench
    HostName 192.168.0.166
    User andon
    IdentityFile ~/.ssh/id_ed25519
```

Artık `ssh bench` yeterli. 16 makinelik filoda bu dosya hayat kurtarır.

### ProxyJump — atlama sunucusu üzerinden bağlanma

Üretim VLAN'ına ofisten doğrudan erişim olmayabilir; bağlantı sunucu üzerinden geçer
(bkz. `07-ag-temelleri.md`). SSH bunu senin yerine yapar:

```
Host andon-server
    HostName 192.168.5.10
    User andon

Host andon-m001
    HostName 192.168.5.11
    User andon
    ProxyJump andon-server

Host andon-m0*
    User andon
    ProxyJump andon-server
```

`ssh andon-m001` yazdığında SSH önce sunucuya bağlanır, oradan Pi'ye atlar — sen aradaki
adımı hiç görmezsin. `scp` ve VS Code Remote-SSH de bu ayarı kullanır.

**Neden bu kalıp:** IT'nin yazacağı firewall kuralı tek hedef ve tek port olur; 16 Pi için
16 kural gerekmez.

## 5. Dosya kopyalama

| Komut | Ne yapar |
|---|---|
| `scp dosya andon@IP:/home/andon/` | Yerelden uzağa kopyala |
| `scp andon@IP:/home/andon/dosya .` | Uzaktan yerele |
| `rsync -av klasor/ andon@IP:/hedef/` | Klasör senkronize et (sadece değişenleri gönderir) |

**Bu projede az kullanacağız:** kod `git pull` ile gidecek, dağıtımı Ansible yapacak
(brief Karar #27). `scp` tek seferlik işler için.

## 6. Port yönlendirme — ne olduğunu bil, şimdilik kullanma

```bash
ssh -L 3000:localhost:3000 andon@IP
```

Uzaktaki bir servisi (örneğin Grafana) kendi bilgisayarında açıkmış gibi gösterir. İleride
sunucudaki Grafana'ya ofisten bakarken işe yarayabilir. Şimdi gerek yok.

## 7. Uzak masaüstü değil, uzak editör

"GUI Pi'de değil laptop'ta olsun" derken kastedilen uzak masaüstü **değil**:

| Ne istiyorum | Nasıl |
|---|---|
| Komut çalıştır, log oku, servis yönet | SSH — üretimde de tek yol |
| Dosya düzenle, kod yaz, klasör ağacını gör | **VS Code Remote-SSH** — editör laptop'ta, dosyalar Pi'de |
| Pi'nin masaüstünü görmek | Monitörü var |

Teknik olarak mümkün ama kullanmayacağımız iki yol:
- **X11 forwarding** (`ssh -X`) — tek bir grafik pencereyi taşır. Yavaş, gereksiz.
- **VNC** — tam masaüstü. Yerel ağda politika ihlali değil ama filoda kapalı olacak: bir
  Pi'ye masaüstünden bağlanıp elle düzeltme yapmak o üniteyi diğerlerinden farklılaştırır.

**Neden önemli:** üretimdeki 16 Pi'de bakabileceğin bir masaüstü zaten olmayacak —
ekranlarında tam ekran Chromium kiosk duracak.

## 8. Bu projede geçerli kurallar (brief §13.1)

- **Sadece key ile giriş.** Parola girişi ve root login kapalı
- **Vendor bulut tüneli yok** — Raspberry Pi Connect dahil (Karar #25). Dışarıya kalıcı
  bağlantı açar, gerektiğinde vendor relay'i üzerinden geçer, erişimi şahsi hesaba bağlar
- **Erişim sadece üretim VLAN'ı içinden**
- **Pi'ler GitHub'a sadece okuma yetkisiyle bağlanır** (read-only deploy key)

## 9. `reboot` SSH oturumunu keser — normaldir

`sudo reboot` yazdığın anda Pi kapanmaya başlar ve SSH bağlantısı **düşer**. Terminal
penceresi kapanır veya `Connection closed by ... port 22` yazar. Bu bir hata değil.

⚠️ **Sonucu:** `sudo reboot` ile başlayan bir komut bloğunu tek seferde yapıştıramazsın —
altındaki komutlar hiç çalışmaz, çünkü onları çalıştıracak oturum ölmüştür.

Doğru yöntem iki adımdır:
1. `sudo reboot` (tek başına)
2. ~30–60 sn bekle, `ssh andon@IP` ile yeniden bağlan, sonra ölçüm komutlarını çalıştır

Aynı şey `poweroff`, ağ ayarı değiştiren komutlar ve `systemctl restart ssh` için de
geçerlidir.

## 10. Bağlantı koparsa

SSH oturumu koparsa çalışan komut da ölür. Uzun süren işler için üç yol:

1. **`systemd` servisi** — kalıcı işler için doğru yol (bkz. `08-systemd-servisler.md`)
2. **`tmux`** — kopan oturumu geri bağlayabildiğin sanal terminal. `tmux` ile başlat,
   `Ctrl+B` sonra `D` ile ayrıl, `tmux attach` ile geri dön
3. **`nohup komut &`** — kaba ama işe yarar

Bu projede birincisini kullanacağız; ikincisi elle uzun iş yaparken (`apt upgrade` gibi) işe
yarar.
