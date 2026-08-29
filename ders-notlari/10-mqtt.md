# MQTT

## 1. MQTT nedir

Cihazların birbirine küçük mesajlar göndermesi için tasarlanmış, hafif bir haberleşme
protokolü. Endüstriyel IoT'nin fiili standardı.

**Otomasyon karşılığı:** Modbus'ta *sen sorarsın, cihaz cevap verir* (poll). MQTT'de cihaz
*kendiliğinden yayınlar*, ilgilenen herkes duyar. Yani polling değil, yayın/abone modeli.

## 2. Üç rol

| Rol | Kim | Ne yapar |
|---|---|---|
| **Publisher** | Pi'deki collector | Mesaj yayınlar |
| **Broker** | Sunucudaki Mosquitto | Mesajları alır, ilgililere dağıtır |
| **Subscriber** | Sunucudaki ingest servisi | İlgilendiği konulara abone olur |

**Publisher ile subscriber birbirini tanımaz.** İkisi de sadece broker'ı bilir. Bu, yeni
tüketici eklemeyi bedava yapar: yarın bir "canlı ekran" servisi yazarsan, collector'da tek
satır değişmez.

## 3. Topic (konu)

Mesajın adresi. `/` ile bölünmüş hiyerarşi:

```
andon/makine/M001/telemetry
andon/makine/M001/state
andon/makine/M001/andon
andon/makine/M001/heartbeat
andon/sunucu/motor_kodu
```

**Joker karakterler (abone olurken):**

| Karakter | Anlamı | Örnek |
|---|---|---|
| `+` | Tek seviye | `andon/makine/+/telemetry` → tüm makinelerin telemetrisi |
| `#` | Bu seviye ve altı | `andon/makine/M001/#` → M001'in her şeyi |

⚠️ Topic'te **boşluk, Türkçe karakter, `+`, `#` olmaz.** `machine_id`'nin anlamsız ve dar
formatlı (`M001`) seçilmesinin bir sebebi de bu (brief §12.4).

**Brief'in topic şeması (§12.1):** hat/bölüm/fabrika topic'e **konmaz** — makine hat
değiştirdiğinde topic değişir, geçmiş kopar, abonelikler bozulur. Bu bilgi veritabanında
yaşar.

## 4. QoS — teslim garantisi

| QoS | Anlamı | Riski |
|---|---|---|
| 0 | "Gönder, unut" | Mesaj kaybolabilir |
| 1 | "En az bir kez ulaşsın" | **Kopya oluşabilir** |
| 2 | "Tam olarak bir kez" | Ağır, yavaş |

**Bu projenin kararı: her yerde QoS 1** (brief §12.1). QoS 0 kaybeder, QoS 2 gereksiz ağırdır.

QoS 1'in kopya üretme riskini **idempotency ile** çözüyoruz, retry mekanizmasını
zayıflatarak değil: her mesajda `msg_id = machine_id + boot_id + seq` var, veritabanında
`UNIQUE(msg_id)` ve `INSERT ... ON CONFLICT DO NOTHING`. Aynı mesaj kaç kez gelirse gelsin
tek satır olur (brief Karar #18).

## 5. Retained mesaj

Broker, topic'in **son** mesajını saklar ve yeni abone bağlandığında ona hemen verir.

**Kural (brief §12.1):**
- **Durum bilgisinde kullan** — `state`, `heartbeat`, `motor_kodu`. Yeni bağlanan istemci
  güncel durumu hemen öğrenir
- **Telemetride kullanma** — yeni bağlanan bir istemci eski bir ölçümü canlı sanır

## 6. LWT — Last Will and Testament

Bağlanırken broker'a bir "vasiyet" bırakırsın: *"benim bağlantım koparsa şu topic'e şu
mesajı sen yayınla."*

```
heartbeat topic'i, retained:
  bağlanırken   → {"status": "online"}
  LWT (vasiyet) → {"status": "offline"}
```

Pi'nin fişi çekilirse broker `offline` mesajını **kendisi** yayınlar. Sunucuda zaman aşımı
mantığı yazmana gerek kalmaz.

**Bu projede:** filo sağlık sayfasındaki 16 yeşil/kırmızı nokta böyle çalışacak (brief §3.4).
Heartbeat sayacı yazmaktan daha basit ve daha güvenilir.

## 7. Oturum ve yeniden bağlanma

- **`clean_session = False`** ve sabit `client_id = machine_id` (brief §12.1). Broker kısa
  kesintilerde oturumu ve bekleyen mesajları tutar.
- İstemci kütüphanesi (`paho-mqtt`) kopan bağlantıyı otomatik yeniden kurar.
- Ağ uzun süre kopuksa mesajlar **Pi'deki yerel kuyrukta** birikir (store-and-forward) ve
  bağlantı gelince topluca gönderilir. Bu yüzden idempotency şart.

## 8. Kimlik doğrulama ve güvenlik

- **Anonim erişim kapalı.** Her Pi için ayrı kullanıcı/parola (brief Karar #20)
- **Topic ACL:** her Pi sadece kendi `machine_id`'sine publish edebilir. Yanlış yapılandırılmış
  bir Pi başka makinenin verisini bozamaz
- **TLS (8883):** üretim VLAN'ında zorunlu değil ama Mosquitto'da açmak ucuz; IT isterse hazır ol
- Parolalar **Ansible Vault**'ta, git'te değil

## 9. Elle deneme — `mosquitto_pub` / `mosquitto_sub`

Öğrenmenin en hızlı yolu iki terminal açmak:

```bash
sudo apt install mosquitto-clients

# Terminal 1 — dinle
mosquitto_sub -h 192.168.0.10 -t "andon/makine/+/telemetry" -v

# Terminal 2 — gönder
mosquitto_pub -h 192.168.0.10 -t "andon/makine/M001/telemetry" -m '{"state":2}'
```

`-v` = topic'i de göster. Bu iki komutla MQTT'nin tamamını yarım saatte anlarsın.

## 10. Mesaj biçimi — bu projede

```json
{
  "msg_id":    "M001-8f2a1c-000148213",
  "machine_id":"M001",
  "boot_id":   "8f2a1c",
  "seq":       148213,
  "ts":        "2026-08-25T09:14:03.000Z",
  "ts_synced": true,
  "counters":  { "total": 1048576, "scrap": 312 },
  "state":     2,
  "fault":     0,
  "cycle_ms":  4120
}
```

| Alan | Neden var |
|---|---|
| `msg_id` | Idempotency anahtarı — kopya kayıt oluşmasın |
| `boot_id` | Her açılışta yeniden üretilir; `seq` sıfırlanınca çakışma olmasın |
| `seq` | Monoton artan sıra numarası |
| `ts` | Pi'nin ürettiği zaman, **UTC** |
| `ts_synced` | Saat NTP ile senkron değilken üretilen kayıtları işaretler |

**Neden Sparkplug B değil:** MQTT üzerine oturan, daha yapılandırılmış bir endüstriyel
standart. Bu ölçekte gereksiz karmaşıklık — düz MQTT + JSON yeterli (brief §6).

## 11. Neyi çözmez

MQTT bir **taşıma** protokolüdür; veri saklamaz, sorgulanamaz, geçmiş tutmaz. Onun için
TimescaleDB var. Broker'a "dün saat 3'te ne olmuştu" diye soramazsın.
