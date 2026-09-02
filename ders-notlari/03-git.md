# Git

## 1. Git ile GitHub aynı şey değil

| | Git | GitHub |
|---|---|---|
| Ne | Bilgisayarında çalışan bir program | İnternette bir site |
| Ne yapar | Dosyalarının her sürümünü diskte saklar | Git deposunun kopyasını barındırır |
| İnternet gerekir mi | **Hayır** | Evet |
| Hesap gerekir mi | **Hayır** | Evet |
| Alternatifi | (yok, standart bu) | GitLab, Gitea, Bitbucket, kendi sunucun |

`git init` çalıştırdığında klasörde `.git` adında gizli bir klasör oluşur ve o klasörün tüm
geçmişi orada tutulur. **Hiçbir yere bağlanmaz.**

**Otomasyon karşılığı:** ISPSoft projesini `proje_v1.isp`, `proje_v2_son.isp`,
`proje_v2_son_GERCEK.isp` diye kaydetmek yerine, tek dosyada "hangi tarihte neyi neden
değiştirdim" geçmişini tutan sistem.

## 2. Neden git (bu projede)

- **Geri alabilmek.** "Dün çalışıyordu" durumuna dönebilmek.
- **Neden yaptığımı hatırlamak.** Commit mesajları altı ay sonraki senin için.
- **Tek doğru kaynak.** 16 Pi aynı koddan çalışacak; kaynağı tek yerde tutmazsan filo dağılır.
- **Bus factor** (brief §15). Sen yokken sistemin nasıl kurulduğu yazılı olsun.
- **PLC programları da git'te olacak** (brief §11 madde 1) — en az bir makinede diskteki
  dosyanın PLC'dekiyle uyuşmadığını göreceksin.

## 3. Kavramlar

| Terim | Ne demek |
|---|---|
| **repository (repo)** | Geçmişi tutulan klasör |
| **commit** | Anlık fotoğraf: "şu dosyalar, şu anda, şu sebeple böyleydi" |
| **staging area (index)** | Bir sonraki commit'e hangi değişikliklerin gireceğini seçtiğin ara alan |
| **HEAD** | Şu an nerede olduğun — genelde son commit |
| **branch (dal)** | Paralel geliştirme çizgisi. Bizde şimdilik tek dal: `main` |
| **remote** | Uzaktaki kopya. Bizde adı `origin` (GitHub) |
| **clone** | Uzak repo'nun tam kopyasını indirmek |
| **merge** | İki dalı birleştirmek |
| **conflict** | Aynı satır iki yerde farklı değişmiş; git karar veremez, sen çözersin |

**Üç alan:**

```
çalışma klasörü  ──git add──▶  staging  ──git commit──▶  repo geçmişi  ──git push──▶  GitHub
   (düzenlediğin)               (seçtiğin)                 (kaydedilen)
```

**`add` → `commit` neden iki adım:** on dosyayı değiştirdin ama sadece üçü aynı işe ait.
`add` ile o üçünü seçer, `commit` ile tek anlamlı kayıt yaparsın.

## 4. Günlük komutlar

### Kurulum (repo başına bir kez)

```bash
git init                                  # bu klasörü repo yap
git config user.name "Baris Yurtman"      # commit'lere yazılacak isim
git config user.email "..."               # ve e-posta
git branch -M main                        # ana dalın adı main olsun
```

`--global` eklersen ayar tüm repo'lar için geçerli olur. İsim/e-posta hiçbir yere
gönderilmez, hesap açmaz — sadece commit'in içine yazılır.

### Akış

```bash
git status              # ne değişti, ne staged
git diff                # staged olmayan değişiklikler
git diff --staged       # staged olanlar
git add .               # tüm değişiklikleri staging'e al
git add machines.yaml   # ya da tek dosya
git commit -m "Mesaj"   # kaydet
git log --oneline       # geçmiş, kısa
git log -p dosya        # bir dosyanın değişim geçmişi, satır satır
git show COMMIT         # bir commit'te tam olarak ne değişmiş
```

### Geri alma

| Durum | Komut |
|---|---|
| Dosyadaki değişikliği çöpe at (henüz `add` yapılmadı) | `git restore dosya` |
| Staging'den çıkar ama değişikliği koru | `git restore --staged dosya` |
| Son commit'in mesajını düzelt | `git commit --amend -m "Yeni mesaj"` |
| Bir commit'i geri alan **yeni** commit üret | `git revert COMMIT` |
| Geçmişi geri sar (**dikkat**) | `git reset --hard COMMIT` |

**Kural:** paylaşılmış (push edilmiş) geçmişte `reset --hard` kullanma; `revert` kullan.
Tek kişilik repo'da bile alışkanlık böyle olsun.

### GitHub ile

```bash
git remote add origin https://github.com/KULLANICI/repo.git
git remote -v                 # bağlı uzak repo'ları göster
git push -u origin main       # ilk gönderim (-u = bundan sonra varsayılan)
git push                      # sonrakiler
git pull                      # uzaktaki değişiklikleri al (fetch + merge)
git fetch                     # sadece indir, birleştirme
git clone URL                 # uzak repo'yu ilk kez indir
```

