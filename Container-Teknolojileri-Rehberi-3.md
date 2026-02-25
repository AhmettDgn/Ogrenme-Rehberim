# 🐳 Container Teknolojileri - Profesyonel Mühendis Rehberi

> **Hazırlayan:** 10 yıllık deneyimli Container & DevOps Mühendisi
> **Hedef:** Gerçek dünya senaryolarıyla, mühendislik kafasıyla Container teknolojilerini öğrenmek

---

## 📋 İçindekiler
1. [Container Teknolojisi Nedir? Neden Önemli?](#container-nedir)
2. [Container vs Virtual Machine - Mühendislik Perspektifi](#container-vs-vm)
3. [Docker Mimarisi - Derinlemesine](#docker-mimarisi)
4. [Docker Images ve Layers - Optimization Teknikleri](#docker-images)
5. [Dockerfile Yazımı - Production Ready](#dockerfile-yazimi)
6. [Docker Volumes - Data Persistence Stratejileri](#docker-volumes)
7. [Docker Networking - Gerçek Dünya Senaryoları](#docker-networking)
8. [Docker Compose - Mikroservis Orchestration](#docker-compose)
9. [Multi-Stage Builds - Optimization Masters](#multi-stage-builds)
10. [Container Güvenliği - Security Best Practices](#container-guvenligi)
11. [Production Deployment Stratejileri](#production-deployment)
12. [Troubleshooting ve Debugging](#troubleshooting)
13. [Pratik Projeler](#pratik-projeler)

---

## 🎯 Container Teknolojisi Nedir? Neden Önemli? {#container-nedir}

### Gerçek Dünyada Yaşanan Problem

10 yıl önce şöyle bir senaryo vardı:
```
Developer: "Benim makinemde çalışıyor!"
Operations: "Production'da çalışmıyor!"
```

**Neden çalışmıyordu?**
- Farklı OS versiyonları
- Farklı kütüphane versiyonları
- Farklı environment variables
- Dependency hell (Python 2 vs 3, Node 12 vs 14 vs 16)

### Container'ın Getirdiği Çözüm

Container'lar şu prensibi getirdi:
> "Uygulamanı ve tüm bağımlılıklarını bir arada taşı, her yerde aynı şekilde çalıştır"

**Mühendis Kafasıyla Düşünelim:**
- **Immutability**: Bir kez oluşturduğun image, her yerde aynı şekilde çalışır
- **Reproducibility**: Production'daki sorunu local'de birebir reproduce edebilirsin
- **Consistency**: Dev, Test, Staging, Production hepsi aynı environment

---

## ⚔️ Container vs Virtual Machine - Mühendislik Perspektifi {#container-vs-vm}

### Teknik Farklar

```
┌─────────────────────────────────┐  ┌─────────────────────────────────┐
│     VIRTUAL MACHINE             │  │         CONTAINER               │
├─────────────────────────────────┤  ├─────────────────────────────────┤
│  App A  │  App B  │  App C      │  │  App A  │  App B  │  App C      │
│  Bins   │  Bins   │  Bins       │  │  Bins   │  Bins   │  Bins       │
│  Libs   │  Libs   │  Libs       │  │  Libs   │  Libs   │  Libs       │
├─────────┼─────────┼─────────────┤  ├─────────────────────────────────┤
│ Guest OS│ Guest OS│ Guest OS    │  │     Container Runtime           │
├─────────────────────────────────┤  │      (Docker Engine)            │
│       Hypervisor (Type 2)       │  ├─────────────────────────────────┤
├─────────────────────────────────┤  │        Host OS (Linux)          │
│        Host OS                  │  ├─────────────────────────────────┤
└─────────────────────────────────┘  └─────────────────────────────────┘
```

### Ne Zaman Hangisini Kullanmalıyız?

#### Virtual Machine Kullan:
✅ **Farklı OS'ler gerektiğinde** (Windows app + Linux app aynı fiziksel sunucuda)
✅ **Güçlü izolasyon gerektiğinde** (Multi-tenant sistemler, farklı müşteriler)
✅ **Eski legacy uygulamalar** (Modernize edilemeyenler)
✅ **Tam kernel kontrolü gerektiğinde** (Custom kernel modules)

**Gerçek Dünya Örneği:**
```
AWS'de farklı müşterilerin workload'ları EC2 instance'larında
(VM'lerde) çalışır. Güvenlik izolasyonu kritik.
```

#### Container Kullan:
✅ **Mikroservis mimarileri** (Her servis bir container)
✅ **Hızlı deployment** (Saniyeler içinde başlatma)
✅ **Yoğun resource kullanımı** (1 sunucuda yüzlerce container)
✅ **CI/CD pipeline'ları** (Her commit için test container'ı)
✅ **Development environment standardizasyonu**

**Gerçek Dünya Örneği:**
```
Netflix, Spotify gibi şirketler binlerce mikroservisi
container'lar ile deploy ediyor. Kubernetes orchestration ile.
```

### Performance Karşılaştırması (Gerçek Metrikler)

| Metrik | Virtual Machine | Container |
|--------|----------------|-----------|
| **Başlatma Süresi** | 30-60 saniye | 1-5 saniye |
| **Disk Kullanımı** | 10-100 GB | 100 MB - 2 GB |
| **Memory Overhead** | 2-8 GB (Guest OS) | 10-100 MB |
| **Density** | 10-20 VM/sunucu | 100-1000 container/sunucu |
| **Izolasyon Seviyesi** | Çok Güçlü | Güçlü (ama VM kadar değil) |

---

## 🏗️ Docker Mimarisi - Derinlemesine {#docker-mimarisi}

### Core Components

```
┌──────────────────────────────────────────────────────────┐
│                    DOCKER ARCHITECTURE                   │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌─────────────┐         ┌──────────────────────┐        │
│  │             │  REST   │                      │        │
│  │   Docker    │  API    │   Docker Daemon      │        │
│  │   Client    ├─────── ▶│   (dockerd)          │        │
│  │   (CLI)     │         │                      │        │
│  │             │         │  ┌────────────────┐  │        │
│  └─────────────┘         │  │  containerd    │  │        │
│                          │  │  (runtime)     │  │        │
│                          │  └────────┬───────┘  │        │
│                          │           │          │        │
│                          │  ┌────────▼───────┐  │        │
│                          │  │    runc        │  │        │
│                          │  │  (OCI runtime) │  │        │
│                          │  └────────────────┘  │        │
│                          └──────────────────────┘        │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │            Docker Registry (Docker Hub)          │    │
│  │              - Public Images                     │    │
│  │              - Private Repositories              │    │
│  └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

### Mühendislik Detayları

#### 1. **Docker Client (CLI)**
- Kullanıcı komutlarını alır (`docker run`, `docker build`, vb.)
- REST API ile daemon'a iletir
- Local veya remote daemon ile konuşabilir

**Production Tip:**
```bash
# Remote Docker daemon'a bağlanma
export DOCKER_HOST="tcp://production-server:2376"
docker ps  # Production sunucusundaki container'ları görürsün
```

#### 2. **Docker Daemon (dockerd)**
- Container lifecycle management
- Image management
- Network ve volume yönetimi
- containerd'yi orkestre eder

**Kritik Bilgi:**
Daemon root olarak çalışır! Bu yüzden Docker security önemli.

#### 3. **containerd**
- Industry-standard container runtime
- Docker'dan bağımsız (Kubernetes de kullanır)
- Image push/pull, storage, network primitive'leri

#### 4. **runc**
- OCI (Open Container Initiative) spec implementation
- Actual container process'i başlatır
- Linux namespaces ve cgroups kullanır

### Linux Namespaces ve Cgroups (Altında Ne Oluyor?)

**Container izolasyonu bu iki Linux özelliği ile sağlanır:**

#### Namespaces (İzolasyon)
```
PID Namespace    → Her container kendi process tree'sini görür
NET Namespace    → Her container kendi network interface'ini görür
MNT Namespace    → Her container kendi filesystem mount'larını görür
UTS Namespace    → Her container kendi hostname'ini görür
IPC Namespace    → Her container kendi inter-process communication
USER Namespace   → User ID mapping (container içinde root ≠ host root)
```

**Gerçek Örnek:**
```bash
# Container içinde
root@container$ ps aux
USER       PID  COMMAND
root         1  /bin/bash
root        15  ps aux

# Host'ta
root@host$ ps aux | grep bash
root     12345  containerd-shim ... /bin/bash
```

#### Cgroups (Resource Limiting)
```
CPU Limit        → Container max %50 CPU kullanabilir
Memory Limit     → Container max 512MB RAM kullanabilir
I/O Limit        → Container max 100 IOPS yapabilir
Network Limit    → Bandwidth throttling
```

**Production Örneği:**
```bash
docker run -d \
  --cpus="1.5" \              # Max 1.5 CPU core
  --memory="2g" \             # Max 2GB RAM
  --memory-swap="2g" \        # Swap disable
  --oom-kill-disable=false \  # OOM durumunda kill et
  nginx
```

---

## 🖼️ Docker Images ve Layers - Optimization Teknikleri {#docker-images}

### Image Layer Sistemi

Docker image'lar **Union File System** kullanır. Her komut bir layer oluşturur.

```dockerfile
FROM ubuntu:22.04          # Layer 1: Base OS (77 MB)
RUN apt-get update         # Layer 2: Package index (22 MB)
RUN apt-get install -y \   # Layer 3: Packages (150 MB)
    python3 python3-pip
COPY app.py /app/          # Layer 4: Application code (1 MB)
CMD ["python3", "/app/app.py"]  # Layer 5: Metadata (0 MB)
```

**Önemli:** Her layer **immutable** ve **cached**!

### Layer Caching - Optimization Stratejisi

**❌ Kötü Örnek (Her değişiklikte tüm dependencies yeniden install):**
```dockerfile
FROM node:18
COPY . /app
WORKDIR /app
RUN npm install        # Her kod değişikliğinde çalışır!
CMD ["npm", "start"]
```

**✅ İyi Örnek (Cache-friendly):**
```dockerfile
FROM node:18
WORKDIR /app

# Önce dependencies (nadiren değişir)
COPY package.json package-lock.json ./
RUN npm install        # Cache'lenir!

# Sonra kod (sık değişir)
COPY . .
CMD ["npm", "start"]
```

**Sonuç:**
- İlk build: 5 dakika
- Sonraki build'ler (kod değişikliğinde): 10 saniye

### Image Size Optimization - Gerçek Dünya Teknikleri

#### Teknik 1: Doğru Base Image Seçimi

```dockerfile
# ❌ Kötü: 1.2 GB
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y python3

# ✅ Daha İyi: 900 MB
FROM python:3.11

# ✅✅ En İyi: 50 MB
FROM python:3.11-slim

# ✅✅✅ Minimal: 15 MB
FROM python:3.11-alpine
```

**Ama dikkat!** Alpine musl libc kullanır (glibc değil). Bazı binary'ler çalışmayabilir.

**Production Kararı:**
- **Alpine kullan**: Stateless apps, simple dependencies
- **Slim kullan**: Complex dependencies, glibc gereken apps
- **Full kullan**: Development environment, debugging tools gerekli

#### Teknik 2: Multi-Stage Builds (Detaylı anlatılacak)

#### Teknik 3: Layer Minimization

```dockerfile
# ❌ Kötü: 3 layer
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get clean

# ✅ İyi: 1 layer
RUN apt-get update && \
    apt-get install -y curl && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

#### Teknik 4: .dockerignore Kullanımı

`.dockerignore` dosyası:
```
node_modules
.git
.env
*.log
tests/
docs/
README.md
.vscode
```

**Neden önemli?**
- Build context küçülür (hız artar)
- Hassas bilgiler image'a girmez
- Gereksiz dosyalar layer'a eklenmez

### Image Naming ve Tagging Stratejisi

**Production Best Practice:**
```bash
# ❌ Kötü
docker build -t myapp .

# ✅ İyi (Semantic Versioning)
docker build -t mycompany/myapp:1.2.3 .
docker build -t mycompany/myapp:1.2 .
docker build -t mycompany/myapp:1 .
docker build -t mycompany/myapp:latest .

# ✅✅ Production Ready (Git SHA + Timestamp)
docker build -t mycompany/myapp:v1.2.3-$(git rev-parse --short HEAD)-$(date +%Y%m%d) .
# Sonuç: mycompany/myapp:v1.2.3-a3f2b1c-20260214
```

**Neden önemli?**
- Rollback yapabilirsin (hangi version çalışıyordu?)
- Debugging kolaylaşır (production'da hangi commit var?)
- Immutability sağlanır (`:latest` her zaman değişir, `:v1.2.3` hiç değişmez)

---

## 📝 Dockerfile Yazımı - Production Ready {#dockerfile-yazimi}

### Anatomy of a Professional Dockerfile

```dockerfile
# ==========================================
# Multi-Stage Build: Node.js Production App
# ==========================================

# ──────────────────────────────────────────
# Stage 1: Dependencies
# ──────────────────────────────────────────
FROM node:18-alpine AS dependencies

# Install security updates
RUN apk update && apk upgrade && \
    apk add --no-cache dumb-init

WORKDIR /app

# Copy package files
COPY package.json package-lock.json ./

# Install production dependencies only
RUN npm ci --only=production && \
    npm cache clean --force

# ──────────────────────────────────────────
# Stage 2: Build
# ──────────────────────────────────────────
FROM node:18-alpine AS build

WORKDIR /app

# Copy package files
COPY package.json package-lock.json ./

# Install all dependencies (including dev)
RUN npm ci

# Copy source code
COPY . .

# Build application
RUN npm run build

# ──────────────────────────────────────────
# Stage 3: Production
# ──────────────────────────────────────────
FROM node:18-alpine AS production

# Add metadata (OCI labels)
LABEL maintainer="devops@company.com"
LABEL version="1.0.0"
LABEL description="Production Node.js application"

# Install dumb-init (PID 1 problem için)
RUN apk add --no-cache dumb-init

# Create non-root user (Security!)
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# Copy dependencies from dependencies stage
COPY --from=dependencies --chown=nodejs:nodejs /app/node_modules ./node_modules

# Copy built application from build stage
COPY --from=build --chown=nodejs:nodejs /app/dist ./dist
COPY --chown=nodejs:nodejs package.json ./

# Switch to non-root user
USER nodejs

# Expose port (documentation purpose)
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node healthcheck.js

# Use dumb-init to handle signals properly
ENTRYPOINT ["dumb-init", "--"]

# Start application
CMD ["node", "dist/index.js"]
```

### Dockerfile Best Practices - Detaylı Açıklamalar

#### 1. **Base Image Seçimi**

**Kriterlere göre karar:**

| Kriter | Alpine | Slim | Full |
|--------|--------|------|------|
| Image Size | ✅ 15-50MB | ⚠️ 100-200MB | ❌ 500MB-1GB |
| Security Surface | ✅ Minimal | ⚠️ Medium | ❌ Large |
| Compatibility | ⚠️ musl libc | ✅ glibc | ✅ Full |
| Debug Tools | ❌ Minimal | ⚠️ Some | ✅ All |
| Build Speed | ✅ Fast | ✅ Fast | ⚠️ Slow |

**Production Tavsiyem:**
```dockerfile
# Development
FROM node:18-bullseye  # Full image, debug tools

# Production
FROM node:18-alpine    # Minimal, fast
```

#### 2. **USER Directive (Kritik Güvenlik)**

**Neden root kullanmamalıyız?**

```dockerfile
# ❌ TEHLIKE: Container root olarak çalışır
FROM ubuntu
CMD ["bash"]

# Bu container'dan host'a escape edilirse, attacker root access kazanır!
```

**✅ Güvenli Yöntem:**
```dockerfile
FROM ubuntu

# Kullanıcı oluştur
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Gerekli dosyalara ownership ver
COPY --chown=appuser:appuser app /app

# Non-root user'a geç
USER appuser

CMD ["./app"]
```

**Gerçek Dünya Örneği:**
2019'da Kubernetes'te büyük bir security vulnerability bulundu. Root container'lar escape edip host'a sızabiliyordu. Non-root container'lar etkilenmedi.

#### 3. **ENTRYPOINT vs CMD**

**ENTRYPOINT:** Container'ın executable'ı (değiştirilemez)
**CMD:** Default arguments (override edilebilir)

```dockerfile
# Örnek 1: Sadece CMD
CMD ["python", "app.py"]
# Çalıştırma: docker run myapp
# Sonuç: python app.py
# Override: docker run myapp bash  →  bash çalışır

# Örnek 2: ENTRYPOINT + CMD
ENTRYPOINT ["python"]
CMD ["app.py"]
# Çalıştırma: docker run myapp
# Sonuç: python app.py
# Override: docker run myapp test.py  →  python test.py çalışır
```

**Production Kullanımı:**
```dockerfile
# Web server (always nginx, but config can change)
ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]

# docker run myapp -t  →  nginx -t (config test)
# docker run myapp      →  nginx -g daemon off; (normal run)
```

#### 4. **HEALTHCHECK - Production Must-Have**

```dockerfile
# HTTP endpoint health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

# Database health check
HEALTHCHECK --interval=10s --timeout=3s \
  CMD pg_isready -U postgres || exit 1

# Custom script
HEALTHCHECK CMD /app/healthcheck.sh || exit 1
```

**Neden önemli?**
- Kubernetes/Docker Swarm unhealthy container'ları restart eder
- Load balancer unhealthy container'lara traffic göndermez
- Monitoring sistemleri alert oluşturur

#### 5. **ARG vs ENV**

```dockerfile
# ARG: Build-time variable (image'da kalmaz)
ARG NODE_VERSION=18
FROM node:${NODE_VERSION}

ARG BUILD_DATE
LABEL build-date=${BUILD_DATE}

# ENV: Runtime variable (image'da kalır)
ENV NODE_ENV=production
ENV PORT=3000

# Kullanım
RUN echo "Building at ${BUILD_DATE}"  # ARG
CMD node -p "process.env.NODE_ENV"     # ENV
```

**Build:**
```bash
docker build \
  --build-arg BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
  --build-arg NODE_VERSION=20 \
  -t myapp .
```

**Güvenlik Notu:**
```dockerfile
# ❌ Kötü: Secret ARG'da kalabilir
ARG DATABASE_PASSWORD=secret123
RUN echo "Password: ${DATABASE_PASSWORD}"

# ✅ İyi: Secret runtime'da inject et
# docker run -e DATABASE_PASSWORD=secret123 myapp
```

### Advanced Dockerfile Patterns

#### Pattern 1: Distroless Images (Google)

```dockerfile
# Build stage
FROM golang:1.21 AS build
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o app

# Production stage (No shell, no package manager!)
FROM gcr.io/distroless/static-debian11
COPY --from=build /app/app /app
ENTRYPOINT ["/app"]
```

**Avantajları:**
- Image size: ~2MB
- No shell → Remote code execution imkansız
- No package manager → Vulnerability surface minimal
- Google tarafından maintain edilir

**Dezavantajları:**
- Debug edilemez (shell yok)
- Dynamic linking desteklenmez

**Ne zaman kullan:** High-security production workloads (fintech, healthcare)

#### Pattern 2: Development vs Production Dockerfile

```dockerfile
# ──────────────────────────────────────────
# Development
# ──────────────────────────────────────────
FROM node:18 AS development
WORKDIR /app
COPY package*.json ./
RUN npm install  # Dev dependencies dahil
COPY . .
EXPOSE 3000 9229  # App + Debug port
CMD ["npm", "run", "dev"]

# ──────────────────────────────────────────
# Production
# ──────────────────────────────────────────
FROM node:18-alpine AS production
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
USER node
EXPOSE 3000
CMD ["node", "index.js"]
```

**Kullanım:**
```bash
# Development
docker build --target development -t myapp:dev .
docker run -v $(pwd):/app -p 3000:3000 -p 9229:9229 myapp:dev

# Production
docker build --target production -t myapp:prod .
docker run -p 3000:3000 myapp:prod
```

---

## 💾 Docker Volumes - Data Persistence Stratejileri {#docker-volumes}

### Problem: Container'lar Ephemeral (Geçici)

```bash
docker run -d --name db postgres
# Postgres veritabanına data yaz
docker stop db
docker rm db
# ❌ Tüm data kayboldu!
```

### Çözüm: Volumes

Docker'da 3 tip data persistence var:

```
┌────────────────────────────────────────────────────┐
│              DATA PERSISTENCE TYPES                │
├────────────────────────────────────────────────────┤
│  1. Volumes         → Docker manages               │
│  2. Bind Mounts     → Host path mounts             │
│  3. tmpfs Mounts    → Memory (not persistent)      │
└────────────────────────────────────────────────────┘
```

### 1. Volumes (Production Recommended)

**Avantajları:**
- Docker tarafından manage edilir
- Backup ve migration kolay
- Volume driver desteği (NFS, Cloud storage)
- Windows ve Linux'ta çalışır
- Multiple container'lar share edebilir

```bash
# Volume oluştur
docker volume create mydata

# Volume'u kullan
docker run -d \
  --name db \
  -v mydata:/var/lib/postgresql/data \
  postgres

# Volume'u incele
docker volume inspect mydata
```

**Output:**
```json
{
    "CreatedAt": "2026-02-14T10:30:00Z",
    "Driver": "local",
    "Mountpoint": "/var/lib/docker/volumes/mydata/_data",
    "Name": "mydata",
    "Scope": "local"
}
```

**Production Örneği: Database Cluster**
```bash
# Master database
docker run -d \
  --name postgres-master \
  -v pgmaster:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:15

# Replica database (farklı volume)
docker run -d \
  --name postgres-replica \
  -v pgreplica:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:15
```

### 2. Bind Mounts (Development Recommended)

**Avantajları:**
- Host'taki dosyayı direkt mount eder
- Development'ta hot-reload için ideal
- Config dosyalarını inject etmek için

**Dezavantajları:**
- Host path'e bağımlı (portability düşük)
- Security risk (host filesystem'e erişim)

```bash
# Bind mount
docker run -d \
  --name web \
  -v /host/path/nginx.conf:/etc/nginx/nginx.conf:ro \  # :ro = read-only
  -v /host/path/html:/usr/share/nginx/html \
  nginx
```

**Development Hot-Reload Örneği:**
```bash
# Node.js development
docker run -d \
  --name myapp \
  -v $(pwd):/app \          # Source code
  -v /app/node_modules \    # Anonymous volume (override)
  -p 3000:3000 \
  node:18 \
  npm run dev
```

**Dikkat:** `-v /app/node_modules` olmadan host'taki node_modules container'ı override eder!

### 3. tmpfs Mounts (Temporary Data)

**Kullanım Alanları:**
- Sensitive data (memory'de, disk'e yazılmaz)
- Temporary processing
- Cache

```bash
docker run -d \
  --name app \
  --tmpfs /tmp:rw,size=100m,mode=1777 \
  myapp
```

### Volume Drivers - Advanced

**Production Senaryoları:**

#### 1. NFS Volume (Shared Storage)
```bash
# NFS volume oluştur
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/path/to/share \
  nfs-volume

# Multi-node cluster'da aynı data
docker run -v nfs-volume:/data node1
docker run -v nfs-volume:/data node2
docker run -v nfs-volume:/data node3
```

#### 2. Cloud Storage (AWS EFS, Azure Files)

**AWS EFS Örneği:**
```bash
docker volume create \
  --driver local \
  --opt type=nfs4 \
  --opt o=addr=fs-12345.efs.us-east-1.amazonaws.com,rw,nfsvers=4.1 \
  --opt device=:/ \
  efs-volume
```

### Volume Backup ve Restore Stratejileri

**Backup:**
```bash
# Volume'u tar archive'e al
docker run --rm \
  -v mydata:/data \
  -v $(pwd):/backup \
  ubuntu \
  tar czf /backup/mydata-backup-$(date +%Y%m%d).tar.gz -C /data .
```

**Restore:**
```bash
# Yeni volume oluştur
docker volume create mydata-restored

# Backup'tan restore et
docker run --rm \
  -v mydata-restored:/data \
  -v $(pwd):/backup \
  ubuntu \
  tar xzf /backup/mydata-backup-20260214.tar.gz -C /data
```

**Production Automation (Cron Job):**
```bash
#!/bin/bash
# /etc/cron.daily/docker-volume-backup

BACKUP_DIR="/backups/docker-volumes"
DATE=$(date +%Y%m%d-%H%M%S)

for volume in $(docker volume ls -q); do
  docker run --rm \
    -v ${volume}:/data:ro \
    -v ${BACKUP_DIR}:/backup \
    alpine \
    tar czf /backup/${volume}-${DATE}.tar.gz -C /data .
done

# 30 günden eski backupları sil
find ${BACKUP_DIR} -name "*.tar.gz" -mtime +30 -delete
```

### Volume Performance Optimization

**Sorun:** Bind mount'lar macOS ve Windows'ta yavaş (OSXFS, WLS2)

**Çözüm 1: Named Volumes kullan**
```bash
# ❌ Yavaş (bind mount)
docker run -v $(pwd):/app node npm install  # 120 saniye

# ✅ Hızlı (named volume)
docker run -v nodemodules:/app/node_modules node npm install  # 20 saniye
```

**Çözüm 2: Cached/Delegated mounts**
```bash
# macOS/Windows için
docker run -v $(pwd):/app:cached  # Host → Container yavaş olabilir (ama hızlı)
docker run -v $(pwd):/app:delegated  # Container → Host yavaş olabilir
```

---

## 🌐 Docker Networking - Gerçek Dünya Senaryoları {#docker-networking}

### Docker Network Tipleri

```
┌──────────────────────────────────────────────────────┐
│             DOCKER NETWORK DRIVERS                    │
├──────────────────────────────────────────────────────┤
│  1. bridge    → Default, single-host                 │
│  2. host      → No isolation, host network           │
│  3. none      → No networking                        │
│  4. overlay   → Multi-host, Swarm                    │
│  5. macvlan   → Physical network integration         │
└──────────────────────────────────────────────────────┘
```

### 1. Bridge Network (En Yaygın)

**Default Behavior:**
```bash
docker run -d --name web nginx
# Otomatik olarak "bridge" network'e bağlanır
# Internal IP: 172.17.0.2
```

**Custom Bridge (Production Best Practice):**
```bash
# Custom network oluştur
docker network create myapp-network

# Container'ları bağla
docker run -d --name db --network myapp-network postgres
docker run -d --name backend --network myapp-network node-app
docker run -d --name frontend --network myapp-network react-app

# DNS resolution otomatik çalışır!
# backend container'da:
curl http://db:5432  # "db" hostname'i resolve olur
```

**Neden custom bridge?**
- Automatic DNS resolution (container name → IP)
- Network izolasyonu (farklı app'ler farklı network'lerde)
- Better control over network settings

### 2. Host Network (Performance Critical)

```bash
docker run -d --network host nginx
# Container directly binds to host network interface
# nginx http://localhost:80 erişilebilir (port mapping yok)
```

**Ne zaman kullan:**
- Çok yüksek network throughput gerektiğinde
- Network latency kritikse
- Monitoring tools (Prometheus, Grafana)

**Dikkat:** Security riski! Container host network'e tam erişir.

### 3. Overlay Network (Multi-Host, Kubernetes/Swarm)

```bash
# Swarm init
docker swarm init

# Overlay network oluştur
docker network create -d overlay myoverlay

# Service deploy (multiple nodes)
docker service create \
  --name web \
  --network myoverlay \
  --replicas 5 \
  nginx
```

**Kullanım:** Kubernetes, Docker Swarm gibi orchestration sistemlerinde.

### Container-to-Container Communication

**Senaryo:** Microservices mimarisi

```
┌─────────────────────────────────────────────┐
│        Frontend (React) :80                 │
│                 │                           │
│                 ▼                           │
│        Backend (Node.js) :3000              │
│            │           │                    │
│            ▼           ▼                    │
│      Database      Redis Cache             │
│    (Postgres)      :6379                    │
│       :5432                                 │
└─────────────────────────────────────────────┘
```

**Docker Compose ile:**
```yaml
version: '3.8'

services:
  frontend:
    image: myapp/frontend
    ports:
      - "80:80"
    networks:
      - frontend-network
    environment:
      - API_URL=http://backend:3000

  backend:
    image: myapp/backend
    ports:
      - "3000:3000"
    networks:
      - frontend-network
      - backend-network
    environment:
      - DB_HOST=database
      - REDIS_HOST=redis

  database:
    image: postgres:15
    networks:
      - backend-network
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    networks:
      - backend-network

networks:
  frontend-network:
    driver: bridge
  backend-network:
    driver: bridge
    internal: true  # External access engellenir

volumes:
  pgdata:
```

**Güvenlik Katmanı:**
- `frontend-network`: Internet-facing
- `backend-network`: Internal only (database exposed değil)

### Port Publishing Stratejileri

```bash
# 1. Specific port binding
docker run -p 8080:80 nginx
# Host:8080 → Container:80

# 2. Random port binding
docker run -p 80 nginx
# Host:random → Container:80
# Hangi port? → docker port <container>

# 3. Specific IP binding
docker run -p 127.0.0.1:8080:80 nginx
# Sadece localhost:8080 erişebilir (external access yok)

# 4. Multiple port binding
docker run \
  -p 80:80 \
  -p 443:443 \
  -p 8080:8080 \
  nginx

# 5. UDP port
docker run -p 53:53/udp dns-server
```

**Production Tavsiye:**
```bash
# ✅ Load Balancer arkasında (Nginx, HAProxy)
docker run -p 127.0.0.1:3000:3000 backend
docker run -p 127.0.0.1:3001:3000 backend
docker run -p 127.0.0.1:3002:3000 backend
docker run -p 80:80 nginx  # Reverse proxy
```

### Network Debugging

```bash
# Container network bilgisi
docker inspect <container> | jq '.[0].NetworkSettings'

# Container içinde network test
docker exec -it myapp sh
apk add curl tcpdump netcat-openbsd
curl http://backend:3000
nc -zv database 5432

# Network packet capture
docker run --rm --net container:myapp nicolaka/netshoot tcpdump -i eth0

# DNS resolution test
docker exec myapp nslookup backend
```

---

## 🎼 Docker Compose - Mikroservis Orchestration {#docker-compose}

### Docker Compose Nedir?

**Tek komutla multi-container uygulamaları yönet:**

```bash
docker-compose up     # Tüm servisleri başlat
docker-compose down   # Tüm servisleri durdur ve sil
```

### Production-Ready Docker Compose Örneği

**Senaryo:** E-commerce application

```yaml
version: '3.8'

# ================================================
# SERVICES DEFINITION
# ================================================
services:
  # ──────────────────────────────────────────────
  # Reverse Proxy & Load Balancer
  # ──────────────────────────────────────────────
  nginx:
    image: nginx:1.25-alpine
    container_name: ecommerce-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - static-files:/var/www/static:ro
    networks:
      - frontend
    depends_on:
      - frontend
      - backend
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # ──────────────────────────────────────────────
  # Frontend (React SPA)
  # ──────────────────────────────────────────────
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
      target: production
      args:
        - NODE_ENV=production
        - REACT_APP_API_URL=http://nginx/api
    image: ecommerce/frontend:${VERSION:-latest}
    container_name: ecommerce-frontend
    restart: unless-stopped
    networks:
      - frontend
    environment:
      - NODE_ENV=production
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  # ──────────────────────────────────────────────
  # Backend API (Node.js)
  # ──────────────────────────────────────────────
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
      target: production
    image: ecommerce/backend:${VERSION:-latest}
    container_name: ecommerce-backend
    restart: unless-stopped
    ports:
      - "127.0.0.1:3000:3000"  # Sadece localhost'tan erişilebilir
    networks:
      - frontend
      - backend
    environment:
      - NODE_ENV=production
      - DB_HOST=postgres
      - DB_PORT=5432
      - DB_NAME=${DB_NAME}
      - DB_USER=${DB_USER}
      - DB_PASSWORD=${DB_PASSWORD}
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - JWT_SECRET=${JWT_SECRET}
    env_file:
      - .env.production
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 40s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # ──────────────────────────────────────────────
  # PostgreSQL Database
  # ──────────────────────────────────────────────
  postgres:
    image: postgres:15-alpine
    container_name: ecommerce-db
    restart: unless-stopped
    networks:
      - backend
    environment:
      - POSTGRES_DB=${DB_NAME}
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_INITDB_ARGS=--encoding=UTF-8 --lc-collate=C --lc-ctype=C
      - PGDATA=/var/lib/postgresql/data/pgdata
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./postgres/init-scripts:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    command: >
      postgres
      -c max_connections=200
      -c shared_buffers=256MB
      -c effective_cache_size=1GB
      -c maintenance_work_mem=64MB
      -c checkpoint_completion_target=0.9
      -c wal_buffers=16MB
      -c default_statistics_target=100
      -c random_page_cost=1.1
      -c effective_io_concurrency=200
      -c work_mem=1MB
      -c min_wal_size=1GB
      -c max_wal_size=4GB
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # ──────────────────────────────────────────────
  # Redis Cache
  # ──────────────────────────────────────────────
  redis:
    image: redis:7-alpine
    container_name: ecommerce-redis
    restart: unless-stopped
    networks:
      - backend
    command: >
      redis-server
      --appendonly yes
      --requirepass ${REDIS_PASSWORD}
      --maxmemory 512mb
      --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # ──────────────────────────────────────────────
  # Background Worker (Message Queue)
  # ──────────────────────────────────────────────
  worker:
    build:
      context: ./backend
      dockerfile: Dockerfile.worker
    image: ecommerce/worker:${VERSION:-latest}
    container_name: ecommerce-worker
    restart: unless-stopped
    networks:
      - backend
    environment:
      - NODE_ENV=production
      - DB_HOST=postgres
      - REDIS_HOST=redis
    depends_on:
      - postgres
      - redis
    deploy:
      replicas: 2  # Multiple workers
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # ──────────────────────────────────────────────
  # Monitoring: Prometheus
  # ──────────────────────────────────────────────
  prometheus:
    image: prom/prometheus:latest
    container_name: ecommerce-prometheus
    restart: unless-stopped
    ports:
      - "127.0.0.1:9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    networks:
      - monitoring
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'

  # ──────────────────────────────────────────────
  # Monitoring: Grafana
  # ──────────────────────────────────────────────
  grafana:
    image: grafana/grafana:latest
    container_name: ecommerce-grafana
    restart: unless-stopped
    ports:
      - "127.0.0.1:3001:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards:ro
    networks:
      - monitoring
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
      - GF_USERS_ALLOW_SIGN_UP=false

# ================================================
# NETWORKS
# ================================================
networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # External access blocked
  monitoring:
    driver: bridge

# ================================================
# VOLUMES
# ================================================
volumes:
  postgres-data:
    driver: local
  redis-data:
    driver: local
  static-files:
    driver: local
  prometheus-data:
    driver: local
  grafana-data:
    driver: local
```

### Environment Variables (.env dosyası)

```env
# .env.production
VERSION=1.2.3
DB_NAME=ecommerce
DB_USER=ecommerce_user
DB_PASSWORD=super_secure_password_here
REDIS_PASSWORD=redis_secure_password
JWT_SECRET=very_long_random_secret_key
GRAFANA_PASSWORD=grafana_admin_password
```

### Docker Compose Komutları

```bash
# Servisleri başlat (detached mode)
docker-compose up -d

# Servisleri başlat (belirli servis)
docker-compose up -d backend

# Build ve start
docker-compose up -d --build

# Logları izle
docker-compose logs -f
docker-compose logs -f backend  # Sadece backend

# Servisleri durdur (container'lar kalır)
docker-compose stop

# Servisleri durdur ve sil
docker-compose down

# Servisleri durdur, sil ve volume'ları da sil
docker-compose down -v

# Servis scale etme
docker-compose up -d --scale worker=5

# Container'larda komut çalıştır
docker-compose exec backend sh
docker-compose exec postgres psql -U ecommerce_user -d ecommerce

# Servis restart
docker-compose restart backend

# Servis durumunu görüntüle
docker-compose ps

# Resource kullanımı
docker-compose stats
```

### Docker Compose Override Pattern

**Development vs Production:**

**docker-compose.yml** (Base):
```yaml
version: '3.8'
services:
  backend:
    image: myapp/backend
    environment:
      - NODE_ENV=${NODE_ENV}
```

**docker-compose.override.yml** (Development - default):
```yaml
version: '3.8'
services:
  backend:
    build: ./backend
    volumes:
      - ./backend:/app
    ports:
      - "3000:3000"
      - "9229:9229"  # Debug port
    command: npm run dev
```

**docker-compose.prod.yml** (Production):
```yaml
version: '3.8'
services:
  backend:
    image: myapp/backend:${VERSION}
    restart: unless-stopped
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
```

**Kullanım:**
```bash
# Development
docker-compose up  # Otomatik override.yml kullanır

# Production
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## 🏗️ Multi-Stage Builds - Optimization Masters {#multi-stage-builds}

### Problem: Büyük Image Size

**❌ Naive Dockerfile (1.2 GB):**
```dockerfile
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install  # Dev dependencies dahil!
COPY . .
RUN npm run build  # Build tools image'da kalıyor!
CMD ["npm", "start"]
```

### Çözüm: Multi-Stage Build

**✅ Optimized Dockerfile (150 MB):**
```dockerfile
# ──────────────────────────────────────────────
# Stage 1: Build
# ──────────────────────────────────────────────
FROM node:18 AS builder

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci

# Build application
COPY . .
RUN npm run build

# ──────────────────────────────────────────────
# Stage 2: Production
# ──────────────────────────────────────────────
FROM node:18-alpine

WORKDIR /app

# Copy only production dependencies
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Copy built artifacts from builder stage
COPY --from=builder /app/dist ./dist

# Non-root user
USER node

CMD ["node", "dist/index.js"]
```

**Sonuç:**
- Builder stage: 1.2 GB (sadece build sırasında kullanılır)
- Final image: 150 MB (production'a deploy edilen)

### Advanced Multi-Stage Patterns

#### Pattern 1: Go Binary (2 MB Image!)

```dockerfile
# ──────────────────────────────────────────────
# Stage 1: Build
# ──────────────────────────────────────────────
FROM golang:1.21 AS builder

WORKDIR /app

# Download dependencies
COPY go.mod go.sum ./
RUN go mod download

# Build
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .

# ──────────────────────────────────────────────
# Stage 2: Minimal Runtime
# ──────────────────────────────────────────────
FROM scratch

# Copy SSL certificates (HTTPS için gerekli)
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copy binary
COPY --from=builder /app/app /app

EXPOSE 8080

ENTRYPOINT ["/app"]
```

**Sonuç: 2 MB image!** (Sadece binary + SSL certs)

#### Pattern 2: Python with Wheels

```dockerfile
# ──────────────────────────────────────────────
# Stage 1: Build Python Wheels
# ──────────────────────────────────────────────
FROM python:3.11 AS builder

WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Create wheels
COPY requirements.txt .
RUN pip wheel --no-cache-dir --no-deps --wheel-dir /app/wheels -r requirements.txt

# ──────────────────────────────────────────────
# Stage 2: Runtime
# ──────────────────────────────────────────────
FROM python:3.11-slim

WORKDIR /app

# Install runtime dependencies only
RUN apt-get update && apt-get install -y \
    libpq5 \
    && rm -rf /var/lib/apt/lists/*

# Install wheels from builder
COPY --from=builder /app/wheels /wheels
RUN pip install --no-cache /wheels/*

# Copy application
COPY . .

USER nobody

CMD ["python", "app.py"]
```

#### Pattern 3: Testing Stage

```dockerfile
# ──────────────────────────────────────────────
# Stage 1: Base
# ──────────────────────────────────────────────
FROM node:18 AS base
WORKDIR /app
COPY package*.json ./

# ──────────────────────────────────────────────
# Stage 2: Development Dependencies
# ──────────────────────────────────────────────
FROM base AS development
RUN npm install
COPY . .

# ──────────────────────────────────────────────
# Stage 3: Testing (CI/CD için)
# ──────────────────────────────────────────────
FROM development AS testing
RUN npm run lint
RUN npm run test
RUN npm run test:e2e

# ──────────────────────────────────────────────
# Stage 4: Build
# ──────────────────────────────────────────────
FROM development AS builder
RUN npm run build

# ──────────────────────────────────────────────
# Stage 5: Production
# ──────────────────────────────────────────────
FROM node:18-alpine AS production
WORKDIR /app
COPY --from=base /app/package*.json ./
RUN npm ci --only=production
COPY --from=builder /app/dist ./dist
USER node
CMD ["node", "dist/index.js"]
```

**CI/CD'de kullanım:**
```bash
# Test stage'i run et
docker build --target testing -t myapp:test .

# Test pass olduysa, production build
docker build --target production -t myapp:prod .
```

---

## 🔒 Container Güvenliği - Security Best Practices {#container-guvenligi}

### Kritik Güvenlik Konuları

#### 1. Non-Root User (Must-Have!)

**Neden?**
- Container escape durumunda saldırgan root erişimi kazanır
- Principle of least privilege

**❌ Kötü:**
```dockerfile
FROM ubuntu
COPY app /app
CMD ["/app"]
```

**✅ İyi:**
```dockerfile
FROM ubuntu

# Kullanıcı oluştur
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Dosya ownership
COPY --chown=appuser:appuser app /app

# Switch to non-root
USER appuser

CMD ["/app"]
```

#### 2. Image Scanning (Vulnerability Detection)

**Tool: Trivy**
```bash
# Image scan
trivy image myapp:latest

# Critical ve High vulnerabilities için fail
trivy image --severity CRITICAL,HIGH --exit-code 1 myapp:latest
```

**CI/CD Integration:**
```yaml
# .github/workflows/security.yml
name: Security Scan

on: [push]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Run Trivy scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          severity: 'CRITICAL,HIGH'
          exit-code: '1'  # Fail if vulnerabilities found
```

#### 3. Secret Management

**❌ Asla yapma:**
```dockerfile
# Secrets Dockerfile'da OLMAMALI!
ENV DATABASE_PASSWORD=supersecret123
ARG API_KEY=abc123def456
```

**✅ Doğru yaklaşımlar:**

**a) Runtime Environment Variables:**
```bash
docker run -e DATABASE_PASSWORD=$(cat /secure/password) myapp
```

**b) Docker Secrets (Swarm):**
```bash
echo "mysecret" | docker secret create db_password -
docker service create --secret db_password myapp
```

**c) External Secret Management:**
- **AWS Secrets Manager**
- **HashiCorp Vault**
- **Azure Key Vault**

**Dockerfile:**
```dockerfile
# Runtime'da secret al
FROM alpine
RUN apk add --no-cache aws-cli
COPY entrypoint.sh /
ENTRYPOINT ["/entrypoint.sh"]
```

**entrypoint.sh:**
```bash
#!/bin/sh
export DB_PASSWORD=$(aws secretsmanager get-secret-value --secret-id prod/db --query SecretString --output text)
exec "$@"
```

#### 4. Read-Only Filesystem

```bash
# Root filesystem read-only
docker run --read-only \
  --tmpfs /tmp \
  --tmpfs /var/run \
  myapp
```

**Dockerfile:**
```dockerfile
FROM alpine
RUN adduser -D appuser
USER appuser
COPY --chown=appuser:appuser app /app
WORKDIR /app
# Write sadece /tmp'ye
VOLUME /tmp
CMD ["./app"]
```

#### 5. Capabilities Dropping

**Linux capabilities nedir?**
Root privileges'ı parçalara ayırır (network binding, file ownership, vb.)

```bash
# Tüm capabilities'leri drop et, sadece gerekenleri ver
docker run \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \  # Port 80'e bind için
  myapp
```

#### 6. AppArmor / SELinux

```bash
# AppArmor profile kullan
docker run --security-opt apparmor=docker-default nginx

# SELinux context
docker run --security-opt label=level:s0:c100,c200 myapp
```

#### 7. Image Signing (Notary/Cosign)

**Docker Content Trust:**
```bash
export DOCKER_CONTENT_TRUST=1
docker push myregistry/myapp:latest
# Image sign edilir, tampered edilirse pull fail olur
```

### Security Checklist

```markdown
# Container Security Checklist

## Build Time
- [ ] Use official base images (Docker Official Images)
- [ ] Pin image versions (avoid :latest)
- [ ] Scan images for vulnerabilities (Trivy, Snyk)
- [ ] Use multi-stage builds (minimize attack surface)
- [ ] Don't include secrets in image
- [ ] Use .dockerignore (prevent sensitive files)
- [ ] Create non-root user
- [ ] Set file permissions correctly

## Runtime
- [ ] Run as non-root user (USER directive)
- [ ] Use read-only filesystem where possible
- [ ] Drop unnecessary capabilities (--cap-drop=ALL)
- [ ] Set resource limits (CPU, memory)
- [ ] Use AppArmor/SELinux profiles
- [ ] Enable Docker Content Trust
- [ ] Use private registries for sensitive images
- [ ] Implement network segmentation
- [ ] Enable logging and monitoring
- [ ] Regular security updates (rebuild images)

## Orchestration (Kubernetes/Swarm)
- [ ] Use Pod Security Policies / Pod Security Standards
- [ ] Network policies (firewall between pods)
- [ ] RBAC (Role-Based Access Control)
- [ ] Secrets management (Vault, Sealed Secrets)
- [ ] Image pull policies
- [ ] Admission controllers (OPA Gatekeeper)
```

---

## 🚀 Production Deployment Stratejileri {#production-deployment}

### Blue-Green Deployment

**Konsept:** İki identik environment (Blue: Current, Green: New)

```bash
# Current: Blue (v1.0) running
docker-compose -f docker-compose.blue.yml up -d

# Deploy: Green (v2.0)
docker-compose -f docker-compose.green.yml up -d

# Test green
curl http://green.myapp.com/health

# Switch traffic (load balancer config change)
# Blue → Green

# Rollback if needed (revert load balancer)
```

**Avantajları:**
- Zero downtime
- Instant rollback
- Testing in production-like environment

**Dezavantajları:**
- 2x resource gerekir
- Database migration karmaşık

### Rolling Update (Kubernetes/Swarm)

```yaml
# docker-compose.yml (Swarm mode)
version: '3.8'
services:
  web:
    image: myapp:v2
    deploy:
      replicas: 10
      update_config:
        parallelism: 2        # 2'şer 2'şer güncelle
        delay: 10s            # Her update arası 10s bekle
        failure_action: rollback
        monitor: 30s
      rollback_config:
        parallelism: 2
        delay: 5s
```

```bash
docker stack deploy -c docker-compose.yml myapp
# v1 → v2 rolling update başlar
# 10 replica: 2 update, 10s wait, 2 update, ...
```

### Canary Deployment

**Konsept:** Yeni version'u küçük trafik subset'ine deploy et

```
Traffic Distribution:
95% → v1 (stable)
5%  → v2 (canary)

If v2 metrics OK:
50% → v1
50% → v2

If still OK:
0%  → v1
100% → v2
```

**Nginx Load Balancer:**
```nginx
upstream backend {
    server backend-v1:3000 weight=95;
    server backend-v2:3000 weight=5;
}
```

---

## 🔧 Troubleshooting ve Debugging {#troubleshooting}

### Yaygın Problemler ve Çözümler

#### Problem 1: Container Hemen Çıkıyor

```bash
# Container start oluyor ama hemen exit ediyor
docker ps -a  # Status: Exited (0)

# Log kontrol
docker logs <container_id>

# Muhtemel sebepler:
# 1. CMD/ENTRYPOINT hatalı
# 2. Application crash
# 3. Foreground process yok (daemon mode)
```

**Çözüm:**
```dockerfile
# ❌ Kötü (nginx daemon mode çalışır, container exit eder)
CMD ["nginx"]

# ✅ İyi (foreground mode)
CMD ["nginx", "-g", "daemon off;"]
```

#### Problem 2: "No Space Left on Device"

```bash
# Disk dolu
docker system df  # Kullanım görüntüle

# Cleanup
docker system prune -a  # Unused her şeyi sil
docker volume prune     # Unused volumes
docker builder prune    # Build cache
```

#### Problem 3: Network Connectivity

```bash
# Container birbirine erişemiyor
docker network inspect mynetwork

# DNS resolution test
docker exec container1 nslookup container2

# Port listening test
docker exec container1 nc -zv container2 5432
```

### Debugging Tools

```bash
# Container içinde shell
docker exec -it myapp sh

# Container process tree
docker top myapp

# Resource usage
docker stats myapp

# Container'a file kopyala
docker cp local-file.txt myapp:/tmp/

# Container'dan file al
docker cp myapp:/app/logs/error.log ./

# Network packet capture
docker run --rm --net container:myapp nicolaka/netshoot tcpdump -i any port 80

# Container filesystem diff
docker diff myapp  # Container start'tan sonra değişen dosyalar
```

---

## 💻 Pratik Projeler {#pratik-projeler}

### Proje 1: Node.js App Containerization

**app.js:**
```javascript
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

app.get('/health', (req, res) => {
  res.json({ status: 'healthy', uptime: process.uptime() });
});

app.get('/', (req, res) => {
  res.json({ message: 'Hello from Docker!' });
});

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

**Dockerfile:**
```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
USER node
EXPOSE 3000
HEALTHCHECK CMD node healthcheck.js
CMD ["node", "app.js"]
```

**Build & Run:**
```bash
docker build -t myapp:1.0 .
docker run -d -p 3000:3000 --name myapp myapp:1.0
curl http://localhost:3000/health
```

### Proje 2: Full-Stack App (Frontend + Backend + DB)

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  frontend:
    build: ./frontend
    ports:
      - "80:80"
    depends_on:
      - backend

  backend:
    build: ./backend
    environment:
      - DATABASE_URL=postgresql://user:pass@postgres:5432/mydb
    depends_on:
      - postgres

  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_PASSWORD=pass
      - POSTGRES_USER=user
      - POSTGRES_DB=mydb
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

**Çalıştır:**
```bash
docker-compose up -d
docker-compose logs -f
docker-compose ps
```

### Proje 3: Custom Base Image (Alpine)

**Dockerfile:**
```dockerfile
FROM alpine:3.19

# Maintainer info
LABEL maintainer="you@company.com"
LABEL description="Custom Python Alpine base image"

# Install Python and common packages
RUN apk add --no-cache \
    python3 \
    py3-pip \
    curl \
    ca-certificates \
    && ln -sf python3 /usr/bin/python

# Install common Python packages
RUN pip3 install --no-cache-dir \
    requests \
    flask \
    gunicorn

# Create app user
RUN addgroup -g 1001 -S appuser && \
    adduser -u 1001 -S appuser -G appuser

WORKDIR /app
USER appuser

CMD ["python3"]
```

**Build & Push:**
```bash
docker build -t mycompany/python-alpine:3.19 .
docker push mycompany/python-alpine:3.19

# Artık diğer projelerde kullan
FROM mycompany/python-alpine:3.19
COPY app.py .
CMD ["python", "app.py"]
```

### Proje 4: Security Best Practices Implementation

**Secure Dockerfile:**
```dockerfile
# ──────────────────────────────────────────────
# Stage 1: Build
# ──────────────────────────────────────────────
FROM node:18-alpine AS builder

WORKDIR /build

# Install dependencies
COPY package*.json ./
RUN npm ci

# Build
COPY . .
RUN npm run build

# ──────────────────────────────────────────────
# Stage 2: Security Scan
# ──────────────────────────────────────────────
FROM builder AS security-scan
RUN npm audit --audit-level=high
RUN npm run lint

# ──────────────────────────────────────────────
# Stage 3: Production
# ──────────────────────────────────────────────
FROM node:18-alpine AS production

# Install dumb-init
RUN apk add --no-cache dumb-init

# Create user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# Copy dependencies
COPY --from=builder --chown=nodejs:nodejs /build/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /build/dist ./dist
COPY --chown=nodejs:nodejs package.json ./

# Security: read-only root filesystem
# Application writes to /tmp only
RUN chown nodejs:nodejs /tmp

USER nodejs

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s \
  CMD node healthcheck.js || exit 1

ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "dist/index.js"]
```

**Deployment with security:**
```bash
# Build
docker build -t myapp:secure .

# Run with security options
docker run -d \
  --name myapp \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=100m \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --security-opt=no-new-privileges \
  -p 3000:3000 \
  myapp:secure
```

---

## 📚 İleri Seviye Konular

### 1. Container Orchestration

**Ne zaman Kubernetes/Swarm gerekir?**
- 10+ container manage edilecekse
- Auto-scaling gerekiyorsa
- Self-healing (crashed container'ları restart)
- Load balancing
- Rolling updates
- Service discovery

### 2. CI/CD Pipeline Integration

**GitHub Actions Örneği:**
```yaml
name: Docker Build & Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Login to DockerHub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: myapp:${{ github.sha }},myapp:latest

      - name: Deploy to production
        run: |
          ssh user@production-server "docker pull myapp:${{ github.sha }} && docker-compose up -d"
```

### 3. Monitoring ve Logging

**Prometheus + Grafana:**
```yaml
# docker-compose.monitoring.yml
version: '3.8'
services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin

  node-exporter:
    image: prom/node-exporter
    ports:
      - "9100:9100"

  cadvisor:
    image: gcr.io/cadvisor/cadvisor
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
    ports:
      - "8080:8080"
```

---

## 🎓 Öğrenme Yol Haritası

### Hafta 1-2: Temel Konular
- [x] Container vs VM anla
- [x] Docker install ve setup
- [x] Dockerfile yaz
- [x] Image build et
- [x] Container çalıştır
- [x] Volume kullan
- [x] Network oluştur

### Hafta 3-4: Orta Seviye
- [x] Multi-stage builds
- [x] Docker Compose
- [x] Security best practices
- [x] Image optimization
- [x] Debugging techniques

### Hafta 5-6: İleri Seviye
- [ ] Container orchestration (Kubernetes basics)
- [ ] CI/CD integration
- [ ] Monitoring ve logging
- [ ] Production deployment patterns
- [ ] Performance tuning

### Hafta 7-8: Production Ready
- [ ] Kubernetes deep dive
- [ ] Helm charts
- [ ] Service mesh (Istio)
- [ ] Security hardening
- [ ] Disaster recovery

---

## 🔗 Kaynaklar

### Resmi Dokümantasyon
- [Docker Official Documentation](https://docs.docker.com/)
- [Docker Hub](https://hub.docker.com/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)

### Kitaplar
- "Docker Deep Dive" - Nigel Poulton
- "Kubernetes Up & Running" - Kelsey Hightower
- "The DevOps Handbook" - Gene Kim

### Online Platformlar
- [Play with Docker](https://labs.play-with-docker.com/)
- [Katacoda Docker Scenarios](https://www.katacoda.com/courses/docker)
- [Docker Mastery Course - Udemy](https://www.udemy.com/course/docker-mastery/)

### Tools
- **Trivy**: Image vulnerability scanning
- **Dive**: Image layer analysis
- **Hadolint**: Dockerfile linter
- **Docker Bench Security**: Security audit tool

### Topluluk
- [Docker Community Forums](https://forums.docker.com/)
- [r/docker subreddit](https://reddit.com/r/docker)
- [CNCF Slack](https://slack.cncf.io/)

---

## 🎯 Son Tavsiyeler

### Mühendis Kafasıyla Yaklaşım

1. **Her Şeyi Sorgula:**
   - Neden container? VM neden yetmez?
   - Alpine neden Debian'dan küçük?
   - Multi-stage build gerçekten gerekli mi?

2. **Trade-off'ları Anla:**
   - Image size ↔ Compatibility
   - Security ↔ Convenience
   - Performance ↔ Isolation

3. **Production-First Düşün:**
   - Local'de çalışması yetmez
   - Monitoring, logging, security düşün
   - Rollback planın olsun

4. **Sürekli Öğren:**
   - Container teknolojisi hızlı evrim geçiriyor
   - Yeni güvenlik vulnerability'leri keşfediliyor
   - Best practice'ler değişiyor

### Pratik Yaparak Öğren

**Bugün başla:**
```bash
# 1. Docker install
curl -fsSL https://get.docker.com | sh

# 2. İlk container'ını çalıştır
docker run -it --rm alpine sh

# 3. Basit bir uygulama containerize et
# 4. docker-compose ile multi-container app yap
# 5. Production'a deploy et
```

**Hata yapmaktan korkma:**
- Container'lar disposable, test etmek güvenli
- En iyi öğrenme şekli: bozup tamir etmek
- Her hatadan bir security/performance insight çıkar

---

## 📝 Özet

Bu rehberde öğrendiklerimiz:

✅ Container teknolojisinin temelleri ve VM'lerden farkları
✅ Docker mimarisinin derinlemesine analizi
✅ Production-ready Dockerfile yazımı
✅ Volume, network, ve compose kullanımı
✅ Multi-stage build ile optimization
✅ Container security best practices
✅ Production deployment stratejileri
✅ Troubleshooting ve debugging teknikleri

**Bir sonraki adım:** Kubernetes! 🚀

Container'ları master ettikten sonra orchestration (Kubernetes) öğren. Kubernetes, container'ları scale etmek, manage etmek ve production'da çalıştırmak için industry standard.

---

**Hazırlayan:** Deneyimli Container & DevOps Mühendisi
**Son Güncelleme:** 14 Şubat 2026
**Versiyon:** 1.0

*İyi çalışmalar! Happy containerizing! 🐳*
