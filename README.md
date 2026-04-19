
# NATS: Cloud Native Mesajlaşma Sisteminin Teknik Değerlendirmesi

**Sakarya Üniversitesi**  
Bilgisayar ve Bilişim Bilimleri Fakültesi  
Bilgisayar Mühendisliği Anabilim Dalı

**Ders:** Bulut Bilişim  
**Dönem:** 2025–2026 Bahar  
**Seviye:** Yüksek Lisans  
**Öğretim Üyesi:** Dr. Öğr. Üyesi Baran Kaynak  
**Seçenek:** B — CNCF Projesi Keşfi ve Teknik Değerlendirme  
**Seçilen Proje:** NATS (Graduated Statüsü)  
**Teslim Tarihi:** 19 Nisan 2026  

**Sakarya, Nisan 2026**

---

## 📖 İçindekiler

- [Giriş ve Motivasyon](#-giriş-ve-motivasyon)
- [Mimari Analiz](#-mimari-analiz)
- [Kurulum ve Deneyim](#-kurulum-ve-deneyim)
- [Karşılaştırmalı Değerlendirme](#-karşılaştırmalı-değerlendirme)
- [Sonuç ve Öneriler](#-sonuç-ve-öneriler)
- [Kaynakça](#-kaynakça)

---

## 🚀 Giriş ve Motivasyon

### Bulut-Yerel Ekosisteminde Mesajlaşma Sorunu

Modern bulut-yerel mimarilerde, onlarca hatta yüzlerce bağımsız mikro hizmet birbiriyle sürekli iletişim kurmaktadır. Senkron yöntemler (REST, gRPC) **gecikme birikimi**, **kırılgan bağımlılıklar** ve **ölçeklendirme zorluğu** gibi ciddi sorunlara yol açar.

> **Temel Problem:**  
> N adet mikro hizmetin birbirleriyle doğrudan iletişim kurduğu bir sistemde bağlantı sayısı \(\frac{N(N-1)}{2}\) olur. 10 servis için bu 45, 50 servis için 1225 bağlantı anlamına gelir. NATS, tüm bu bağlantıları tek bir merkezi hub üzerinden yöneterek karmaşıklığı \(N\) bağlantıya indirger.

### NATS Nedir?

NATS, açık kaynaklı, yüksek performanslı bir mesajlaşma sistemidir. 2010 yılında Derek Collison tarafından Go ile geliştirilmiş, 2016’da CNCF bünyesine alınmış ve **2022’de CNCF Graduated statüsüne** ulaşmıştır.

- Tek binary, sıfır harici bağımlılık
- Pub/Sub, Request/Reply ve JetStream (persistent streaming) desteği
- ~10 MB RAM tüketimi

### Proje Seçim Gerekçesi

- IoT ve endüstriyel sistemlerde düşük gecikme ihtiyacı
- Mikro hizmet mimarilerinde servisler arası iletişim
- Tek container ile çalışabilmesi
- CNCF Graduated statüsü (uzun vadeli destek)
- Rakipsiz kaynak verimliliği

---

## 🏗️ Mimari Analiz

### İç Mimari

| Katman | Bileşenler |
|--------|------------|
| **İstemci Katmanı** | Go, Python, Node.js client |
| **NATS Server Core** | JetStream Engine, Auth Layer, Monitoring (:8222) |
| **Depolama Katmanı** | File Storage, Memory Storage |

### İletişim Modelleri

| Özellik | Pub/Sub | Request/Reply | JetStream |
|---------|---------|---------------|------------|
| Kalıcılık | Hayır | Hayır | Evet |
| Yön | Tek yön | Çift yön | Tek yön |
| Teslimat garantisi | En fazla bir kez | En fazla bir kez | En az / tam bir kez |
| Gecikme | <1ms | <2ms | <5ms |
| Kullanım alanı | Bildirim, yayın | RPC, sorgulama | Sipariş, ödeme |

### Subject Sistemi

| Subject | Tür | Eşleşme Örneği |
|---------|-----|----------------|
| `orders.new` | Tam eşleşme | Yalnızca `orders.new` |
| `orders.*` | Tek seviye wildcard | `orders.new`, `orders.cancel` |
| `orders.>` | Çok seviyeli wildcard | `orders.new.usa`, `orders.items.list` |

### Tasarım Felsefesi

- **Basitlik:** Tek binary, sıfır harici bağımlılık
- **Performans:** Hashmap tabanlı O(1) subject matching
- **Güvenlik:** TLS, token/NKey/JWT kimlik doğrulama
- **Multi-tenancy:** Account sistemi ile tam izolasyon
- **Dayanıklılık:** At-least-once ve exactly-once teslimat garantisi

---

## 🛠️ Kurulum ve Deneyim

### Ortam Bilgileri

| Parametre | Değer |
|-----------|-------|
| Sunucu İşletim Sistemi | Ubuntu 22.04.5 LTS |
| RAM | 16 GB |
| CPU | 6 Core |
| Depolama | 96 GB SSD |
| Docker Sürümü | 29.1.3 |
| Docker Compose Sürümü | 1.29.2 |
| NATS Sürümü | 2.12.6 (Go 1.25.8) |
| IP Adresi | 51.38.113.222 |

### Kurulum Adımları

#### 1. Docker Kurulumu

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y docker.io docker-compose
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER
newgrp docker
```

#### 2. Proje Yapısı

```bash
mkdir ~/nats-project && cd ~/nats-project
```

#### 3. docker-compose.yml

```yaml
version: '3.8'

services:
  nats:
    image: nats:latest
    container_name: nats-server
    ports:
      - "4222:4222"   # Client connections
      - "8222:8222"   # HTTP Monitoring
      - "6222:6222"   # Cluster routing
    command: ["-m", "8222", "-js"]
    restart: unless-stopped

  nats-box:
    image: natsio/nats-box:latest
    container_name: nats-box
    depends_on:
      - nats
    stdin_open: true
    tty: true
    restart: unless-stopped
```

> **Not:** `-js` flag'i JetStream'i aktifleştirir. `-m 8222` HTTP monitoring endpoint'ini açar.

#### 4. Servisleri Başlatma

```bash
docker-compose up -d
```

### Test 1: Publish/Subscribe

**Subscriber (Terminal 1):**
```bash
docker exec -it nats-box nats sub test.subject --server nats://nats-server:4222
```

**Publisher (Terminal 2):**
```bash
docker exec -it nats-box nats pub test.subject "Merhaba Ben Bilal!" --server nats://nats-server:4222
```

### Test 2: JetStream (Kalıcı Mesajlaşma)

**Stream oluşturma:**
```bash
docker exec -it nats-box nats stream add ORDERS \
  --subjects "orders.*" \
  --storage file \
  --replicas 1 \
  --defaults \
  --server nats://nats-server:4222
```

**Mesaj gönderme:**
```bash
docker exec -it nats-box nats pub orders.new "order-001" --server nats://nats-server:4222
```

**Mesaj görüntüleme:**
```bash
docker exec -it nats-box nats stream view ORDERS --server nats://nats-server:4222
```

### Test 3: Monitoring API

```bash
curl http://localhost:8222/varz | python3 -m json.tool
```

**Metrik Karşılaştırması:**

| Metrik | Test Öncesi | Test Sonrası |
|--------|-------------|--------------|
| Uptime | 34 saniye | 2 dk 28 sn |
| Bellek Kullanımı | 11.15 MB | 16.39 MB |
| Toplam Bağlantı | 0 | 5 |
| Gelen Mesaj Sayısı | 0 | 9 |
| Giden Mesaj Sayısı | 0 | 10 |
| JetStream Storage | 0 B | 49 B |

> **Temel Bulgu:** Bellek kullanımı 9 mesaj sonrasında sadece 5.24 MB arttı.

### Test 4: Consumer Oluşturma

```bash
# Consumer oluştur
docker exec -it nats-box nats consumer add ORDERS CONSUMER1 \
  --deliver all --pull --defaults \
  --server nats://nats-server:4222

# Mesaj tüket
docker exec -it nats-box nats consumer next ORDERS CONSUMER1 \
  --server nats://nats-server:4222
```

### Karşılaşılan Sorunlar

| Sorun | Neden | Çözüm |
|-------|-------|-------|
| `no servers available` | Container içinde localhost NATS'a ulaşmıyor | `--server nats://nats-server:4222` eklendi |
| İlk pull'da yavaşlık | İmajlar ilk kez indiriliyor | İkinci çalıştırmada anlık başladı |

---

## 📊 Karşılaştırmalı Değerlendirme

### Genel Karşılaştırma

| Kriter | NATS | Apache Kafka | RabbitMQ |
|--------|------|--------------|----------|
| Lisans | Apache 2.0 | Apache 2.0 | MPL 2.0 |
| CNCF Statüsü | **Graduated** | — | — |
| Dil | Go | Scala/Java | Erlang |
| Minimum RAM | **~10 MB** | ~512 MB | ~128 MB |
| Harici Bağımlılık | **Yok** | ZooKeeper/KRaft | Erlang Runtime |
| Kalıcı Mesajlaşma | JetStream | Evet | Evet |
| Max Throughput | **~10M msg/s** | ~1M msg/s | ~50K msg/s |
| Gecikme | **<1ms** | 2-10ms | 1-5ms |
| Kurulum Zorluğu | **Kolay** | Zor | Orta |

### Senaryo Bazlı Uyumluluk

| Senaryo | NATS | Kafka | RabbitMQ |
|---------|------|-------|----------|
| IoT / Edge mesajlaşma | **Mükemmel** | Ağır | İyi |
| Mikro hizmet iletişimi | **Mükemmel** | İyi | İyi |
| Büyük veri pipeline | Orta | **Mükemmel** | Zayıf |
| Gerçek zamanlı bildirim | **Mükemmel** | Gecikmeli | İyi |
| Kısıtlı kaynak ortamı | **Mükemmel** | Uygunsuz | Orta |

---

## 📝 Sonuç ve Öneriler

### Güçlü Yönler

1. **Olağanüstü hafiflik:** ~10 MB bellek ile başlar, kaynak kısıtlı ortamlar için ideal
2. **Sıfır bağımlılık:** Tek binary ile çalışır
3. **Çok modlu iletişim:** Pub/Sub, Request/Reply ve JetStream tek altyapıda
4. **CNCF Graduated:** Üretim ortamı olgunluğu ve uzun vadeli destek
5. **Multi-protocol:** MQTT, WebSocket ve core NATS protokolleri

### Zayıf Yönler

1. Büyük veri ekosistemi entegrasyonu (Flink, Spark) Kafka'ya göre sınırlı
2. JetStream, Kafka'nın partition tabanlı parallel processing kapasitesine henüz tam ulaşamıyor
3. Confluent (Kafka) gibi güçlü ticari destek yok

### NATS Hangi Senaryolarda Tercih Edilmeli?

- **IoT ve edge computing:** Düşük bellek ve ağ bant genişliği
- **Mikro hizmet iletişimi:** Hafif, hızlı servisler arası mesajlaşma
- **Gerçek zamanlı sistemler:** Fintech, oyun, bildirim sistemleri
- **Multi-tenant SaaS:** Account sistemi ile izolasyon gereken platformlar

---

## 📚 Kaynakça

1. Synadia Communications. (2024). *NATS Official Documentation*.  
   https://docs.nats.io

2. Synadia Communications. (2024). *NATS Concepts: Subject-Based Messaging, JetStream*.  
   https://docs.nats.io/nats-concepts/overview

3. Synadia Communications. (2024). *JetStream Technical Documentation*.  
   https://docs.nats.io/nats-concepts/jetstream

4. Cloud Native Computing Foundation. (2022). *NATS Graduates within the Cloud Native Computing Foundation*.  
   https://www.cncf.io/projects/nats/

5. Dobbelaere, P., & Esmaili, K. S. (2017). Kafka versus RabbitMQ: A comparative study.  
   *DEBS '17*. https://doi.org/10.1145/3093742.3093908

6. Richardson, C. (2018). *Microservices Patterns: With Examples in Java*. Manning Publications.

7. Burns, B., et al. (2016). Borg, Omega, and Kubernetes. *ACM Queue*, 14(1), 70–93.

---

> **Not:** Bu çalışmada yapay zekâ araçları (Claude - Anthropic) içerik taslağı ve düzenleme süreçlerinde yardımcı amaçlı kullanılmış; tüm teknik kurulum, testler ve değerlendirmeler bizzat gerçekleştirilmiştir.
