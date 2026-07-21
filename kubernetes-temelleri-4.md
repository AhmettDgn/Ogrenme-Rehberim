# 🚀 Kubernetes Temelleri — Hafta 3-4

> "Kubernetes'i anlamak, modern yazılım mühendisliğinin en değerli becerilerinden birini kazanmaktır."

10 yıldır bu sektördeyim ve şunu net söyleyebilirim: Kubernetes öğrenmek zor değil, **doğru sırayla** öğrenmek zor. Çoğu insan dökümantasyona dalıyor, 500 sayfalık kavramlar arasında kayboluyor ve "bu benim için değil" diyerek bırakıyor. Sen öyle yapmayacaksın.

Bu rehber boyunca seninle konuşacağım — sanki yan yana oturuyormuşuz gibi. Deneyimlerimden, hatalarımdan ve "ah be, keşke bunu baştan biri söyleseydi" dediğim anlardan bahsedeceğim.

Hazırsan başlayalım.

---

## 📋 İçindekiler

- [Kubernetes Nedir ve Neden Kullanılır?](#kubernetes-nedir)
- [Kubernetes Mimarisi](#kubernetes-mimarisi)
  - [Control Plane](#control-plane)
  - [Worker Nodes](#worker-nodes)
  - [Bileşenler Arası İletişim](#iletisim)
- [kubectl Kurulumu ve Temel Komutlar](#kubectl)
- [Local Cluster: Minikube ve Kind](#local-cluster)
- [Namespaces](#namespaces)
- [Pratik Projeler](#pratik-projeler)
- [Kaynaklar](#kaynaklar)

---

<a id="kubernetes-nedir"></a>
## 🎯 Kubernetes Nedir ve Neden Kullanılır?

### Problemi Anlayalım Önce

Docker öğrendiğinde "harika, artık uygulamalarımı container içinde çalıştırabiliyorum" dedin. Ama şimdi şu soruları sor kendine:

- 50 tane container'ı nasıl yöneteceksin?
- Bir container çökerse kim onu yeniden başlatacak?
- Trafik artınca nasıl ölçeklendireceksin?
- Güncelleme yaparken kullanıcıların hizmet alması nasıl kesintisiz devam edecek?
- Hangi container hangi sunucuda çalışacak? Kim karar verecek?

Bu sorular "Container Orchestration" problemini tanımlar. Ve Kubernetes bu problemi çözmek için 2014'te Google tarafından tasarlanmış, bugün CNCF bünyesinde geliştirilen açık kaynaklı bir sistemdir.

> **Gerçek Hayat Anekdotu:** Google, Kubernetes'i kendi iç sistemi olan Borg'dan yola çıkarak geliştirdi. Google haftada milyarlarca container başlatıyor. Kubernetes'in temellerini, bu ölçekte sistemi yönetme deneyimi olan mühendisler attı.

### Orkestra Şefi Analojisi

Kubernetes'i bir **orkestra şefi** olarak düşün.

- **Müzisyenler** = Container'larının
- **Partisyon** = Senin tanımladığın YAML dosyaları (istenen durum)
- **Şef** = Kubernetes

Sen şefe "violinler şu partisyonu çalsın, 3 tane violin olsun, biri hastalanırsa yerine başkasını getir" diyorsun. Şef bunu organize ediyor. Bir violinciye bir şey olursa şef hemen yerine başkasını atıyor. Sen sahnede ne olduğuyla tek tek ilgilenmiyorsun.

### Kubernetes'in Çözdüğü 5 Temel Problem

| Problem | Kubernetes Çözümü |
|---------|-------------------|
| Container çöküyor | Self-healing: Otomatik yeniden başlatma |
| Trafik artıyor | Auto-scaling: Pod sayısını otomatik artırma |
| Güncelleme yapmak istiyorum | Rolling update: Kesintisiz deployment |
| Container'lar birbirini bulamıyor | Service discovery: DNS tabanlı iletişim |
| Kaynakları verimli kullanmak | Bin packing: Akıllı kaynak planlaması |

### Kubernetes'in Yapmadığı Şeyler

Dikkat: Kubernetes her şeyi çözmez. O bir **platform**dur, uygulama değil.

- Kod yazmaz
- CI/CD pipeline'ı değildir (ama onunla çalışır)
- Monitoring aracı değildir (ama Prometheus gibi araçları barındırır)
- Veritabanı yönetim aracı değildir (stateful uygulamalar için ek dikkat gerekir)

---

<a id="kubernetes-mimarisi"></a>
## ⚙️ Kubernetes Mimarisi

Şimdi buraya dikkat et. Mimariye bakınca kafan karışabilir çünkü çok fazla bileşen var. Ama şu şekilde düşünürsen her şey yerine oturur:

**Kubernetes bir "cluster"dır** — birden fazla makine bir arada çalışır. Bu makinelerin bir kısmı **yönetici** (Control Plane), diğerleri **işçi** (Worker Node) rolündedir.

```
┌─────────────────────────────────────────────────────────────────┐
│                        KUBERNETES CLUSTER                        │
│                                                                   │
│  ┌──────────────────────────────────┐                           │
│  │         CONTROL PLANE            │                           │
│  │                                  │                           │
│  │  ┌──────────┐  ┌──────────────┐ │                           │
│  │  │API Server│  │    etcd      │ │                           │
│  │  └──────────┘  └──────────────┘ │                           │
│  │  ┌──────────┐  ┌──────────────┐ │                           │
│  │  │Scheduler │  │  Controller  │ │                           │
│  │  │          │  │  Manager     │ │                           │
│  │  └──────────┘  └──────────────┘ │                           │
│  └────────────────┬─────────────────┘                           │
│                   │                                              │
│      ┌────────────┼────────────┐                                │
│      │            │            │                                │
│  ┌───▼──────┐ ┌───▼──────┐ ┌───▼──────┐                       │
│  │  Worker  │ │  Worker  │ │  Worker  │                       │
│  │  Node 1  │ │  Node 2  │ │  Node 3  │                       │
│  │          │ │          │ │          │                       │
│  │ kubelet  │ │ kubelet  │ │ kubelet  │                       │
│  │kube-proxy│ │kube-proxy│ │kube-proxy│                       │
│  │containerd│ │containerd│ │containerd│                       │
│  │  [Pod]   │ │  [Pod]   │ │  [Pod]   │                       │
│  └──────────┘ └──────────┘ └──────────┘                       │
└─────────────────────────────────────────────────────────────────┘
```

---

<a id="control-plane"></a>
### 🧠 Control Plane (Beyin)

Control Plane cluster'ın beynidir. Tüm kararlar burada alınır. Production ortamında Control Plane bileşenleri yüksek erişilebilirlik için genellikle 3 veya 5 ayrı makinede çalışır.

#### 1. API Server — Tek Giriş Noktası

```
kubectl → API Server → etcd
```

API Server, Kubernetes'in **tek giriş kapısıdır**. Sen kubectl ile bir komut çalıştırdığında, bir uygulama Kubernetes API'sini kullandığında, her şey API Server üzerinden geçer.

Belediye başkanlığı olarak düşün: Şehirde ne yapılacaksa belediyeye bildirilir. API Server da böyle — kimse etcd'ye doğrudan yazamaz, kimse scheduler'a doğrudan emir veremez. Her şey API Server'dan geçer.

**Önemli özellikler:**
- Authentication ve Authorization burada yapılır
- Tüm operasyonları kaydeder (audit log)
- RESTful API sağlar
- Watch mekanizmasıyla bileşenlere değişiklikleri bildirir

#### 2. etcd — Hafıza

etcd, Kubernetes'in **tüm durumunu saklayan** dağıtık key-value store'dur. Cluster'ında kaç Pod var, hangi Node'da ne çalışıyor, hangi Service hangi Pod'lara bakıyor — hepsi etcd'de.

> **Dikkat:** etcd'yi direkt değiştirme! Bu hatayı yapan mühendisler gördüm. Her zaman kubectl veya Kubernetes API üzerinden işlem yap. etcd'ye direkt erişim sadece disaster recovery senaryolarında düşünülmeli.

**etcd hakkında bilmen gerekenler:**
- Raft consensus algoritması kullanır
- Kaybedersen cluster'ını kaybedersin → **Yedeklemeyi unutma!**
- etcd cluster'ı için tek sayı (1, 3, 5) tercih et

#### 3. Scheduler — Karar Verici

Yeni bir Pod oluşturulduğunda "bu Pod hangi Node'da çalışsın?" sorusunun cevabını Scheduler verir.

Scheduler'ın değerlendirdiği kriterler:
- Node'un yeterli CPU/Memory'si var mı?
- Pod'un özel Node gereksinimleri var mı? (nodeSelector, affinity)
- Taint/Toleration uyuşuyor mu?
- Mevcut Pod dağılımı nasıl? (Pod anti-affinity)

Scheduler kararı etcd'ye yazar, oradan kubelet bilgilendirilir ve Pod o Node'da başlar.

#### 4. Controller Manager — Durumu İzleyen

Controller Manager, "istenen durum" ile "gerçek durum" arasındaki farkı sürekli izleyen ve kapatan bileşendir.

İçinde birçok controller barınır:
- **ReplicaSet Controller:** "3 Pod çalışmalı" dersen ve 2 tane varsa, 1 tane daha başlatır
- **Deployment Controller:** Rolling update'leri yönetir
- **Node Controller:** Node'ların sağlığını izler
- **Service Account Controller:** Hesap yönetimi

> **Mühendislik kafasıyla düşünelim:** Controller Manager "reconciliation loop" çalıştırır. Sürekli `istenen_durum == gerçek_durum?` diye kontrol eder. Farklıysa farkı kapatır. Bu pattern Kubernetes'in tüm tasarımında var ve siz de kendi operatörlerinizi yazarken bu pattern'ı kullanırsınız.

---

<a id="worker-nodes"></a>
### 💪 Worker Nodes (İşçiler)

Worker Node'lar gerçek iş yüklerinin — yani Pod'larının — çalıştığı makinelerdir.

#### 1. kubelet — Node'un Ajanı

kubelet, her Worker Node'da çalışan ve o Node'u Control Plane'e bağlayan ajandır.

Sorumlulukları:
- API Server'dan kendisine atanan Pod'ları alır
- Container Runtime'a (containerd) Pod'ları başlatmasını söyler
- Pod'ların sağlığını izler, raporlar
- Liveness/Readiness probe'larını çalıştırır

#### 2. kube-proxy — Ağ Sihirbazı

kube-proxy, her Node'da çalışan ve Kubernetes Service'lerinin ağ kurallarını uygulayan bileşendir. Service'e gelen trafiği doğru Pod'a yönlendirir.

> **Gerçek Hayat Notu:** Modern cluster'larda kube-proxy'nin yerini Cilium gibi eBPF tabanlı CNI eklentileri alıyor. Ama temeli anlamak için kube-proxy'yi bilmek şart.

#### 3. Container Runtime — Çalıştırıcı

Container Runtime Interface (CRI) standardını uygulayan yazılım — container'ları gerçekten başlatan, durduran, yöneten katman.

- **containerd** (en yaygın, Kubernetes'in önerdiği)
- **CRI-O** (Red Hat ekosisteminde tercih edilir)
- ~~Docker~~ (Kubernetes 1.24'ten itibaren kaldırıldı — ama containerd hâlâ containerd üzerinde çalışır)

> **Sık Sorulan Soru:** "Docker kaldırıldıysa Docker image'larım çalışmaz mı?" Çalışır! Docker image formatı (OCI) standardı hâlâ geçerli. Sadece Docker'ın Kubernetes ile doğrudan entegrasyonu kalktı, yerini containerd aldı.

---

<a id="iletisim"></a>
### 🔄 Bileşenler Arası İletişim — Bir Pod'un Yolculuğu

`kubectl apply -f deployment.yaml` yazdığında ne olur? Adım adım izleyelim:

```
1. kubectl → API Server'a HTTP POST isteği gönderir
              (authentication, authorization, admission control)

2. API Server → Deployment nesnesini etcd'ye yazar

3. Deployment Controller (Controller Manager içinde)
   → etcd'yi watch eder, yeni Deployment'ı görür
   → ReplicaSet oluşturur
   → API Server üzerinden etcd'ye yazar

4. ReplicaSet Controller
   → ReplicaSet'i görür
   → Pod nesnelerini oluşturur (henüz Node atanmamış)
   → etcd'ye yazar

5. Scheduler
   → Node atanmamış Pod'ları watch eder
   → Uygun Node'u seçer
   → Pod'a Node atar, etcd'ye yazar

6. kubelet (seçilen Node'da)
   → Kendine atanan Pod'ları watch eder
   → containerd'ye Pod'u başlatmasını söyler
   → Pod çalışmaya başlar
   → Durumu API Server'a bildirir

7. API Server → etcd'ye Pod'un Running durumunu yazar
```

Bu akışı anlamak, sorun giderme sırasında çok değerli. Bir Pod başlamıyorsa hangi adımda takıldığını görebilirsin.

---

<a id="kubectl"></a>
## 🛠️ kubectl — Kubernetes'in Komut Satırı

kubectl (okunuşu: "cube-ctl" veya "kube-control") Kubernetes cluster'ını yönetmek için kullandığın komut satırı aracıdır.

### Kurulum

**Linux / macOS:**
```bash
# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# macOS (Homebrew)
brew install kubectl

# Versiyonu kontrol et
kubectl version --client
```

**Windows:**
```powershell
# winget ile
winget install Kubernetes.kubectl

# Ya da Chocolatey
choco install kubernetes-cli
```

### Konfigürasyon: kubeconfig

kubectl hangi cluster'a bağlanacağını `~/.kube/config` dosyasından öğrenir. Bu dosyaya kubeconfig denir.

```bash
# Mevcut context'i gör
kubectl config current-context

# Tüm context'leri listele
kubectl config get-contexts

# Context değiştir
kubectl config use-context minikube

# Kubeconfig dosyasını görüntüle
kubectl config view
```

> **İpucu:** Birden fazla cluster'ı yönetiyorsan kubectx ve kubens araçlarını yükle. Hayatını kolaylaştırır.

---

### 📋 kubectl Cheat Sheet

#### Temel Kaynak Komutları

```bash
# Kaynakları listele
kubectl get pods
kubectl get pods -o wide          # Node bilgisiyle
kubectl get pods -A               # Tüm namespace'lerde
kubectl get pods --watch          # Gerçek zamanlı izle

kubectl get nodes
kubectl get services
kubectl get deployments
kubectl get namespaces
kubectl get all                   # Her şeyi göster

# Detaylı bilgi
kubectl describe pod <pod-adı>
kubectl describe node <node-adı>
kubectl describe service <servis-adı>

# YAML çıktısı al
kubectl get pod <pod-adı> -o yaml
kubectl get deployment <deployment-adı> -o json
```

#### Kaynak Oluşturma ve Silme

```bash
# YAML dosyasından uygula (oluştur veya güncelle)
kubectl apply -f deployment.yaml
kubectl apply -f ./manifests/     # Bir klasördeki tüm YAML'ları uygula

# Sil
kubectl delete pod <pod-adı>
kubectl delete -f deployment.yaml
kubectl delete deployment <deployment-adı>

# Hızlı Pod oluştur (test için)
kubectl run test-pod --image=nginx --restart=Never
```

#### Loglara ve Container'a Erişim

```bash
# Pod logları
kubectl logs <pod-adı>
kubectl logs <pod-adı> -f              # Canlı takip (tail -f gibi)
kubectl logs <pod-adı> --previous      # Önceki container'ın logları (çökmüş pod için)
kubectl logs <pod-adı> -c <container>  # Multi-container pod'da belirli container

# Container içine gir
kubectl exec -it <pod-adı> -- bash
kubectl exec -it <pod-adı> -- sh       # bash yoksa
kubectl exec -it <pod-adı> -c <container> -- bash  # Multi-container

# Tek komut çalıştır
kubectl exec <pod-adı> -- ls /app
```

#### Port Forwarding (Test İçin Çok Kullanışlı)

```bash
# Local port'u pod'a yönlendir
kubectl port-forward pod/<pod-adı> 8080:80
kubectl port-forward service/<servis-adı> 8080:80
kubectl port-forward deployment/<deployment-adı> 8080:80
```

#### Düzenleme ve Güncelleme

```bash
# Canlı olarak düzenle
kubectl edit deployment <deployment-adı>

# Scale etme
kubectl scale deployment <deployment-adı> --replicas=5

# Image güncelle
kubectl set image deployment/<deployment-adı> <container-adı>=<yeni-image>

# Rollout yönetimi
kubectl rollout status deployment/<deployment-adı>
kubectl rollout history deployment/<deployment-adı>
kubectl rollout undo deployment/<deployment-adı>
kubectl rollout undo deployment/<deployment-adı> --to-revision=2
```

#### Kaynak Kullanımı

```bash
# Node ve Pod kaynak kullanımı (metrics-server gerektirir)
kubectl top nodes
kubectl top pods
kubectl top pods -A
```

#### Faydalı Kısayollar

```bash
# Kısa isimler
kubectl get po          # pods
kubectl get svc         # services
kubectl get deploy      # deployments
kubectl get ns          # namespaces
kubectl get ing         # ingresses
kubectl get pv          # persistentvolumes
kubectl get pvc         # persistentvolumeclaims
kubectl get cm          # configmaps
kubectl get secret

# Namespace belirterek
kubectl get pods -n kube-system
kubectl get all -n my-namespace

# Label ile filtrele
kubectl get pods -l app=nginx
kubectl get pods -l environment=production,tier=frontend
```

---

<a id="local-cluster"></a>
## 🖥️ Local Cluster: Minikube ve Kind

Production cluster'ına geçmeden önce local'de pratik yapmak şart. Bunun için iki popüler araç var: **Minikube** ve **Kind**.

### Minikube vs Kind: Hangisini Seçmeli?

| Özellik | Minikube | Kind |
|---------|----------|------|
| Kurulum kolaylığı | Kolay | Kolay |
| Multi-node cluster | Evet | Evet |
| Hız | Orta | Hızlı |
| Kaynak kullanımı | Daha fazla | Az |
| Add-on desteği | Zengin | Sınırlı |
| CI/CD entegrasyonu | Orta | Mükemmel |
| Driver seçeneği | Çok (VM, Docker, bare-metal) | Sadece Docker |
| Kubernetes versiyonu seçimi | Evet | Evet |

**Ne zaman Minikube?**
- İlk defa Kubernetes öğreniyorsan
- Dashboard ve add-on'ları kolayca denemek istiyorsan
- LoadBalancer tipinde Service'leri test etmek istiyorsan

**Ne zaman Kind?**
- CI/CD pipeline'larında
- Çok hızlı cluster oluşturman gerekiyorsa
- Multi-node senaryolar test etmek istiyorsan
- Docker zaten kuruluysa ve hafif bir şey istiyorsan

---

### 🔵 Minikube

#### Kurulum

```bash
# Linux
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# macOS
brew install minikube

# Windows (winget)
winget install Kubernetes.minikube
```

#### Temel Kullanım

```bash
# Cluster başlat (Docker driver ile — en kolay)
minikube start --driver=docker

# Belirli Kubernetes versiyonuyla başlat
minikube start --kubernetes-version=v1.28.0

# Kaynakları belirle
minikube start --cpus=4 --memory=8192

# Cluster durumu
minikube status

# Dashboard aç (browser'da Kubernetes UI)
minikube dashboard

# Cluster'ı durdur
minikube stop

# Cluster'ı sil
minikube delete
```

#### Faydalı Eklentiler

```bash
# Mevcut eklentileri listele
minikube addons list

# Metrics Server aç (kubectl top için şart)
minikube addons enable metrics-server

# Ingress Controller
minikube addons enable ingress

# Storage provisioner
minikube addons enable storage-provisioner
```

#### İlk Pod'unu Minikube'de Çalıştır

```bash
# Cluster'ı başlat
minikube start

# Bir nginx pod'u çalıştır
kubectl run ilk-pod --image=nginx

# Pod'un çalışmasını bekle
kubectl get pods --watch

# Pod'a eriş
kubectl port-forward pod/ilk-pod 8080:80
# Şimdi tarayıcıda http://localhost:8080 aç!

# Temizle
kubectl delete pod ilk-pod
```

---

### 🟡 Kind (Kubernetes in Docker)

Kind, Kubernetes node'larını Docker container'ı olarak çalıştırır. Bu yüzden çok hızlıdır.

#### Kurulum

```bash
# Linux / macOS
# Go ile
go install sigs.k8s.io/kind@latest

# macOS Homebrew
brew install kind

# Linux binary
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Windows
choco install kind
# ya da winget
winget install Kubernetes.kind
```

#### Temel Kullanım

```bash
# Basit cluster oluştur
kind create cluster

# İsim ver
kind create cluster --name ogrenme-cluster

# Cluster listele
kind get clusters

# Cluster sil
kind delete cluster --name ogrenme-cluster
```

#### Multi-node Cluster (Gerçek Senaryolar İçin)

Kind ile 1 control plane + 3 worker node cluster'ı şöyle kurarsın:

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
```

```bash
kind create cluster --name multi-node --config kind-config.yaml

# Kontrol et
kubectl get nodes
```

---

<a id="namespaces"></a>
## 📦 Namespaces

### Namespace Nedir?

Namespace, tek bir Kubernetes cluster'ını birden fazla sanal cluster'a bölme mekanizmasıdır. Mantıksal bir izolasyon sağlar.

Somut örnek: Aynı cluster'da hem `development` hem `staging` hem `production` ortamlarını çalıştırmak istiyorsun. Namespace'ler bu ortamları birbirinden izole eder.

```
cluster
├── namespace: development
│   ├── deployment: my-app (image: my-app:dev)
│   └── service: my-app
├── namespace: staging
│   ├── deployment: my-app (image: my-app:v1.5)
│   └── service: my-app
└── namespace: production
    ├── deployment: my-app (image: my-app:v1.4)
    └── service: my-app
```

Her namespace'de aynı isimde kaynak olabilir — çakışma olmaz.

### Default Namespace'ler

Yeni bir cluster'da şu namespace'ler gelir:

```bash
kubectl get namespaces
```

| Namespace | Amacı |
|-----------|-------|
| `default` | Namespace belirtmeden oluşturulan kaynaklar buraya gider |
| `kube-system` | Kubernetes sistem bileşenleri (API Server, DNS, vb.) |
| `kube-public` | Herkese açık kaynaklar (genelde boş) |
| `kube-node-lease` | Node heartbeat nesneleri |

> **Dikkat:** `kube-system` içinde hiçbir şeyi silme veya değiştirme! Yüksek ihtimalle cluster'ını kırarsın.

### Namespace Oluşturma

```bash
# Komutla oluştur
kubectl create namespace development
kubectl create namespace staging
kubectl create namespace production

# YAML ile oluştur (tavsiye edilen yöntem)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: development
  labels:
    environment: dev
    team: backend
EOF
```

### Namespace ile Kaynak Yönetimi

```bash
# Belirli namespace'de kaynak oluştur
kubectl apply -f deployment.yaml -n development

# Belirli namespace'deki kaynakları listele
kubectl get pods -n development
kubectl get all -n development

# Tüm namespace'lerde listele
kubectl get pods --all-namespaces
kubectl get pods -A

# Default namespace'i değiştir (her komutta -n yazmaktan kurtar)
kubectl config set-context --current --namespace=development

# Şu anki namespace'i kontrol et
kubectl config view --minify | grep namespace
```

### Resource Quota — Namespace'e Kaynak Sınırı

Gerçek hayatta namespace'ler arasında kaynak sınırı koymak istersin. Örneğin development ekibi cluster'ın hepsini tüketmesin.

```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: development-quota
  namespace: development
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 4Gi
    limits.cpu: "8"
    limits.memory: 8Gi
    pods: "20"
    services: "10"
```

```bash
kubectl apply -f resource-quota.yaml

# Quota kullanımını gör
kubectl describe resourcequota -n development
```

### LimitRange — Container Başına Varsayılan Sınırlar

```yaml
# limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: development
spec:
  limits:
  - default:
      memory: 512Mi
      cpu: 500m
    defaultRequest:
      memory: 256Mi
      cpu: 250m
    type: Container
```

Bu sayede namespace içinde Pod oluştururken limit belirtmeyi unutsan bile otomatik varsayılan değerler atanır.

---

<a id="pratik-projeler"></a>
## 💻 Pratik Projeler

Şimdi burada dur. Sadece okumak yetmez. Her bir projeyi gerçekten yap. Hata yapacaksın — bu normaldir. Ben de yaptım. Hatalar öğretir.

### Proje 1: İlk Cluster'ını Kur ve Keşfet

```bash
# Minikube başlat
minikube start

# Cluster bilgilerini incele
kubectl cluster-info
kubectl get nodes
kubectl get nodes -o wide

# Sistem pod'larına bak
kubectl get pods -n kube-system

# Her bir pod'u describe et ve ne olduklarını anlamaya çalış
kubectl describe pod -n kube-system coredns-<hash>
```

**Görev:** kube-system namespace'indeki her pod'un ne iş yaptığını araştır ve bir not defterine yaz.

---

### Proje 2: kubectl ile Cluster'ı Keşfet

```bash
# İlk pod'unu çalıştır
kubectl run nginx-test --image=nginx

# Pod'u izle
kubectl get pods --watch

# Pod detaylarına bak
kubectl describe pod nginx-test

# Loglara bak
kubectl logs nginx-test

# Pod içine gir
kubectl exec -it nginx-test -- bash

# İçeriden çıkmak için
exit

# Temizle
kubectl delete pod nginx-test
```

**Görev:** nginx yerine `busybox` image'ını kullan. `kubectl run busybox-test --image=busybox --restart=Never -- sleep 3600` komutuyla başlat ve içine girerek birkaç Linux komutu dene.

---

### Proje 3: Namespace'ler Oluştur ve Yönet

```bash
# 3 namespace oluştur
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace prod

# Her namespace'de farklı bir pod çalıştır
kubectl run dev-pod --image=nginx -n dev
kubectl run staging-pod --image=httpd -n staging
kubectl run prod-pod --image=nginx:alpine -n prod

# Her namespace'i ayrı ayrı kontrol et
kubectl get pods -n dev
kubectl get pods -n staging
kubectl get pods -n prod

# Hepsine bir bakış at
kubectl get pods -A
```

**Görev:** `dev` namespace'ine ResourceQuota ekle (maksimum 5 pod, 2 CPU, 2Gi memory). Sonra 6. pod'u oluşturmayı dene ve ne olduğunu gözlemle.

---

### Proje 4: kubectl Cheat Sheet'ini Kendi Yaz

Yukarıdaki cheat sheet'e bakarak **kendi kelimelerinle** ve **kendi deneyimlediğin örneklerle** bir cheat sheet oluştur. Dosya adı: `kubectl-notlarim.md`

Mutlaka şunları ekle:
- En çok kullanacağın komutlar
- Seni şaşırtan veya ezberlemeye değer komutlar
- Kendi örneklerin

**Neden?** Başkasının notlarını okumak ile kendi notlarını yazmak arasında öğrenme açısından dağlar kadar fark var.

---

### Proje 5: Kind ile Multi-Node Cluster

```bash
# kind-cluster.yaml oluştur
cat > kind-cluster.yaml <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF

# Cluster oluştur
kind create cluster --name multi-node --config kind-cluster.yaml

# Node'ları gör
kubectl get nodes

# 3 replikalı bir deployment çalıştır
kubectl create deployment web --image=nginx --replicas=3

# Pod'ların farklı node'lara dağıldığını gözlemle
kubectl get pods -o wide

# Bir worker node'u "bozuk" simüle et
kubectl cordon <worker-node-adı>

# Yeni pod'ların bu node'a gitmediğini gözlemle
kubectl scale deployment web --replicas=6
kubectl get pods -o wide

# Node'u geri al
kubectl uncordon <worker-node-adı>

# Temizle
kind delete cluster --name multi-node
```

---

<a id="kaynaklar"></a>
## 📚 Kaynaklar

### Resmi Dokümantasyon
- **Kubernetes Official Documentation** — kubernetes.io/docs (en güncel kaynak, her şey burada)
- **kubectl Reference** — kubernetes.io/docs/reference/kubectl (tüm komutlar)

### Kitaplar
- **"Kubernetes in Action"** — Marko Luksa (en kapsamlı, biraz eski ama temeller için mükemmel)
- **"The Kubernetes Book"** — Nigel Poulton (hızlı başlangıç için ideal)
- **"Production Kubernetes"** — Josh Rosso et al. (ileri seviye, production için)

### Video Kurslar
- **KodeKloud — Kubernetes for Beginners** (uygulamalı, çok iyi)
- **TechWorld with Nana — Kubernetes Tutorial for Beginners** (YouTube, ücretsiz, Türkçe altyazı mevcut)
- **A Cloud Guru / Pluralsight** (daha sistematik öğrenme için)

### İnteraktif Öğrenme
- **Killercoda** (killercoda.com) — Browser'da Kubernetes ortamı, ücretsiz
- **Play with Kubernetes** — labs.play-with-k8s.com — Ücretsiz, 4 saatlik session
- **Kubernetes Playground** — Katacoda senaryoları

### Topluluk
- **CNCF Slack** — #kubernetes-users kanalı
- **Kubernetes Forum** — discuss.kubernetes.io
- **Reddit** — r/kubernetes

---

## ✅ Hafta 3-4 Başarı Kriterleri

Bu bölümü tamamladığında şunları yapabiliyor olmalısın:

- [ ] Kubernetes'in neden var olduğunu ve hangi problemi çözdüğünü kendi kelimelerinle açıklayabilmek
- [ ] Control Plane ve Worker Node bileşenlerini ve her birinin rolünü açıklayabilmek
- [ ] Bir Pod oluşturulduğunda hangi bileşenlerin devreye girdiğini sırayla anlatabilmek
- [ ] kubectl ile temel CRUD operasyonları yapabilmek (oluştur, listele, sil, describe)
- [ ] Pod loglarını görebilmek ve container içine girebilmek
- [ ] Minikube veya Kind ile local cluster kurabilmek
- [ ] Namespace oluşturabilmek ve kaynakları namespace'e göre yönetebilmek
- [ ] ResourceQuota ile namespace'e kaynak sınırı koyabilmek

---

## 🔮 Sonraki Adım

Bu haftalarda öğrendiklerinin üzerine inşa edeceğiz:

- **Pod** nedir, nasıl tanımlanır (YAML yapısı)
- **ReplicaSet** ve **Deployment** — uygulamaları yönetme
- **Service** — Pod'lara nasıl erişilir, load balancing
- **ConfigMap ve Secret** — konfigürasyon yönetimi

Hızlıca ilerlemek istersen şimdiden "Kubernetes Deployment ve Service" konularına göz atabilirsin. Ama önce bu haftanın pratik projelerini bitir. Temel sağlam olmazsa üst katlar sallanır.

Başarılar! 🚀

---

*Sorularını, takıldığın yerleri not et. Bir sonraki hafta birlikte üstüne inşa edeceğiz.*
