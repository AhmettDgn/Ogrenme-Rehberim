# Network Temelleri Rehberi - Cloud Native Perspektifinden

Merhaba! Ben 10 yillik network deneyimimle, ozellikle cloud-native dunyasinda karsiniza cikacak network konularini size ogretici bir sekilde anlatacagim. Bu rehberi yazarken "bunu bilmeden Kubernetes/Cloud ortaminda rahat edemezsin" dediklerim konulari bir araya getirdim. Her konuyu gercek hayat ornekleriyle pekistirerek anlatacagim.

> Bu rehber, cloudnative dosyasindaki Faz 0 - Hafta 2-3: Networking Temelleri bolumunu kapsamli sekilde tamamlar.

---

## ICINDEKILER

1. [OSI ve TCP/IP Modelleri](#osi-tcpip)
2. [IP Adresleme ve Subnetting](#ip-subnetting)
3. [DNS Calisma Mantigi](#dns)
4. [HTTP/HTTPS Protokolleri](#http-https)
5. [Load Balancing Kavrami](#load-balancing)
6. [Reverse Proxy Nedir](#reverse-proxy)
7. [Firewall Temelleri](#firewall)
8. [Cloud Native Networking Ekstra Konulari](#cloud-native-extras)

---

## <a id="osi-tcpip"></a>1. OSI VE TCP/IP MODELLERI

### Neden Bu Kadar Onemli?

Sunu soyleyeyim: network sorunlarini debug ederken "sorun hangi katmanda?" sorusunu soramazsan, karanlikta el yordamiyla yurursun. OSI ve TCP/IP modelleri sana network iletisimini katman katman dusunmeyi ogretir. Kubernetes'te bir pod baska bir pod'a ulasamamissa, sorunun DNS'te mi (Layer 7), IP routing'de mi (Layer 3), yoksa fiziksel baglantida mi (Layer 1) oldugunu anlamak icin bu modeli bilmen sart.

### OSI Modeli (7 Katman)

OSI (Open Systems Interconnection) modeli, ISO tarafindan 1984'te yayinlandi. Teorik bir referans modelidir. Gercek hayatta birebir uygulanmaz ama sorun giderme ve iletisim icin harika bir cercevedir.

```
Katman 7 - Application (Uygulama)     : HTTP, HTTPS, DNS, FTP, SSH, gRPC
Katman 6 - Presentation (Sunum)       : SSL/TLS, sifrele/coz, veri formati
Katman 5 - Session (Oturum)           : Baglanti yonetimi, oturum takibi
Katman 4 - Transport (Tasima)         : TCP, UDP - port numaralari
Katman 3 - Network (Ag)              : IP adresleri, routing, ICMP
Katman 2 - Data Link (Veri Baglantisi): MAC adresleri, Ethernet, ARP
Katman 1 - Physical (Fiziksel)        : Kablolar, Wi-Fi sinyalleri, voltaj
```

### Gercek Hayat Analojisi

Bir mektup gonderdigini dusun:

```
Layer 7 (Application)    : Mektubu yaziyorsun (icerik)
Layer 6 (Presentation)   : Mektubu sifreliyorsun (gizlilik)
Layer 5 (Session)        : Yazisma basliyor/bitiyor (oturum)
Layer 4 (Transport)      : Iadeli taahhutlu mu, adi posta mi? (TCP vs UDP)
Layer 3 (Network)        : Adres yaziyorsun - hangi sehir, hangi mahalle (IP)
Layer 2 (Data Link)      : Postacinin mahallede dogru kapiya gitmesi (MAC)
Layer 1 (Physical)       : Postacinin fiziksel olarak yurumesi (kablo/sinyal)
```

### TCP/IP Modeli (4 Katman) - Gercek Hayatta Kullanilan

TCP/IP, OSI'nin pratik versiyonudur. Internet'in gercekten uzerinde calistigi modeldir.

```
TCP/IP Katmani          OSI Karsiligi           Protokol Ornekleri
----------------------------------------------------------------------
Application             Layer 5-6-7             HTTP, DNS, SSH, TLS
Transport               Layer 4                 TCP, UDP
Internet                Layer 3                 IP, ICMP, ARP
Network Access          Layer 1-2               Ethernet, Wi-Fi
```

### Cloud Native'de Neden Onemli?

Kubernetes'te sorun giderirken su sekilde dusunursun:

```
Senaryo: Pod A, Pod B'ye HTTP istegi atiyor ama cevap gelmiyor.

Adim 1 (Layer 3 - Network): Pod'larin IP'leri dogru mu?
  $ kubectl get pod -o wide
  # Pod IP'lerini kontrol et

Adim 2 (Layer 4 - Transport): Port acik mi? TCP baglantisi kuruluyor mu?
  $ kubectl exec pod-a -- nc -zv pod-b-ip 8080
  # TCP baglantisi test et

Adim 3 (Layer 7 - Application): HTTP cevabi ne diyor?
  $ kubectl exec pod-a -- curl -v http://pod-b:8080/health
  # HTTP seviyesinde kontrol et
```

KRITIK BILGI: Kubernetes'teki Service, Ingress ve Network Policy kaynaklari farkli katmanlarda calisir:
- Service (ClusterIP): Layer 4 (TCP/UDP port yonlendirme)
- Ingress: Layer 7 (HTTP/HTTPS routing)
- Network Policy: Layer 3-4 (IP ve port bazli filtreleme)

---

## <a id="ip-subnetting"></a>2. IP ADRESLEME VE SUBNETTING

### IP Adresi Nedir?

IP adresi, bir cihazin aglardaki kimlik numarasidir. Iki versiyonu var:

**IPv4:** 32 bit, 4 oktet, noktalarla ayrilir
```
Ornek: 192.168.1.100
Binary: 11000000.10101000.00000001.01100100
```

**IPv6:** 128 bit, 8 grup, iki noktayla ayrilir
```
Ornek: 2001:0db8:85a3:0000:0000:8a2e:0370:7334
Kisaltilmis: 2001:db8:85a3::8a2e:370:7334
```

### Ozel (Private) vs Genel (Public) IP Adresleri

Bu ayrim cloud ortaminda hayati onem tasir:

```
Ozel IP Araliklari (Internetten erisim YOK):
  10.0.0.0     - 10.255.255.255    (10.0.0.0/8)      -> Buyuk sirketler, cloud VPC
  172.16.0.0   - 172.31.255.255    (172.16.0.0/12)    -> Docker default
  192.168.0.0  - 192.168.255.255   (192.168.0.0/16)   -> Ev/ofis agi

Genel IP: Internet uzerinden dogrudan erisilebilir adresler
```

CLOUD NATIVE BILGISI: AWS VPC genelde 10.0.0.0/16 kullanir. Kubernetes pod networku icin 10.244.0.0/16 veya 192.168.0.0/16 gibi ozel araliklar atanir. Docker ise default olarak 172.17.0.0/16 kullanir.

### Subnetting - Alt Ag Olusturma

Subnetting, buyuk bir agi kucuk parcalara boluyor. Bunu neden yapariz?

1. Guvenligi artirmak (her alt ag izole)
2. Trafigi yonetmek (broadcast domain'i kucultur)
3. IP'leri verimli kullanmak

### CIDR Notasyonu

CIDR (Classless Inter-Domain Routing) modern IP adreslemenin temelidir:

```
Notasyon: IP/prefix_uzunlugu

Ornek: 10.0.1.0/24
  10.0.1.0   = Network adresi
  /24        = Ilk 24 bit network kismini tanimlar
             = 32 - 24 = 8 bit host kismi
             = 2^8 - 2 = 254 kullanilabilir adres

Subnet Mask: 255.255.255.0
```

### Subnetting Tablosu (Ezberle!)

```
CIDR    Subnet Mask         Kullanilabilir Host    Kullanim Alani
----------------------------------------------------------------------
/32     255.255.255.255      1                      Tek host (pod IP)
/31     255.255.255.254      2                      Point-to-point link
/30     255.255.255.252      2                      Kucuk link
/28     255.255.255.240      14                     Kucuk subnet
/27     255.255.255.224      30                     Kucuk ofis
/26     255.255.255.192      62                     Orta ofis
/25     255.255.255.128      126                    Buyuk ofis
/24     255.255.255.0        254                    Standart subnet
/20     255.255.240.0        4094                   Buyuk subnet
/16     255.255.0.0          65534                  VPC / buyuk ag
/8      255.0.0.0            16 milyon+             Dev ag (10.0.0.0/8)
```

### Pratik Ornek: Cloud VPC Tasarimi

Diyelim ki bir Kubernetes cluster'i icin VPC tasarliyorsun:

```
VPC CIDR: 10.0.0.0/16 (65,534 IP)
|
|-- Public Subnet AZ-a:   10.0.1.0/24   (254 IP) -> Load Balancer, NAT GW
|-- Public Subnet AZ-b:   10.0.2.0/24   (254 IP) -> Load Balancer, NAT GW
|
|-- Private Subnet AZ-a:  10.0.10.0/24  (254 IP) -> Worker Node'lar
|-- Private Subnet AZ-b:  10.0.11.0/24  (254 IP) -> Worker Node'lar
|
|-- DB Subnet AZ-a:       10.0.20.0/24  (254 IP) -> RDS, ElastiCache
|-- DB Subnet AZ-b:       10.0.21.0/24  (254 IP) -> RDS, ElastiCache
|
|-- Pod Network:           10.244.0.0/16 (65,534 IP) -> Kubernetes pod'lari
```

ONEMLI: Kubernetes'te her pod'un kendi IP adresi vardir. 100 pod calistiriyorsan, 100 IP adresine ihtiyacin var. Pod networku icin yeterli buyuklukte CIDR blogu planlamak zorundasin. AWS EKS'te /16 veya /18 CIDR blogu oneriliyor.

### Subnetting Hesaplama Ornegi

Soru: 10.0.0.0/22 networkunde kac host olabilir?

```
/22 = 32 - 22 = 10 bit host kismi
2^10 = 1024 toplam adres
1024 - 2 = 1022 kullanilabilir host (network + broadcast adresi cikar)

IP Araligi: 10.0.0.1 - 10.0.3.254
Network:    10.0.0.0
Broadcast:  10.0.3.255
Subnet Mask: 255.255.252.0
```

---

## <a id="dns"></a>3. DNS CALISMA MANTIGI

### DNS Nedir?

DNS (Domain Name System), internetin telefon rehberidir. Insanlar isimleri hatilar (google.com), bilgisayarlar sayilari anlar (142.250.185.14). DNS bu ikisi arasinda cevirmenlik yapar.

### DNS Sorgu Sureci (Adim Adim)

Tarayiciya "www.example.com" yazdiginda ne olur:

```
                          Kullanici
                             |
                    "www.example.com nedir?"
                             |
                             v
                  +---------------------+
              1.  |  Browser Cache      |  -> Onceden cozulmus mu?
                  +---------------------+
                             |  (bulunamazsa)
                             v
                  +---------------------+
              2.  |  OS Cache           |  -> isletim sistemi bilir mi?
                  |  (/etc/hosts)       |
                  +---------------------+
                             |  (bulunamazsa)
                             v
                  +---------------------+
              3.  |  Recursive Resolver |  -> ISP'nin DNS sunucusu
                  |  (8.8.8.8 gibi)    |     veya Cloudflare 1.1.1.1
                  +---------------------+
                             |  (bulunamazsa)
                             v
                  +---------------------+
              4.  |  Root DNS Server    |  -> "com'u kim biliyor?"
                  |  (. root zone)      |     13 root server grubu var
                  +---------------------+
                             |
                             v
                  +---------------------+
              5.  |  TLD DNS Server     |  -> ".com alan adi sunucusu"
                  |  (.com, .org, .tr)  |     "example.com icin su NS'e sor"
                  +---------------------+
                             |
                             v
                  +---------------------+
              6.  |  Authoritative DNS  |  -> "example.com = 93.184.216.34"
                  |  (yetkili sunucu)   |     Kesin cevap burada
                  +---------------------+
                             |
                             v
                  Cevap: 93.184.216.34
```

### DNS Kayit Tipleri

```
Kayit Tipi   Aciklama                       Ornek
----------------------------------------------------------------------
A            IPv4 adresine esler             example.com -> 93.184.216.34
AAAA         IPv6 adresine esler             example.com -> 2606:2800:220:1::
CNAME        Baska bir isme yonlendirir      www.example.com -> example.com
MX           Mail sunucusu                   example.com -> mail.example.com
NS           Yetkili isim sunucusu           example.com -> ns1.example.com
TXT          Metin bilgisi (SPF, DKIM vs.)   example.com -> "v=spf1 ..."
SRV          Servis konumu (port dahil)      _http._tcp.example.com
PTR          Ters DNS (IP -> isim)           34.216.184.93 -> example.com
SOA          Zone yetkisi baslangici         Zone bilgileri
```

### TTL (Time To Live)

TTL, bir DNS kaydinin ne kadar sure cache'de tutulacagini belirler:

```
TTL=300   -> 5 dakika  (Sik degisen kayitlar icin, ornegin failover)
TTL=3600  -> 1 saat    (Normal web siteleri)
TTL=86400 -> 24 saat   (Nadiren degisen kayitlar)

IPUCU: DNS degisikligi yapacaksan ONCE TTL'i dusur (ornegin 60 saniye),
degisikligi yap, sonra tekrar yukselt. Boylece eski kayit hizla duser.
```

### Kubernetes'te DNS - CoreDNS

Bu kisim cloud native icin cok kritik. Kubernetes, cluster icindeki DNS islemlerini CoreDNS ile yonetir.

```
Kubernetes DNS Formati:
  <servis-adi>.<namespace>.svc.cluster.local

Ornek:
  my-api.production.svc.cluster.local  -> ClusterIP adresine cozulur
  my-db.database.svc.cluster.local     -> Database servisinin IP'si
```

Gercek Senaryo:

```yaml
# Backend servisi
apiVersion: v1
kind: Service
metadata:
  name: backend-api
  namespace: production
spec:
  selector:
    app: backend
  ports:
    - port: 8080

# Frontend pod'u icinden backend'e erisim:
# curl http://backend-api.production.svc.cluster.local:8080
# veya ayni namespace icindeyse kisaca:
# curl http://backend-api:8080
```

ONEMLI BILGI: Kubernetes'te bir pod baska bir namespace'deki servise ulasmak istiyorsa, tam DNS adini (FQDN) kullanmak zorundadir. Ayni namespace icindeyse sadece servis adi yeter.

### CoreDNS Nasil Calisir?

```
Pod "backend-api" adresini cozumlemek istiyor
         |
         v
Pod'un /etc/resolv.conf dosyasi
  nameserver 10.96.0.10        <- CoreDNS servis IP'si
  search default.svc.cluster.local svc.cluster.local cluster.local
         |
         v
CoreDNS (kube-dns servisi)
  - Kubernetes API'den servis bilgilerini alir
  - Eger cluster ici adresse -> ClusterIP dondurur
  - Eger dis adresse -> upstream DNS'e yonlendirir (8.8.8.8 gibi)
```

### Modern DNS Guvenligi

```
DNS over HTTPS (DoH):   DNS sorgularini HTTPS uzerinden sifreler (RFC 8484)
DNS over TLS (DoT):     DNS sorgularini TLS uzerinden sifreler (RFC 7858)

Neden onemli?
  - Normal DNS sorguari duz metin (plaintext) gider
  - Aradaki herkes hangi sitelere girdiginizi gorebilir
  - DoH/DoT ile sorgular sifrelenir

CoreDNS, DNS over HTTPS ve DNS over TLS destekler.
Kubernetes cluster'larinda ozellikle multi-cloud veya public-facing
deployment'larda DNS sorgularini sifrelemeyi degerlendirebilirsiniz.
```

---

## <a id="http-https"></a>4. HTTP/HTTPS PROTOKOLLERI

### HTTP Nedir?

HTTP (HyperText Transfer Protocol), web'in iletisim dilidir. Client (istemci) bir istek atar, server (sunucu) bir yanit dondurur. Basit, stateless (durumsuz) bir protokoldur.

### HTTP Request (Istek) Yapisi

```
GET /api/users HTTP/1.1          <- Method + Path + Protokol versiyonu
Host: api.example.com            <- Hangi sunucu
Authorization: Bearer eyJhbG...  <- Kimlik dogrulama
Content-Type: application/json   <- Veri formati
Accept: application/json         <- Kabul edilen format
User-Agent: curl/7.68.0          <- Istemci bilgisi
                                 <- Bos satir (header sonu)
{"name": "ahmet"}               <- Body (POST/PUT icin)
```

### HTTP Methods (Metodlari)

```
Method   Amac                Ornek                           Idempotent?
----------------------------------------------------------------------
GET      Veri oku            GET /api/users                  Evet
POST     Yeni kayit olustur  POST /api/users                 Hayir
PUT      Kaydi tamamen guncelle  PUT /api/users/1            Evet
PATCH    Kaydi kismen guncelle   PATCH /api/users/1          Hayir
DELETE   Kayit sil           DELETE /api/users/1             Evet
HEAD     Sadece header al    HEAD /api/users                 Evet
OPTIONS  Izin verilen metodlari sor  OPTIONS /api/users      Evet
```

IDEMPOTENT NE DEMEK? Ayni istegi 1 kere de atsan 10 kere de atsan sonuc ayni olur. GET ile bir kullaniciyi 10 kere sorgulasan hep ayni sonucu alirsin. Ama POST ile 10 kere "kullanici olustur" dersen 10 tane kullanici olusur. Bu kavram API tasariminda ve retry mekanizmalarinda kritiktir.

### HTTP Status Codes (Durum Kodlari)

```
Kod Ailesi   Anlami           Sik Karsilasilan Kodlar
----------------------------------------------------------------------
1xx          Bilgilendirme    101 Switching Protocols (WebSocket)
2xx          Basarili         200 OK, 201 Created, 204 No Content
3xx          Yonlendirme      301 Moved Permanently, 302 Found, 304 Not Modified
4xx          Client Hatasi    400 Bad Request, 401 Unauthorized
                              403 Forbidden, 404 Not Found
                              429 Too Many Requests (rate limit)
5xx          Server Hatasi    500 Internal Server Error
                              502 Bad Gateway (upstream sorun)
                              503 Service Unavailable
                              504 Gateway Timeout
```

CLOUD NATIVE IPUCU:
- 502 Bad Gateway: Genelde load balancer arkasindaki servis cokmus demek
- 503 Service Unavailable: Servis var ama hazirlari degil (pod starting)
- 504 Gateway Timeout: Backend cok yavas, timeout doldu
- 429 Too Many Requests: Rate limiting devrede, istekleri azalt

Bu kodlari Kubernetes'te Ingress, Service ve Pod durumlarini debug ederken surekli goreceksin.

### HTTPS ve TLS

HTTPS = HTTP + TLS (Transport Layer Security). Veriyi sifrelleyerek guvenli iletisim saglar.

```
TLS Handshake Sureci (Basitlestirilmis):

Client                                    Server
  |                                          |
  |------- ClientHello (TLS versiyon) ------>|
  |                                          |
  |<------ ServerHello + Sertifika ----------|
  |                                          |
  |  (Sertifikayi dogrula:                   |
  |   - CA tarafindan imzalanmis mi?         |
  |   - Sure gecmemis mi?                    |
  |   - Domain adi uyuyor mu?)               |
  |                                          |
  |------- Key Exchange (sifreleme) -------->|
  |                                          |
  |<======= Sifrelenmis Iletisim ===========>|
```

TLS 1.3 (Guncel Standart):
- 1-RTT handshake (eskisi 2-RTT idi) -> daha hizli baglanti
- Eski guvenli olmayan algoritmalar kaldirildi
- 0-RTT resumption destegi (daha da hizli yeniden baglanti)

### HTTP Versiyonlari Karsilastirmasi

```
Ozellik          HTTP/1.1        HTTP/2           HTTP/3
----------------------------------------------------------------------
Yil              1997            2015             2022 (RFC 9114)
Transport        TCP             TCP              QUIC (UDP uzerinde)
Multiplexing     Yok (pipeline)  Evet             Evet
Header Sikistirma Yok            HPACK            QPACK
Sifreleme        Opsiyonel       Pratikte zorunlu Zorunlu
Head-of-line     Var             Kismi cozum      Tamamen cozuldu
Baglanti kurma   Yavas (TCP+TLS) Daha iyi         En hizli (1-RTT)
```

HTTP/3 ve QUIC protokolu 2025 itibariyle global web trafiginin %35'ini olusturuyor (Cloudflare verileri). Mobil cihazlarda gecikmeyi %30'a kadar azaltiyor (Akamai raporu 2025). Chrome, Firefox, Safari ve Edge tamami HTTP/3 destekliyor.

### gRPC - Mikroservisler Icin

Cloud native dunyada REST API'nin yaninda gRPC cok yaygin. Ozellikle servisler arasi (east-west) iletisimde tercih edilir:

```
Ozellik          REST              gRPC
----------------------------------------------------------------------
Protokol         HTTP/1.1, HTTP/2  HTTP/2 (zorunlu)
Format           JSON (text)       Protocol Buffers (binary)
Hiz              Normal            Cok hizli (10x'e kadar)
Streaming        Sinirli           Tam destek (bi-directional)
Kod uretimi      Manuel            Otomatik (proto dosyasindan)
Browser destegi  Tam               Sinirli (gRPC-Web gerekir)
Kullanim         Dis API           Servisler arasi iletisim
```

Ornek gRPC kullanim senaryosu:
```
Mikroservis mimarisi:
  Frontend (React) --REST/JSON--> API Gateway --gRPC--> User Service
                                               --gRPC--> Order Service
                                               --gRPC--> Payment Service

Dis dunyaya REST, ic iletisimde gRPC. Bu pattern cloud native'de standart.
```

---

## <a id="load-balancing"></a>5. LOAD BALANCING KAVRAMI

### Load Balancing Nedir?

Load balancer (yuk dengeleyici), gelen trafigi birden fazla sunucuya dagitir. Amac:
- Tek bir sunucuyu asiri yukten korumak
- Yuksek erisilebilirlik (high availability) saglamak
- Olceklenebilirlik (scalability) sunmak

### Gercek Hayat Analojisi

```
Bir bankayi dusun:

Load Balancer OLMADAN:
  Tek gise acik -> 100 kisi sirada bekliyor -> 1 saat bekleme

Load Balancer ILE:
  5 gise acik -> Kapidaki gorevli (load balancer) musteri yonlendiriyor
  -> Her gisede ~20 kisi -> 12 dk bekleme

  Bir gise kapanirsa? -> Diger 4 gise devam ediyor (yuksek erisilebilirlik)
  Cok kalabalik mi? -> 3 gise daha ac (olceklendirme / auto-scaling)
```

### Layer 4 vs Layer 7 Load Balancing

Bu ayrimi bilmek cloud native'de zorunlu:

```
Layer 4 (Transport) Load Balancing:
  - TCP/UDP seviyesinde calisir
  - IP adresi ve port numarasina bakar
  - Paket icerigini BILMEZ (HTTP header'lari gormez)
  - Daha hizli (daha az islem)
  - Ornek: AWS NLB, Kubernetes Service (ClusterIP, NodePort)

  Client --[TCP paket]--> L4 LB --[TCP paket]--> Server1
                                --[TCP paket]--> Server2

Layer 7 (Application) Load Balancing:
  - HTTP/HTTPS seviyesinde calisir
  - URL path, header, cookie, hostname bilir
  - Akilli yonlendirme yapabilir
  - Daha yavas (paketi acip okumasi lazim) ama daha yetenekli
  - Ornek: AWS ALB, Nginx, Kubernetes Ingress

  Client --[HTTP GET /api]--> L7 LB ---> API Server
  Client --[HTTP GET /web]--> L7 LB ---> Web Server
  Client --[Host: a.com ]--> L7 LB ---> Tenant A Server
```

### Load Balancing Algoritmalari

```
Algoritma           Nasil Calisir                    Ne Zaman Kullanilir
----------------------------------------------------------------------
Round Robin         Sirayla dagitir (1,2,3,1,2,3)    Sunucular esit gucluyse
Weighted Round      Agirlikli sirayla                 Farkli guclerde sunucular
  Robin             (guclu olana daha fazla)
Least Connections   En az baglantisi olana            Uzun sureli baglantilar
                    yonlendirir
IP Hash             Client IP'sine gore sabit         Session persistence
                    sunucu atar                       (sticky session)
Random              Rastgele secer                    Basit durumlar
Least Response      En hizli cevap verene             Performans odakli
  Time              yonlendirir
```

### Kubernetes'te Load Balancing

```
                    Internet
                       |
              +--------v--------+
              |  Cloud Load     |  <- AWS ALB/NLB, GCP LB
              |  Balancer (L4/7)|
              +--------+--------+
                       |
              +--------v--------+
              |  Ingress        |  <- Nginx Ingress, Traefik (L7)
              |  Controller     |     path/host bazli routing
              +--------+--------+
                       |
              +--------v--------+
              |  Service        |  <- ClusterIP (L4)
              |  (kube-proxy)   |     iptables/IPVS ile dagitim
              +--------+--------+
                    /     \
              +----v-+   +-v----+
              | Pod1 |   | Pod2 |  <- Gercek uygulama pod'lari
              +------+   +------+
```

BILMEN GEREKEN: Kubernetes'te 3 katman load balancing vardir:
1. Cloud LB (dis dunyadan cluster'a)
2. Ingress Controller (L7 - URL/host bazli routing)
3. kube-proxy/Service (L4 - pod'lar arasi dagitim)

### Health Check Turleri

Load balancer'in arkasindaki sunucularin sagligini kontrol etmesi lazim:

```yaml
# Kubernetes'te health check ornegi
livenessProbe:        # Pod canli mi? (degilse restart et)
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10

readinessProbe:       # Pod trafik almaya hazir mi?
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5

startupProbe:         # Uygulama baslatildi mi? (yavas baslayan uygulamalar)
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

KRITIK FARK:
- livenessProbe basarisiz -> Pod restart edilir
- readinessProbe basarisiz -> Pod Service'den cikarilir (trafik gelmez ama restart OLMAZ)
- startupProbe basarisiz -> Diger probe'lar calismaz, beklenir

---

## <a id="reverse-proxy"></a>6. REVERSE PROXY NEDIR?

### Forward Proxy vs Reverse Proxy

Bu ikisi cok karistirilir, net ayiralim:

```
FORWARD PROXY (Ileri Vekil):
  Client biliniyor, Server bilinmiyor.
  Client'in kimligini gizler.

  [Kullanici] --> [Forward Proxy] --> [Internet/Sunucular]
  Ornek: Sirket proxy'si, VPN
  "Ben (client) proxy uzerinden dis dunyaya cikiyorum"


REVERSE PROXY (Ters Vekil):
  Client bilinmiyor, Server biliniyor.
  Server'in kimligini gizler.

  [Internet/Kullanicilar] --> [Reverse Proxy] --> [Backend Sunucular]
  Ornek: Nginx, Traefik, Envoy
  "Dis dunya benim proxy'me geliyor, ben arkadaki sunuculara yonlendiriyorum"
```

### Reverse Proxy Ne Ise Yarar?

```
1. YUKU DAGITMA (Load Balancing)
   Gelen istekleri birden fazla backend'e dagitir.

2. SSL SONLANDIRMA (SSL Termination)
   HTTPS baglantisini reverse proxy karsilar, backend'e duz HTTP gider.
   Backend'ler sertifika yonetimiyle ugrasmaz.

   [Client] --HTTPS--> [Reverse Proxy] --HTTP--> [Backend]

3. ONBELLEKLEME (Caching)
   Sik istenen icerikleri cache'ler, backend'e gereksiz istek gitmez.

4. SIKISTIRMA (Compression)
   Yanit verilerini gzip/brotli ile sikistirir, bant genisligi kazandirir.

5. GUVENLIK
   - Backend IP adreslerini gizler
   - DDoS korumasi saglayabilir
   - WAF (Web Application Firewall) ozelligi olabilir
   - Rate limiting uygulayabilir

6. A/B TESTING ve CANARY DEPLOYMENT
   Trafigi yuzdesel olarak farkli versiyonlara yonlendirir.
```

### Cloud Native'de Reverse Proxy Secenekleri

#### Nginx

```
En bilinen ve yaygin kullanilan reverse proxy/web server.
Kubernetes'te Nginx Ingress Controller olarak calisir.

Artilari:
  + Cok stabil ve olgun
  + Genis topluluk ve dokumantasyon
  + Dusuk kaynak tuketimi
  + Yaygin bilgi birikimi

Eksileri:
  - Config dosyasi ile yapilandirilir (dinamik degil)
  - Guncelleme icin reload gerekir
  - Modern cloud native ozellikler sinirli
```

Basit Nginx reverse proxy ornegi:
```
# /etc/nginx/conf.d/reverse-proxy.conf
server {
    listen 80;
    server_name api.example.com;

    location /api/users {
        proxy_pass http://user-service:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /api/orders {
        proxy_pass http://order-service:8080;
    }
}
```

#### Envoy

```
Modern, cloud native icin tasarlanmis yuksek performansli proxy.
Istio service mesh'in data plane'i Envoy uzerine kuruludur.

Artilari:
  + xDS API ile dinamik konfigürasyon (restart gerekmez)
  + Detayli metrik ve tracing (Prometheus, Jaeger entegrasyonu)
  + gRPC native destegi
  + HTTP/3 destegi
  + Service mesh'lerin tercihi

Eksileri:
  - Konfigurasyonu daha karmasik
  - Nginx'e gore daha fazla kaynak tuketir
  - Ogrenme egrisi daha dik
```

#### Traefik

```
Cloud native ve container odakli modern reverse proxy.
Docker ve Kubernetes ile otomatik entegrasyon.

Artilari:
  + Otomatik servis kesfetme (auto-discovery)
  + Let's Encrypt ile otomatik SSL sertifikasi
  + Dashboard ile gorsel yonetim
  + Kubernetes Ingress olarak dogrudan calisir
  + Konfigurasyonu basit

Eksileri:
  - Cok yuksek trafik altinda Nginx/Envoy kadar performansli degil
  - Enterprise ozellikleri ucretli
```

### Kubernetes Ingress Ornegi (Nginx Ingress)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-api
                port:
                  number: 8080
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
```

Bu ornekte:
- myapp.example.com/api -> backend-api servisine gider
- myapp.example.com/ -> frontend servisine gider
- TLS sertifikasi ile HTTPS zorunlu
- Ingress Controller (Nginx) burada reverse proxy gorevi gorur

---

## <a id="firewall"></a>7. FIREWALL TEMELLERI

### Firewall Nedir?

Firewall (ates duvari), ag trafigini belirlenen kurallara gore filtreler. Izin verilen trafigi gecirir, izin verilmeyeni engeller. Bir binanin guvenlik gorevlisi gibi dusun: kimlik kontrol eder, listede olani icceri alir, olmayani almaz.

### Firewall Calisma Prensibi

```
              INTERNET
                 |
         +-------v-------+
         |   FIREWALL    |
         |               |
         | Kural 1: IZIN | -> Port 80 (HTTP) dis dunyadan gelsin
         | Kural 2: IZIN | -> Port 443 (HTTPS) dis dunyadan gelsin
         | Kural 3: IZIN | -> Port 22 (SSH) sadece 10.0.0.0/24'ten
         | Kural 4: REDDET| -> Geri kalan her sey ENGELLE
         |               |
         +-------+-------+
                 |
           IC NETWORK
```

ALTIN KURAL: Default olarak her seyi engelle (deny all), sadece gereken trafige izin ver (whitelist yaklasimi). Buna "least privilege" prensibi denir ve guvenligin temelidir.

### Stateful vs Stateless Firewall

```
STATELESS FIREWALL:
  - Her paketi bagimsiz degerlendirir
  - Baglanti durumunu takip etmez
  - Daha hizli ama daha az akilli
  - Ornek: AWS NACL (Network ACL)

  Kural: "Port 80'e gelen trafige izin ver"
  Sorun: Yanit trafigi icin de ayri kural yazmak lazim!

STATEFUL FIREWALL:
  - Baglanti durumunu takip eder
  - Giden istege karsilik gelen yaniti otomatik gecirir
  - Daha akilli
  - Ornek: AWS Security Group, iptables (default)

  Kural: "Port 80'e gelen trafige izin ver"
  Yanit trafigi: Otomatik olarak gecirilir (stateful)
```

### Linux'ta iptables

iptables, Linux cekirdegindeki Netfilter framework'unun kullanici araci. Kubernetes'in kube-proxy bileseni iptables kurallarini kullanarak Service routing'i yapar.

```
iptables Zincirleri (Chains):
  INPUT    -> Makineye gelen trafik
  OUTPUT   -> Makineden cikan trafik
  FORWARD  -> Makineden gecen trafik (router gibi)

iptables Tablolari:
  filter   -> Paket filtreleme (default)
  nat      -> Adres cevirmesi (NAT)
  mangle   -> Paket degistirme

Ornek Kurallar:
  # Port 80'e gelen trafige izin ver
  iptables -A INPUT -p tcp --dport 80 -j ACCEPT

  # Belirli IP'den SSH'a izin ver
  iptables -A INPUT -p tcp -s 10.0.1.50 --dport 22 -j ACCEPT

  # Geri kalan her seyi engelle
  iptables -A INPUT -j DROP

  # NAT kurali (Kubernetes Service benzeri)
  iptables -t nat -A PREROUTING -p tcp --dport 80 \
    -j DNAT --to-destination 10.244.1.5:8080
```

BILGI: nftables, iptables'in modern halefidir. Daha iyi performans ve daha temiz syntax sunar. Yeni Linux dagitimlarinda (Debian 10+, RHEL 8+) nftables default olarak gelir. Ancak Kubernetes ekosisteminde hala iptables yaygin sekilde kullanilmaktadir.

### Cloud Ortaminda Guvenlik Katmanlari

```
+------------------------------------------------------------------+
|                        CLOUD GUVENLIK KATMANLARI                  |
|                                                                    |
|  1. Cloud Firewall / Security Group                               |
|     AWS: Security Groups + NACLs                                  |
|     Azure: NSG (Network Security Group)                           |
|     GCP: Firewall Rules                                           |
|     -> VM/instance seviyesinde trafik kontrolu                    |
|                                                                    |
|  2. Kubernetes Network Policy                                     |
|     -> Pod seviyesinde trafik kontrolu                            |
|     -> Namespace izolasyonu                                       |
|                                                                    |
|  3. Service Mesh (Istio/Linkerd)                                  |
|     -> Servis seviyesinde trafik kontrolu                         |
|     -> mTLS ile sifreleme                                         |
|     -> Authorization Policy                                       |
|                                                                    |
|  4. Application Level                                             |
|     -> WAF (Web Application Firewall)                             |
|     -> API Gateway rate limiting                                  |
|     -> Input validation                                           |
+------------------------------------------------------------------+
```

### Kubernetes Network Policy

Bu konu cloud native icin zorunlu bilgi:

```yaml
# Ornek: Sadece frontend pod'larindan backend'e erisime izin ver
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend          # Bu policy backend pod'larina uygulanir
  policyTypes:
    - Ingress               # Gelen trafigi kontrol et
    - Egress                # Giden trafigi kontrol et
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend   # Sadece frontend'den gelen trafige izin ver
        - namespaceSelector:
            matchLabels:
              env: production # Sadece production namespace'inden
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database   # Backend sadece database'e gidebilir
      ports:
        - protocol: TCP
          port: 5432          # PostgreSQL portu
    - to:                     # DNS erisimi (zorunlu!)
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

DIKKAT: Network Policy uygularken DNS (port 53) erisimini UNUTMA! Aksi halde pod'lar servis isimlerini cozemez ve hicbir yere baglanamaz. Bu en sik yapilan hatadir.

### AWS Security Group Ornegi

```
Inbound Rules (Gelen):
  Tip       Protokol  Port   Kaynak              Aciklama
  ----------------------------------------------------------------
  HTTPS     TCP       443    0.0.0.0/0           Herkesten HTTPS
  HTTP      TCP       80     0.0.0.0/0           Herkesten HTTP
  SSH       TCP       22     10.0.0.0/24         Sadece VPN'den SSH
  Custom    TCP       8080   sg-0abc123           LB security group'undan

Outbound Rules (Giden):
  Tip       Protokol  Port   Hedef               Aciklama
  ----------------------------------------------------------------
  All       All       All    0.0.0.0/0           Tum cikis trafigine izin
```

---

## <a id="cloud-native-extras"></a>8. CLOUD NATIVE NETWORKING - EKSTRA KONULAR

Bu bolumde cloud native alaninda calisan birinin mutlaka bilmesi gereken ek network konularini anlatiyorum. Bunlar standart network mufredatinda yer almaz ama Kubernetes ve cloud ortaminda kritik oneme sahiptir.

### CNI (Container Network Interface)

Kubernetes, pod networking'i kendisi yapmaz. CNI eklentileri uzerinden yapar. CNI secimin cluster'in performansini, guvenligini ve ozelliklerini dogrudan etkiler.

```
Populer CNI Eklentileri:

  Calico:
    + Network Policy destegi (en kapsamli)
    + BGP routing destegi
    + Genis kullanim alani
    + On-premise ve cloud uyumlu
    -> En yaygin secim, cogu durumda iyi calisir

  Cilium:
    + eBPF tabanli (kernel seviyesinde calisir)
    + kube-proxy'yi tamamen degistirebilir
    + Hubble ile detayli ag gozlemleme
    + HTTP/gRPC bazli network policy
    + En yuksek performans
    -> Modern, performans odakli tercih (2025-2026 trendi)

  Flannel:
    + Basit ve hafif
    + Kurulumu kolay
    - Network Policy destegi yok (baska araclara ihtiyac)
    -> Ogrenme ve test ortamlari icin

  Weave Net:
    + Kurulumu basit
    + Sifreleme destegi
    - Performansi digerleri kadar iyi degil
    -> Kucuk cluster'lar icin

  AWS VPC CNI:
    + AWS native (VPC IP'lerini dogrudan pod'lara atar)
    + En iyi AWS performansi
    - Sadece AWS'te calisir
    - IP adresi sinirlamasi (instance basina IP limiti)
    -> EKS kullaniyorsan varsayilan secim
```

TREND BILGISI: 2025-2026 itibariyle Cilium, CNCF graduated projesi olarak cloud native networking'in gelecegi olarak gorulmektedir. eBPF teknolojisi sayesinde iptables'a bagimliligi ortadan kaldirarak belirgin performans artisi saglar. Azure AKS artik varsayilan olarak Azure CNI powered by Cilium sunuyor.

### eBPF (Extended Berkeley Packet Filter)

```
eBPF Nedir?
  Linux cekirdegi icinde sanal makine gibi calisir.
  Kernel'i degistirmeden, kernel seviyesinde program calistirmanizi saglar.

Network icin ne yapar?
  - Paket islemlerini kernel seviyesinde hizlandirir
  - iptables kurallarini devre disi birakabilir (daha hizli)
  - Detayli network gozlemleme (hangi pod nereye ne kadar trafik gonderiyor)
  - Guvenlik politikalarini kernel'de uygular

Cloud Native'deki Kullanim Alanlari:
  1. Cilium (networking + guvenlik)
  2. Hubble (network gozlemleme)
  3. Falco (runtime guvenlik)
  4. Pixie (uygulama izleme)
  5. Tetragon (guvenlik olaylari)

Neden onemli?
  Geleneksel: Paket -> iptables kurallarini gez (yavas) -> Karar ver
  eBPF:       Paket -> eBPF programi (kernel icinde, cok hizli) -> Karar ver
```

### Service Mesh Networking

Service mesh, mikroservisler arasi iletisimi yonetir. Her pod'un yanina bir sidecar proxy (genelde Envoy) koyar:

```
Service Mesh Olmadan:
  [Service A] ----dogrudan----> [Service B]
  Sorun: Sifreleme yok, retry yok, observability yok

Service Mesh Ile:
  [Service A] -> [Sidecar Proxy] --mTLS--> [Sidecar Proxy] -> [Service B]
  Kazanim: Otomatik sifreleme, retry, circuit breaker, metrikler

Karsilastirma:
  Istio:
    + En kapsamli ozellik seti
    + Trafik yonetimi (A/B testing, canary, fault injection)
    + Guclu guvenlik (mTLS, authorization policy)
    - Kaynak tuketimi yuksek
    - Karmasiklik

  Linkerd:
    + Hafif ve basit
    + Dusuk kaynak tuketimi
    + Rust tabanli proxy (daha hizli)
    - Istio kadar ozellik zengin degil
```

### mTLS (Mutual TLS)

```
Normal TLS:
  Client sunucunun kimligini dogrular.
  "Sen gercekten google.com musun?"

mTLS (Karsilikli TLS):
  Hem client hem sunucu birbirinin kimligini dogrular.
  "Sen gercekten User Service misin?" + "Sen gercekten Order Service misin?"

Kubernetes'te neden onemli?
  - Cluster icinde tum trafik sifrelenir
  - Sahte servisler iletisim kuramaz
  - Zero-trust security modeli
  - Service mesh (Istio/Linkerd) ile otomatik mTLS

  Istio'da mTLS:
    apiVersion: security.istio.io/v1beta1
    kind: PeerAuthentication
    metadata:
      name: default
      namespace: production
    spec:
      mtls:
        mode: STRICT   # Tum trafik sifrelenmek ZORUNDA
```

### VXLAN ve Overlay Networking

```
Overlay Network Nedir?
  Fiziksel ag uzerine kurulmus sanal ag.
  Farkli fiziksel makinelerdeki pod'lar ayni sanal agdaymis gibi iletisim kurar.

  Fiziksel Ag:   Node1 (10.0.1.5)  <-------->  Node2 (10.0.1.6)
  Overlay Ag:    Pod1 (10.244.1.2)  <-------->  Pod2 (10.244.2.3)

  Pod1'in paketi -> VXLAN ile kapsullenir -> Fiziksel ag uzerinden gider
  -> Node2'de acilir -> Pod2'ye ulasir

VXLAN (Virtual Extensible LAN):
  - Layer 2 frame'i UDP paketine kapsulleyerek Layer 3 uzerinden tasir
  - Port: UDP 4789
  - 16 milyon sanal ag destekler (VLAN'in 4096 limitini asar)
  - Kubernetes CNI eklentilerinin cogu VXLAN kullanir

Overlay vs Native Routing:
  Overlay: Daha esnek, her yerde calisir, biraz performans kaybi
  Native:  Daha hizli, bulut saglayicilarina ozel (AWS VPC CNI gibi)
```

### Onemli Network Portlari (Ezberle!)

```
Port      Protokol/Servis        Cloud Native Kullanim
----------------------------------------------------------------------
22        SSH                    Sunucu yonetimi
53        DNS                    CoreDNS, servis kesfetme
80        HTTP                   Web trafigi
443       HTTPS                  Guvenli web trafigi
2379      etcd client            Kubernetes veri deposu
2380      etcd peer              etcd cluster iletisimi
5432      PostgreSQL             Veritabani
6379      Redis                  Cache/session store
6443      Kubernetes API Server  kubectl, tum kontrol islemleri
8080      HTTP alternatif        Uygulama sunuculari
8443      HTTPS alternatif       Guvenli uygulama sunuculari
9090      Prometheus             Metrik toplama
3000      Grafana                Dashboard
10250     Kubelet API            Node ile iletisim
10255     Kubelet read-only      Kubelet okuma
30000-    NodePort araligi       Kubernetes NodePort servisleri
  32767
```

### Network Troubleshooting Araclari

Cloud native ortaminda sorun giderme icin bilmen gereken araclar:

```
ARAC           KULLANIM                              ORNEK
----------------------------------------------------------------------
ping           Hedefe ulasilabilirlik testi           ping 10.0.1.5
traceroute     Paket rotasini goster                  traceroute google.com
nslookup/dig   DNS sorgusu                            dig backend-api.default.svc.cluster.local
curl           HTTP istegi gonder                     curl -v http://service:8080/health
wget           Dosya indir / HTTP test                wget -qO- http://service/api
nc (netcat)    Port acik mi kontrol                   nc -zv 10.0.1.5 8080
ss / netstat   Acik portlar ve baglantilar            ss -tlnp
tcpdump        Paket yakalama (detayli analiz)        tcpdump -i eth0 port 8080
ip             IP/route bilgisi                       ip addr show; ip route
iptables       Firewall kurallarini goruntule         iptables -L -n
nmap           Port tarama                            nmap -sT 10.0.1.5

Kubernetes ozel:
  kubectl exec <pod> -- curl ...          Pod icinden test
  kubectl exec <pod> -- nslookup ...      DNS testi
  kubectl logs <pod>                      Uygulama loglari
  kubectl describe service <svc>          Servis detaylari
  kubectl get endpoints <svc>             Servis endpoint'leri
```

### Pratik Senaryo: Kubernetes Network Sorun Giderme

```
SENARYO: Frontend pod'u backend servisine ulasamiyor.
HATA: "curl: (7) Failed to connect to backend-api port 8080"

ADIM 1: DNS cozumleniyor mu?
  $ kubectl exec frontend-pod -- nslookup backend-api
  # Sonuc: "backend-api.default.svc.cluster.local = 10.96.50.100"
  # DNS OK ise -> Adim 2

ADIM 2: Service endpoint'leri var mi?
  $ kubectl get endpoints backend-api
  # Sonuc: "10.244.1.5:8080, 10.244.2.8:8080"
  # Endpoint varsa -> Adim 3
  # Endpoint YOKSA -> Pod label'lari Service selector'u ile eslesmiyor!

ADIM 3: Pod'a dogrudan ulasilabiliyor mu?
  $ kubectl exec frontend-pod -- curl -v http://10.244.1.5:8080
  # Basariliysa -> Service veya DNS sorunu
  # Basarisizsa -> Adim 4

ADIM 4: Network Policy engelliyor mu?
  $ kubectl get networkpolicy -n default
  # Varsa kurallari incele: frontend -> backend trafigine izin var mi?

ADIM 5: Pod calisiyor mu?
  $ kubectl get pod -l app=backend
  $ kubectl logs backend-pod
  # CrashLoopBackOff ise uygulama hatasi var
  # Running ise ama cevap vermiyorsa uygulama icinde port dogru mu?
```

---

## OZET VE TAVSIYELER

### Cloud Native Icin En Kritik Network Bilgileri (Oncelik Sirasi)

```
1. DNS calisma mantigi + Kubernetes DNS (CoreDNS)
   -> Pod'lar arasi iletisimin temeli

2. IP adresleme + CIDR/Subnetting
   -> VPC, subnet, pod network tasarimi

3. HTTP/HTTPS + TLS
   -> Tum web iletisiminin temeli, Ingress konfigurasyonu

4. Load Balancing (L4 vs L7)
   -> Service, Ingress, Cloud LB katmanlari

5. Network Policy + Firewall
   -> Guvenligi saglamak (zero-trust)

6. CNI ve Overlay Networking
   -> Pod networking altyapisi

7. Service Mesh + mTLS
   -> Mikroservis guvenligi ve yonetimi
```

### Pratik Gorevler Listesi

Bu konulari ogrendikten sonra asagidaki pratik gorevleri tamamla:

```
- [ ] Nginx web server kurup reverse proxy yapilandirmasi yap
- [ ] DNS kayitlari olustur (A, CNAME, MX)
- [ ] Port forwarding ile uzak servise eris
- [ ] tcpdump ile HTTP trafigini yakala ve incele
- [ ] Kubernetes'te Network Policy yaz ve test et
- [ ] Ingress Controller kur ve path-based routing yap
- [ ] curl ile HTTP status code'larini test et
- [ ] nslookup/dig ile DNS sorgulari yap
- [ ] Subnetting hesaplamalari yap (en az 5 ornek)
- [ ] Bir servisin Layer 4 ve Layer 7'de nasil farklilastigini goster
```

### Onerilen Kaynaklar

```
Kitaplar:
  - "Computer Networking: A Top-Down Approach" - Kurose & Ross
  - "TCP/IP Illustrated" - W. Richard Stevens

Online Kaynaklar:
  - Cloudflare Learning Center (cloudflare.com/learning)
  - Practical Networking YouTube kanali
  - Julia Evans'in networking zine'leri (wizardzines.com)
  - Learnk8s.io (Kubernetes networking makaleleri)

Araclar (pratik icin):
  - Wireshark (gorsel paket analizi)
  - Postman/Insomnia (HTTP API testi)
  - subnet calculator (online araçlar)
  - Kind/Minikube (Kubernetes lab ortami)

Hands-on Lab:
  - "Kubernetes the Hard Way" by Kelsey Hightower
    (Network altyapisini sifirdan kurarak ogrenmek icin muhtesem)
```

---

Son Guncelleme: Subat 2026
Kubernetes Version: 1.29+
Bu rehber, cloudnative dosyasindaki Faz 0 - Networking Temelleri bolumunu kapsamli sekilde tamamlar.

---

*"Network'u anlamadan cloud native uzmanı olamazsın. Her sorunun kokunun %70'i network'tur."*
