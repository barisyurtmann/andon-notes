# SSH ve Uzaktan Erişim

## SSH nedir

**S**ecure **Sh**ell. Başka bir bilgisayarda komut çalıştırmanı sağlayan şifreli bağlantı.

**Önemli:** SSH karşı tarafa **klavye uzatır, ekran uzatmaz.** Pi'nin masaüstü kaybolmaz,
monitöründe durmaya devam eder — sen sadece metin kanalından bağlanırsın.

**Otomasyon karşılığı:** PLC'ye programlama kablosuyla bağlanıp online olmak gibi; ama
kablo yerine ağ, ve tek bir program yerine tüm makine.

## Bağlanma

```bash
ssh andon@192.168.0.166
```

Kalıp: `ssh KULLANICI@ADRES`.

### İlk bağlantıda çıkan parmak izi sorusu

```
The authenticity of host '192.168.0.166' can't be established.
ED25519 key fingerprint is SHA256:...
Are you sure you want to continue connecting (yes/no)?
```

Tam olarak **`yes`** yazılır — `y` kabul edilmez. SSH karşıdaki makinenin kimlik parmak izini
henüz bilmiyor, sana gösterip onaylatıyor. `yes` dedikten sonra parmak izi laptop'ındaki
`~/.ssh/known_hosts` dosyasına kaydedilir ve bir daha sorulmaz.

⚠️ **Aynı adreste bu uyarı tekrar çıkarsa dur.** İki ihtimal var: SD kart yeniden imajlandı
(normal), ya da o IP'de artık başka bir makine var (normal değil). Bu mekanizma tam olarak
ikincisini yakalamak için var.

### Parola yazarken ekranda hiçbir şey görünmez

Yıldız yok, imleç ilerlemez, satır boş kalır. **Hata değil, kasıtlı** — omzunun üstünden
bakan biri parolanın kaç karakter olduğunu bile göremesin diye. Körlemesine yaz, Enter'a bas.
`sudo` da parolayı böyle sorar.

## SSH key — parola yerine anahtar

Parolayla giriş çalışır ama üretimde kapatılacak (brief §13.1). Yerine **anahtar çifti**
kullanılır.

```bash
ssh-keygen -t ed25519 -C "andon-bench"
```

Üç soru sorar: dosya yolu (Enter = varsayılan), passphrase (Enter = boş), tekrar (Enter).

İki dosya oluşur:

| Dosya | Nedir | Kural |
|---|---|---|
| `~/.ssh/id_ed25519` | **Özel anahtar** | Asla paylaşılmaz, kopyalanmaz, git'e girmez |
| `~/.ssh/id_ed25519.pub` | **Açık anahtar** | Paylaşılmak için var |

**Zihin modeli:** açık anahtar bir kilittir, özel anahtar onu açan tek anahtardır. Kilidi
istediğin kapıya takarsın (sunucu, GitHub); anahtar sende kalır.

Açık anahtarı görmek için:

```bash
cat ~/.ssh/id_ed25519.pub
```

Tek satırlık çıktı `ssh-ed25519` ile başlar, sondaki yorum (`andon-bench`) sadece hangi
makinenin anahtarı olduğunu hatırlamak içindir.

### `ed25519` neden

Modern, kısa ve hızlı bir anahtar tipi. Eski `rsa`'ya göre daha güvenli ve daha küçük.
Bir sebep yoksa hep bunu kullan.

### Passphrase

Anahtarın kendisini koruyan ikinci bir parola. Boş bırakırsan her bağlantıda parola sorulmaz
(pratik), ama anahtar dosyası çalınırsa doğrudan kullanılabilir. Bench makinesinde ve kendi
repo'na erişen bir anahtar için boş kabul edilebilir; üretim sunucusunda değil.

## Bağlantıyı test etme

```bash
ssh -T git@github.com
```

`-T` = "terminal açma, sadece kimliğimi doğrula". GitHub şunu döner:

```
Hi KULLANICI! You've successfully authenticated, but GitHub does not provide shell access.
```

Bu **başarı** mesajıdır. "Giriş yaptın ama burada komut çalıştıramazsın" diyor — zaten
istediğimiz de o.

## Uzak masaüstü değil, uzak editör

"GUI Pi'de değil laptop'ta olsun" derken kastedilen uzak masaüstü **değil**:

| Ne istiyorum | Nasıl |
|---|---|
| Komut çalıştır, log oku, servis yönet | SSH — üretimde de tek yol bu |
| Dosya düzenle, kod yaz, klasör ağacını gör | **VS Code Remote-SSH** — editör laptop'ta, dosyalar Pi'de |
| Pi'nin masaüstünü görmek | Monitörü var |

Teknik olarak mümkün ama bu projede kullanmayacağımız iki yol:
- **X11 forwarding** (`ssh -X`) — tek bir grafik pencereyi laptop'a taşır. Yavaş, gereksiz.
- **VNC** — tam masaüstünü verir. Yerel ağda politika ihlali değil ama filoda kapalı olacak:
  bir Pi'ye masaüstünden bağlanıp elle düzeltme yapmak o üniteyi diğerlerinden farklılaştırır.

**Neden önemli:** üretimdeki 16 Pi'de zaten bakabileceğin bir masaüstü olmayacak — ekranlarında
tam ekran Chromium kiosk duracak. Metin üzerinden çalışmayı öğrenmek zorluk değil, doğrudan
üretim pratiği.

## Bağlantı kesilirse

SSH oturumu koparsa çalışan komut da ölür. Uzun süren işler için `systemd` servisi
(ilerideki not) veya `tmux` kullanılır — sırası gelince.
