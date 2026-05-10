# OnlineCourse — DevOps Altyapı Dokümantasyonu

Bu doküman, **OnlineCourse** projesinin Azure DevOps ortamında kurulumu, yapılandırması ve otomatik deployment süreçlerini kapsamaktadır. Pipeline dosyaları ve Dockerfile'lar `/pipelines` klasöründe ayrıca yer almaktadır.

---

## 📌 Proje Genel Bakış

| Bileşen | Teknoloji |
|---|---|
| Backend | Django (Python) |
| Frontend | Next.js (Node.js) |
| Veritabanı | Azure PostgreSQL |
| Container Registry | Azure Container Registry (ACR) |
| Sunucu | Azure Virtual Machine (Ubuntu 24.04) |
| Depolama  | Azure Blob Storage |
| CI/CD | Azure DevOps Pipelines |
| Reverse Proxy | Nginx |
| SSL | Let's Encrypt (Certbot) |
| Domain | onlinecourses.site |

---

## 🏗️ Mimari

```
Kullanıcı
   │
   ▼
onlinecourses.site (DNS → 20.56.30.152)
   │
   ▼
Nginx (80/443)
   ├── onlinecourses.site     → localhost:3000 (Frontend Container)
   └── api.onlinecourses.site → localhost:8000 (Backend Container)
         │
         ▼
   Azure PostgreSQL
```

Güvenlik gereği backend'in çalıştığı **8000 portu dışarıya kapalıdır**. Kullanıcıların ve frontend'in backend API'sine erişebilmesi için `api.onlinecourses.site` adında ayrı bir subdomain oluşturulmuştur. Bu subdomain aynı VM IP'sine yönlendirilmiş, Nginx üzerinde ise gelen istekleri VM içindeki `localhost:8000`'e iletecek şekilde yapılandırılmıştır. Böylece backend dışarıya hiç açılmadan, domain üzerinden güvenli şekilde erişilebilir hale gelmiştir.

---

## ☁️ Azure Servisleri

### 1. Azure Container Registry (ACR)

Her build sonrası Docker image'larının güvenli şekilde saklanması ve VM'den çekilebilmesi için Azure Container Registry kullanılmıştır. Pipeline build aşamasında image ACR'ye push edilir, deploy aşamasında VM bu image'ı ACR'den çeker ve container olarak ayağa kaldırır.

- **Registry adı:** `acroninecourse`
- **Image'lar:**
  - `acroninecourse.azurecr.io/backend-app:latest`
  - `acroninecourse.azurecr.io/frontend-app:latest`

### 2. Azure PostgreSQL

Üretim ortamı için yönetilen bir PostgreSQL veritabanı servisi kullanılmıştır. Veritabanı bilgileri kod içinde yer almaz; pipeline üzerinden container'a environment variable olarak iletilir.

- **Host:** `db-online-course.postgres.database.azure.com`
- **Veritabanı:** `production_db`

### 3. Azure Virtual Machine

Backend ve frontend container'larının çalıştığı sunucudur. Güvenlik için yalnızca 22 (SSH), 80 (HTTP) ve 443 (HTTPS) portları dışarıya açılmıştır. Uygulama portları (3000, 8000) yalnızca VM içinden erişilebilir durumdadır. 22 portu da SSH bağlantısı için açık durumdadır.

- **İşletim sistemi:** Ubuntu 24.04 LTS

### 4. Azure Blob Storage

Kurs görselleri ve medya dosyaları Azure Blob Storage üzerinde tutulmaktadır. Frontend bu URL'yi build sırasında environment variable olarak alır ve görselleri doğrudan Blob Storage'dan çeker.

- **URL:** `https://bsonlinecourse.blob.core.windows.net/media/`

---

## 🔄 CI/CD Pipeline

### Azure DevOps Variable Group

Veritabanı şifresi, Django secret key ve ACR kimlik bilgileri gibi hassas veriler kod içinde tutulmaz. Bu değerler Azure DevOps Library'de `online-course-project-varible` adlı Variable Group'ta saklanır. Gizli değerler şifrelenerek tutulur ve yalnızca pipeline çalışırken ilgili adımlara iletilir. Bu sayede hassas bilgiler ne repoda ne de pipeline loglarında görünür.


### Backend Pipeline

`main` branch'e her push geldiğinde otomatik tetiklenir. Pipeline'ın ihtiyaç duyduğu tüm değerler Azure DevOps Library'deki Variable Group'tan otomatik olarak alınır. Pipeline üç aşamadan oluşur:

