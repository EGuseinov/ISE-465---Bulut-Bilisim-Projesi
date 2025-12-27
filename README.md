# ISE 465 - Bulut Bilişim Projesi: OpenStack Üzerinde Web Uygulaması Dağıtımı

**Ders:** ISE 465 - Bulut Bilişim  
**Dönem:** 2025-2026 Güz  
**Öğrenci Adı:** Elvin Guseinov  
**Proje Konusu:** OpenStack (MicroStack) üzerinde Ölçeklenebilir Web Uygulaması


---

##  Proje Özeti
Bu proje kapsamında, IaaS (Infrastructure as a Service) katmanında **OpenStack** kullanılarak özel bir bulut ortamı kurulmuştur. Bu ortam üzerinde, **HTML5 , CSS3 ve JavaScript** teknolojileriyle geliştirilmiş, işlemci gücünü istemci tarafında (Client-Side) kullanan bir "PDF Dönüştürücü" uygulaması dağıtılmıştır. Bulut bilişimin yetenekleri olan sanallaştırma, ağ yönetimi ve otomatik ölçekleme (Auto-Scaling) senaryoları değerlendirilmiştir.

---

##  Kullanılan Teknolojiler ve Mimari
* **Bulut Platformu:** OpenStack (MicroStack sürümü via Snap)
* **Sanallaştırma:** KVM / QEMU (VirtualBox üzerinde Nested Virtualization)
* **İşletim Sistemi:** Ubuntu 20.04 LTS (Cloud Image)
* **Uygulama Mimarisi:** Statik Web Uygulaması (HTML5, CSS3, JavaScript - jsPDF Kütüphanesi)
* **Web Sunucusu:** Python `http.server` (Sadece statik dosyaları yayınlamak için kullanılmıştır)
* **Dağıtım Yöntemi:** Infrastructure as Code (User Data / Cloud-Init)

  <img width="5959" height="4359" alt="mimari_sema" src="https://github.com/user-attachments/assets/def7fe4c-6220-4c51-985c-367ca7fc9198" />


---

##  1. Kurulum ve Altyapı Hazırlığı (Zorluklar ve Çözümler)

Projenin en kritik aşaması, kendi bilgisayarımızda çalışan bir OpenStack veri merkezi kurmaktı. Bütün işlemleri bilgisayarımdaki virtual box üzerindeki Ubuntu sanal makinesinde gerçekleştirdim. Bu süreçte karşılaşılan teknik sorunlar ve mühendislik çözümleri aşağıdadır:

### 1.1. Servis Kesintileri ve LVM Yapılandırması
Kurulum sonrası Dashboard'a (Horizon) erişimde `502 Bad Gateway` hataları alındı. İncelemelerde `cinder-volume` servislerinin çalışmadığı görüldü.
* **Sorun:** MicroStack, disk yönetimi için gerekli LVM (Logical Volume Manager) grubunu oluşturamamıştı.

<img width="841" height="639" alt="neutronerror" src="https://github.com/user-attachments/assets/6ca15611-3a7c-44df-b81d-8111d99a06e4" />
<img width="825" height="588" alt="badgateway" src="https://github.com/user-attachments/assets/a519ba4f-ce9e-45c6-bb25-6983d2874ec8" />


* **Çözüm:** Manuel olarak loop-device oluşturulup LVM grubuna dahil edildi:
    ```bash
    sudo truncate -s 20G /var/snap/microstack/common/cinder-volumes
    sudo losetup /dev/loop15 /var/snap/microstack/common/cinder-volumes
    sudo vgcreate cinder-volumes /dev/loop15
    sudo snap restart microstack.cinder-volume
    ```

### 1.2. İmaj ve Network Sorunları
* **Sorun:** `Invalid image identifier` hatası ve internet bağlantısındaki kopmalar nedeniyle imaj yüklenemedi.
  <img width="440" height="70" alt="pingerror" src="https://github.com/user-attachments/assets/fb8cae46-1404-473a-b71f-8ba3dd3d881c" />

* **Çözüm:** Bozuk imajlar veritabanından temizlendi. `wget -c` parametresi ile kesintiye dayanıklı indirme yapılarak önce CirrOS (test için), ardından Ubuntu 20.04 Cloud imajları sisteme dahil edildi ve OpenStack (Glance) servisine tanıtıldı:

    ```bash
    # --- ADIM 1: İmajların İndirilmesi (Download) ---
    # CirrOS (Test İmajı) İndirme
    wget -c http://download.cirros-cloud.net/0.5.2/cirros-0.5.2-x86_64-disk.img

    # Ubuntu 20.04 (Proje İmajı) İndirme
    wget -c https://cloud-images.ubuntu.com/releases/focal/release/ubuntu-20.04-server-cloudimg-amd64.img

    # --- ADIM 2: İmajların OpenStack'e Yüklenmesi (Import) ---
    # CirrOS'u Sisteme Tanıtma
    microstack.openstack image create --public --disk-format qcow2 --container-format bare --file cirros-0.5.2-x86_64-disk.img cirros-final

    # Ubuntu'yu Sisteme Tanıtma
    microstack.openstack image create --public --disk-format qcow2 --container-format bare --file ubuntu-20.04-server-cloudimg-amd64.img ubuntu-20
    ```

### 1.3. Depolama Stratejisi (Ephemeral Disk)
* **Sorun:** Cinder servisi stabil çalışmadığı için Volume oluşturma işlemleri zaman aşımına uğruyordu.  
* **Strateji:** Bulut mimarisinde "Stateless" prensibine uygun olarak, "Create New Volume: No" seçeneği ile **Ephemeral (Uçucu) Disk** kullanımına geçildi. Bu sayede instance başlatma süresi hızlandı ve hatalar giderildi.