## 5. Dallar (branch) — temel

```bash
git branch                    # dalları listele
git switch -c yeni-ozellik    # yeni dal aç ve geç
git switch main               # ana dala dön
git merge yeni-ozellik        # dalı main'e birleştir
git branch -d yeni-ozellik    # birleşmiş dalı sil
```

**Bu projede şimdilik gerek yok.** Tek kişi, tek çizgi. Faz 2'de "üretime çıkan sürüm" ile
"denenen sürüm" ayrımı gerekirse dal açarız. Erken karmaşıklık eklemenin anlamı yok.

**Conflict nedir:** iki dalda aynı satır farklı değişmişse git birleştiremez, dosyaya
`<<<<<<<` `=======` `>>>>>>>` işaretleri koyar. Sen doğru hali bırakır, işaretleri siler,
`git add` + `git commit` yaparsın.

## 6. Commit mesajı nasıl yazılır

- İngilizce, emir kipi, ne yapıldığını söyler (brief kuralı)
- İlk satır kısa (50–70 karakter), gerekirse boş satır sonra detay

İyi: `Add machine config template for M001`
İyi: `Fix word order in 32-bit counter read`
Kötü: `değişiklik`, `fix`, `asdf`, `son hali`

**Neden önemli:** altı ay sonra "bu adres neden değişmiş" sorusunun cevabı `git log` içinde
olacak. Mesaj kötüyse cevap yok demektir.

## 7. `.gitignore`

Git'in **takip etmemesi** gereken dosyalar burada listelenir:

```
.env            # parolalar, token'lar
*.key  *.pem    # anahtarlar
secrets.yaml
vault-password*
*.bak           # yedek kopyalar
__pycache__/    # Python geçici dosyaları
.venv/
```

**Kural (brief §13.1): secrets asla git'e girmez.** Bir kez commit'lersen geçmişten silmek
zordur — parolayı değiştirmek gerekir. O yüzden `.gitignore` ilk commit'te yazılır.

## 8. Bu projedeki akış

```
PC (yaz, commit, push) ──▶ GitHub (private) ──▶ Pi: git pull (sadece okuma)
                                                 Faz 2'den itibaren: Ansible
```

- **Tek repo seti var, 16 tane değil.** Üretimdeki Pi'lerde git bulunmaz.
- **Pi'ler repo'ya yazamaz** — read-only **deploy key** alacaklar. Deploy key = tek bir
  repo'ya erişen, hesaba bağlı olmayan anahtar.
- **Elle düzenleme yok.** 16 Pi'de ayrı geçmiş olsaydı, altı ay sonra hangi Pi'de hangi
  sürümün çalıştığını kimse bilemezdi (brief §4.3).
- **Yeni hat için repo kopyalanmaz** — kod hattan bağımsız; her hat sadece kendi envanter
  klasörünü ekler (brief §12.4).

### Repo'larımız

| Repo | İçerik |
|---|---|
| `andon-collector` | Collector kodu ve `machines.yaml` |
| `andon-notes` | Çalışma günlüğü ve ders notları |
| `andon-webapp` | (ileride) Makine ekranı |
| `andon-ansible` | (Faz 2) Dağıtım ve hat envanterleri |
| `plc-programs` | (ileride) PLC yedekleri, hat klasörleriyle |

İkisi de **private**. Şahsi hesap geçici; kalıcı olarak firma organizasyonuna taşınacak
(brief §15).

## 9. Faydalı küçük şeyler

```bash
git log --oneline --graph --all     # geçmişi görsel olarak
git blame dosya                     # her satırı kim, ne zaman, hangi commit'te yazmış
git stash                           # yarım işi geçici olarak rafa kaldır
git stash pop                       # geri al
git tag v1.0                        # sürüm etiketi (PLC program sürümüyle eşleşecek)
```

`git tag` bu projede işe yarayacak: brief §5.1'deki öneri, PLC programına git tag'iyle
eşleşen bir sürüm numarası (D308) yazmak.

## 10. `origin` gerçekte nedir — ve `ahead` / `behind` ne demek

`git clone` veya `git remote add` yaptığında git, uzaktaki repo için yerel diskinde bir
**kopya işaretçi** tutar: `origin/main`. Bu, "GitHub'ın şu andaki hali" **değildir** —
"git'in en son konuştuğunda GitHub böyleydi" bilgisidir. İnternet olmadan da okunur.

```
main          ← senin çalıştığın dal (yereldeki gerçek)
origin/main   ← GitHub'ın en son bilinen hali (yerelde tutulan not)
```

Bu ikisini karşılaştıran komut:

```bash
git status -sb        # -s kisa, -b dal bilgisini de goster
```

