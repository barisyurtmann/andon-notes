# Docker

## 1. Docker nedir

Bir programı, ihtiyaç duyduğu her şeyle (kütüphaneler, ayarlar, işletim sistemi parçaları)
birlikte tek bir pakete koyup her yerde aynı şekilde çalıştırma yöntemi.

**Otomasyon karşılığı:** hazır bir pano modülü. İçini bilmene gerek yok, klemenslerini
bağlarsın çalışır. Aynı modülü başka bir panoya taktığında aynı davranır.

**Çözdüğü asıl problem:** "benim makinemde çalışıyordu". Konteyner her yerde aynı sürümü,
aynı bağımlılıklarla taşır.

## 2. Kavramlar

| Terim | Ne demek |
|---|---|
| **image (imaj)** | Şablon. Salt okunur, değişmez paket. Örn: `timescale/timescaledb:latest-pg16` |
| **container (konteyner)** | İmajdan üretilmiş, çalışan örnek |
| **volume** | Konteyner silinince kaybolmayacak kalıcı veri alanı |
| **port mapping** | Konteynerin içindeki portu dışarıya açmak |
| **registry** | İmajların indirildiği depo (Docker Hub) |

**Kritik ayrım:** konteyner silinince içindekiler gider. Kalması gereken her şey (veritabanı
dosyaları, Mosquitto'nun kalıcı verisi) **volume**'da durmalı.

## 3. Neden bu projede

Sunucuda çalışacak her şey konteynerde olacak (brief §6):

| Servis | İmaj |
|---|---|
| MQTT broker | `eclipse-mosquitto` |
| Veritabanı | `timescale/timescaledb` |
| Dashboard | `grafana/grafana` |
| Ingest servisi | kendi yazacağımız |

**Kazancı:**
- Sunucu ölürse tüm yığın **tek dosyadan** yeniden kurulur — brief §8.2'nin "sunucu tek arıza
  noktası" riskini azaltan şey bu
- Sürümler sabitlenir; "güncelleme sonrası bozuldu" olmaz
- Kurulum adımları kafanda değil, dosyada yazılı (bus factor, §15)

**Pi'lerde Docker yok.** Orada `systemd` servisleri çalışacak — 2 GB RAM'de gereksiz katman
eklemenin anlamı yok.

## 4. Temel komutlar

```bash
docker ps                        # çalışan konteynerler
docker ps -a                     # duranlar dahil
docker images                    # indirilmiş imajlar
docker logs KONTEYNER            # logunu oku
docker logs -f KONTEYNER         # canlı izle
docker exec -it KONTEYNER bash   # konteynerin içine gir
docker stop / start / rm KONTEYNER
docker volume ls                 # volume'lar
docker system df                 # ne kadar yer kaplıyor
```

## 5. `docker-compose.yml`

Birden fazla konteyneri tek dosyada tanımlar. **Sunucudaki tüm yığın bu dosyada olacak.**

```yaml
services:
  mosquitto:
    image: eclipse-mosquitto:2
    restart: unless-stopped
    ports:
      - "1883:1883"                       # dışarı:içeri
    volumes:
      - ./mosquitto/config:/mosquitto/config
      - mosquitto_data:/mosquitto/data

  timescaledb:
    image: timescale/timescaledb:latest-pg16
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}   # .env dosyasından gelir
      POSTGRES_DB: andon
    ports:
      - "5432:5432"
    volumes:
      - tsdb_data:/var/lib/postgresql/data

volumes:
  mosquitto_data:
  tsdb_data:
```

```bash
docker compose up -d          # başlat (-d = arka planda)
docker compose ps             # durum
docker compose logs -f        # tüm logları izle
docker compose down           # durdur ve kaldır (volume'lar kalır)
docker compose pull           # imajları güncelle
```

### Satır satır ne yapıyor

| Satır | Anlamı |
|---|---|
| `image:` | Hangi imaj, hangi sürüm. **`latest` yerine sürüm sabitle** |
| `restart: unless-stopped` | Çökerse ve sunucu yeniden başlarsa geri gelsin |
| `ports: "1883:1883"` | Sunucunun 1883 portu → konteynerin 1883 portu |
| `volumes:` | Kalıcı veri. `./yerel:/konteyner` veya isimli volume |
| `environment:` | Konteynere verilen ayarlar |
| `${DB_PASSWORD}` | `.env` dosyasından okunur — **parola dosyaya yazılmaz** |

**Kural (brief §13.1): secrets git'e girmez.** `.env` `.gitignore`'da olacak;
`docker-compose.yml` git'te olacak.

## 6. Volume — nerede saklanıyor

İki tür:
- **İsimli volume** (`tsdb_data:`) — Docker yönetir, `/var/lib/docker/volumes` altında
- **Bind mount** (`./config:/etc/config`) — sunucudaki gerçek bir klasör

Veritabanı için isimli volume, ayar dosyaları için bind mount kullanılır.

⚠️ **`docker compose down -v` volume'ları da siler.** Veritabanını uçurur. `-v` yazmadan önce
iki kez düşün.

## 7. Yedekleme

Konteyner yedeklenmez, **volume yedeklenir.** TimescaleDB için doğrusu veritabanı dump'ı:

```bash
docker compose exec timescaledb pg_dump -U postgres andon > yedek.sql
```

**Kural (brief §8.2): test edilmemiş yedek, yedek değildir.** Geri yükleme prosedürü yılda
en az bir kez gerçekten denenecek.

## 8. Öğrenirken nereye kadar

**Yeterli olan:** image, container, volume, port mapping, `docker compose up/down/logs`.

**Bakma:** Kubernetes, Docker Swarm, kendi imajını optimize etme, multi-stage build. Bu
ölçekte zaman kaybı (brief §7.1).

**Öğrenme yöntemi:** "Docker öğreneyim" değil, **"Mosquitto'yu Docker'da ayağa kaldırayım"**.
Brief §9.1 adım D tam olarak bu.
