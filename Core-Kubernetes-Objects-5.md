# ⚙️ Core Kubernetes Objects — Hafta 5-6

> "Kubernetes'i gerçekten öğrenmek istiyorsan, bileşenlerini ezberleme — ne yaptıklarını, neden var olduklarını anla."

10 yıldır Kubernetes ile çalışıyorum. Bu sürede şunu fark ettim: İnsanlar Pod nedir, ReplicaSet nedir diye ezberliyor ama aralarındaki ilişkiyi anlamıyorlar. Bu rehberde sana sadece tanımları değil, "neden bu nesne var?", "gerçekte ne işe yarıyor?", "bir şeyler bozulduğunda nasıl düşünürüm?" sorularının cevaplarını vereceğim.

Hazırsan başlayalım.

---

## 📋 İçindekiler

- [Pod: Kubernetes'in Temel Birimi](#pod)
  - [Pod Lifecycle](#pod-lifecycle)
  - [Multi-Container Pods](#multi-container)
  - [Init Containers](#init-containers)
  - [Pod Resource Defaults](#pod-presets)
- [ReplicaSets ve Deployments](#replicaset-deployment)
  - [Rolling Updates](#rolling-updates)
  - [Rollback Stratejileri](#rollback)
  - [Scaling](#scaling)
- [Services: Pod'lara Nasıl Ulaşırsın?](#services)
  - [ClusterIP](#clusterip)
  - [NodePort](#nodeport)
  - [LoadBalancer](#loadbalancer)
  - [ExternalName](#externalname)
- [ConfigMaps ve Secrets](#configmaps-secrets)
  - [Environment Variables](#env-vars)
  - [Volume Mounts](#volume-mounts)
  - [Secret Types](#secret-types)
- [Pratik Projeler](#pratik-projeler)

---

<a id="pod"></a>
## 🫘 Pod: Kubernetes'in Temel Birimi

### Önce Şunu Anla: Neden "Pod" Var?

Docker öğrendiğinde container kavramını öğrendin. Kubernetes'e gelince şöyle bir soru gelir akla: "Container'ı direkt çalıştırsam olmaz mı?" Olmaz. Çünkü Kubernetes tek bir container'la değil, **Pod** adı verilen bir sarmalayıcıyla çalışır.

Pod, **bir veya daha fazla container'ı bir arada tutan en küçük dağıtım birimidir.**

Neden böyle tasarlandı? Google Borg'un deneyiminden: Bazen birbirine sıkı bağlı iki süreç aynı ağı ve depolamayı paylaşmak zorunda kalıyor. Bunları ayrı container'lara koyup ama aynı "birim" gibi davranmalarını sağlamak için Pod soyutlaması geldi.

```
┌─────────────────────────────────────────────┐
│                    POD                       │
│                                              │
│  ┌──────────────┐    ┌──────────────┐        │
│  │  Container 1 │    │  Container 2 │        │
│  │  (app)       │    │  (sidecar)   │        │
│  └──────────────┘    └──────────────┘        │
│                                              │
│  Shared: Network (localhost), Volumes        │
│  IP: 10.244.1.5 (tek IP, iki container)      │
└─────────────────────────────────────────────┘
```

### En Basit Pod Tanımı

```yaml
# basit-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: benim-ilk-pod
  labels:
    app: web
    ortam: test
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

```bash
# Pod oluştur
kubectl apply -f basit-pod.yaml

# Durumunu izle
kubectl get pod benim-ilk-pod
kubectl describe pod benim-ilk-pod

# Pod içine gir
kubectl exec -it benim-ilk-pod -- /bin/bash

# Logları gör
kubectl logs benim-ilk-pod
```

> **Gerçek Hayat Notu:** Production'da Pod'ları direkt yaratmazsın. Neden? Bir Node çökerse Pod kaybolur, kimse onu geri getirmez. Bu işi Deployment veya diğer controller'lar yapar. Ama Pod YAML'ını anlamak şart — Deployment da sonunda Pod tanımı içeriyor.

---

<a id="pod-lifecycle"></a>
### 🔄 Pod Lifecycle (Yaşam Döngüsü)

Pod'un hayatı 5 aşamadan geçer. Sorun giderirken hangi aşamada takıldığını bilmek inanılmaz değerli.

```
Pending → Running → Succeeded / Failed / Unknown
```

| Durum | Anlamı | Ne Yapmalısın? |
|-------|--------|----------------|
| `Pending` | Pod kabul edildi ama container henüz başlamadı | `kubectl describe pod` ile sebebe bak |
| `Running` | En az bir container çalışıyor | Normal durum |
| `Succeeded` | Tüm container'lar başarıyla tamamlandı (Job'lar için) | Beklenen durum |
| `Failed` | En az bir container hatayla çıktı | Log'lara bak |
| `Unknown` | Pod'un durumu bilinemiyor (Node ile iletişim koptu) | Node durumunu kontrol et |

#### Container State (Container Durumu)

Pod durumunun altında her container'ın kendi durumu var:

| Container State | Anlamı |
|-----------------|--------|
| `Waiting` | Container başlamayı bekliyor (image çekiliyor vs.) |
| `Running` | Çalışıyor |
| `Terminated` | Durdu (başarılı veya hatalı) |

```bash
# Detaylı container durumlarını gör
kubectl get pod benim-pod -o jsonpath='{.status.containerStatuses}'
```

#### Restart Politikası

Pod'un container'ı çökerse ne olur? Bu `restartPolicy` ile belirlenir:

```yaml
spec:
  restartPolicy: Always    # Her durumda yeniden başlat (default)
  # restartPolicy: OnFailure  # Sadece hata durumunda
  # restartPolicy: Never      # Asla yeniden başlatma (Job'larda kullanılır)
```

#### Probe'lar: Kubernetes Sağlık Kontrolleri

Kubernetes bir container'ın "sağlıklı" olduğunu nasıl anlıyor? Probe'larla:

```yaml
spec:
  containers:
  - name: uygulama
    image: benim-uygulama:1.0

    # Liveness Probe: "Container hâlâ yaşıyor mu?"
    # Başarısız olursa container YENİDEN BAŞLATILIR
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 15   # Başlamak için 15 sn bekle
      periodSeconds: 10         # Her 10 saniyede bir kontrol et
      failureThreshold: 3       # 3 başarısız = yeniden başlat

    # Readiness Probe: "Container trafik almaya hazır mı?"
    # Başarısız olursa Service'ten ÇIKARILIR, trafik gelmez
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5

    # Startup Probe: "Container ilk başlarken hazır mı?"
    # Yavaş başlayan uygulamalar için (Java uygulamaları gibi)
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30    # 30 * 10s = 300 saniye tolerans
      periodSeconds: 10
```

> **Kritik Fark:** Liveness ve Readiness probe'larını karıştırma. Liveness başarısız → container restart. Readiness başarısız → sadece trafik kesilir, container çalışmaya devam eder. Bir veritabanı bağlantısı koptuğunda container'ı restart etmek değil, trafik almayı durdurmasını istiyorsun. Bunun için Readiness kullan.

---

<a id="multi-container"></a>
### 🤝 Multi-Container Pods

Bazen iki container'ın aynı Pod'da çalışması gerekir. Bu pattern'ların isimleri var:

#### Sidecar Pattern

Ana uygulamanın yanında onu destekleyen container. Log toplayıcı, servis mesh proxy'si (Envoy/Istio), secret rotator...

```yaml
# sidecar-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: uygulama-ile-log-toplayici
spec:
  # Paylaşılan Volume: iki container da buraya yazıp okur
  volumes:
  - name: paylasilan-loglar
    emptyDir: {}

  containers:
  # 1. Ana uygulama
  - name: uygulama
    image: benim-uygulama:1.0
    volumeMounts:
    - name: paylasilan-loglar
      mountPath: /var/log/uygulama

  # 2. Sidecar: Log toplayıcı
  - name: log-toplayici
    image: fluentd:v1.16
    volumeMounts:
    - name: paylasilan-loglar
      mountPath: /var/log/uygulama
      readOnly: true
    env:
    - name: FLUENTD_ARGS
      value: "--no-supervisor"
```

#### Ambassador Pattern

Ana uygulamanın dış dünyayla iletişimini yöneten proxy container. Örneğin: Uygulama `localhost:5432`'ye bağlanıyor sandığında aslında bir proxy'e bağlanıyor, proxy bağlantıyı doğru veritabanına yönlendiriyor.

#### Adapter Pattern

Ana uygulamanın çıktısını dışarıya uygun formatta sunan container. Prometheus metriklerini eski sisteme uygun formata çevirmek gibi.

```bash
# Multi-container pod'da belirli bir container'ın loglarını görmek
kubectl logs pod-adi -c container-adi

# Belirli container'a exec
kubectl exec -it pod-adi -c container-adi -- /bin/sh
```

---

<a id="init-containers"></a>
### 🚀 Init Containers

Ana container başlamadan önce çalışması gereken hazırlık işleri var mı? Init container'lar bunun için:

```
Init Container 1 → Init Container 2 → Ana Container Başlar
   (Sırayla)         (Önceki bitmeden başlamaz)
```

**Kullanım senaryoları:**
- Veritabanının hazır olmasını beklemek
- Config dosyalarını başlamadan önce çekmek
- Şifreli dosyaların şifresini çözmek
- Uygulama başlamadan önce migration çalıştırmak

```yaml
# init-container-ornek.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-uygulamasi
spec:
  initContainers:

  # Init 1: Veritabanı hazır olana kadar bekle
  - name: veritabani-bekle
    image: busybox:1.36
    command: ['sh', '-c',
      'until nc -z veritabani-service 5432; do echo "DB bekleniyor..."; sleep 2; done; echo "DB hazır!"']

  # Init 2: Config dosyasını çek
  - name: config-cek
    image: alpine/curl:latest
    command: ['sh', '-c',
      'curl -s https://config-server/app-config > /config/app.properties']
    volumeMounts:
    - name: config-volume
      mountPath: /config

  # Ana container: Config hazır, DB hazır, şimdi çalışabilir
  containers:
  - name: web
    image: benim-web-uygulamam:1.0
    volumeMounts:
    - name: config-volume
      mountPath: /app/config

  volumes:
  - name: config-volume
    emptyDir: {}
```

> **İnce Nokta:** Init container'lar başarıyla tamamlanmalı. Başarısız olursa Pod `Init:Error` veya `Init:CrashLoopBackOff` durumuna girer ve ana container hiç başlamaz. `kubectl describe pod` ile init container'ın loglarını incele.

---

<a id="pod-presets"></a>
### 📋 Pod Resource Defaults (LimitRange)

Pod Presets eski bir özellik ve deprecated oldu. Ama arkasındaki ihtiyaç hâlâ geçerli: "Her Pod'a varsayılan kaynak limitleri nasıl atarım?" Bunun için bugün **LimitRange** kullanılıyor.

```yaml
# limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: varsayilan-limitler
  namespace: uygulama
spec:
  limits:
  - default:           # Container başına varsayılan limit
      memory: 256Mi
      cpu: 500m
    defaultRequest:    # Container başına varsayılan request
      memory: 128Mi
      cpu: 250m
    type: Container
```

Bu namespace'e atandıktan sonra resources belirtmemiş her Pod otomatik olarak bu değerleri alır.

---

<a id="replicaset-deployment"></a>
## 🔄 ReplicaSets ve Deployments

### ReplicaSet: Pod Sayısını Garantile

ReplicaSet'in tek görevi: "Her zaman X tane Pod çalışsın" garantisini vermek.

```yaml
# replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-replicaset
spec:
  replicas: 3    # Her zaman 3 Pod çalışsın
  selector:
    matchLabels:
      app: web   # Bu etikete sahip Pod'ları yönet
  template:      # Yeni Pod oluşturulacaksa bu şablonu kullan
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

```bash
# Bir Pod'u sil — ReplicaSet hemen yenisini oluşturur
kubectl delete pod web-replicaset-xxxxx

# Pod sayısını değiştir
kubectl scale replicaset web-replicaset --replicas=5
```

> **Ama direkt ReplicaSet kullanma!** Neden? Güncelleme yönetimi yok. `nginx:1.25`'i `nginx:1.26`'ya geçirmek istersen ne yaparsın? ReplicaSet bilmiyor. Bunun için **Deployment** var.

---

### Deployment: Gerçek Hayatta Kullandığın Nesne

Deployment, ReplicaSet'in üstüne **versiyonlama ve güncelleme yönetimi** ekler.

```
Deployment → ReplicaSet (v1) → Pod, Pod, Pod
           ↘ ReplicaSet (v2) → Pod, Pod, Pod  ← güncelleme sırasında
```

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
  labels:
    app: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web

  # Güncelleme stratejisi
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Güncelleme sırasında max +1 Pod ekstra
      maxUnavailable: 0  # Sıfır downtime: hiçbir Pod hazırsız olmasın

  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
```

```bash
# Deployment oluştur
kubectl apply -f deployment.yaml

# Durumu izle
kubectl get deployment web-deployment
kubectl rollout status deployment/web-deployment

# ReplicaSet'leri gör (deployment her güncellemede yenisini yaratır)
kubectl get replicasets
```

---

<a id="rolling-updates"></a>
### 🔃 Rolling Updates

Uygulamayı güncellerken hiç downtime olmaması için Kubernetes Rolling Update yapar.

**Nasıl Çalışır:**

```
Başlangıç:   [v1] [v1] [v1]

Adım 1:      [v1] [v1] [v1] [v2]   ← maxSurge=1, yeni Pod eklendi
Adım 2:      [v1] [v1] [v2]        ← eski Pod kaldırıldı
Adım 3:      [v1] [v1] [v2] [v2]   ← yeni Pod eklendi
Adım 4:      [v1] [v2] [v2]        ← eski Pod kaldırıldı
...
Son:         [v2] [v2] [v2]        ← güncelleme tamamlandı
```

```bash
# Image güncelle — Rolling Update başlar
kubectl set image deployment/web-deployment nginx=nginx:1.26

# Güncellemeyi izle
kubectl rollout status deployment/web-deployment

# Güncelleme geçmişi
kubectl rollout history deployment/web-deployment

# Güncellemeyi durdur (sorun çıktıysa)
kubectl rollout pause deployment/web-deployment

# Devam ettir
kubectl rollout resume deployment/web-deployment
```

#### `maxSurge` ve `maxUnavailable` Ayarları

Bu iki parametre performans ve güvenlik arasındaki dengeyi belirler:

```yaml
strategy:
  rollingUpdate:
    maxSurge: 25%        # Sayı veya yüzde olabilir
    maxUnavailable: 25%  # Sayı veya yüzde olabilir
```

| Senaryo | maxSurge | maxUnavailable | Sonuç |
|---------|----------|----------------|-------|
| Sıfır downtime | 1 | 0 | Yavaş ama güvenli |
| Hızlı güncelleme | 3 | 1 | Hızlı ama riskli |
| Kaynak tasarrufu | 0 | 1 | Yavaş, kaynak harcamaz |

---

<a id="rollback"></a>
### ⏪ Rollback Stratejileri

Güncelleme sonrası sorun çıktı. Geri dönmek çok kolay:

```bash
# Bir önceki versiyona dön
kubectl rollout undo deployment/web-deployment

# Belirli bir revizyona dön
kubectl rollout undo deployment/web-deployment --to-revision=2

# Tüm geçmişi gör
kubectl rollout history deployment/web-deployment

# Belirli revizyonun detaylarını gör
kubectl rollout history deployment/web-deployment --revision=2
```

**Kaç revision saklanır?**

```yaml
spec:
  revisionHistoryLimit: 10  # Default 10 — geçmişte 10 ReplicaSet tutar
```

> **Gerçek Hayat Deneyimi:** Production'da bir güncelleme yaptıktan sonra hemen monitor et. CPU, memory, error rate, response time — bunların hepsi anormal gidiyorsa `kubectl rollout undo` ile 30 saniyede geri dönersin. CI/CD pipeline'larında otomatik rollback da ekleyebilirsin.

---

<a id="scaling"></a>
### 📈 Scaling

**Manuel Scaling:**

```bash
# Anında 5'e çıkar
kubectl scale deployment/web-deployment --replicas=5

# YAML'dan da yapabilirsin
kubectl patch deployment web-deployment -p '{"spec":{"replicas":5}}'
```

**Horizontal Pod Autoscaler (HPA) — Otomatik Scaling:**

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # CPU %70'i geçince scale up
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

```bash
kubectl apply -f hpa.yaml
kubectl get hpa
kubectl describe hpa web-hpa
```

> **Önemli:** HPA çalışması için Pod'ların `resources.requests` tanımlanmış olması şart. Requests tanımlanmamışsa HPA neye göre hesaplama yapacağını bilemiyor.

---

<a id="services"></a>
## 🌐 Services: Pod'lara Nasıl Ulaşırsın?

### Problem: Pod IP'leri Değişiyor

Her Pod başladığında yeni bir IP alıyor. Bir Pod ölüp yenisi doğduğunda IP değişiyor. Peki diğer uygulamalar sana nasıl ulaşacak?

**Service**, sabit bir IP ve DNS adı vererek Pod'ları erişilebilir kılar. Arkasında Pod'lar değişse bile Service adresi sabit kalır.

```
Kullanıcı → Service (sabit IP/DNS) → Pod 1 (10.244.1.2)
                                   → Pod 2 (10.244.1.3)
                                   → Pod 3 (10.244.1.4)
```

Service, hangi Pod'lara yönleneceğini **label selector** ile bulur:

```yaml
# Service selector
selector:
  app: web        # app=web labelı olan Pod'lara yönlen
  ortam: prod
```

---

<a id="clusterip"></a>
### 1️⃣ ClusterIP — Cluster İçi Erişim

**Sadece cluster içinden** erişilebilen servis. Default türdür. Microservice'ler birbirleriyle konuşurken kullanır.

```
Dış Dünya ✗    →    ClusterIP Service    →    Pod'lar
Cluster İçi ✓
```

```yaml
# clusterip-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: ClusterIP  # Default, yazmasan da olur
  selector:
    app: web
  ports:
  - port: 80         # Service portu (diğerleri buna bağlanır)
    targetPort: 8080  # Pod'un gerçek portu
    protocol: TCP
```

```bash
kubectl apply -f clusterip-service.yaml
kubectl get service web-service

# DNS ile erişim (cluster içinden):
# web-service                             → aynı namespace
# web-service.default                     → namespace belirt
# web-service.default.svc.cluster.local  → tam DNS adı
```

> **DNS Mekaniği:** Kubernetes her namespace'te bir DNS resolver çalıştırır (CoreDNS). `web-service` diye eriştiğinde aslında `web-service.default.svc.cluster.local` çözülüyor. Bu yüzden servis adını değiştirirsen, onu kullanan uygulamaları da güncellemelisin.

---

<a id="nodeport"></a>
### 2️⃣ NodePort — Node'dan Dışarıya Aç

Cluster'ın **her Node'unda** belirli bir port açar. Bu port üzerinden dışarıdan erişim mümkün olur.

```
Dış Dünya → Node IP:30080 → Service → Pod
```

```yaml
# nodeport-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80          # Cluster içi erişim portu
    targetPort: 8080  # Pod portu
    nodePort: 30080   # Dış erişim portu (30000-32767 arası)
```

```bash
# Node IP'ini öğren
kubectl get nodes -o wide

# Erişim:
# http://<herhangi-bir-node-ip>:30080
```

**Ne zaman kullanılır?**
- Geliştirme ve test ortamları
- Bare metal cluster'lar (cloud provider yok)
- Hızlı prototip

**Ne zaman kullanılmaz?**
- Production cloud ortamlarında (LoadBalancer daha iyi)
- Her servis için ayrı port yönetmek karmaşık

---

<a id="loadbalancer"></a>
### 3️⃣ LoadBalancer — Cloud'da Production

Cloud provider'ın (AWS, GCP, Azure) load balancer'ını otomatik oluşturur. Dışarıya **tek, sabit bir IP** ile servis sunar.

```
İnternet → Cloud Load Balancer (external IP) → NodePort → Service → Pod
```

```yaml
# loadbalancer-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-loadbalancer
  annotations:
    # AWS ELB ayarları gibi cloud-specific annotation'lar
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

```bash
kubectl get service web-loadbalancer
# EXTERNAL-IP sütununda cloud'un sana atadığı IP gelir
# Pending görüyorsan cloud entegrasyonu bekleniyor demektir
```

**Önemli:** Her LoadBalancer Service ayrı bir cloud load balancer oluşturur → para! 10 servisin varsa 10 load balancer ödersin. Bunun yerine **Ingress** kullan: tek bir load balancer, tüm servislere routing. (Ingress sonraki haftalarda.)

---

<a id="externalname"></a>
### 4️⃣ ExternalName — Dış Servise Alias

Cluster dışındaki bir servise Kubernetes DNS üzerinden erişmek için kullanılır. Pod aynı cluster içindeymiş gibi bir DNS adıyla dış servise bağlanır.

```
Pod → veritabani.default.svc.cluster.local → DNS CNAME → prod-db.amazonaws.com
```

```yaml
# externalname-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: veritabani
  namespace: default
spec:
  type: ExternalName
  externalName: prod-db.us-east-1.rds.amazonaws.com
```

**Neden kullanılır?**
- Migration sırasında: Önce dış DB, sonra Kubernetes içine taşı — kod değişmez
- Cluster dışı servisleri cluster içindeymiş gibi adresle
- Environment'a göre farklı dış servislere yönlendir

```yaml
# Uygulama kodu her ortamda aynı adrese bağlanır:
# "veritabani" → dev'de: local PostgreSQL
#              → prod'da: RDS instance
```

---

### Service Özeti

```
┌──────────────────────────────────────────────────────────────────┐
│                    SERVİS TİPLERİ                                │
├──────────────────┬───────────────────────────────────────────────┤
│ ClusterIP        │ Sadece cluster içi. Default. Microservice.    │
├──────────────────┼───────────────────────────────────────────────┤
│ NodePort         │ Node IP:Port ile dışarıdan erişim. Dev/Test.  │
├──────────────────┼───────────────────────────────────────────────┤
│ LoadBalancer     │ Cloud LB. Production. Para ödersin.           │
├──────────────────┼───────────────────────────────────────────────┤
│ ExternalName     │ Dış servise DNS alias. Migration/Hybrid.      │
└──────────────────┴───────────────────────────────────────────────┘
```

---

<a id="configmaps-secrets"></a>
## 🔧 ConfigMaps ve Secrets

### Problem: Config'i Kod'dan Ayır

12-Factor App prensibinin en önemli kurallarından biri: **Konfigürasyonu kod'dan ayır.**

Neden?
- Aynı image, farklı ortamlarda (dev/staging/prod) farklı config ile çalışır
- Config değiştiğinde yeniden build etmek istemezsin
- Şifreler kod repo'suna girmemeli

Kubernetes bunu iki nesneyle çözer:
- **ConfigMap:** Hassas olmayan config verisi
- **Secret:** Hassas veri (şifre, API key, sertifika)

---

### ConfigMap

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: uygulama-config
data:
  # Key-value çiftleri
  VERITABANI_HOST: "postgres-service"
  VERITABANI_PORT: "5432"
  LOG_SEVIYESI: "INFO"
  CACHE_TTL: "300"

  # Büyük config dosyası da ekleyebilirsin
  app.properties: |
    max.connections=100
    timeout.seconds=30
    feature.dark-mode=true

  nginx.conf: |
    server {
      listen 80;
      location / {
        proxy_pass http://backend;
      }
    }
```

```bash
# Oluştur
kubectl apply -f configmap.yaml

# Görüntüle
kubectl get configmap uygulama-config
kubectl describe configmap uygulama-config

# İçeriği göster
kubectl get configmap uygulama-config -o yaml

# Hızlı oluştur (YAML olmadan)
kubectl create configmap hizli-config \
  --from-literal=DB_HOST=localhost \
  --from-literal=PORT=5432

# Dosyadan oluştur
kubectl create configmap nginx-config --from-file=nginx.conf
```

---

<a id="env-vars"></a>
### 📌 Environment Variables Olarak Kullanım

```yaml
# deployment-with-config.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: uygulama
        image: benim-uygulama:1.0

        # Yöntem 1: Tek tek seç
        env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: uygulama-config
              key: VERITABANI_HOST
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: uygulama-config
              key: VERITABANI_PORT

        # Yöntem 2: Hepsini birden al (ConfigMap'teki tüm key'ler env olur)
        envFrom:
        - configMapRef:
            name: uygulama-config
```

---

<a id="volume-mounts"></a>
### 📂 Volume Mounts Olarak Kullanım

Özellikle config dosyaları için kullanılır:

```yaml
spec:
  volumes:
  # ConfigMap'i volume olarak tanımla
  - name: nginx-config-vol
    configMap:
      name: uygulama-config
      items:
      - key: nginx.conf          # ConfigMap'teki key
        path: nginx.conf         # Container içindeki dosya adı

  containers:
  - name: nginx
    image: nginx:1.25
    volumeMounts:
    - name: nginx-config-vol
      mountPath: /etc/nginx/conf.d  # Bu klasöre mount et
      readOnly: true
```

> **Süper Özellik:** ConfigMap'i volume olarak mount ettiğinde, ConfigMap değiştirilirse Kubernetes otomatik olarak dosyayı günceller — Pod yeniden başlamadan! (birkaç dakika gecikme olabilir). Bu özellikle nginx config için çok kullanışlı.

---

### Secret

Secret, ConfigMap'e benzer ama hassas veriler için. Kubernetes etcd'de base64 ile encode eder (dikkat: bu şifreleme değil, sadece encoding!).

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: veritabani-sifresi
type: Opaque
data:
  # base64 encode edilmiş değerler
  # echo -n "gizli123" | base64  →  Z2l6bGkxMjM=
  VERITABANI_SIFRE: Z2l6bGkxMjM=
  API_KEY: c3VwZXJnaXpsaWFwaWtleQ==
```

```bash
# Hızlı oluştur (base64'ü Kubernetes halleder)
kubectl create secret generic veritabani-sifresi \
  --from-literal=VERITABANI_SIFRE=gizli123 \
  --from-literal=API_KEY=supergizliapikey

# Değerleri gör (base64 decode edilmiş)
kubectl get secret veritabani-sifresi \
  -o jsonpath='{.data.VERITABANI_SIFRE}' | base64 -d
```

#### Secret'ı Deployment'ta Kullan

```yaml
spec:
  containers:
  - name: uygulama
    image: benim-uygulama:1.0

    # Environment variable olarak
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: veritabani-sifresi
          key: VERITABANI_SIFRE

    # Veya envFrom ile hepsini al
    envFrom:
    - secretRef:
        name: veritabani-sifresi

    # Veya volume olarak mount et (daha güvenli)
    volumeMounts:
    - name: secret-vol
      mountPath: /app/secrets
      readOnly: true

  volumes:
  - name: secret-vol
    secret:
      secretName: veritabani-sifresi
      defaultMode: 0400  # Sadece okunabilir, sadece owner
```

> **Güvenlik Notu:** Volume mount yöntemi environment variable'dan daha güvenli. Environment variable'lar process listesinde görünebilir, log'lara sızabilir. Volume'daki dosyalar daha kontrollü. Production'da Vault, AWS Secrets Manager gibi harici secret yönetim araçları entegre et — bu konuya ilerleyen haftalarda değineceğiz.

---

<a id="secret-types"></a>
### 🔐 Secret Types

Kubernetes'te farklı amaçlar için farklı Secret type'ları var:

| Type | Kullanım |
|------|----------|
| `Opaque` | Default. Her türlü key-value. |
| `kubernetes.io/service-account-token` | Service account token'ı |
| `kubernetes.io/dockerconfigjson` | Docker registry kimlik bilgisi |
| `kubernetes.io/tls` | TLS sertifika ve private key |
| `kubernetes.io/ssh-auth` | SSH kimlik doğrulama |
| `kubernetes.io/basic-auth` | Kullanıcı adı / şifre |

#### Docker Registry Secret (Özel Registry'den Image Çekmek İçin)

```bash
# Private registry için secret oluştur
kubectl create secret docker-registry benim-registry-sirrim \
  --docker-server=registry.sirketim.com \
  --docker-username=kullanici \
  --docker-password=sifre123 \
  --docker-email=kullanici@sirket.com
```

```yaml
# Deployment'ta kullan
spec:
  imagePullSecrets:
  - name: benim-registry-sirrim
  containers:
  - name: ozel-uygulama
    image: registry.sirketim.com/ozel-uygulama:1.0
```

#### TLS Secret

```bash
# Sertifikadan oluştur
kubectl create secret tls web-tls-sirri \
  --cert=sertifika.crt \
  --key=ozel-anahtar.key
```

---

## 🏗️ Hepsini Birleştiren Tam Örnek

Gerçek dünya senaryosu: Bir web uygulaması deployment'ı — ConfigMap, Secret, Init Container, Deployment ve Service hepsi bir arada.

```yaml
# ── 1. ConfigMap ──────────────────────────────────────
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config
  namespace: uygulama
data:
  DB_HOST: "postgres-service"
  DB_PORT: "5432"
  LOG_LEVEL: "INFO"
---
# ── 2. Secret ─────────────────────────────────────────
apiVersion: v1
kind: Secret
metadata:
  name: web-secrets
  namespace: uygulama
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQxMjM=    # password123
---
# ── 3. Deployment ─────────────────────────────────────
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deployment
  namespace: uygulama
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: web
    spec:
      initContainers:
      - name: db-bekle
        image: busybox:1.36
        command: ['sh', '-c',
          'until nc -z postgres-service 5432; do echo "DB bekleniyor..."; sleep 1; done']

      containers:
      - name: web
        image: benim-web:2.0
        ports:
        - containerPort: 8080
        envFrom:
        - configMapRef:
            name: web-config
        - secretRef:
            name: web-secrets
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 15
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
# ── 4. Service ────────────────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: uygulama
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

```bash
# Namespace oluştur
kubectl create namespace uygulama

# Hepsini uygula
kubectl apply -f tam-ornek.yaml

# Durumu izle
kubectl get all -n uygulama

# Deployment durumunu takip et
kubectl rollout status deployment/web-deployment -n uygulama

# Test et
kubectl port-forward service/web-service 8080:80 -n uygulama
# Tarayıcıdan: http://localhost:8080
```

---

<a id="pratik-projeler"></a>
## 🛠️ Pratik Projeler

### Proje 1: Stateless Web Uygulaması

**Hedef:** Nginx + custom ConfigMap + ClusterIP Service

1. ConfigMap ile özel `nginx.conf` oluştur
2. Nginx Deployment (3 replica) oluştur, ConfigMap'i volume mount et
3. ClusterIP Service oluştur
4. Port-forward ile test et: `kubectl port-forward service/nginx-service 8080:80`

### Proje 2: Rolling Update Pratiği

**Hedef:** Güncelleme ve rollback deneyimle

1. `nginx:1.24` ile Deployment oluştur
2. HPA ekle (min 2, max 5, CPU %70)
3. `nginx:1.25`'e güncelle ve `kubectl rollout status` ile izle
4. Kasıtlı hatalı image ver: `nginx:var-olmayan-tag`
5. `kubectl rollout undo` ile geri dön

### Proje 3: Tam Stack Uygulama

**Hedef:** Frontend + Backend + Secret yönetimi

1. Backend API Deployment (env'den DB bağlantısı alıyor)
2. Secret ile DB şifresi
3. ConfigMap ile API endpoint'leri
4. ClusterIP Service (backend için)
5. NodePort Service (frontend için, tarayıcıdan test)

---

## 📚 Yararlı kubectl Komutları — Özet Tablo

```bash
# Pod işlemleri
kubectl get pods                          # Pod listesi
kubectl get pods -o wide                  # IP ve Node bilgisiyle
kubectl describe pod <pod-adı>            # Detaylı bilgi + Events
kubectl logs <pod-adı>                    # Son loglar
kubectl logs <pod-adı> --follow           # Canlı log
kubectl logs <pod-adı> -c <container>     # Multi-container log
kubectl exec -it <pod-adı> -- /bin/bash   # Pod içine gir
kubectl delete pod <pod-adı>              # Pod sil (RS yenisini yaratır)

# Deployment işlemleri
kubectl get deployments
kubectl rollout status deployment/<isim>
kubectl rollout history deployment/<isim>
kubectl rollout undo deployment/<isim>
kubectl scale deployment/<isim> --replicas=5
kubectl set image deployment/<isim> <container>=<yeni-image>

# Service işlemleri
kubectl get services
kubectl describe service <isim>
kubectl port-forward service/<isim> <local-port>:<service-port>

# ConfigMap / Secret işlemleri
kubectl get configmaps
kubectl get secrets
kubectl describe configmap <isim>
kubectl edit configmap <isim>             # Canlı düzenle

# Genel
kubectl get all                           # Her şeyi göster
kubectl get all -n <namespace>            # Namespace bazlı
kubectl apply -f <dosya.yaml>             # Uygula
kubectl delete -f <dosya.yaml>            # Sil
kubectl explain deployment.spec           # API açıklaması
```

---

## ✅ Bu Haftanın Kontrol Listesi

Bu haftayı bitirdiğinde şunları yapabiliyor olman gerek:

- [ ] Pod lifecycle'ı anlatabilmek ve probe'ları konfigüre edebilmek
- [ ] Multi-container pod yazabilmek (sidecar pattern)
- [ ] Init container ile bağımlılık yönetimi yapabilmek
- [ ] Deployment ile Rolling Update ve Rollback yapabilmek
- [ ] HPA ile otomatik scaling ayarlayabilmek
- [ ] 4 Service tipini ayırt edebilmek ve doğru tip seçebilmek
- [ ] ConfigMap ve Secret ile config yönetimi yapabilmek
- [ ] Hem env variable hem de volume mount yöntemini uygulayabilmek

Hafta 7-8'de bu temeller üzerine **Ingress, Storage (PV/PVC), StatefulSets** konularını öğreneceğiz. O hafta bu haftaki bilgileri hazır varsayacağız.

---

> **Son Söz:** Kubernetes'te her nesne bir amaca hizmet eder. Pod, ReplicaSet, Deployment — bunlar tesadüfen katmanlı değil. Her biri bir öncekinin eksikliğini gidermek için var. Bunu anlarsan, yeni bir Kubernetes nesnesiyle karşılaştığında "bu neden var, hangi problemi çözüyor?" diye sorarsın ve hızla öğrenirsin.