| Çıktı | Anlamı | Ne yapılır |
|---|---|---|
| `## main...origin/main` | Aynı noktadalar | Bir şey yok |
| `## main...origin/main [ahead 3]` | Sende 3 commit fazla, GitHub'da yok | `git push` |
| `## main...origin/main [behind 2]` | GitHub'da 2 commit var, sende yok | `git pull` |
| `## main...origin/main [ahead 1, behind 2]` | İkisi de ayrışmış | `git pull` sonra `git push` |

**`ahead` sayısı commit sayar, dosya değil.** 17 commit tek bir dosyaya da dokunmuş olabilir,
21 dosyaya da. Neyin gideceğini görmek için:

```bash
git log --oneline origin/main..main      # push edilecek commit'ler
git diff --stat origin/main..main        # push edilecek dosyalar ve satır sayilari
```

İki nokta (`..`) "şundan şuna kadar olan fark" demek. Sırası önemli:
`origin/main..main` = "bende olup orada olmayanlar".

**`git fetch` işaretçiyi tazeler, çalışma klasörüne dokunmaz.** `git pull` = `fetch` + `merge`.
Tek kişilik repo'da `pull` genelde gereksizdir; başka bir makineden commit atmadıysan
`origin/main` zaten senin bıraktığın yerdedir.

## 11. Kimlik doğrulama — kim push edebilir

Git'in kendisi kimlik bilmez. GitHub bilir. HTTPS ile push ederken GitHub **parola kabul
etmez**; token ister.

| Yöntem | Nerede durur | Bu projede |
|---|---|---|
| **Credential Manager** | Windows'un kimlik kasasında, kullanıcı hesabına bağlı | **PC'de kullanılan yöntem.** İlk push'ta tarayıcıdan giriş yapılır, sonrası sessizdir |
| **PAT** (personal access token) | Elle saklanan uzun bir dize | Yedek yöntem. Süresi biter, yenilenir |
| **SSH anahtarı** | `~/.ssh/id_ed25519` | Remote `git@github.com:...` biçiminde olsaydı bu kullanılırdı |
| **Deploy key** | Tek repo'ya bağlı SSH anahtarı, hesaba bağlı değil | **Pi'ler bunu alacak** — sadece okuma (brief §4.7, §12.4) |

**Pratik sonucu:** kimlik bilgisi Windows kullanıcı hesabının kasasında durduğu için, push
sadece o oturumdan çıkar. Başka bir ortamdan (başka makine, sanal ortam, Claude'un eriştiği
kabuk) çalıştırıldığında git şunu der:

```
fatal: could not read Username for 'https://github.com': terminal prompts disabled
```

Bu bir hata değil, beklenen davranış: **kasa taşınmaz.** Dosyaları düzenlemek ve commit'lemek
her yerden yapılabilir, `push` PC'den yapılır.

## 12. Orphan branch — 2026-09-02'de karşılaşıldı

`andon-collector` içinde `master` adında ikinci bir dal vardı ve `main` ile **ortak atası
yoktu**. Sebebi: repo hem yerelde `git init` ile hem GitHub'da "initialize with README" ile
başlatılınca iki ayrı başlangıç noktası oluşur.

Teşhis:

```bash
git merge-base master main
# cikti yoksa: ortak ata yok -> orphan
git log --oneline main..master        # master'da olup main'de olmayanlar
git diff --stat master 61c3503        # icerikleri ayni mi (bos cikti = ayni)
```

Sonuç: `master`'ın tek commit'i, `main`'in ilk commit'i ile **birebir aynı içerikteydi** —
sadece farklı hash. Silmek güvenliydi:

```bash
git branch -D master
```

`-d` birleşmemiş dalı silmeyi reddeder, `-D` zorlar. **`-D` kullanmadan önce her zaman
`git log main..DAL` çalıştır** ve o dalda kaybolacak benzersiz bir şey olmadığını gör.

**Genel kural:** repo'yu ya yerelde `git init` ile aç, ya GitHub'da boş (README'siz) aç.
İkisini birden yaparsan bu durum çıkar.

## 13. Push öncesi kontrol listesi

Filoya çıkacak kod için alışkanlık hâline gelmeli:

```bash
git status -sb                          # temiz mi, kac commit ahead
git diff --stat origin/main..main       # ne gidiyor
git log --oneline origin/main..main     # hangi commit'ler
cat .gitignore                          # secret bloğu yerinde mi
grep -rniE "password|token|api[_-]?key|BEGIN .*PRIVATE KEY" . --include="*.py" --include="*.yaml"
```

Son satır **push edilmeden önce** çalışır. Bir secret bir kez GitHub'a gittiyse geçmişten
silmek yetmez — o parolayı/anahtarı **değiştirmek** gerekir (brief §13.1). Private repo bunu
değiştirmez: repo sonradan organizasyona taşınacak, erişen kişi sayısı artacak.

Push'tan sonra doğrulama:

```bash
git status -sb                    # 'ahead' ifadesi kaybolmali
git log --oneline -1 origin/main  # uzak dal artik yeni commit'te mi
```
