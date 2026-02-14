# Linux Rehberi - 10 Yıllık Deneyim ile Öğretme Temelli Kılavuz

Merhaba! Ben, 10 yıllık Linux deneyimim ile sizi Linux dünyasına ve bu harika işletim sisteminin derinliklerine davet ediyorum. Bu rehberi yazarken başlangıç seviyesinden ileri seviyeye kadar herkesin anlayabileceği şekilde hazırladım.

---

## 1. LINUX DOSYA SİSTEMİ YAPISI

### Dosya Sistemi Hiyerarşisi

Linux, ağaç yapısına benzer bir hiyerarşide dosyaları organize eder. Tüm yollar `/` (root) ile başlar.

```
/
├── bin         → Temel sistem komutları (ls, cp, rm vb.)
├── boot        → Sistem önyükleme dosyaları
├── dev         → Cihaz dosyaları (disk, terminal vb.)
├── etc         → Yapılandırma dosyaları
├── home        → Kullanıcı ana dizinleri
├── lib         → Kütüphane dosyaları
├── media       → Çıkarılabilir medya bağlama noktaları
├── mnt         → Geçici bağlama noktaları
├── opt         → İsteğe bağlı yazılımlar
├── proc        → Sistem bilgileri (RAM'de dinamik)
├── root        → root kullanıcısının ana dizini
├── run         → Çalışan işlemlerin bilgileri
├── sbin        → Sistem yönetimi komutları
├── srv         → Servis verileri
├── tmp         → Geçici dosyalar (sistem kapanınca silinir)
├── usr         → Kullanıcı yazılımları ve belgeleri
├── var         → Değişken veriler (log dosyaları vb.)
└── sys         → Sistem bilgileri
```

### Önemli Konseptler

**Gizli Dosyalar**: Noktayla (.) başlayan dosyalar gizlidir.
```bash
.bashrc         # Gizli bash yapılandırma dosyası
.ssh            # SSH anahtarları gizli klasöründe tutulur
```

**Dosya Yolu Türleri**:
```bash
/home/user/dosya.txt     # Mutlak yol (/ ile başlar)
./document/file.txt      # Göreli yol (./ ile başlar)
../parent_dosya.txt      # Üst dizine gitme (..)
```

---

## 2. TEMEL KOMUTLAR

### ls - Dosyaları Listele

```bash
ls                       # Mevcut dizini listele
ls -l                    # Detaylı listeyle
ls -la                   # Gizli dosyalar dahil, detaylı
ls -lh                   # İnsan tarafından okunabilir boyutlarla
ls -ltr                  # Zaman sırasına göre (en eski ilk)
ls -R                    # Rekürsif (alt klasörleri de göster)
ls /home/user/Masaüstü   # Belirli klasörü listele
```

**Örnek Analiz**:
```
-rw-r--r-- 1 user group 2048 Jan 15 10:30 dosya.txt
│││││││││ │ │    │     │    │  │   │  │      │
││ bit   │ │    │     │    │  │   │  │      └─ Dosya adı
││ sayısı│ │    │     │    │  │   │  └──── Saat:Dakika
││       │ │    │     │    │  │   └─────── Gün
││       │ │    │     │    │  └─────────── Ay
││       │ │    │     │    └────────────── Yıl
││       │ │    │     └─────────────────── Boyut (Byte)
││       │ │    └────────────────────────── Grup adı
││       │ │───────────────────────────── Sahibi (user)
││       └────────────────────────────── Link sayısı
└└─ Dosya izinleri (sahip-grup-diğerleri)
```

### cd - Dizini Değiştir

```bash
cd /home/user            # Belirli klasöre git
cd ~                     # Ana dizine git (home)
cd ..                    # Üst dizine git
cd -                     # Önceki dizine geri dön
cd                       # Direkt çağırılırsa home'a gider
```

### mkdir - Klasör Oluştur

