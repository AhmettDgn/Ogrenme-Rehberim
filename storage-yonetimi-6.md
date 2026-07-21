# 💾 Kubernetes Storage Yönetimi

> "Stateless uygulama çalıştırmak kolaydır. Veritabanını, dosyalarını, durumunu Kubernetes'te yönetmek — işte asıl ustalık burada."

10 yıldır Kubernetes ile çalışıyorum ve şunu rahatlıkla söyleyebilirim: Storage, Kubernetes'in en çok yanlış anlaşılan konusudur. "Pod çalıştı, her şey bitti" zanneden mühendislerin production'da veritabanı verilerini kaybettiğini gördüm. Bir Pod yeniden başladığında içindeki veriler gider. Bu rehberde bunu nasıl çözeceğini — geçici bellekten tam donanımlı kalıcı depolamaya kadar — adım adım anlatacağım.

Hazırsan başlayalım.

---

## 📋 İçindekiler

- [Neden Storage Zor?](#neden-zor)
- [Volume Tipleri](#volume-tipleri)
  - [emptyDir](#emptydir)
  - [hostPath](#hostpath)
  - [configMap / secret](#configmap-secret-volume)
  - [Diğer Volume Tipleri](#diger-tipler)
- [PersistentVolume (PV)](#pv)
- [PersistentVolumeClaim (PVC)](#pvc)
- [PV ve PVC İlişkisi](#pv-pvc-iliski)
- [StorageClass](#storageclass)
- [Dynamic Provisioning](#dynamic-provisioning)
- [StatefulSets](#statefulsets)
- [Volume Snapshots](#volume-snapshots)
- [Pratik Projeler](#pratik-projeler)

---

<a id="neden-zor"></a>
## 🧩 Neden Storage Zor?

Kubernetes **stateless** düşünülerek tasarlandı. Pod öldü mü? Yenisini başlat. Hangi Node'da çalışıyor? Önemli değil. Ama veritabanı için bu yaklaşım çalışmıyor:

```
Pod A (MySQL) → Node 1'de çalışıyor, /var/lib/mysql'de veri var
      ↓
Pod A çöktü, Kubernetes yeniden başlattı → ama Node 2'de!
      ↓
/var/lib/mysql BOŞ → Tüm veriler gitti!
```

Kubernetes storage sistemini anlamak için şu soruların cevaplarını bilmen gerekiyor:

| Soru | Çözüm |
|------|-------|
| Pod yeniden başlayınca veri kaybolmasın | PersistentVolume |
| Farklı Pod'lar aynı veriye erişsin | ReadWriteMany PVC |
| Storage otomatik oluşturulsun | StorageClass + Dynamic Provisioning |
| Veritabanı Pod'ları sıralı başlasın | StatefulSet |
| Veriyi yedeklemek/geri almak | Volume Snapshots |

---

<a id="volume-tipleri"></a>
## 📦 Volume Tipleri

Volume, Pod'a bağlanan bir depolama alanıdır. Pod'un spec'inde tanımlanır ve container'lara mount edilir.

```
Pod
├── Container 1  ──→  /data  (volume mount)
├── Container 2  ──→  /cache (volume mount)
└── Volumes
    ├── data-vol  (emptyDir)
    └── cache-vol (hostPath)
```

---

<a id="emptydir"></a>
### 1️⃣ emptyDir — Geçici Ortak Alan

Pod başladığında boş oluşur, Pod silindiğinde kaybolur. Aynı Pod içindeki container'lar arasında veri paylaşmak için kullanılır.

**Ne zaman kullanılır?**
- Sidecar pattern'da container'lar arası paylaşım (önceki hafta gördük)
- Geçici işlem dosyaları
- Cache

```yaml
# emptydir-ornek.yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-pod
spec:
  volumes:
  - name: paylasilan-alan
    emptyDir: {}

  # Bellek tabanlı emptyDir (tmpfs - çok hızlı ama RAM kullanır)
  - name: bellek-alani
    emptyDir:
      medium: Memory
      sizeLimit: 256Mi

  containers:
  - name: yazici
    image: busybox
    command: ['sh', '-c', 'while true; do date >> /data/zaman.txt; sleep 5; done']
    volumeMounts:
    - name: paylasilan-alan
      mountPath: /data

  - name: okuyucu
    image: busybox
    command: ['sh', '-c', 'while true; do cat /data/zaman.txt; sleep 10; done']
    volumeMounts:
    - name: paylasilan-alan
      mountPath: /data
      readOnly: true
```

> **Gerçek Dünya:** Machine learning pipeline'larında ara sonuçları emptyDir'e yazarız. Ana model container işler, sidecar container sonuçları S3'e upload eder. Pod bittiğinde geçici dosyalar otomatik temizlenir — disk dolmaz.

---

<a id="hostpath"></a>
### 2️⃣ hostPath — Node Dosya Sistemine Eriş

Pod'un çalıştığı **Node'un dosya sistemini** container'a mount eder.

```yaml
# hostpath-ornek.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  volumes:
  - name: node-logs
    hostPath:
      path: /var/log          # Node'daki gerçek yol
      type: Directory         # Tip kontrolü

  # type seçenekleri:
  # Directory         - var olmalı
  # DirectoryOrCreate - yoksa oluştur
  # File              - dosya olmalı
  # FileOrCreate      - yoksa oluştur
  # Socket            - Unix socket
  # CharDevice        - karakter aygıtı
  # BlockDevice       - blok aygıtı

  containers:
  - name: log-okuyucu
    image: busybox
    command: ['sh', '-c', 'tail -f /host-logs/syslog']
    volumeMounts:
    - name: node-logs
      mountPath: /host-logs
      readOnly: true
```

> **UYARI — Güvenlik Riski:** hostPath production'da çok dikkatli kullanılmalı. Container, Node'un dosya sistemine erişiyor. Kötü yapılandırılmış bir hostPath ile container host'u ele geçirebilir. Production'da yalnızca DaemonSet'lerde (log toplayıcı gibi) ve güvenilir sistem uygulamalarında kullan.

**Ne zaman kullanılır?**
- Log toplayıcı DaemonSet'ler (Fluentd, Filebeat)
- Node monitoring araçları
- Geliştirme ortamında local code mount etmek

---

<a id="configmap-secret-volume"></a>
### 3️⃣ configMap / secret — Config Dosyaları Olarak

Önceki haftada gördük ama kısaca hatırlatalım: ConfigMap ve Secret'ları dosya sistemi olarak mount edebilirsin.

```yaml
volumes:
- name: uygulama-config
  configMap:
    name: benim-configmap
- name: gizli-anahtarlar
  secret:
    secretName: benim-secret
    defaultMode: 0400  # Dosya izinleri
```

---

<a id="diger-tipler"></a>
### 4️⃣ Diğer Volume Tipleri

| Tip | Açıklama | Kullanım |
|-----|----------|----------|
| `nfs` | NFS sunucusu mount et | Paylaşımlı storage, ReadWriteMany |
| `iscsi` | iSCSI block storage | Yüksek performans gereken DB |
| `cephfs` | Ceph distributed storage | On-premise büyük cluster |
| `projected` | Birden fazla kaynağı tek volume'a birleştir | Token + ConfigMap + Secret aynı dizinde |
| `downwardAPI` | Pod metadata'yı dosyaya yaz | Pod adı, namespace, label'ları container'a ilet |

```yaml
# projected volume örneği - token + configmap tek dizinde
volumes:
- name: birlesik-volume
  projected:
    sources:
    - serviceAccountToken:
        path: token
        expirationSeconds: 3600
    - configMap:
        name: uygulama-config
    - secret:
        name: tls-sirri
```

---

<a id="pv"></a>
## 🗄️ PersistentVolume (PV) — Kalıcı Depolama Birimi

emptyDir ve hostPath geçici ya da Node'a bağımlı. Gerçekten kalıcı ve taşınabilir depolama için **PersistentVolume** gerekiyor.

PersistentVolume, cluster yöneticisinin (veya dynamic provisioner'ın) oluşturduğu **gerçek depolama kaynağının Kubernetes'teki temsilidir.**

```
Gerçek Dünya                    Kubernetes
─────────────────               ──────────────────
AWS EBS Disk        ←────────→  PersistentVolume
GCP Persistent Disk ←────────→  PersistentVolume
NFS Share           ←────────→  PersistentVolume
Ceph RBD            ←────────→  PersistentVolume
```

### PV Tanımı

```yaml
# persistentvolume.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: veritabani-pv
  labels:
    tip: hdd
    ortam: prod
spec:
  capacity:
    storage: 10Gi           # Toplam boyut

  accessModes:
    - ReadWriteOnce         # Erişim modu (aşağıda açıklanıyor)

  persistentVolumeReclaimPolicy: Retain   # PVC silinince ne olsun?

  storageClassName: manual  # StorageClass ile ilişki

  # Gerçek depolama backend'i — burada NFS örneği
  nfs:
    server: 192.168.1.100
    path: /exports/veritabani

  # Ya da hostPath (test için):
  # hostPath:
  #   path: /data/veritabani
```

### Access Modes (Erişim Modları)

Bu çok önemli — yanlış seçersen uygulaman çalışmaz:

| Mod | Kısa | Anlamı |
|-----|------|--------|
| `ReadWriteOnce` | RWO | Tek Node okur+yazar. Çoğu block storage (EBS, disk) |
| `ReadOnlyMany` | ROX | Birden fazla Node okur. Çok Pod paylaşır ama yazmaz |
| `ReadWriteMany` | RWX | Birden fazla Node okur+yazar. NFS, CephFS |
| `ReadWriteOncePod` | RWOP | Sadece tek **Pod** okur+yazar (K8s 1.22+) |

```
RWO: Node1[Pod yazıyor] ✓    Node2[Pod yazmak istiyor] ✗
RWX: Node1[Pod yazıyor] ✓    Node2[Pod yazıyor] ✓
```

### Reclaim Policy (Geri Kazanım Politikası)

PVC silindiğinde PV'ye ne olur?

| Politika | Davranış | Ne Zaman? |
|----------|----------|-----------|
| `Retain` | PV kalır, veri dokunulmaz. Manuel temizlik gerekir | Production verisi — yanlışlıkla silme |
| `Delete` | PV ve arkasındaki depolama silinir | Cloud diskler, geliştirme |
| `Recycle` | İçerik silinir, PV yeniden kullanılabilir | **Deprecated**, kullanma |

> **Gerçek Hayat Dersi:** Production'da her zaman `Retain` kullan. Bir mühendis PVC'yi yanlışlıkla sildi, `Delete` politikasıyla beraber 2 yıllık müşteri verisi gitti. `Retain` olsaydı PV disk orada dururdu, veriyi geri alırdık.

---

<a id="pvc"></a>
## 📋 PersistentVolumeClaim (PVC) — Depolama Talebi

Uygulama geliştiricisi olarak sen PV'nin detaylarını bilmek zorunda değilsin. "Bana 5GB, okuma-yazma yapabilen bir disk ver" diyorsun — bu iş **PVC** ile yapılıyor.

PVC bir **depolama talebidir**. Kubernetes bu talebi karşılayan bir PV ile eşleştirir.

```yaml
# persistentvolumeclaim.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: veritabani-pvc
  namespace: uygulama
spec:
  accessModes:
    - ReadWriteOnce         # İstediğim erişim modu

  resources:
    requests:
      storage: 5Gi          # İstediğim boyut

  storageClassName: manual  # Hangi StorageClass (boş bırakırsan default)

  # İsteğe bağlı: Belirli bir PV'yi seç
  selector:
    matchLabels:
      tip: hdd
```

### PVC'yi Pod'da Kullan

```yaml
# pvc-kullanan-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql-pod
spec:
  volumes:
  - name: mysql-verisi
    persistentVolumeClaim:
      claimName: veritabani-pvc   # PVC adı

  containers:
  - name: mysql
    image: mysql:8.0
    env:
    - name: MYSQL_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysql-secret
          key: root-password
    volumeMounts:
    - name: mysql-verisi
      mountPath: /var/lib/mysql   # MySQL veri dizini
```

```bash
# PVC durumunu kontrol et
kubectl get pvc
# NAME              STATUS   VOLUME         CAPACITY   ACCESS MODES
# veritabani-pvc    Bound    veritabani-pv  10Gi       RWO

# PV durumunu kontrol et
kubectl get pv
# NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS
# veritabani-pv   10Gi       RWO            Retain           Bound
```

**STATUS değerleri:**
- `Pending` → Uygun PV bulunamadı, bekliyor
- `Bound` → PV ile eşleşti, kullanıma hazır
- `Lost` → Bağlı PV kayboldu (ciddi sorun!)

---

<a id="pv-pvc-iliski"></a>
## 🔗 PV ve PVC İlişkisi — Büyük Resim

```
┌─────────────────────────────────────────────────────────────────┐
│                       KUBERNETES CLUSTER                         │
│                                                                   │
│  Cluster Yöneticisi              Geliştirici                     │
│  ─────────────────               ────────────                    │
│  PV oluşturur         ←eşleşir→  PVC oluşturur                  │
│                                                                   │
│  ┌──────────────┐               ┌──────────────┐                 │
│  │     PV       │               │     PVC      │                 │
│  │  10Gi, RWO   │ ──── Bound ──→│   5Gi, RWO   │                │
│  │  NFS backend │               │  uygulama ns │                │
│  └──────────────┘               └──────┬───────┘                │
│                                         │                        │
│                                    kullanır                      │
│                                         │                        │
│                                  ┌──────▼───────┐               │
│                                  │     Pod      │               │
│                                  │  /var/lib/db │               │
│                                  └──────────────┘               │
└─────────────────────────────────────────────────────────────────┘

Fiziksel Dünya:
┌──────────────┐
│  NFS Server  │ ←── PV bu diski temsil eder
│  /exports/db │
└──────────────┘
```

### Eşleşme Kuralları

Kubernetes PVC ile PV'yi eşleştirirken şunlara bakar:

1. `storageClassName` eşleşmeli
2. `accessModes` PVC'nin istediği PV'de mevcut olmalı
3. PV kapasitesi PVC talebinden **büyük veya eşit** olmalı
4. PVC'nin `selector`'ı varsa PV'nin label'ları uymalı

```bash
# Eşleşmeyen PVC sorununu teşhis et
kubectl describe pvc veritabani-pvc
# Events bölümünde "no persistent volumes available" gibi hata görürsün
```

---

<a id="storageclass"></a>
## 🏷️ StorageClass — Depolama Profili

Şimdiye kadar PV'yi elle yönetici oluşturdu. 100 uygulama için 100 PV manuelden oluşturmak pratik değil. **StorageClass** bu işi otomatikleştirir.

StorageClass, "bu profilde disk isteyenlere bu şekilde disk oluştur" diyen bir şablondur.

```yaml
# storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: hizli-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  # Default yap

provisioner: kubernetes.io/aws-ebs    # Kimin oluşturacağı

parameters:
  type: gp3               # EBS disk tipi
  iopsPerGB: "10"
  encrypted: "true"
  fsType: ext4

reclaimPolicy: Delete     # PVC silinince diski de sil
allowVolumeExpansion: true  # Sonradan boyut artırma izni
volumeBindingMode: WaitForFirstConsumer  # Pod başlayana kadar disk oluşturma
```

### Yaygın Provisioner'lar

| Cloud/Platform | Provisioner | Disk Tipi |
|----------------|-------------|-----------|
| AWS | `kubernetes.io/aws-ebs` | gp2, gp3, io1 |
| GCP | `kubernetes.io/gce-pd` | pd-standard, pd-ssd |
| Azure | `kubernetes.io/azure-disk` | Standard_LRS, Premium_LRS |
| vSphere | `csi.vsphere.volume` | - |
| On-premise | `ceph.rook.io/block` | RBD, CephFS |
| Local test | `rancher.io/local-path` | hostPath tabanlı |

```bash
# Cluster'daki StorageClass'ları listele
kubectl get storageclass
# NAME                 PROVISIONER             RECLAIMPOLICY
# standard (default)  kubernetes.io/gce-pd    Delete
# premium-ssd         kubernetes.io/gce-pd    Retain

# Default StorageClass'ı gör
kubectl get storageclass -o wide
```

### `volumeBindingMode` Neden Önemli?

```yaml
volumeBindingMode: Immediate         # PVC oluşturulunca anında disk oluştur
volumeBindingMode: WaitForFirstConsumer  # Pod başlayana kadar bekle
```

`WaitForFirstConsumer` neden tercih edilir? Çünkü EBS diskler sadece aynı AZ'deki Pod'lara mount edilebilir. Pod'u önce oluşturur, hangi Node'a (dolayısıyla hangi AZ'a) schedule edildiğini görür, sonra o AZ'de disk oluşturur. Yoksa disk us-east-1a'da oluşur, Pod us-east-1b'ye schedule edilirse hiç mount edilemez!

---

<a id="dynamic-provisioning"></a>
## ⚡ Dynamic Provisioning — Otomatik Disk Yönetimi

StorageClass + PVC = Otomatik disk oluşturma. Artık yöneticinin PV oluşturmasına gerek yok.

```
PVC oluşturuldu
    ↓
Kubernetes: "Bu StorageClass için provisioner var mı?"
    ↓ Evet
Provisioner: AWS'den EBS disk oluştur
    ↓
PV otomatik oluşturuldu
    ↓
PVC → PV Bound
    ↓
Pod kullanmaya başlar
```

```yaml
# dynamic-pvc.yaml
# StorageClass belirtiyoruz, PV YOK — otomatik oluşturulacak
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: otomatik-disk
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: hizli-ssd    # StorageClass adı
  resources:
    requests:
      storage: 20Gi
```

```bash
kubectl apply -f dynamic-pvc.yaml

# İlk başta Pending
kubectl get pvc otomatik-disk
# NAME           STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS
# otomatik-disk  Pending                                      hizli-ssd

# Pod bağlayınca (WaitForFirstConsumer) veya anında (Immediate):
# NAME           STATUS   VOLUME                                     CAPACITY
# otomatik-disk  Bound    pvc-a1b2c3d4-xxxx-yyyy-zzzz-aabbccdd1234   20Gi
```

### Sonradan Boyut Artırma

StorageClass `allowVolumeExpansion: true` ise:

```bash
# PVC'yi edit et
kubectl edit pvc otomatik-disk
# spec.resources.requests.storage değerini 20Gi'den 50Gi'ye çek

# Durumu kontrol et
kubectl get pvc otomatik-disk
# Condition: FileSystemResizePending → Pod yeniden başlayınca tamamlanır
```

> **Dikkat:** Volume'u küçültemezsin, sadece büyütebilirsin. Ve tüm storage backend'ler bu özelliği desteklemez.

---

<a id="statefulsets"></a>
## 🎯 StatefulSets — Durumlu Uygulamalar İçin

Deployment, Pod'larına `web-deployment-abc123` gibi rastgele isimler verir. Sırası önemli değil. Ama veritabanları için:

- **MySQL master**: önce başlamalı, `mysql-0` adını almalı
- **MySQL replica**: master hazır olduktan sonra başlamalı, `mysql-1`, `mysql-2`
- Bir replica çökerse yerine `mysql-1` adıyla tam aynı yere mount edilmeli

Bunu Deployment yapamaz. **StatefulSet** yapar.

### StatefulSet'in Guarantees (Garantileri)

1. **Sıralı, benzersiz kimlik:** Pod'lar `<isim>-0`, `<isim>-1`, `<isim>-2` şeklinde adlandırılır
2. **Sıralı başlatma:** `<isim>-0` Running olmadan `<isim>-1` başlamaz
3. **Sıralı silme:** Ters sırayla silinir (`<isim>-2` önce)
4. **Kararlı DNS:** `<pod-adı>.<servis-adı>.<namespace>.svc.cluster.local`
5. **Kararlı depolama:** Her Pod kendi PVC'sine sahip, Pod silinse de PVC kalır

```yaml
# statefulset-mysql.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-headless    # Headless service adı (zorunlu)
  replicas: 3
  selector:
    matchLabels:
      app: mysql

  # Güncelleme stratejisi
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0   # 0 = hepsini güncelle, 2 = sadece index>=2'yi güncelle

  template:
    metadata:
      labels:
        app: mysql
    spec:
      initContainers:
      # Her Pod kendisinin master mı replica mı olduğuna karar veriyor
      - name: init-mysql
        image: mysql:8.0
        command:
        - bash
        - "-c"
        - |
          set -ex
          # Hostname'den index çıkar: mysql-0 → 0
          [[ $(hostname) =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          # 0 index = master, diğerleri = replica
          if [[ $ordinal -eq 0 ]]; then
            cp /mnt/config-map/master.cnf /mnt/conf.d/
          else
            cp /mnt/config-map/replica.cnf /mnt/conf.d/
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map

      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: data          # Her Pod'un kendi PVC'si (aşağıda tanımlı)
          mountPath: /var/lib/mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 2

      volumes:
      - name: conf
        emptyDir: {}
      - name: config-map
        configMap:
          name: mysql-config

  # Her Pod için otomatik PVC oluştur
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: hizli-ssd
      resources:
        requests:
          storage: 10Gi
```

### Headless Service — StatefulSet'in DNS'i

StatefulSet ile beraber bir **Headless Service** gerekiyor. Bu, her Pod'un kendi DNS adını almasını sağlar:

```yaml
# headless-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
spec:
  clusterIP: None    # Headless! IP yok, sadece DNS
  selector:
    app: mysql
  ports:
  - port: 3306
    name: mysql
```

Bu sayede Pod'lar şu DNS adresleriyle erişilebilir olur:

```
mysql-0.mysql-headless.default.svc.cluster.local  → master
mysql-1.mysql-headless.default.svc.cluster.local  → replica 1
mysql-2.mysql-headless.default.svc.cluster.local  → replica 2
```

```bash
# StatefulSet oluştur
kubectl apply -f headless-service.yaml
kubectl apply -f statefulset-mysql.yaml

# Sıralı başlamayı izle
kubectl get pods -w
# mysql-0   0/1   Pending   →  Running  (önce başlar)
# mysql-1   0/1   Pending   →  Running  (mysql-0 hazır olduktan sonra)
# mysql-2   0/1   Pending   →  Running  (mysql-1 hazır olduktan sonra)

# Her Pod'un kendi PVC'si var
kubectl get pvc
# data-mysql-0   Bound   pvc-xxx   10Gi
# data-mysql-1   Bound   pvc-yyy   10Gi
# data-mysql-2   Bound   pvc-zzz   10Gi

# StatefulSet Scale
kubectl scale statefulset mysql --replicas=5

# Sil ama PVC koru (veri kaybolmaz)
kubectl delete statefulset mysql
kubectl get pvc   # PVC'ler hâlâ duruyor
```

### Deployment vs StatefulSet

```
┌────────────────────────────────────────────────────────────────┐
│              DEPLOYMENT           │          STATEFULSET        │
├───────────────────────────────────┼────────────────────────────┤
│ Pod isimleri rastgele             │ Pod isimleri sıralı (0,1,2)│
│ Herhangi sırada başlar/durur      │ Sıralı başlar ve durur     │
│ Tek paylaşımlı PVC (varsa)        │ Her Pod'un kendi PVC'si    │
│ DNS yok (Service üzerinden)       │ Her Pod kendi DNS adresine │
│ Stateless uygulamalar             │ Stateful: DB, Cache, Queue │
│ nginx, api, frontend              │ MySQL, Redis, Kafka, Zk    │
└───────────────────────────────────┴────────────────────────────┘
```

---

<a id="volume-snapshots"></a>
## 📸 Volume Snapshots — Yedek ve Geri Yükleme

Production'da veritabanı diskinin anlık görüntüsünü almak (snapshot) ve gerektiğinde geri yüklemek hayat kurtarır.

### VolumeSnapshotClass

StorageClass'ın snapshot versiyonu:

```yaml
# volumesnapshotclass.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-aws-vsc
driver: ebs.csi.aws.com
deletionPolicy: Delete    # Snapshot silinince arkasındaki data da silinsin
```

### VolumeSnapshot — Anlık Görüntü Al

```yaml
# snapshot-al.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mysql-snapshot-2024-01-15
spec:
  volumeSnapshotClassName: csi-aws-vsc
  source:
    persistentVolumeClaimName: data-mysql-0   # Hangi PVC'nin snapshot'ı
```

```bash
# Snapshot al
kubectl apply -f snapshot-al.yaml

# Durumu kontrol et
kubectl get volumesnapshot
# NAME                       READYTOUSE   SOURCEPVC       RESTORESIZE   AGE
# mysql-snapshot-2024-01-15  true         data-mysql-0    10Gi          2m
```

### Snapshot'tan Geri Yükleme

```yaml
# restore-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-restore-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: hizli-ssd
  resources:
    requests:
      storage: 10Gi
  dataSource:
    name: mysql-snapshot-2024-01-15   # Hangi snapshot'tan
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

```bash
# Restore PVC oluştur
kubectl apply -f restore-pvc.yaml

# Bu PVC'yi bir Pod'a bağla
kubectl run mysql-restore \
  --image=mysql:8.0 \
  --env="MYSQL_ROOT_PASSWORD=sifre" \
  --overrides='{"spec":{"volumes":[{"name":"data","persistentVolumeClaim":{"claimName":"mysql-restore-pvc"}}],"containers":[{"name":"mysql-restore","image":"mysql:8.0","volumeMounts":[{"mountPath":"/var/lib/mysql","name":"data"}]}]}}'
```

### Otomatik Snapshot — CronJob ile

```yaml
# snapshot-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mysql-otomatik-yedek
spec:
  schedule: "0 2 * * *"   # Her gece 02:00'de
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: snapshot-creator
          containers:
          - name: snapshot-creator
            image: bitnami/kubectl:latest
            command:
            - bash
            - -c
            - |
              TARIH=$(date +%Y%m%d-%H%M%S)
              kubectl apply -f - <<EOF
              apiVersion: snapshot.storage.k8s.io/v1
              kind: VolumeSnapshot
              metadata:
                name: mysql-otomatik-$TARIH
              spec:
                volumeSnapshotClassName: csi-aws-vsc
                source:
                  persistentVolumeClaimName: data-mysql-0
              EOF
              echo "Snapshot oluşturuldu: mysql-otomatik-$TARIH"
          restartPolicy: OnFailure
```

---

## 🏗️ Tam Örnek: MySQL StatefulSet Production Setup

Gerçek bir MySQL production kurulumu — tüm parçalar bir arada:

```yaml
# ── 1. Namespace ──────────────────────────────────────
apiVersion: v1
kind: Namespace
metadata:
  name: veritabani
---
# ── 2. Secret ─────────────────────────────────────────
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: veritabani
type: Opaque
data:
  root-password: c3VwZXJnaXpMaXNpZnJlMTIz    # base64
  replication-password: cmVwbFBhc3MxMjM=
---
# ── 3. ConfigMap ──────────────────────────────────────
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
  namespace: veritabani
data:
  master.cnf: |
    [mysqld]
    log-bin
    server-id=1
  replica.cnf: |
    [mysqld]
    super-read-only
    server-id=2
---
# ── 4. StorageClass ───────────────────────────────────
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: mysql-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
# ── 5. Headless Service ───────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: veritabani
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
  - port: 3306
    name: mysql
---
# ── 6. Read Service (replica'lara yönlen) ─────────────
apiVersion: v1
kind: Service
metadata:
  name: mysql-read
  namespace: veritabani
spec:
  selector:
    app: mysql
  ports:
  - port: 3306
    name: mysql
---
# ── 7. StatefulSet ────────────────────────────────────
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: veritabani
spec:
  serviceName: mysql
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
        - name: config
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping", "-h", "localhost"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 2
          timeoutSeconds: 1
      volumes:
      - name: config
        configMap:
          name: mysql-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: mysql-storage
      resources:
        requests:
          storage: 10Gi
```

```bash
# Uygula
kubectl apply -f mysql-production.yaml

# İzle
kubectl get all -n veritabani
kubectl get pvc -n veritabani

# Master'a bağlan
kubectl exec -it mysql-0 -n veritabani -- mysql -uroot -p

# Replica'ya bağlan
kubectl exec -it mysql-1 -n veritabani -- mysql -uroot -p

# DNS test et (Pod içinden)
kubectl exec -it mysql-0 -n veritabani -- bash -c \
  "nslookup mysql-1.mysql.veritabani.svc.cluster.local"
```

---

<a id="pratik-projeler"></a>
## 🛠️ Pratik Projeler

### Proje 9: MySQL StatefulSet ile Deploy

**Hedef:** 1 master + 2 replica MySQL cluster kur

1. Namespace ve Secret oluştur
2. Headless Service + ClusterIP Service oluştur
3. StatefulSet (3 replica) kur ve sıralı başlamayı izle
4. Her Pod'un DNS adresini test et
5. Master'a veri yaz, replica'dan oku

```bash
# Test:
kubectl exec -it mysql-0 -n veritabani -- mysql -uroot -psifre -e "CREATE DATABASE test; USE test; CREATE TABLE t1 (id INT); INSERT INTO t1 VALUES (1);"
kubectl exec -it mysql-1 -n veritabani -- mysql -uroot -psifre -e "USE test; SELECT * FROM t1;"
```

### Proje 10: Dynamic Volume Provisioning

**Hedef:** StorageClass ile otomatik disk yönetimi

1. `rancher.io/local-path` provisioner kur (Minikube/Kind için)
2. StorageClass oluştur
3. PVC oluştur — PV otomatik oluşmasını izle
4. Pod bağla, veri yaz
5. Pod sil ve yenisini başlat — verinin kaldığını doğrula
6. PVC boyutunu büyüt

```bash
# Minikube ile local-path provisioner
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
kubectl get storageclass
```

### Proje 11: Volume Backup ve Restore

**Hedef:** Snapshot al, veri boz, geri yükle

1. MySQL'e test verisi yaz
2. VolumeSnapshot al
3. Kasıtlı olarak tabloyu sil (felaket simülasyonu!)
4. Snapshot'tan yeni PVC oluştur
5. Yeni PVC ile MySQL Pod başlat
6. Verinin geri geldiğini doğrula

### Proje 12: Multi-Zone Storage Replication

**Hedef:** Bölgeler arası veri replikasyonu

1. Node'ları farklı zone'lara dağıt (Minikube'de simüle et)
2. `volumeBindingMode: WaitForFirstConsumer` ile StorageClass kur
3. Pod'un schedule edildiği zone'da disk oluştuğunu gözlemle
4. Anti-affinity ile Pod'ları farklı zone'lara dağıt
5. Zone failure simülasyonu: bir Node'u `kubectl cordon` et, StatefulSet'in nasıl davrandığını gözlemle

---

## 📚 Yararlı kubectl Komutları — Storage Özet

```bash
# PV işlemleri
kubectl get pv                          # Tüm PV'ler
kubectl describe pv <pv-adı>            # Detay
kubectl delete pv <pv-adı>              # Sil (Retain politikasında manuel)

# PVC işlemleri
kubectl get pvc                         # Tüm PVC'ler
kubectl get pvc -n <namespace>          # Namespace bazlı
kubectl describe pvc <pvc-adı>          # Detay + Events (bağlanma sorunu için)
kubectl delete pvc <pvc-adı>            # Sil (Delete politikasında disk de gider!)

# StorageClass işlemleri
kubectl get storageclass                # StorageClass'lar
kubectl describe storageclass <isim>    # Detay

# StatefulSet işlemleri
kubectl get statefulset                 # StatefulSet listesi
kubectl scale statefulset <isim> --replicas=5
kubectl rollout status statefulset/<isim>
kubectl rollout history statefulset/<isim>
kubectl rollout undo statefulset/<isim>
kubectl delete statefulset <isim> --cascade=orphan  # Pod sil, PVC koru

# Volume Snapshot işlemleri
kubectl get volumesnapshot
kubectl get volumesnapshotclass
kubectl describe volumesnapshot <isim>

# Disk kullanımını gör (Pod içinden)
kubectl exec -it <pod-adı> -- df -h

# PVC'ye bağlı Pod'u bul
kubectl get pods -o json | \
  jq '.items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName=="pvc-adim") | .metadata.name'
```

---

## ✅ Bu Haftanın Kontrol Listesi

- [ ] emptyDir, hostPath, configMap volume tiplerini açıklayabilmek
- [ ] PV oluşturup PVC ile eşleştirme yapabilmek
- [ ] Access mode farkları (RWO, ROX, RWX) ve ne zaman kullanılacağını bilmek
- [ ] Reclaim policy farkları (Retain vs Delete) ve production tercihini bilmek
- [ ] StorageClass ile dynamic provisioning kurabilmek
- [ ] StatefulSet ile MySQL gibi stateful uygulama deploy edebilmek
- [ ] Headless Service ve StatefulSet DNS adreslerini anlayabilmek
- [ ] Deployment vs StatefulSet ne zaman hangisi kullanılır bilmek
- [ ] Volume snapshot alıp geri yükleme yapabilmek

Bir sonraki haftada **Ingress, Network Policies ve RBAC** konularına geçeceğiz. Bu hafta öğrendiğin PVC ve StatefulSet bilgileri, o haftaki uygulamalarda veritabanı katmanı olarak kullanılacak.

---

> **Son Söz:** Storage, Kubernetes'in en sık "sonraya bırakılan" konusudur. "Önce stateless şeyleri öğreneyim" diyerek ertelenir. Ama ilk production veritabanını Kubernetes'e taşıyacağın gün bu bilgiler olmadan çaresiz kalırsın. PV/PVC/StorageClass üçlüsünü kavradıysan ve StatefulSet'in neden Deployment'tan farklı olduğunu anladıysan, production-grade Kubernetes kurulumu yapabilecek seviyeye geldin.
