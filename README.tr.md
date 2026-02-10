# aws-wordpress-scalable-architecture

**AWS üzerinde ölçeklenebilir, production‑grade WordPress için yalnızca tasarım odaklı referans mimari.**  
Edge: CloudFront + WAF + ALB · Uygulama: Auto Scaling EC2 (PHP 8.3) · Veri: RDS (MariaDB) · Paylaşılan medya: EFS · Cache: ElastiCache (Memcached).  
**Yüksek trafik (10k+ eşzamanlı)**, **düşük gecikme**, **SEO** ve **günlük yedekler** için optimize edilmiştir.

> **Diyagram:** `docs/aws_proje.drawio.pdf`
---

## İçindekiler
- [1. Amaç & Kapsam](#1-amaç--kapsam)
- [2. Hedefler & Varsayımlar](#2-hedefler--varsayımlar)
- [3. Mimari Genel Bakış](#3-mimari-genel-bakış)
- [4. Bileşen Dağılımı](#4-bileşen-dağılımı)
- [5. VPC & Ağ Topolojisi](#5-vpc--ağ-topolojisi)
- [6. Güvenlik & Uyumluluk](#6-güvenlik--uyumluluk)
- [7. Performans & SEO](#7-performans--seo)
- [8. Yedekleme & Felaket Kurtarma](#8-yedekleme--felaket-kurtarma)
- [9. Gözlemlenebilirlik](#9-gözlemlenebilirlik)
- [10. Maliyet Değerlendirmeleri](#10-maliyet-değerlendirmeleri)
- [11. Riskler & Trade‑off’lar](#11-riskler--trade-offlar)
- [12. Gelecek Geliştirmeler](#12-gelecek-geliştirmeler)
- [13. Depo Yapısı](#13-depo-yapısı)
- [14. Lisans](#14-lisans)

---

## 1. Amaç & Kapsam
Bu depo, AWS üzerinde ölçeklenebilir bir WordPress platformunun **sistem tasarımını** dokümante eder.  
Kararları, gerekçeleri ve referans topolojiyi kapsar. **Kurulum/deploy kodu içermez** (Terraform, CloudFormation, script vb. yoktur).

**Kapsam dışı:** adım adım kurulum, CI/CD, OS hardening script’leri, cache rehberliği dışındaki tema/eklenti detayları.

---

## 2. Hedefler & Varsayımlar
**Hedefler**
- **10k+ eşzamanlı istek** için stabil gecikme.
- **SEO performansını** maksimize etmek (TTFB, cache, görsel optimizasyon).
- Edge ve uygulama katmanlarında **güvenlik** (WAF, TLS, least privilege).
- **Günlük yedekler** ve net bir **DR** yolu.
- **Maliyetleri** gözeten pragmatik tasarım (cache/CDN ile origin yükünü azaltma).

**Varsayımlar**
- WordPress (en güncel sürüm) ve PHP **8.3**.
- Veritabanı: **Amazon RDS** üzerinde **MariaDB**.
- Paylaşılan medya ≈ **30 GB**; çok‑AZ uygulama filosu için **EFS**.
- Nesne/oturum cache’i için **ElastiCache (Memcached)**.
- Tek bir AWS Region içinde **Multi‑AZ** kurulum.

---

## 3. Mimari Genel Bakış
- **Edge:** Route 53 → **CloudFront (CDN)** → **AWS WAF / Shield Standard**  
- **Giriş:** **Application Load Balancer (ALB, HTTPS)** + HTTP→HTTPS yönlendirme  
- **Uygulama:** **EC2 Auto Scaling** (PHP 8.3, Nginx/Apache, stateless)  
- **Paylaşılan Medya:** **Amazon EFS** (`wp-content/uploads`)  
- **Veri:** **Amazon RDS (MariaDB)** (+ opsiyonel **Read Replica**)  
- **Cache:** **Amazon ElastiCache (Memcached)**  
- **Objeler & Loglar:** **Amazon S3** (Versioning + Lifecycle)  
- **Operasyon:** CloudWatch / CloudTrail / GuardDuty, **AWS Backup**; erişim **Bastion** veya **SSM Session Manager**

---

## 4. Bileşen Dağılımı
**Route 53** – CloudFront’a ALIAS; opsiyonel health check’ler.  
**CloudFront** – Global edge cache; TLS termination; origin’i korur; minimum header/cookie iletimi.  
**AWS WAF** – Managed kurallar (SQLi/XSS), rate limit, allow/deny list’ler.  
**ALB** – L7 routing, ACM’den TLS, HTTP→HTTPS, erişim logları S3’e.  
**EC2 ASG** – Stateless uygulama katmanı; CPU/istek bazlı ölçekleme; immutable image (AMI bake) önerilir.  
**EFS** – Paylaşılan medya; AZ başına mount target; at‑rest encryption; IA lifecycle.  
**RDS (MariaDB)** – Multi‑AZ HA veya Single‑AZ + Read Replica; encryption; otomatik yedekler.  
**ElastiCache (Memcached)** – Nesne/fragment cache; DB ve uygulama yükünü azaltır.  
**S3** – Obje depolama, log arşivi, versioned yedekler; Glacier lifecycle.  
**SSM** – Parametre/secret yönetimi; SSH yerine session‑based erişim.

---

## 5. VPC & Ağ Topolojisi
- **VPC CIDR:** `10.0.0.0/16` (örnek)
- **AZ başına subnetler:** Public (`10.0.0.0/24`, `10.0.1.0/24`), App (`10.0.10.0/24`, `10.0.11.0/24`), Data (`10.0.20.0/24`, `10.0.21.0/24`)
- **Routing:** Public → IGW; App/Data → NAT GW (yalnızca çıkış)
- **Erişim:** DB/Cache yalnızca App SG’den; EFS yalnızca App SG; public DB/EFS yok.

---

## 6. Güvenlik & Uyumluluk
- **Edge güvenliği:** WAF managed + custom kurallar; rate limit; Shield Standard.  
- **Kimlik:** **Least privilege** IAM roller; instance’larda uzun ömürlü anahtar yok.  
- **Şifreleme:** In‑transit TLS; at‑rest KMS (EBS/EFS/S3/RDS).  
- **Gizli bilgiler:** SSM Parameter Store / Secrets Manager (DB creds, WP salts).  
- **Denetim:** CloudTrail; opsiyonel VPC Flow Logs; ALB/CloudFront logları S3 + lifecycle.  
- **Erişim:** **SSM Session Manager** tercih edilir; public SSH ve yönetim portlarından kaçınılır.

---

## 7. Performans & SEO
- **CDN** ile TTFB düşürme; statik varlıklar için uzun TTL.  
- **Autoscaling** (istek/CPU); hızlı scale‑out için AMI bake.  
- **Memcached** ile object/fragment cache; cookie/header/query forwarding’i minimize etme.  
- **HTTP/2**, gzip/brotli, doğru `Cache-Control`, görsel optimizasyon (WebP/AVIF, lazy‑load).  
- Gelecekte **S3 media offload** ile EFS ve egress maliyetlerini düşürme.

---

## 8. Yedekleme & Felaket Kurtarma
- **Günlük** RDS snapshot’ları (7–30 gün saklama); automated backups açık.  
- **EFS** günlük yedekler (AWS Backup); S3 versioning + Glacier lifecycle.  
- **Failover:** Primary arızasında **RDS Read Replica** promote edilir; RTO/RPO belgelenir.  
- **Opsiyonel:** Cross‑region replikasyon (S3 CRR, RDS RR).

---

## 9. Gözlemlenebilirlik
- **Metrikler/Alarmlar:** ALB 5xx/latency/healthy targets; EC2 CPU/memory; RDS CPU/free storage; ElastiCache eviction.  
- **Loglar:** ALB & CloudFront access logları S3; analiz için CloudWatch Logs Insights/Athena.  
- **Dashboard’lar:** Edge, app ve data katmanları için CloudWatch dashboard’ları.

---

## 10. Maliyet Değerlendirmeleri
Başlıca maliyetler: **NAT Gateway**, **ALB**, **EC2/EBS**, **EFS**, **RDS**, **ElastiCache**, **CloudFront**, **S3 transfer**, **WAF**.  
Tasarruf için: CDN/cache hit oranını artırma, doğru instance boyutları, düşük ASG min capacity, EFS IA, minimal header/cookie forwarding, güvenliyse NAT konsolidasyonu.

---

## 11. Riskler & Trade‑off’lar
- **EFS gecikmesi vs. sadelik:** Paylaşılan medya kolaylığı; gecikme ekleyebilir → ileride S3 offload.  
- **Memcached vs. Redis:** Memcached basit/hızlı; Redis daha fazla özellik (persistency vb.) ama maliyet/karmaşıklık artar.  
- **Tek Region:** Operasyonel sadelik; cross‑region DR daha dayanıklı ama pahalı.  
- **ASG scale‑out süresi:** AMI bake ve warm pool değerlendirin.

---

## 12. Gelecek Geliştirmeler
- **S3 media offload** + CloudFront signed URL’ler.  
- **Blue/Green / Canary** deploy (ALB/CodeDeploy).  
- **IaC** (Terraform/CloudFormation) ve **CI/CD**.  
- **WAF custom rule**’ların trafik desenlerine göre ayarlanması.  
- **Containerization** (ECS/EKS).

---

## 13. Depo Yapısı
```
.
├── docs/
│   └── aws_proje.drawio.pdf   # mimari diyagram
└── README.md
```

---

## 14. Lisans
MIT