```bash
mkdir yeni_klasor          # Basit klasör oluştur
mkdir -p a/b/c/d           # İç içe klasör oluştur (-p recurrence için)
mkdir -v klasor1 klasor2   # Oluşturma sırasında bildiriş ver
```

### rm - Dosya/Klasör Sil

```bash
rm dosya.txt             # Dosyayı sil
rm -r klasor/            # Klasörü ve içeriğini sil (recursive)
rm -f dosya.txt          # Onay istemeden sil (force)
rm -i dosya.txt          # Her silme işleminden önce onay iste (interactive)
rm *.txt                 # Tüm .txt dosyalarını sil
```

**⚠️ Dikkat**: `rm` geri alınamaz! Çöp kutusuna gitmiyor, direkt silinir.

### cp - Dosya/Klasör Kopyala

```bash
cp dosya.txt kopyasi.txt      # Dosya kopyala
cp -r klasor/ klasor_kopya/   # Klasörü rekürsif olarak kopyala
cp dosya.txt /home/user/      # Başka klasöre kopyala
cp *.txt /backup/             # Tüm txt dosyalarını kopyala
```

### mv - Dosya/Klasör Taşı veya Adlandır

```bash
mv eski_ad.txt yeni_ad.txt      # Dosyayı adlandır
mv dosya.txt /home/user/        # Dosyayı taşı
mv klasor/ /yeni/konum/         # Klasörü taşı
mv -i dosya.txt /tmp/           # Hedef varsa sorma (-i)
```

### cat - Dosya İçeriğini Göster

```bash
cat dosya.txt                # Dosyayı ekrana yazdır
cat dosya1.txt dosya2.txt    # Birden fazla dosyayı göster
cat > yeni_dosya.txt         # Yeni dosya oluştur (Ctrl+D ile bitir)
cat >> dosya.txt             # Dosyanın sonuna ekle
cat dosya.txt | less          # Sayfa sayfa göster
```

### grep - Metin Ara

```bash
grep "arama_kelimesi" dosya.txt        # Basit arama
grep -i "HELLO" dosya.txt              # Büyük-küçük harf duyarlılığı yoksay
grep -n "pattern" dosya.txt            # Satır numarası ile göster
grep -r "pattern" /home/user/          # Rekürsif arama
grep -v "pattern" dosya.txt            # Ters eşleştirme (dışlama)
grep "^ERROR" log.txt                  # Satırın başında "ERROR" ile başlayanlar
grep "\.txt$" dosya.txt                # Satırın sonunda ".txt" ile bitenler
cat dosya.txt | grep "pattern"         # Pipe ile başka komuttan gelen veriyi ara
```

**Örnek Kullanım**:
```bash
# log.txt dosyasında ERROR içeren satırları bul
grep ERROR log.txt

# Dosyasında "user@domain" paternini ara
grep "user@domain" contacts.txt

# /etc klasöründe "ssh" içeren satırları tüm dosyalarda ara
grep -r "ssh" /etc/
```

### find - Dosya Bul

```bash
find /home/user -name "*.txt"              # İsim ile bul
find /home/user -type f -name "*.pdf"      # Dosya tipi belirterek bul (f=file)
find /home/user -type d -name "Masaüstü"   # Klasör tipi (d=directory)
find /home/user -size +100M                # 100MB'den büyük dosyalar
find /home/user -size -10k                 # 10KB'den küçük dosyalar
find /home -mtime -7                       # Son 7 gün içinde değiştirilen
find /home/user -name "*.log" -delete      # Bulunanları sil
find /home/user -name "*.txt" -exec rm {} \;  # Her bulu dosyaya komut uygula
```

### Combine: Pipe (|) ile Komutları Zincirle

```bash
cat dosya.txt | grep "ERROR" | wc -l      # ERROR'in kaç kere geçtiğini say
ls -l | grep ".txt"                       # txt dosyalarını listele
find /home -name "*.pdf" | wc -l          # PDF sayısını say
cat /var/log/syslog | head -50            # İlk 50 satırı göster
ps aux | grep "firefox"                   # Firefox işlemini bul
```