1. **Build:** Django uygulaması Docker image'ına dönüştürülür.Gerekli değerler library'den alınarak Docker BuildKit secret mekanizmasıyla build ortamına güvenli şekilde iletilir; image'a gömülmez, logda görünmez.
2. **Push:** Oluşan image ACR'ye gönderilir.
3. **Deploy:** SSH bağlantısıyla VM'e bağlanılır. Eski container durdurulur, yeni image ACR'den çekilir. Veritabanı bilgileri ve diğer environment variable'lar Library'den alınarak container'a iletilir ve uygulama ayağa kaldırılır.

### Frontend Pipeline

Backend pipeline'ına benzer şekilde çalışır. Tüm değerler yine Library'den alınır. Temel fark, `NEXT_PUBLIC_API_URL` ve `NEXT_PUBLIC_ASSET_URL` değerlerinin build sırasında `--build-arg` olarak Next.js uygulamasının içine gömülmesidir. Next.js bu değerleri runtime'da değil build anında kodun içine işlediği için bu yöntem kullanılmıştır.

---

## 🌐 Nginx Yapılandırması

VM üzerinde Nginx reverse proxy olarak çalışır. Dışarıya yalnızca 80 ve 443 portları açıktır. Gelen her istek domain adına göre Nginx tarafından ilgili container'a yönlendirilir:

- `onlinecourses.site` → Frontend container (localhost:3000)
- `api.onlinecourses.site` → Backend container (localhost:8000)

HTTP üzerinden gelen tüm istekler otomatik olarak HTTPS'e yönlendirilir.

---

## 🔒 SSL Sertifikası

`onlinecourses.site` ve `api.onlinecourses.site` domainleri için Let's Encrypt üzerinden ücretsiz SSL sertifikası alınmıştır. Certbot, Nginx yapılandırmasını otomatik olarak günceller ve sertifikaları 90 günde bir otomatik yeniler.

---
## 🐳 Sanal Makineye Docker Kurulumu
 
Azure VM oluşturulduktan sonra SSH ile bağlanılarak Docker kurulumu yapılmıştır. Ubuntu üzerinde Docker'ın resmi deposu eklenerek kurulum gerçekleştirilir. Pipeline'ın SSH üzerinden VM'e bağlanıp Docker komutları çalıştırabilmesi bu kurulum sayesinde mümkün olmaktadır.

---
## 🌍 Domain ve DNS Yapılandırması

Domain `guzelhosting` üzerinden satın alınmıştır.  hem ana domain hem de API subdomaini aynı VM I'sine yönlendirilmiştir.


---

## 📋 Azure Boards — Görev Yönetimi

Proje görevleri Azure Boards üzerinde takip edilmiştir. Görevler Epic,Feature, User Story ve Task hiyerarşisinde organize edilmiştir. PM olarak görev atamaları, sprint planlaması ve iş takibi bu süreç boyunca yönetilmiştir.

---

## 🔁 Deployment Akışı

```
Geliştirici → Pull Request açılır
                   │
                   ▼
            Code Review & Merge → main
                   │
                   ▼
            Azure Pipeline tetiklenir
                   │
          ┌────────┴────────┐
          ▼                 ▼
    Docker Build       Docker Build
    (Backend)          (Frontend)
          │                 │
          ▼                 ▼
    ACR'ye Push        ACR'ye Push
          │                 │
          └────────┬────────┘
                   ▼
            SSH ile VM'e bağlan
                   │
                   ▼
            Eski container durdur
                   │
                   ▼
            Yeni image çek (docker pull)
                   │
                   ▼
            Yeni container başlat
                   │
                   ▼
            Uygulama canlıya alındı ✓
```

---

Bu projede **Project Manager** olarak aşağıdaki sorumluluklar üstlenilmiştir:

- Azure DevOps ortamının kurulumu ve yapılandırması
- Variable Group ve pipeline secret yönetimi
- Backend ve frontend CI/CD pipeline yazımı
- Azure Container Registry bağlantısı kurulumu
- Azure PostgreSQL veritabanı yapılandırması
- Azure Virtual Machine kurulumu ve güvenlik ayarları
- Nginx reverse proxy yapılandırması
- SSL sertifikası kurulumu
- Domain alımı ve DNS yapılandırması
- Azure Boards üzerinde görev ataması 
