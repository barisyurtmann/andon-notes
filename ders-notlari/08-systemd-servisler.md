# systemd ve Servisler

## 1. `systemd` nedir

Linux açılırken **her şeyi başlatan** program. Açılıştan kapanışa kadar sistemdeki tüm
servisleri yöneten yönetici.

**"Servis" (unit), ona verdiğin bir talimat dosyasıdır:**

> Şu programı çalıştır. Açılışta otomatik başlat. Çökerse 5 saniye sonra geri getir.
> Şu kullanıcı adına çalıştır. Çıktısını log sistemine yaz.

**PLC karşılığı:** ladder programını PLC'ye download edip **RUN**'a almak. Script'i elle
`python3 x.py` diye çalıştırmak ise sadece simülatörde koşturmaktır — terminalden çıkınca
ölür, elektrik gidince geri gelmez.

**Bu projede neden en önemli konu:** collector, Andon servisi ve ingest servisi — üçü de bu
kalıpla yazılacak. Bunu anlayınca projenin yazılım iskeleti eline geçer.

## 2. Servis dosyası

Yeri: `/etc/systemd/system/andon-button.service`

```ini
[Unit]
Description=Andon button service
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /home/andon/andon/button.py
WorkingDirectory=/home/andon/andon
Restart=always
RestartSec=5
User=andon
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

### Satır satır

| Satır | Ne yapar |
|---|---|
| `[Unit]` | Kimlik ve bağımlılıklar bölümü |
| `Description=` | `systemctl status`'ta görünen açıklama |
| `After=` | Bu hedeften **sonra** başlat (sıralama) |
| `Wants=` | Bu hedefi de başlatmaya çalış (zayıf bağımlılık) |
| `[Service]` | Nasıl çalıştırılacağı |
| `Type=simple` | Program ön planda çalışır ve çıkmaz (en yaygın) |
| `ExecStart=` | Çalıştırılacak komut. **Tam yol yazılır**, `python3` yetmez |
| `WorkingDirectory=` | Hangi klasörde çalışacak (göreli dosya yolları buna göre) |
| `Restart=always` | Çökerse veya çıkarsa geri getir |
| `RestartSec=5` | Yeniden başlatmadan önce 5 sn bekle |
| `User=andon` | Root olarak değil, bu kullanıcı adına çalıştır |
| `[Install]` | `enable` edildiğinde nereye bağlanacağı |
| `WantedBy=multi-user.target` | Normal açılışta başlasın |

**`Restart=always` neden önemli:** brief'in "bir vardiya gecesi 02:00'de tek kişilik ekiple
ayakta kalmak" ilkesi. Servis çöktüğünde birinin fark edip elle başlatmasını beklemek plan
değildir.

## 3. Komutlar

```bash
sudo systemctl daemon-reload            # servis dosyasını değiştirdiysen ÖNCE bu
sudo systemctl start andon-button       # şimdi başlat
sudo systemctl stop andon-button        # durdur
sudo systemctl restart andon-button     # yeniden başlat
sudo systemctl enable andon-button      # açılışta başlasın
sudo systemctl disable andon-button     # açılışta başlamasın
systemctl status andon-button           # durum + son log satırları
systemctl is-active andon-button        # sadece "active" / "inactive"
systemctl list-units --type=service     # çalışan tüm servisler
```

⚠️ **`enable` ile `start` farklıdır.** `start` şimdi çalıştırır, `enable` açılışta
çalışmasını sağlar. İkisi de gerekir — sadece `start` yaparsan reboot sonrası servis gelmez.

⚠️ Servis dosyasını düzenledikten sonra `daemon-reload` yapmazsan `systemd` eski hali
kullanmaya devam eder ve "değişikliğim neden işe yaramadı" diye yarım saat kaybedersin.

## 4. Loglar — `journalctl`

`systemd`'nin log sistemi. Servisin `print`/`logging` çıktıları buraya düşer.

```bash
journalctl -u andon-button              # servisin tüm logu
journalctl -u andon-button -f           # canlı izle (-f = follow)
journalctl -u andon-button -n 50        # son 50 satır
journalctl -u andon-button --since today
journalctl -u andon-button -p err       # sadece hata seviyesi
journalctl -b                           # bu açılıştan itibaren tüm sistem
journalctl --disk-usage                 # loglar ne kadar yer kaplıyor
```

⚠️ **Read-only root açıldığında loglar RAM'de tutulur ve reboot'ta kaybolur.** Bu yüzden
merkezî log toplama (Loki veya rsyslog) **zorunlu** hale gelir (brief §4.3).

## 5. Timer — cron yerine

Belirli aralıklarla iş çalıştırmak için. `cron`'un `systemd` karşılığı.

```ini
# /etc/systemd/system/kiosk-reboot.timer
[Unit]
Description=Nightly reboot for the kiosk

[Timer]
OnCalendar=*-*-* 03:30:00
Persistent=true

[Install]
WantedBy=timers.target
```

**Bu projede kullanılacak:** Chromium kiosk modunda haftalar içinde bellek sızdırır →
gecelik reboot (brief §4.2). Read-only root reboot'u zaten bedava yaptığı için ucuz sigorta.

```bash
systemctl list-timers          # tanımlı timer'lar ve sıradaki çalışma
```

## 6. Servis yazma sırası (alıştırma kalıbı)

1. Python script'ini yaz, elle çalıştır, çalıştığını gör
2. `sudo nano /etc/systemd/system/ad.service` ile servis dosyasını yaz
3. `sudo systemctl daemon-reload`
4. `sudo systemctl start ad` → `systemctl status ad` ile kontrol
5. `journalctl -u ad -f` ile logu izle
6. `sudo systemctl enable ad`
7. **`sudo reboot`** → servis kendiliğinden kalkmalı
8. Script'e kasten hata koy, çökert → `Restart=always` geri getirmeli

**Bitiş kriteri 7 ve 8'dir.** Piyasadaki hiçbir Raspberry Pi kursu buraya kadar getirmiyor
(brief Karar #23).

## 7. Sık yapılan hatalar

| Hata | Sebep |
|---|---|
| `status` → `code=exited, status=203/EXEC` | `ExecStart` yolu yanlış veya dosya çalıştırılabilir değil |
| Servis başlıyor, hemen ölüyor | Script bir hata verip çıkıyor — `journalctl` ile bak |
| Reboot'ta gelmiyor | `enable` yapılmamış |
| Değişiklik işe yaramıyor | `daemon-reload` yapılmamış |
| Dosyaya yazamıyor | `User=` yetkisi yok, ya da read-only root |
| Ağ hazır değilken çöküyor | `After=network-online.target` eksik |

## 8. Bu projede kurulacak servisler

| Servis | İş | Ne zaman |
|---|---|---|
| `andon-button` | GPIO butonunu dinle, MQTT'ye çağrı yay | Faz 1 adım C |
| `andon-collector` | PLC'yi 1 Hz poll et, kuyrukla, MQTT'ye gönder | Faz 1 adım E |
| Chromium kiosk | Operatör ekranı (autostart veya servis) | Faz 1–2 |
| `andon-ingest` | Sunucuda: MQTT → TimescaleDB | Faz 1 adım D (Docker içinde) |

Sunucudaki servisler Docker konteynerlerinde çalışacağı için orada `systemd` yerine
`docker compose` devreye girer — mantık aynı: "çöktüğünde geri gelsin, logu okunabilsin".