---

## 3. VIM VE NANO TEXT EDITOR KULLANIMI

### Nano - Kolay Metin Editörü (Başlangıç için Önerilir)

```bash
nano dosya.txt              # Dosyayı aç veya oluştur
nano +10 dosya.txt          # 10. satırdan başla
```

**Nano Kısayolları**:
```
Ctrl+X      → Çık (değişiklik kaydedilsin mi diye sorar)
Ctrl+S      → Kaydet
Ctrl+A      → Satırın başına git
Ctrl+E      → Satırın sonuna git
Ctrl+V      → Bir sayfa aşağı git
Ctrl+Y      → Bir sayfa yukarı git
Ctrl+W      → Ara
Ctrl+\      → Değiştir
```

**Örnek Nano Kullanışı**:
```bash
$ nano script.sh
# Editörde dosya açılır, yazıyı yaz, Ctrl+X ile çık
```

### Vim - Güçlü Editör (Öğrenme Eğrisi Yüksek ama Çok Güçlü)

```bash
vim dosya.txt               # Dosyayı aç
vim +10 dosya.txt           # 10. satırdan başla
vim +/pattern dosya.txt     # Pattern'i ara ve aç
```

**Vim Modları**:

1. **Normal Mod** (Dosya açılıyor):
   - Komutları yazarız
   - `i` tuşuyla **Insert Mod**'a geçeriz

2. **Insert Mod**:
   - Metin yazarız
   - `Esc` tuşuyla **Normal Mod**'a geri döneriz

