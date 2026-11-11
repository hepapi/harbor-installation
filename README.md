# Harbor — RHEL Üzerinde Resmi Kurulum Rehberi

Son kontrol: **11 Kasım 2026**. Güncel kararlı sürüm: **v2.16.0** ([resmi release](https://github.com/goharbor/harbor/releases/tag/v2.16.0)). Bu rehber, resmi Docker Compose tabanlı installer ile **yeni bir Harbor kurulumu** içindir.

## 1. Sunucu gereksinimleri

Komutları **RHEL 9 sunucusunda, Bash ile** çalıştır. Sunucu mimarisine uygun amd64 veya arm64 Harbor paketini kullan.

[Resmi gereksinimler](https://goharbor.io/docs/main/install-config/installation-prereqs/):

| Kaynak | Minimum | Önerilen |
|---|---|---|
| CPU | 2 vCPU | 4 vCPU |
| Bellek | 4 GB | 8 GB |
| Disk | 40 GB | 160 GB |

Docker Engine > 20.10 ve Docker Compose > 2.3 gerekir. Sunucu adresi istemcilerden erişilebilir olmalı; TCP 80 ve 443 portları boş ve istemciden erişilebilir olmalı.

Örnek hostname: **harbor.example.com**. Bu bir placeholder'dır. Kendi hostname'inle değiştir; `localhost` veya `127.0.0.1` değerini Harbor hostname'i olarak kullanma ([resmi v2.16.0 şablonu](https://github.com/goharbor/harbor/blob/v2.16.0/make/harbor.yml.tmpl)).

RHEL BaseOS ve AppStream depolarına erişim sağlayan abonelik veya kurum içi mirror hazır olmalı. Bu akış Docker CE + Compose kullanır; Podman ile değiştirilmemiştir.

Sunucu IP adresini ve işletim sistemini kontrol et:

```bash
cat /etc/redhat-release
hostname -I
uname -m
free -h
df -h
sudo ss -ltnp '( sport = :80 or sport = :443 )'
```

`harbor.example.com` için sunucunun IP adresine yönlenen DNS kaydını oluştur. Sunucuda ve Docker istemcilerinde çözümlemeyi doğrula:

```bash
getent hosts harbor.example.com
```

## 2. Docker Engine ve Compose kurulumu

Yeni RHEL 9 sunucu üzerinde [Docker'ın resmi RHEL RPM deposu](https://docs.docker.com/engine/install/rhel/) kullanılır. Önce mevcut paketleri kontrol et:

```bash
rpm -q podman podman-docker runc docker-ce docker-ce-cli
sudo dnf repolist
```

Podman/runc veya eski Docker paketleri kuruluysa Docker'ın resmi sayfasındaki **Uninstall old versions** bölümüne göre çakışmaları gider. Mevcut container iş yükleri olan bir sunucuda paketleri doğrudan kaldırma; bu rehber yeni bir sunucuyu esas alır.

```bash
sudo dnf install -y dnf-plugins-core ca-certificates curl openssl tar gzip
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo docker version
sudo docker compose version
sudo docker run --rm hello-world
```

GPG anahtarı sorulduğunda resmi Docker fingerprint değerini doğrula: `060A 61C5 1B55 8A7F 742B 77AA C52F EB6B 621E 9F35`.

Tamamen offline hedefte Docker'ın aynı sayfadaki **Install from a package** yöntemini uygula. Hedef RHEL sürümü ve mimarisi için beş Docker RPM paketini ve tüm eksik RHEL bağımlılıklarını internetli ortamda hazırla. Yalnızca bu paketlerin bulunduğu dizinde, hedef Sunucuda kur:

```bash
sudo dnf --disablerepo='*' install ./*.rpm
sudo systemctl enable --now docker
sudo docker version
sudo docker compose version
```

Eksik bağımlılık hatası varsa ilgili RPM'leri aynı RHEL sürümü/mimarisi için tamamla. Harbor offline arşivi Docker Engine veya RHEL bağımlılıklarını içermez. Offline `hello-world` testi de önceden taşınmış imaj gerektirir.

### firewalld ve SELinux

Sunucuda firewalld kullanılıyorsa aktif zone'u kontrol et:

```bash
sudo firewall-cmd --state
sudo firewall-cmd --get-active-zones
getenforce
```

Aşağıdaki `public` değerini Sunucunun istemciye bakan arayüzünün gerçek zone'uyla değiştir:

```bash
sudo firewall-cmd --permanent --zone=public --add-service=http
sudo firewall-cmd --permanent --zone=public --add-service=https
sudo firewall-cmd --reload
sudo firewall-cmd --zone=public --list-services
```

Zone seçimi için [Red Hat firewalld rehberi](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_firewalls_and_packet_filters/using-and-configuring-firewalld_firewall-packet-filters) esas alınır. Sunucunun bulunduğu ağ da TCP 80/443 erişimine izin vermeli. Docker kendi ağ kurallarını yönetir; erişimi ayrıca istemciden test et. SELinux'u mevcut enforcing durumunda bırak; erişim reddi olursa 9. bölümdeki AVC kontrolünü kullan.

## 3. Resmi Harbor offline installer'ı indir

Aşağıdaki komutlar sunucu üzerinde çalışır. Offline hedef için indirme ve SHA-256 kontrolünü internetli RHEL makinede yapıp doğrulanmış paketi sunucuya taşı; indirme mimarisini hedef sunucu'ye göre seç.

[Resmi paketler ve SHA-256 değerleri](https://github.com/goharbor/harbor/releases/expanded_assets/v2.16.0). v2.16.0 paket adlarında mimari bulunur.

```bash
mkdir -p "$HOME/harbor-download"
cd "$HOME/harbor-download"
HARBOR_VERSION=v2.16.0
case "$(uname -m)" in
  x86_64) HARBOR_ARCH=amd64; HARBOR_SHA256=643c137d8e829c857ef0ba45c46b5d6d8553077340bbd4929ded7e92808e77ff ;;
  aarch64) HARBOR_ARCH=arm64; HARBOR_SHA256=6525af9c26cc24e48ecf37d3f7d9d139895b39a037a3de697f995171ce65524a ;;
  *) echo "Desteklenmeyen hedef mimari: $(uname -m)"; exit 1 ;;
esac
HARBOR_PACKAGE="harbor-offline-installer-${HARBOR_VERSION}-${HARBOR_ARCH}.tgz"
curl -fL --retry 3 -O "https://github.com/goharbor/harbor/releases/download/${HARBOR_VERSION}/${HARBOR_PACKAGE}"
printf '%s  %s\n' "$HARBOR_SHA256" "$HARBOR_PACKAGE" | sha256sum --check --strict || exit 1
sudo tar -xzf "$HARBOR_PACKAGE" -C /opt
```

`OK` çıktısı olmadan devam etme. `/opt/harbor` mevcut bir kurulum içeriyorsa üzerine açma; bu rehber yükseltme prosedürü değildir. SHA-256 dosya bütünlüğünü doğrular; release sayfasında ayrıca her pakete ait Sigstore bundle bulunur.

## 4. HTTPS sertifikasını hazırla

[Resmi HTTPS rehberine](https://goharbor.io/docs/2.15.0/install-config/configure-https/) göre kurum CA'sından veya güvenilir bir sertifika otoritesinden alınmış sertifika ve özel anahtarı hazırla. Sertifikanın SAN alanı Harbor hostname'ini içermeli. `server.crt` sunucu sertifikasını ve varsa ara CA zincirini, `ca.crt` güvenilecek kök CA sertifikasını içermelidir.

Sertifika dosyalarını `$HOME/harbor-tls` altına yerleştir ve doğrula:

```bash
cd "$HOME/harbor-tls"
openssl verify -CAfile ca.crt -untrusted server.crt -verify_hostname harbor.example.com server.crt
openssl x509 -in server.crt -noout -dates -ext subjectAltName
openssl x509 -in server.crt -pubkey -noout | openssl sha256
openssl pkey -in server.key -pubout | openssl sha256
```

Son iki komutun çıktıları eşleşmelidir. Ardından dosyaları kur:

```bash
sudo install -d -m 0755 /data/cert /data/harbor
sudo install -m 0644 server.crt /data/cert/harbor.example.com.crt
sudo install -m 0600 server.key /data/cert/harbor.example.com.key
```

## 5. CA güvenini yapılandır

Kurum CA'sı kullanılıyorsa RHEL sunucusunda güven deposuna ekle:

```bash
sudo install -m 0644 "$HOME/harbor-tls/ca.crt" /etc/pki/ca-trust/source/anchors/harbor-ca.crt
sudo update-ca-trust extract
sudo install -d -m 0755 /etc/docker/certs.d/harbor.example.com
sudo install -m 0644 "$HOME/harbor-tls/ca.crt" /etc/docker/certs.d/harbor.example.com/ca.crt
sudo systemctl restart docker
```

Harbor'a bağlanacak Docker istemcilerinin ve tarayıcıların da ilgili CA'ya güvenmesi gerekir. Güvenilir genel bir CA zaten sistem güven deposunda bulunuyorsa ayrıca CA eklemek gerekmez.

## 6. Resmi harbor.yml şablonunu düzenle

```bash
cd /opt/harbor
sudo cp harbor.yml.tmpl harbor.yml
sudo chmod 600 harbor.yml
sudoedit harbor.yml
```

**Dosyanın tamamını aşağıdaki parçayla değiştirme.** Paketin kendi `harbor.yml.tmpl` dosyasındaki diğer ayarları koruyarak mevcut alanları düzenle. `<...>` değerlerini gerçek ve birbirinden farklı güçlü parolalarla değiştir:

```yaml
hostname: harbor.example.com
http:
  port: 80
https:
  port: 443
  certificate: /data/cert/harbor.example.com.crt
  private_key: /data/cert/harbor.example.com.key
harbor_admin_password: '<set-admin-password-from-secure-source>'
database:
  password: '<set-database-password-from-secure-source>'
data_volume: /data/harbor
```

`database` altındaki diğer mevcut anahtarları koru; aynı YAML anahtarını ikinci kez ekleme. `harbor_admin_password` yalnızca ilk kurulumda uygulanır. Harbor'ın kendi nginx container'ı HTTPS'i sunar. Bu senaryoda `external_url` veya ayrı reverse proxy gerekmez. Yapılandırma referansı: [v2.16.0 resmi şablon](https://github.com/goharbor/harbor/blob/v2.16.0/make/harbor.yml.tmpl).

## 7. Harbor'ı kur

```bash
cd /opt/harbor
sudo ./install.sh
sudo docker compose ps
```

İlk başlangıç birkaç dakika sürebilir. Servisler çalışır ve sağlık kontrolleri başarılı olduktan sonra devam et. Resmi installer imajları offline arşivinden yükler, yapılandırmayı hazırlar ve Compose servislerini başlatır ([v2.16.0 install.sh](https://github.com/goharbor/harbor/blob/v2.16.0/make/install.sh)).

İmaj taraması da istiyorsan yukarıdaki kurulum komutunun yerine `sudo ./install.sh --with-trivy` kullan. Tamamen offline ortamda Trivy veritabanları ayrıca önceden hazırlanmalıdır; offline installer tek başına güncel zafiyet veritabanı sağlamaz. İlk registry testi için Trivy zorunlu değildir.

## 8. Sağlık, login, push ve pull testi

sunucu üzerinde HTTPS ve API sağlık durumunu kontrol et:

```bash
curl --fail --silent --show-error https://harbor.example.com/api/v2.0/ping
curl --fail --silent --show-error https://harbor.example.com/api/v2.0/health
```

Ping için `Pong`, sağlık yanıtında `status: healthy` beklenir. Sertifika doğrulamasını kapatan `curl -k` kullanma; güven hatası varsa CA ve hostname/SAN eşleşmesini düzelt.

1. Tarayıcıda `https://harbor.example.com` adresini aç.
2. `admin` ve belirlediğin parolayla giriş yap.
3. **Projects → New Project** üzerinden **private** `registry-validation` projesi oluştur. İlk test için Proxy Cache gerekmez.

Docker komutlarını sunucu üzerinde çalıştır. Login parolayı etkileşimli sorar:

```bash
sudo docker login harbor.example.com --username admin
```

İnternet bağlantısı varsa test imajını çek:

```bash
sudo docker pull busybox:1.37.0
```

Offline hedefte bunun yerine internetli makinede hedef mimariye uygun imajı indirip arşivle (`linux/arm64` veya `linux/amd64`), dosyayı sunucuya taşı ve yükle:

```bash
# İnternetli makine — hedef sunucu mimarisine göre --platform değerini değiştir.
sudo docker pull --platform linux/amd64 busybox:1.37.0
sudo docker save -o busybox-1.37.0.tar busybox:1.37.0
# Dosya hedef sunucu'ye taşındıktan sonra, sunucu üzerinde:
sudo docker load -i busybox-1.37.0.tar
```

Her iki yolun ardından sunucu üzerinde push/pull testini tamamla:

```bash
sudo docker tag busybox:1.37.0 harbor.example.com/registry-validation/busybox:1.37.0
sudo docker push harbor.example.com/registry-validation/busybox:1.37.0
sudo docker image rm harbor.example.com/registry-validation/busybox:1.37.0
sudo docker pull harbor.example.com/registry-validation/busybox:1.37.0
sudo docker run --rm harbor.example.com/registry-validation/busybox:1.37.0 echo 'Harbor testi basarili'
sudo docker logout harbor.example.com
```

UI'da `registry-validation/busybox` altında `1.37.0` etiketi görünmeli; container `Harbor testi basarili` yazmalıdır. Push ve yeniden pull çıktılarındaki digest değerleri aynı olmalıdır. Docker tag/push işlemleri için [resmi Harbor rehberi](https://goharbor.io/docs/2.15.0/install-config/run-installer-script/) esas alınmıştır.

## 9. Loglar ve yeniden başlatma

```bash
cd /opt/harbor
sudo docker compose ps
sudo docker compose logs --tail=100
sudo ls -lh /var/log/harbor
```

Harbor servis logları varsayılan olarak `/var/log/harbor` altında tutulur. Sorun yaşanan servisin logunu incele. Durdurma ve tekrar başlatma:

```bash
cd /opt/harbor
sudo docker compose stop
sudo docker compose start
```

RHEL'de `permission denied` hatası varsa SELinux kayıtlarını kontrol et (`ausearch`, `audit` paketiyle gelir):

```bash
getenforce
sudo ausearch -m AVC,USER_AVC -ts recent -i
sudo ls -ldZ /data /data/harbor /data/cert /var/log/harbor
sudo journalctl -u docker --since '10 minutes ago' --no-pager
```

[Red Hat SELinux sorun giderme rehberine](https://access.redhat.com/articles/2191331) göre ilgili AVC kaydını incele. İzin/etiket düzeltmesini reddedilen yol ve işlem için belirle; SELinux'u kapatmak veya körlemesine `audit2allow` politikası üretmek kurulum adımı değildir.

`/data/harbor` kalıcı verileri içerir. Bu rehber veri silme veya sıfırlama adımı içermez.

## Resmi kaynaklar ve doğrulama sınırı

- [Güncel kararlı release](https://github.com/goharbor/harbor/releases/latest) — bu rehber v2.16.0'a sabitlenmiştir; sürüm değişirse paket adı ve SHA-256 değerleri birlikte güncellenmelidir.
- [Harbor kurulum dokümanı](https://goharbor.io/docs/main/install-config/) — `main` hareketli dokümantasyondur. Kontrol sırasında 2.16.0 doküman adresleri erişilebilir olmadığı için sürüme özgü ayarlar release'in kendi şablon ve installer kaynaklarından doğrulandı.
- [Docker RHEL kurulumu](https://docs.docker.com/engine/install/rhel/)
- [Harbor Red Hat CA güven deposu adımları](https://goharbor.io/docs/main/install-config/troubleshoot-installation/)
- [Harbor HTTPS yapılandırması](https://goharbor.io/docs/2.15.0/install-config/configure-https/)

Bu dosyadaki komutlar kaynak ve Bash sözdizimi açısından kontrol edilmiştir. Bu çalışma sırasında sunucu üzerinde Harbor kurulumu ve uçtan uca runtime testi yapılmamıştır; 8. bölüm kurulum doğrulama adımlarını içerir.
