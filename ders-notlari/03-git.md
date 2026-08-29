# Git

## Git ile GitHub aynı şey değil

| | Git | GitHub |
|---|---|---|
| Ne | Bilgisayarında çalışan bir program | İnternette bir site |
| Ne yapar | Dosyalarının her sürümünü diskte saklar | Git deposunun bir kopyasını barındırır |
| İnternet gerekir mi | **Hayır** | Evet |
| Hesap gerekir mi | **Hayır** | Evet |

`git init` çalıştırdığında klasörde `.git` adında gizli bir klasör oluşur ve o klasörün tüm
geçmişi orada tutulur. **Hiçbir yere bağlanmaz.**

**Otomasyon karşılığı:** ISPSoft projesini `proje_v1.isp`, `proje_v2_son.isp`,
`proje_v2_son_GERCEK.isp` diye kaydetmek yerine, tek dosyada "hangi tarihte neyi değiştirdim"
geçmişini tutan sistem. GitHub ise o dosyayı bir sunucuya yedeklemek — ayrı bir iş.

## Kavramlar

| Terim | Ne demek |
|---|---|
| **repository (repo)** | Geçmişi tutulan klasör |
| **commit** | Bir anlık fotoğraf: "şu dosyalar, şu anda, şu sebeple böyleydi" |
| **staging area** | Bir sonraki commit'e hangi değişikliklerin gireceğini seçtiğin ara alan |
| **branch** | Paralel geliştirme dalı. Bizde şimdilik tek dal: `main` |
| **remote** | Uzaktaki kopya (GitHub'daki repo). Bizde adı `origin` |

**`add` → `commit` neden iki adım:** on dosyayı değiştirdin ama sadece üçü aynı işe ait.
`git add` ile o üçünü seçer, `git commit` ile onları tek bir anlamlı kayıt yaparsın.
Küçük projede fark etmez, alışkanlık olarak doğru.

## Kullandığımız komutlar

### Kurulum (repo başına bir kez)

```bash
git init                                    # bu klasörü repo yap
git config user.name "Baris Yurtman"        # commit'lere yazılacak isim
git config user.email "..."                 # ve e-posta
git branch -M main                          # ana dalın adı main olsun
```

`user.name` ve `user.email` sadece "bu değişikliği kim yaptı" bilgisidir; hiçbir yere
gönderilmez, hesap açmaz.

### Günlük akış

```bash
git status              # ne değişti, ne staged
git add .               # tüm değişiklikleri staging'e al ("." = bu klasör)
git add dosya.yaml      # ya da tek dosya
git commit -m "Mesaj"   # kaydet
git log --oneline       # geçmişi kısa listele
git diff                # henüz staged olmayan değişiklikleri göster
```

**Commit mesajı nasıl yazılır:** İngilizce, emir kipi, ne yapıldığını söyler.
İyi: `Add machine config template for M001`
Kötü: `değişiklik`, `fix`, `asdf`

### GitHub'a bağlama

```bash
git remote add origin https://github.com/KULLANICI/andon-collector.git
git push -u origin main     # ilk gönderim; -u = "bundan sonra varsayılan bu olsun"
git push                    # sonraki gönderimler
git pull                    # uzaktaki değişiklikleri al
```

### `.gitignore`

Git'in **takip etmemesi** gereken dosyalar bu dosyada listelenir. Bizde:

```
.env            # parolalar, token'lar
*.key  *.pem    # anahtarlar
secrets.yaml
*.bak           # yedek kopyalar
__pycache__/    # Python'un ürettiği geçici dosyalar
```

**Kural (brief §13.1): secrets asla git'e girmez.** Bir kez commit'lersen geçmişten
silmek zordur — o yüzden `.gitignore` ilk commit'te yazılır.

## Bu projedeki akış

```
PC (yaz, commit, push) ──▶ GitHub (private) ──▶ Pi: git pull (sadece okuma)
```

- **Tek repo seti var, 16 tane değil.** Üretimdeki Pi'lerde git bulunmaz.
- **Pi'ler repo'ya yazamaz** — read-only deploy key alacaklar.
- **Elle düzenleme yok.** 16 Pi'de ayrı geçmiş olsaydı, altı ay sonra hangi Pi'de hangi
  sürümün çalıştığını kimse bilemezdi.
- **Yeni hat için repo kopyalanmaz** — kod hattan bağımsız, her hat sadece kendi envanter
  klasörünü ekler.

## Repo'larımız

| Repo | İçerik |
|---|---|
| `andon-collector` | Collector kodu ve `machines.yaml` |
| `andon-notes` | Çalışma günlüğü ve bu ders notları |

İkisi de **private**. Şahsi hesap geçici; kalıcı olarak firma organizasyonuna taşınacak
(brief §15 — bus factor).