3. **Command Mode** (Normal Mod'dan `:` yazarak):
   - Dosya yönetimi komutları

**Temel Vim Komutları**:
```
INSERT MOD'A GIRMEK:
i               → İmleç konumundan önce yazılı başla
I               → Satırın başından yazılı başla
a               → İmleç konumundan sonra yazılı başla
A               → Satırın sonundan yazılı başla
o               → Aşağıya yeni satır ekle
O               → Yukarıya yeni satır ekle

NORMAL MOD KISAYOLARı:
h, j, k, l      → Sol, aşağı, yukarı, sağ (ok tuşları gibi)
w               → Sonraki kelimeye git
b               → Önceki kelimeye git
gg              → Dosyanın başına git
G               → Dosyanın sonuna git
dd              → Satırı sil
yy              → Satırı kopyala
p               → Yapıştır
u               → Geri al (undo)
Ctrl+R          → İleri al (redo)
/pattern        → Arama yap (n ile sonraki, N ile önceki)
:%s/old/new/g   → Tümünü değiştir (old'u new'e)

COMMAND MODE (:):
:w              → Kaydet
:q              → Çık
:wq             → Kaydet ve çık
:q!             → Kaydetsiz çık
:set number     → Satır numarası göster
:set nonumber   → Satır numarasını gizle
:e dosya.txt    → Başka dosya aç
:split dosya    → Dosyayı ayrı pencerede aç
```

**Vim Başlayanlar İçin Pratik**:
```bash
vim test.txt
# i yazıp Insert Mod'a gir
# "Vim öğreniyorum" yaz
# Esc ile Normal Mod'a dön
# :wq yazıp Enter ile kaydet ve çık
```

**Vim'e Alışmak İçin**: `vimtutor` komutunu çalıştır!

---

## 4. PROCESS YÖNETİMİ

### ps - Process'leri Göster

```bash
ps                          # Mevcut terminallerdeki process'leri göster
ps aux                      # Tüm process'leri detaylı göster
ps aux | grep "firefox"     # Belirli process'i bul
ps -ef --forest             # Process'leri ağaç formatında göster
ps -u username              # Belirli kullanıcının process'lerini göster
```

**ps aux Çıktısı Analizi**:
```
USER  PID  %CPU %MEM   VSZ   RSS TTY STAT START   TIME COMMAND
root   1    0.0  0.0  19240 1620 ?   Ss   10:00   0:01 /sbin/init

USER     → Kullanıcı adı
PID      → Process ID (kimlik numarası)
%CPU     → CPU kullanım yüzdesi
%MEM     → RAM kullanım yüzdesi
VSZ      → Sanal bellek (KB)
RSS      → Fiziksel bellek (KB)
STAT     → Process durumu (R=çalışıyor, S=uyuyor, Z=zombie)
COMMAND  → Komut satırı
```

### top - İnteraktif Sistem Monitörü

```bash
top                         # top başlat
top -u username             # Belirli kullanıcının process'lerini göster
top -p 1234                 # Belirli PID'yi izle
```

**top İçinde Komutlar**:
```
q               → Çık
u               → Kullanıcıya göre filtrele
M               → Bellek kullanımına göre sırala
P               → CPU kullanımına göre sırala
k               → Process öldür (PID ister)
h               → Yardım
```

### htop - top'un Daha Güzel Versiyonu

```bash
htop                        # htop başlat (daha modern arayüz)
htop -u username            # Belirli kullanıcıyı izle
```

### kill - Process Öldür

```bash
kill 1234                   # PID 1234 olan process'i tersiyle öldür
kill -9 1234                # Zorla öldür (SIGKILL)
kill -l                     # Bütün signal'leri listele
killall firefox             # İsme göre tüm process'leri öldür
```

**Sık Kullanılan Signaller**:
```
SIGTERM (15)    → Zarif kapatış (default)
SIGKILL (9)     → Zorla kapatış (dinlemez)
SIGHUP (1)      → Configuration reload (bazı servislerde)
SIGSTOP (19)    → Process'i durdur
SIGCONT (18)    → Process'i devam ettir
```

### &, fg, bg - Background/Foreground İşleri

```bash
./program &                 # Program'ı background'da çalıştır
Ctrl+Z                      # Çalışan process'i durdur (suspend)
bg                          # Durumuş process'i background'da çalıştır
fg                          # Background process'i foreground'a getir
fg %1                       # İş 1'i foreground'a getir
jobs                        # Background'daki işleri listele
```

**Örnek**:
```bash
# Uzun işlemler için:
tar -czf backup.tar.gz /home &    # Arkaplan'da başla
jobs                              # İşleri göster
# Sonra fg ile karşı planını getir veya devam et
```

---

## 5. KULLANICI VE İZİN YÖNETİMİ

### chmod - İzinleri Değiştir

*Linux'te her dosya 3 izin türü vardır: Read(R), Write(W), Execute(X)*

```bash
chmod 755 script.sh        # Sahip: rwx, Grup: r-x, Diğerleri: r-x
chmod 644 dosya.txt        # Sahip: rw-, Grup: r--, Diğerleri: r--
chmod 777 dosya.txt        # Herkese tam izin
chmod +x script.sh         # Execute izni ekle
chmod -x script.sh         # Execute izni kaldır
chmod u+w dosya.txt        # Sahibine write izni ekle
chmod g-r dosya.txt        # Gruptan read izni al
chmod o+x script.sh        # Diğerlerine execute izni ekle
chmod -R 755 klasor/       # Klasör ve içeriğe izin ver
```

**İzin Sayıları**:
```
4 = Read (r)
2 = Write (w)
1 = Execute (x)

755 = 7(sahip: r+w+x=4+2+1) 5(grup: r+x=4+1) 5(diğerleri: r+x=4+1)
644 = 6(sahip: r+w=4+2) 4(grup: r=4) 4(diğerleri: r=4)
600 = 6(sahip: r+w=4+2) 0(grup:hiçbir şey) 0(diğerleri: hiçbir şey)
```

### chown - Dosya Sahibi Değiştir

```bash
sudo chown user dosya.txt                    # Sahibi değiştir
sudo chown user:group dosya.txt              # Sahip ve grup değiştir
sudo chown -R user:group klasor/             # Rekürsif değiştir
sudo chown :group dosya.txt                  # Sadece grup değiştir
```

### Kullanıcı Yönetimi Komutları

```bash
whoami                      # Şu anki kullanıcı
id                          # Detaylı ID bilgisi
id username                 # Belirli kullanıcı bilgisi
sudo useradd yeni_user      # Yeni kullanıcı ekle
sudo usermod -aG sudo user  # Kullanıcıyı sudo grubuna ekle
sudo userdel user           # Kullanıcıyı sil
sudo passwd user            # Kullanıcının parolasını değiştir
groups                      # Mevcut kullanıcı grupları
groups username             # Belirli kullanıcının grupları
```

### sudo - Yükseltilmiş İzinler

```bash
sudo komut                  # Komut'u root olarak çalıştır
sudo -u user komut          # Komut'u başka user olarak çalıştır
sudo -i                     # Root shell'ine gir
sudo -l                     # Yapabileceğin sudo komutlarını listele
```

---

## 6. SSH BAĞLANTILAR

### SSH Bağlantısı

```bash
ssh user@hostname               # Uzak sunucuya bağlan
ssh user@192.168.1.100          # IP adresi kullanarak bağlan
ssh -p 2222 user@hostname       # Farklı port kullan
ssh -v user@hostname            # Verbose mode (hata avlamaya yardımcı)
exit                            # SSH oturumundan çık
```

### SSH Anahtarları (Parola Yerine Güvenli)

```bash
ssh-keygen -t rsa -b 4096       # RSA anahtarı oluştur
ssh-keygen -t ed25519           # Modern Ed25519 anahtarı oluştur
```

**Anahtar Kurulumu**:
```bash
# 1. Yerel makinede anahtar oluştur
ssh-keygen -t ed25519
# ~/.ssh/id_ed25519 (private) ve ~/.ssh/id_ed25519.pub (public) oluşur

# 2. Public anahtarı sunucuya kopyala
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@hostname
# VEYA manuel olarak:
cat ~/.ssh/id_ed25519.pub | ssh user@hostname "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

# 3. Artık password'suz bağlanabilirsin
ssh user@hostname
```

### SSH Dosyası Transferi

```bash
scp dosya.txt user@hostname:/home/user/         # Yukarı yükle
scp user@hostname:/home/user/dosya.txt ./       # İndir
scp -r klasor/ user@hostname:/home/user/        # Rekürsif kopyala
scp -P 2222 dosya.txt user@hostname:/tmp/       # Farklı port
```

### SSH Config Dosyası (~/.ssh/config)

```bash
# ~/.ssh/config dosyasını düzen:
Host myserver
    HostName example.com
    User myuser
    Port 22
    IdentityFile ~/.ssh/id_ed25519

# Şimdi sadece bu komutla bağlan:
ssh myserver
```

### SSH Basit Komut Çalıştırma

```bash
ssh user@hostname "ls -la"                      # Uzak komut çalıştır
ssh user@hostname "tar -czf backup.tar.gz /home" # Uzakta backup al
```

---

## 7. BASH SCRIPTING TEMELLERİ

### İlk Script'in

```bash
#!/bin/bash
# Bu ilk satır (shebang) script'in bash ile çalışacağını gösterir

echo "Merhaba Dünya!"     # Ekrana yazdır
echo "Bugün: $(date)"     # Komutu çalıştır ve sonucunu yazdır
```

**Çalıştır**:
```bash
chmod +x script.sh        # Execute izni ver
./script.sh               # Çalıştır
```

### Değişkenler

```bash
#!/bin/bash

# Değişken tanımla (boşluk yok!)
isim="Ahmet"
yaş=25
dosya="$(pwd)/backup.txt"    # Komut sonucu

# Değişkenleri kullan
echo "İsim: $isim"
echo "Yaş: $yaş"
echo "Dosya yolu: $dosya"

# ${} ile uzun vari isimleri koru
echo "${isim}_yedek"    # Yedek.txt yerine isim_yedek
```

### Argümanlar

```bash
#!/bin/bash

# Script'e argümanları geç:
# ./script.sh arg1 arg2 arg3

echo "İlk argüman: $1"
echo "İkinci argüman: $2"
echo "Tüm argümanlar: $@"
echo "Argüman sayısı: $#"
echo "Script adı: $0"
```

**Örnek**:
```bash
#!/bin/bash
# backup_user.sh
user=$1
if [ -z "$user" ]; then
    echo "Kullanım: backup_user.sh <username>"
    exit 1
fi
tar -czf backup_$user.tar.gz /home/$user
echo "Backup tamamlandı!"
```

### Koşullar (if/else)

```bash
#!/bin/bash

# Sayı karşılaştırması
if [ $1 -gt 10 ]; then          # -gt = greater than
    echo "10'dan büyük"
elif [ $1 -eq 10 ]; then        # -eq = equal
    echo "10'a eşit"
else
    echo "10'dan küçük"
fi

# Dosya kontrolü
if [ -f "dosya.txt" ]; then     # -f = file exists
    echo "Dosya var"
fi

if [ -d "klasor" ]; then        # -d = directory exists
    echo "Klasör var"
fi

# String karşılaştırması
if [ "$user" = "root" ]; then   # = = string equals
    echo "Süper kullanıcı"
fi

if [ -z "$user" ]; then         # -z = string is empty
    echo "User tanımlanmamış"
fi
```

**Karşılaştırma Operatörleri**:
```
Sayılar:
-eq     Equal
-ne     Not equal
-lt     Less than
-le     Less than or equal
-gt     Greater than
-ge     Greater than or equal

Dosyalar:
-f      File exists
-d      Directory exists
-e      Exists (file or directory)
-r      Readable
-w      Writable
-x      Executable

Stringler:
=       Equal
!=      Not equal
-z      Empty string
-n      Non-empty string
```

### Döngüler

```bash
#!/bin/bash

# for döngüsü
for i in 1 2 3 4 5; do
    echo "Sayı: $i"
done

# Dosyalar üzerinde döngü
for dosya in *.txt; do
    echo "İşleme alınıyor: $dosya"
done

# C-style for döngüsü
for ((i=1; i<=5; i++)); do
    echo "Artış: $i"
done

# while döngüsü
sayac=1
while [ $sayac -le 5 ]; do
    echo "Sayac: $sayac"
    ((sayac++))
done

# until döngüsü (koşul yanlış olana kadar)
sayac=1
until [ $sayac -gt 5 ]; do
    echo "Sayac: $sayac"
    ((sayac++))
done
```

### Fonksiyonlar

```bash
#!/bin/bash

# Fonksiyon tanımla
greet() {
    echo "Merhaba $1!"
}

backup() {
    local source=$1          # Local değişken
    local dest=$2

    if [ ! -d "$dest" ]; then
        mkdir -p "$dest"
    fi

    cp -r "$source" "$dest"
    echo "Backup tamamlandı!"
}

# Fonksiyonları çağır
greet "Ahmet"
backup "/home/user" "/backup/user"
```

### Hata Kontrolü

```bash
#!/bin/bash

# Komutun başarılı olması gerekirse
if ! mkdir /root/test 2>/dev/null; then
    echo "Hata: Klasör oluşturulamadı!"
    exit 1
fi

# Exit code kontrolü
cp dosya.txt /tmp/
if [ $? -eq 0 ]; then
    echo "Kopyalama başarılı"
else
    echo "Kopyalama başarısız: $?"
    exit 1
fi

# set -e ile herhangi bir hata script'i durduruyor
set -e
mkdir /tmp/test
cd /tmp/test        # Buraya gelmezsek, script durur
```

### Gerçek Hayat Örneği: Backup Script'i

```bash
#!/bin/bash
# backup.sh - Günlük backup scripti

BACKUP_DIR="/backups"
SOURCE="/home/user/documents"
DATE=$(date +%Y-%m-%d_%H-%M-%S)
BACKUP_FILE="$BACKUP_DIR/backup_$DATE.tar.gz"

# Hata kontrolü
if [ ! -d "$SOURCE" ]; then
    echo "Hata: Kaynak klasör bulunamadı: $SOURCE"
    exit 1
fi

# Backup dizini oluştur
mkdir -p "$BACKUP_DIR"

# Backup al
echo "Backup başlanıyor: $SOURCE -> $BACKUP_FILE"
tar -czf "$BACKUP_FILE" "$SOURCE" 2>/dev/null

if [ $? -eq 0 ]; then
    SIZE=$(du -h "$BACKUP_FILE" | cut -f1)
    echo "Backup başarılı! Boyut: $SIZE"

    # Eski backupları sil (7 günden eski)
    find "$BACKUP_DIR" -name "backup_*.tar.gz" -mtime +7 -delete
    echo "Eski backuplar temizlendi"
else
    echo "Hata: Backup başarısız oldu!"
    exit 1
fi
```

---

## 8. PAKET YÖNETİMİ (BONUS)

### APT (Debian/Ubuntu için)

```bash
sudo apt update                 # Paket listesini güncelle
sudo apt upgrade                # Paketleri güncelle
sudo apt install paketismi      # Paket yükle
sudo apt remove paketismi       # Paket kaldır
sudo apt search arama_terimi    # Paket ara
sudo apt list --installed       # Yüklü paketleri listele
apt-cache show paketismi        # Paket bilgisi
```

### YUM (RedHat/CentOS için)

```bash
sudo yum update                 # Güncellemeleri yükle
sudo yum install paketismi      # Paket yükle
sudo yum remove paketismi       # Paket kaldır
sudo yum search arama_terimi    # Paket ara
```

### Kaynak'tan Derleme

```bash
tar -xzf program.tar.gz
cd program/
./configure                     # Konfigüre et
make                            # Derle
sudo make install               # Yükle
```

---

## 9. SISTEM YÖNETİMİ İPUÇLARI (BONUS)

### Disk Kullanımını Kontrol Et

```bash
df -h                          # Disk alanı göster
du -sh /home/user              # Klasörün boyutu
du -sh /home/user/*            # Klasördeki öğelerin boyutu
du -sh /home/user/* | sort -h  # Büyükten küçüğe sırala
find / -type f -size +100M     # 100MB'den büyük dosyalar
```

### RAM ve CPU Kullanımı

```bash
free -h                        # RAM kullanımı
top                            # Canlı monitör
htop                           # Daha güzel monitör
watch -n 1 free -h             # Her 1 saniyede güncelle
```

### Sistem Bilgisi

```bash
uname -a                       # Genel bilgi
lsb_release -a                 # Linux dağıtım bilgisi
cat /proc/cpuinfo              # CPU bilgisi
cat /proc/meminfo              # RAM bilgisi
hostnamectl                    # Bilgisayar adı
```

### Dosya İzinleri Hızlı Ref

```
rwx rwx rwx     755 (tamamen açık)
rw- r-- r--     644 (sadece sahip yazabilir)
r-x r-x r-x     555 (sadece okuma ve çalıştırma)
rwx --- ---     700 (sadece sahip erişebilir)
--- --- ---     000 (hiç kimse erişemez)
```

---

## 10. SÖZ DİZİMİ CHEAT SHEET

### Joker Karakterler

```bash
*       Herhangi sayıda karakter (dosya*)
?       Tek karakter (dosya?.txt)
[abc]   a, b veya c ([aeiou]= sesli harfler)
[a-z]   a'dan z'ye ([0-9]= sayılar)
[^abc]  a, b, c dışında (olumsuz)
```

### Yönlendirme (Redirection)

```bash
>       Çıktıyı dosyaya yönlendir (üzerine yaz)
>>      Çıktıyı dosyaya ekle
<       Dosyayı input olarak gönder
2>      Hataları yönlendir
2>&1    Hataları stdout'a yönlendir
|       Output'u başka komuta gönder
```

### Özel Karakterler

```bash
;       Komutları ayır (sırasıyla çalıştır)
&&      Öncekisi başarılıysa çalıştır
||      Öncekisi başarısızsa çalıştır
!       Geçmiş komutunu tekrar çalıştır
~       Ana dizin (/home/user/)
.       Mevcut dizin
..      Üst dizin
```

---

## 11. SIK YAPILAN HATALAR VE ÇÖZÜMLERI

### Hata 1: "Permission Denied"

```bash
# Sorun: Dosya/klasörde izin yok
-rw-r--r-- 1 user group 1024 dosya.txt

# Çözüm:
chmod u+x dosya.txt       # Execute izni ekle
sudo chown user:user dosya.txt  # Sahibi değiştir
```

### Hata 2: Yanlışlıkla "rm -rf" Kullanmak

```bash
# Önlem:
# Hiçbir zaman çoğaltıcı işemi test etmeden çalıştırma!
rm -rf /                           # ÇOK TEHLIKELI!

# Düşün: iki kere kontrol et, bir kere sil
rm -i *.txt                        # -i flag'ı ile onay iste
```

### Hata 3: SSH Anahtarı İzinleri Yanlış

```bash
# Sorun: SSH private anahtarı herkese açık
# Hata: "Permissions 0644 for '.ssh/id_ed25519' are too open"

# Çözüm:
chmod 600 ~/.ssh/id_ed25519        # Sadece sahip okuyabilir
chmod 644 ~/.ssh/id_ed25519.pub    # Public anahtarı herkese aç
chmod 700 ~/.ssh                   # SSH dizini
```

### Hata 4: Sudo Parola Sorması

```bash
# sudo'yu password'suz kullanmak için:
sudo visudo

# Dosyada şu satırı ekle:
username ALL=(ALL) NOPASSWD: ALL

# UYARI: Güvenlik risktir!
```

---

## KÖK TABLOSU

| Görev | Komut |
|-------|-------|
| Dosya Listele | `ls -la` |
| Klasör Oluştur | `mkdir klasor` |
| Dosya Sil | `rm dosya.txt` |
| İçeriği Göster | `cat dosya.txt` |
| Arama Yap | `grep "pattern" dosya.txt` |
| Dosya Bul | `find /home -name "*.txt"` |
| Process Göster | `ps aux` |
| Canlı Monitör | `top` veya `htop` |
| Process Öldür | `kill PID` |
| İzin Ver | `chmod 755 dosya` |
| SSH Bağlan | `ssh user@host` |
| Dosya Transferi | `scp dosya.txt user@host:~` |
| Backup Al | `tar -czf backup.tar.gz dosya` |
| Disk Alan | `df -h` |
| Klasör Boyutu | `du -sh klasor` |

---

## BİTİRİŞ SÖZLERİ

Linux öğrenme yolculuğunda sabırlı olun. Başlarda zor görünse de, pratik yaptıkça o muhabbet hastanesinden çıkacağız. En önemli şey, hatalar yapmanız ve hatalarınızdan öğrenmenizdir.

İpuçları:
1. **Pratik Yapın**: Komutları terminal'de kendiniz yazın
2. **Man Sayfalarını Okuyun**: `man komut` ile detaylı bilgi alın
3. **Scriptler Yazın**: Bash scripting pratiği en iyi öğreticidir
4. **Hata Ayıklamayı Öğrenin**: `set -x` ile debug modu kullanın
5. **Bir Linux Sunucu Kurun**: VirtualBox'ta Linux VM'si çalıştırın

Sorularınız varsa, `man` komutuna güvenin:
```bash
man ls          # ls komutu hakkında
man bash        # bash scripting hakkında
man grep        # grep hakkında
```

**Başarılar! 🐧**

---

*Son Güncelleme: 2026 - 10 Yıllık Linux Kullanıcısından*
