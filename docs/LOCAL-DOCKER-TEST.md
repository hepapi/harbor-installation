# Harbor — Mac üzerinde yerel Docker testi

Bu doküman RHEL kurulumundan bağımsızdır. Mac üzerinde **OrbStack veya Docker Desktop** ile resmi Harbor installer'ını çalıştırır. Kontrol tarihi: 11 Kasım 2026; Harbor **v2.16.0**. Mevcut ortamda Docker Engine 29.8.2, Compose 5.5.0, OrbStack ve arm64 doğrulandı.

HTTPS için kendi self-signed kök CA'mızı ve bu CA'nın imzaladığı IP SAN içeren sunucu sertifikasını kullanacağız. Adres `https://<Mac-LAN-IP>:8443` olacak; DNS, hosts kaydı veya gerçek domain gerekmiyor. Mac ve Docker daemon aynı IP'ye erişebilmelidir. Ağ değişip IP değişirse sertifika ve Harbor ayarları yeniden hazırlanmalıdır.

Kaynaklar: [resmi v2.16.0 release](https://github.com/goharbor/harbor/releases/tag/v2.16.0), [Harbor HTTPS rehberi](https://goharbor.io/docs/main/install-config/configure-https/), [macOS path desteği bulunan prepare scripti](https://github.com/goharbor/harbor/blob/v2.16.0/make/prepare). Bu bir yerel geliştirme akışıdır; RHEL rehberi [README.md](../README.md) içinde bulunur.

## 1. Ortamı kontrol et

Bütün komutları Mac Terminal'de **aynı Bash oturumunda**, sudo olmadan çalıştır (Keychain adımı hariç). En az 4 CPU, 8 GB RAM ve 40 GB boş disk ayır. Bu akış internet bağlantısı kullanır.

```bash
bash
docker context show
docker version
docker compose version
docker ps --format 'table {{.Names}}\t{{.Ports}}'
lsof -nP -iTCP:8080 -iTCP:8443 -iTCP:1514 -sTCP:LISTEN
```

Server bilgisi görünmeli. 8080, 8443 ve 1514 portları başka uygulama tarafından kullanılıyorsa devam etmeden çakışmayı çöz. Docker context'ini test boyunca değiştirme. Kurulum sırasında Docker'ın yayınladığı portlara yerel ağdaki diğer cihazlar da erişebilir; güvenilir bir test ağı kullan.

Mac'in ağ IP adresini bul:

```bash
HARBOR_INTERFACE=$(route -n get default | awk '/interface:/{print $2; exit}')
HARBOR_HOST=$(ipconfig getifaddr "$HARBOR_INTERFACE")
printf 'Arayuz: %s\nHarbor IP: %s\n' "$HARBOR_INTERFACE" "$HARBOR_HOST"
```

VPN nedeniyle boş veya yanlış adres çıkarsa Wi-Fi/Ethernet IPv4 adresini Sistem Ayarları → Ağ'dan al ve `HARBOR_HOST=192.168.1.50` gibi elle ata. Gerçek adresini kullan. `localhost`, `127.0.0.1` ve `0.0.0.0` kullanma. Sonraki adımlar IPv4 adresi bekler.

## 2. Resmi paketi indir

Tüm kalıcı dosyalar `$HOME/harbor-local-test` altında tutulur; macOS için sertifika ve veri yollarının `$HOME` altında olması resmi prepare scriptiyle uyumludur. Boşluksuz bir dizin kullan.

```bash
HARBOR_LAB="$HOME/harbor-local-test"
mkdir -p "$HARBOR_LAB"
cd "$HARBOR_LAB"
case "$(docker info --format '{{.Architecture}}')" in
  aarch64|arm64)
    HARBOR_ARCH=arm64
    HARBOR_SHA256=3646b2917b14715a9835fcb3b492fce521d3b32e5a9ee898720534423bf9bf64 ;;
  x86_64|amd64)
    HARBOR_ARCH=amd64
    HARBOR_SHA256=c8972341e93e0752bd840855eb008c6dccfa6d6debf6942ce809d05e8c781621 ;;
  *) echo 'Docker server mimarisi desteklenmiyor'; exit 1 ;;
esac
HARBOR_PACKAGE="harbor-online-installer-v2.16.0-${HARBOR_ARCH}.tgz"
curl -fL --retry 3 -O "https://github.com/goharbor/harbor/releases/download/v2.16.0/$HARBOR_PACKAGE"
printf '%s  %s\n' "$HARBOR_SHA256" "$HARBOR_PACKAGE" | shasum -a 256 --check || exit 1
```

`OK` çıktısını gör. Değerler [resmi release varlıklarından](https://github.com/goharbor/harbor/releases/expanded_assets/v2.16.0) alınmıştır. Online paket küçüktür; Harbor imajları kurulumda indirilir. Bu sürüm için SHA-256 sabittir.

İlk kurulumda arşivi aç:

```bash
cd "$HARBOR_LAB"
if [ -e harbor ]; then
  echo 'harbor dizini zaten var; mevcut kurulumu korumak icin duruldu.'
else
  tar -xzf "$HARBOR_PACKAGE"
fi
```

Mevcut dizin uyarısı aldıysan şablon ve sertifikaları üzerine yazmadan önce mevcut kurulumu incele; kalan adımlar yeni kurulum içindir.

## 3. Self-signed CA ve IP sertifikası oluştur

```bash
mkdir -p "$HARBOR_LAB/tls" "$HARBOR_LAB/data" "$HARBOR_LAB/log"
cd "$HARBOR_LAB/tls"
umask 077
cat > ca.cnf <<'CA_CONFIG'
[req]
prompt = no
distinguished_name = dn
x509_extensions = ca_ext
[dn]
CN = Harbor Local Test Root CA
[ca_ext]
basicConstraints = critical,CA:TRUE
keyUsage = critical,keyCertSign,cRLSign
subjectKeyIdentifier = hash
CA_CONFIG
openssl req -x509 -newkey rsa:4096 -nodes -sha256 -days 3650 \
  -config ca.cnf -keyout ca.key -out ca.crt
openssl req -new -newkey rsa:2048 -nodes \
  -keyout server.key -out server.csr -subj "/CN=$HARBOR_HOST"
cat > server.ext <<SERVER_CONFIG
basicConstraints=critical,CA:FALSE
keyUsage=critical,digitalSignature,keyEncipherment
extendedKeyUsage=serverAuth
subjectAltName=IP:$HARBOR_HOST
SERVER_CONFIG
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server.crt -days 365 -sha256 -extfile server.ext
openssl verify -CAfile ca.crt server.crt
openssl x509 -in server.crt -noout -text | sed -n '/Subject Alternative Name/,+1p'
```

`OK` ve SAN alanında gerçek IP adresini görmelisin. `ca.key` ve `server.key` özeldir; paylaşma veya Git'e ekleme.

## 4. harbor.yml ayarla

```bash
cd "$HARBOR_LAB/harbor"
cp harbor.yml.tmpl harbor.yml
chmod 600 harbor.yml
printf 'hostname: %s\ncertificate: %s/tls/server.crt\nprivate_key: %s/tls/server.key\ndata_volume: %s/data\nlog location: %s/log\n' \
  "$HARBOR_HOST" "$HARBOR_LAB" "$HARBOR_LAB" "$HARBOR_LAB" "$HARBOR_LAB"
nano harbor.yml
```

Şablonun tamamını koru; aşağıdaki mevcut alanları güncelle. `<mac-ip>` ve `<absolute-lab-path>` yerine yukarıda yazdırılan gerçek değerleri koy. `${HARBOR_LAB}` veya `~` gibi shell ifadelerini YAML içine yazma; Harbor bunları genişletmez.

```yaml
hostname: <mac-ip>
http:
  port: 8080
https:
  port: 8443
  certificate: <absolute-lab-path>/tls/server.crt
  private_key: <absolute-lab-path>/tls/server.key
harbor_admin_password: '<choose-a-strong-admin-password>'
database:
  password: '<choose-a-different-strong-database-password>'
data_volume: <absolute-lab-path>/data
```

`database` altındaki diğer alanları koru. Ayrıca mevcut `log.local.location` değerini `<absolute-lab-path>/log` yap; log bölümünün diğer ayarlarını koru. `external_url` yorum satırı olarak kalsın. Başka bir port belirtmedikçe registry adresi her yerde **IP:8443** olmalı. Şifreleri yerel dosyaya yaz; terminal komutuna veya Git'e koyma.

## 5. Kur ve HTTPS'i kontrol et

```bash
cd "$HARBOR_LAB/harbor"
bash ./install.sh
docker compose ps
curl --fail --show-error --cacert "$HARBOR_LAB/tls/ca.crt" \
  "https://$HARBOR_HOST:8443/api/v2.0/ping"
curl --fail --show-error --cacert "$HARBOR_LAB/tls/ca.crt" \
  "https://$HARBOR_HOST:8443/api/v2.0/health"
```

İlk kurulum imajları indirir ve birkaç dakika sürebilir. Ping `Pong`, health yanıtı `healthy` olmalı. Bu test `--cacert` ile gerçek TLS doğrulaması yapar. `curl -k` veya insecure registry ayarı gerekmez. İlk testte Trivy kapalıdır.

## 6. Docker ve tarayıcı için CA güveni

**OrbStack kullanıyorsan** registry'ye özel CA dosyasını ekle ([resmi OrbStack dokümanı](https://docs.orbstack.dev/docker/network)):

```bash
mkdir -p "$HOME/.docker/certs.d/$HARBOR_HOST:8443"
cp "$HARBOR_LAB/tls/ca.crt" "$HOME/.docker/certs.d/$HARBOR_HOST:8443/ca.crt"
```

Tarayıcı güveni için CA'yı Keychain Access üzerinden System anahtarlığına içe aktar, sertifikayı aç ve SSL güvenini **Always Trust** yap. Alternatif Terminal komutu aşağıdadır; Mac yönetici parolanı sorar:

```bash
sudo security add-trusted-cert -d -r trustRoot \
  -k /Library/Keychains/System.keychain "$HARBOR_LAB/tls/ca.crt"
```

**Docker Desktop kullanıyorsan** aynı Keychain güven adımını uygula ve Docker Desktop'ı yeniden başlat ([resmi Docker Mac sertifika rehberi](https://docs.docker.com/desktop/troubleshoot-and-support/faqs/macfaqs/)). OrbStack'te registry CA değişikliği algılanmazsa uygulamayı yeniden başlat. Bu işlem diğer container'ları da etkileyebileceği için açık işlerini kontrol et. Yeniden başlattıktan sonra gerekirse Harbor dizininde `docker compose up -d` çalıştır.

CA güveni eklendikten sonra tarayıcıda `https://<Mac-LAN-IP>:8443` adresini aç. `admin` ve seçtiğin şifreyle giriş yap. **Projects → New Project** ile private **local-test** projesi oluştur.

## 7. Push/pull kabul testi

```bash
docker login "$HARBOR_HOST:8443" --username admin
docker pull busybox:1.37.0
docker tag busybox:1.37.0 "$HARBOR_HOST:8443/local-test/busybox:1.37.0"
docker push "$HARBOR_HOST:8443/local-test/busybox:1.37.0"
docker image rm "$HARBOR_HOST:8443/local-test/busybox:1.37.0"
docker pull "$HARBOR_HOST:8443/local-test/busybox:1.37.0"
docker run --rm "$HARBOR_HOST:8443/local-test/busybox:1.37.0" echo 'Harbor local test OK'
docker logout "$HARBOR_HOST:8443"
```

Kabul kriterleri: HTTPS ping/health başarılı, login başarılı, UI'da `local-test/busybox:1.37.0` görünür, push ve yeniden pull digest değerleri eşleşir, container `Harbor local test OK` yazar.

## 8. Sorun giderme ve durdurma

| Belirti | Kontrol |
|---|---|
| `x509: unknown authority` | CA dosyası Docker'ın kullandığı runtime'a eklenmiş mi? Registry dizini IP ve `:8443` içeriyor mu? |
| IP SAN hatası | Mac IP'si değişmiş mi? Sertifikayı doğru IP ile yeniden üret. |
| Timeout / connection refused | IP, VPN, macOS firewall ve yayınlanan 8443 portunu kontrol et. |
| `unauthorized` / `denied` | Doğru parola, proje adı ve kullanıcı yetkisini kontrol et. |
| Bind mount / permission hatası | Tüm yollar mutlak ve `$HOME` altında mı? Docker Desktop dosya paylaşımı bu dizine izin veriyor mu? |

```bash
cd "$HARBOR_LAB/harbor"
docker compose ps
docker compose logs --tail=100
ls -lh "$HARBOR_LAB/log"
```

Veriyi koruyarak durdur ve tekrar başlat:

```bash
cd "$HARBOR_LAB/harbor"
docker compose stop
docker compose start
```

Yeni terminalde değişkenleri tekrar tanımla: `HARBOR_LAB="$HOME/harbor-local-test"` ve `HARBOR_HOST=<gercek-mac-ip>`. Kurulumu yeniden çalıştırmak için şablonu ve CA'yı tekrar oluşturma. Veriler `data`, loglar `log`, sertifikalar `tls` dizinindedir.

Bu doküman hazırlanırken yerel Docker/Compose bağlantısı doğrulandı ve Bash blokları kontrol edildi. Harbor kurulumu, Keychain değişikliği ve push/pull kabul testi henüz çalıştırılmadı.
