# Docker Üzerinde Çalışan Nginx İçin Log Rotation

## 🎯 Neden Log Rotation Önemlidir?

Web uygulamaları, özellikle yoğun trafiğe sahip ortamlarda çalışırken, Reverse proxy görevini üstlenen Nginx tarafından çok sayıda log üretir. Bu loglar zaman içinde ciddi boyutlara ulaşabilir. Eğer log dosyaları zamanında döndürülmez (rotate edilmez) ve yönetilmezse, sistemde disk alanı dolabilir, performans düşebilir veya servisler kesintiye uğrayabilir.

Docker konteynerlerinde bu problem daha da karmaşık hale gelir çünkü konteynerler genellikle kısa ömürlüdür ve sistem logları dış dünyadan izole olabilir.

---

## 🧱 Temel Mimarinin Anlaşılması

Docker konteynerleri içinde çalışan uygulamalar genellikle loglarını dosya sistemi üzerine ya da doğrudan standart çıkışa (stdout) yazarlar. Nginx'in log yönetimi de bu iki yöntemden biriyle sağlanır:

### 1. Dosya Bazlı Loglama

```nginx
access_log /var/log/nginx/access.log;
error_log /var/log/nginx/error.log;
```

Bu klasik yöntem, logların doğrudan dosya sistemi üzerinde tutulmasını sağlar. Ancak Docker konteyneri içinde kalan bu log dosyalarına host (sunucu) üzerinden erişilemez. Bu yüzden **volume mount** yapılması gerekir.

### 2. Stdout/Stderr Yönlendirmesi

```nginx
access_log /dev/stdout;
error_log /dev/stderr;
```

Bu yöntemle loglar, Docker motorunun log yönetimine teslim edilir. Böylece JSON formatında döndürme (rotate) işlemleri Docker tarafından yapılabilir.

---

## 🔧 Yöntem 1: Dosya Bazlı Loglama + Host Üzerinden Logrotate

### Adım 1: Nginx Loglarını Host’a Mount Et

Docker Compose ile log klasörü konteyner dışına taşınır:

```yaml
services:
  nginx:
    image: nginx:stable
    volumes:
      - ./logs:/var/log/nginx
```

Bu yapılandırma ile `/var/log/nginx` dizini, host üzerindeki `./logs` klasörüne bağlanır. Böylece host loglara erişebilir.

### Adım 2: logrotate Yapılandırması

Linux sistemlerde `logrotate`, belirli aralıklarla log dosyalarını döndürür, sıkıştırır ve temizler. Aşağıdaki yapılandırma örneği, günlük döndürme sağlar:

```bash
sudo nano /etc/logrotate.d/nginx-docker
```

İçerik:

```text
/home/kullanici/nginx-logs/*.log {
    daily
    missingok
    rotate 7
    compress
    delaycompress
    notifempty
    create 0640 root adm
    sharedscripts
    postrotate
        docker exec nginx-container-name nginx -s reopen
    endscript
}
```

- `daily`: Her gün log döndürülür
- `rotate 7`: 7 gün boyunca eski loglar saklanır
- `compress`: Eski loglar gzip ile sıkıştırılır
- `postrotate`: Nginx'e log dosyasını yeniden aç komutu gönderilir

### Adım 3: nginx -s reopen Mantığı

Logrotate eski log dosyasını yeniden adlandırır. Ancak Nginx hâlâ eski dosyaya yazmaya devam eder. Bu yüzden şu komutla Nginx yeniden log dosyasını açmaya zorlanır:

```bash
docker exec nginx-container-name nginx -s reopen
```

Alternatif olarak:

```bash
kill -USR1 $(cat /var/run/nginx.pid)
```

Bu sinyal `access.log` ve `error.log` dosyalarının yeniden oluşturulmasını sağlar.

---

## 🛠️ Yöntem 2: Logları stdout/stderr’a Yönlendirme (Docker Log Driver)

Logları `/dev/stdout` ve `/dev/stderr`'e yönlendirerek Docker’ın kendi log yönetim sürücüsünden yararlanabilirsiniz.

### Docker Compose Log Driver Ayarı:

```yaml
services:
  nginx:
    image: nginx
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "5"
```

Bu konfigürasyonla:

- Her log dosyası 10MB'a ulaştığında döndürülür
- En fazla 5 döndürülmüş log dosyası tutulur

Loglar şu dizinde tutulur:

```
/var/lib/docker/containers/<container-id>/<container-id>-json.log
```

Bu yöntem sistem tarafından otomatik olarak yönetildiği için `logrotate` gibi ek işlemlere gerek kalmaz.

---

## 📦 Üçüncü Parti Araçlarla Yönetim

- **Logrotate Container**: logrotate işlemini kendi başına yapan bağımsız bir konteyner çalıştırabilirsiniz.
- **Rsyslog + Volume Mount**: Nginx loglarını syslog soketine gönderip, syslog üzerinden döndürmek mümkündür.
- **Centralized Logging**: Fluentd, Filebeat, Logstash gibi ajanlar ile logları merkezi sunuculara yönlendirebilir, daha ileri analiz yapılabilir.

---

## 🧪 Test ve Doğrulama

### Test Log Üretimi:

```bash
docker exec nginx-container-name bash -c "for i in {1..10000}; do echo \"$(date) log test\" >> /var/log/nginx/access.log; done"
```

### Elle logrotate Çalıştırma:

```bash
sudo logrotate -f /etc/logrotate.d/nginx-docker
```

Yeni log dosyasının oluştuğunu ve Nginx’in yeni dosyaya yazmaya devam ettiğini kontrol edin.

---

## 🧩 Sık Karşılaşılan Hatalar

| Hata | Açıklama |
|------|----------|
| logrotate çalışmıyor | Dosya host'a mount edilmemiş olabilir |
| Yeni log dosyası oluşmuyor | `nginx -s reopen` komutu gönderilmemiş olabilir |
| Disk alanı hala dolu | `.gz` loglar silinmemiştir veya log driver düzgün çalışmıyordur |
| logrotate işlemiyor | PID dosyası yanlış veya container erişilemiyor olabilir |

---

## 🔚 Sonuç

Docker ortamında çalışan Nginx servislerinde log yönetimi, sistem kararlılığı için hayati önem taşır. Gerek klasik `logrotate` ile, gerekse Docker’ın sunduğu `json-file` log driver üzerinden log döndürme işlemleri sağlıklı bir şekilde yapılandırılmalıdır.

Yazılımcılar ve sistem yöneticileri için bu yapıların doğru şekilde konumlandırılması, logların hem arşivlenmesi hem de merkezi izleme sistemlerine aktarılması için temel oluşturur.

---

*Oluşturulma Tarihi: 2025-05-2025

