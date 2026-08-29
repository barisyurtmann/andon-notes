# Ansible ve Filo Yönetimi

**Ne zaman lazım: Faz 2.** Tek Pi varken öğrenmek boşa gider (brief §7.2). Bu not temel
kavramları verir; sırası gelince derinleşeceğiz.

## 1. Problem

16 (sonra daha fazla) Pi var. Hepsinde aynı yazılım, aynı ayar, aynı sürüm olmalı.

**Elle yapılırsa ne olur:** 9 numaralı makinede bir gece hızlıca bir düzeltme yaparsın,
yazmayı unutursun. Altı ay sonra o Pi diğer 15'ten farklıdır ve neden farklı olduğunu kimse
bilmez. Bu, OEE sistemlerinin sessizce çürüdüğü yerdir.

**Brief §4.3'ün kuralı:** *"9 numaralı makinede dokümansız elle düzenleme olamaz."*

## 2. Ansible nedir

Bir merkezden, SSH üzerinden, birden fazla makineye **aynı işi** yapan araç.

- **Ajan gerekmez** — hedef makinede özel bir yazılım kurulu olması şart değil, SSH yeter
- **Bildirimsel (declarative)** — "şu paket kurulu olsun" dersin, Ansible kurulu değilse
  kurar, kuruluysa dokunmaz
- **Idempotent** — aynı playbook'u iki kez çalıştırmak zarar vermez, sonuç aynıdır

**Otomasyon karşılığı:** aynı ladder programını 16 PLC'ye tek tek download etmek yerine, bir
listeye bakıp hepsine aynı sürümü yükleyen bir araç. Üstelik "zaten doğru sürüm varsa
dokunma" diyebilen.

## 3. Kavramlar

| Terim | Ne demek |
|---|---|
| **inventory** | Hangi makineler var, IP'leri ve grupları |
| **playbook** | Yapılacak işlerin YAML dosyası |
| **task** | Tek bir iş: paket kur, dosya kopyala, servis başlat |
| **module** | Görevi yapan hazır bileşen (`apt`, `copy`, `systemd`, `template`) |
| **role** | Tekrar kullanılabilir görev paketi |
| **vars / group_vars** | Değişkenler; makineye veya gruba özel değerler |
| **template** | İçinde değişken olan dosya şablonu (`.j2`) |
| **Vault** | Parola ve secret şifreleme |

## 4. Inventory — hat yapısıyla

Brief §12.4'ün repo yapısı:

```
andon-ansible/
├─ playbooks/
│   ├─ site.yml
│   └─ collector.yml
├─ inventory/
│   ├─ hat-4inc/
│   │   ├─ hosts.yml
│   │   └─ machines.yaml
│   └─ hat-3inc/          ← yeni hat = sadece yeni klasör
└─ group_vars/
```

```yaml
# inventory/hat-4inc/hosts.yml
all:
  children:
    andon_nodes:
      hosts:
        andon-m001:
          ansible_host: 192.168.10.11
          machine_id: M001
        andon-m002:
          ansible_host: 192.168.10.12
          machine_id: M002
```

**Yeni hat için kod repo'su kopyalanmaz** — kod hattan bağımsız, sadece envanter klasörü
eklenir. Kodu fork'lamak bir hafta sonra iki farklı yazılıma ve iki katına çıkmış bakım
yüküne yol açar.

## 5. Playbook örneği

```yaml
- name: Configure andon nodes
  hosts: andon_nodes
  become: true                       # sudo ile çalıştır
  tasks:

    - name: Install packages
      apt:
        name: [python3-pip, chrony]
        state: present
        update_cache: true

    - name: Deploy machine config
      template:
        src: machines.yaml.j2
        dest: /etc/andon/machines.yaml
        owner: andon
        mode: "0644"
      notify: restart collector

    - name: Enable collector service
      systemd:
        name: andon-collector
        enabled: true
        state: started

  handlers:
    - name: restart collector
      systemd:
        name: andon-collector
        state: restarted
```

`notify` + `handlers` = "dosya gerçekten değiştiyse servisi yeniden başlat". Değişmediyse
dokunmaz — idempotency budur.

```bash
ansible-playbook -i inventory/hat-4inc playbooks/site.yml
ansible-playbook -i inventory/hat-4inc playbooks/site.yml --check   # deneme, değişiklik yapmaz
ansible -i inventory/hat-4inc andon_nodes -m ping                   # hepsi ayakta mı
```

`--check` (dry run) alışkanlığı: önce ne olacağını gör, sonra çalıştır.

## 6. Bu projede özel durumlar

### Read-only root

Overlay FS açıkken dosya sistemi salt okunur. Ansible playbook'unun sırası:
overlay'i kapat → değişiklikleri yap → overlay'i aç → reboot.

Bu bir zorluk değil, **dayatılan disiplin**: elle düzenleme fiilen imkânsız hale gelir.

### Secrets

Parolalar (MQTT kullanıcı/parola, veritabanı) **Ansible Vault**'ta şifreli durur:

```bash
ansible-vault encrypt group_vars/all/secrets.yml
ansible-playbook ... --ask-vault-pass
```

**Kural (brief §13.1): secrets asla düz metin olarak git'e girmez.**

### Golden image + Ansible birlikte

- **Golden image** = temel kurulum (OS, paketler, kullanıcı, SSH key). 18 karta yazılır
- **Ansible** = makineye özel ayarlar (hostname, `machine_id`, config) ve sonraki güncellemeler

İkisi birbirini tamamlar: imaj başlangıç noktası, Ansible sürekliliği sağlar.

## 7. IT'ye verilen taahhüdün karşılığı

Brief §3.6'da IT'ye şu söylenecek: *"18 node'u tek komutla, aynı image'dan, hiçbir yerde elle
düzenleme olmadan yamalayabilirim."*

**Ansible bu cümlenin karşılığıdır.** Onsuz o cümle bir iddia, onunla bir olgu.

## 8. Öğrenirken nereye kadar

**Yeterli:** inventory, playbook, `apt`/`copy`/`template`/`systemd` modülleri, handlers,
group_vars, Vault temel kullanımı.

**Bakma:** AWX/Tower, dinamik inventory, custom module yazma, collections derinliği.

**Süre:** ~1 hafta, Faz 2'de. Şimdi öğrenmenin bir faydası yok — uygulayacak makine yok.