### 1.4. Bağlantı Testi (CirrOS Denemesi)
Ubuntu gibi büyük imajları kurmadan önce, sistem servislerinin çalıştığını doğrulamak için **CirrOS** kullanıldı.
* CirrOS imajı sisteme yüklendi ve bir instance oluşturuldu.

---

##  2. Uygulama Dağıtımı ve Konfigürasyon Adımları

Web uygulamasının çalışacağı sanal sunucu (Instance) oluşturulurken aşağıdaki OpenStack bileşenleri yapılandırılmıştır:

### 2.1. Ağ ve Güvenlik Ayarları (Security Groups)
Varsayılan olarak tüm dış erişimlere kapalı olan sunucuya erişebilmek için özel bir **Security Group** oluşturuldu ve aşağıdaki Ingress (Giriş) kuralları tanımlandı:
* **SSH Port 22 :** Uzaktan yönetim ve terminal bağlantısı için.
* **TCP Port 80 (HTTP):** Web arayüzüne tarayıcı üzerinden erişim için.

### 2.2. Ana Sunucunun (Instance) Oluşturulması
"Launch Instance" sihirbazı ile `web-main-server` aşağıdaki parametrelerle oluşturuldu:
* **Source:** Ubuntu 20.04 Cloud Image
* **Flavor:** m1.small (1 VCPU, 2GB RAM, 20GB Disk)
* **Network:** Test Network (Private)
* **Key Pair:** SSH bağlantısı için oluşturulan `.pem` anahtarı sisteme tanımlandı.

### 2.3. Dış Erişim (Floating IP)
Sanal makine oluşturulduğunda sadece OpenStack iç ağından (Private IP) erişilebiliyordu. Dış dünyadan (Host makine ve tarayıcı) erişim sağlayabilmek için:
1.  OpenStack havuzundan bir **Floating IP** (Örn: `10.20.20.168`) tahsis edildi.
2.  Bu IP, oluşturulan `web-main-server` instance'ına "Associate" edildi.

<img width="757" height="336" alt="mainserver" src="https://github.com/user-attachments/assets/3dc44c59-0d65-465f-88e9-1c4d48bc14fa" />


### 2.4. Uygulamanın Yüklenmesi ve SSH Bağlantısı
Sunucu hazır olduktan sonra yerel terminal üzerinden SSH protokolü ile bağlantı sağlandı:

```bash
# SSH Anahtarı yetkilendirmesi ve bağlantı
chmod 400 new-pair-key.pem
#Sistem Tarafından Oluşturulan Floating IP kullanılmalı
ssh -i new-pair-key.pem ubuntu@10.20.20.168
```
# Bağlantı yapıldıktan sonra :
* Uygulama için **website** dizini oluşturuldu.
* Geliştirilen HTML5/JS PDF Dönüştürücü kodları **index.html** dosyasına yazıldı.
* Python'un yerleşik sunucusu ile uygulama 80 portundan yayına alındı:
```bash
sudo python3 -m http.server 80
```


### 2.1. İlk Dağıtım (Main Server)
Ana sunucu (`web-main-server`), manuel olarak oluşturuldu, Security Group (Port 80/22) ayarları yapıldı ve Floating IP (`10.20.20.168`) atandı. Uygulama başarıyla çalıştırıldı.

### 2.2. Ölçekleme Kriz Senaryosu: "Host Disk Full"
Proje senaryosu gereği, ana sunucunun yedeğinin (Snapshot) alınarak çoğaltılması planlandı.
* **Kritik Hata:** Snapshot işlemi sırasında Host makinenin (Windows) disk alanı doldu (`DrvVD_DISKFULL`). Sanal makine "Suspend" moduna geçti ve işlem başarısız oldu.

<img width="780" height="553" alt="diskfull" src="https://github.com/user-attachments/assets/5f155ccc-9383-4993-87ed-8987eae44d67" />

  
* **Analiz:** Snapshot alma işlemi, ayrılan sanal disk boyutu kadar (20GB) ek alana ihtiyaç duyuyordu ve fiziksel donanım limitlerine takıldık.

---

##  3. Çözüm: Infrastructure as Code (Otomasyon)

Snapshot alarak disk alanını şişirmek yerine, modern DevOps yaklaşımı olan **Infrastructure as Code (IaC)** yöntemine geçiş yapıldı.

**User Data Script (Cloud-Init)** kullanılarak, yeni bir sunucu başlatıldığı anda uygulamanın otomatik olarak kurulması sağlandı. Bu yöntem disk kullanımını minimize etti ve operasyonel verimliliği artırdı.

### Kullanılan Otomasyon Kodu (User Data Script)
Aşağıdaki Bash scripti, OpenStack "Configuration" bölümüne eklenerek **Replika Sunucu** ortamda otomatik olarak ayağa kaldırıldı:

```bash
#!/bin/bash

# --- 1. Erişim Güvenliği ---
# Kullanıcı şifresini belirle ve SSH erişimini aç
echo "ubuntu:123456" | chpasswd
sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
service ssh restart

# --- 2. Uygulama Kurulumu ---
mkdir -p /home/ubuntu/website
cd /home/ubuntu/website

# HTML Dosyasını Oluştur (Uygulama Kodu Gömüldü)
cat <<EOF > index.html
<!DOCTYPE html>
<html lang="tr">
 ....
</html>
EOF

# --- 3. Servisi Başlat ---
# Python HTTP sunucusunu arka planda 80 portunda çalıştır
nohup sudo python3 -m http.server 80 > /dev/null 2>&1 &
```
<img width="868" height="632" alt="mainserver_replica" src="https://github.com/user-attachments/assets/b55f3279-d84a-4716-b26e-1e2ef6ddad28" />

