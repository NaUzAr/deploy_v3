# 🐳 Docker Multi-Website Deployment (VPS Linux)

Deploy multiple websites ke satu VPS dengan Docker + Cloudflare. Panduan ini menggunakan struktur `shared/` terbaru yang menyatukan Postgres, MQTT, dan Nginx Proxy, serta mendukung Laravel Reverb, Queue, dan Vite.

## 📋 Arsitektur

```text
                              ┌─────────────────────────────────────┐
                              │              SERVER                 │
                              │          (VPS/Cloud)                │
                              └──────────────┬──────────────────────┘
                                             │
              ┌──────────────────────────────┼──────────────────────────────┐
              │                              │                              │
              ▼                              ▼                              ▼
    ┌─────────────────┐           ┌─────────────────┐           ┌─────────────────┐
    │  Nginx Proxy    │           │   PostgreSQL    │           │  MQTT Broker    │
    │   (Port 80)     │           │   (Port 5432)   │           │  (Port 1883)    │
    └────────┬────────┘           └────────┬────────┘           └─────────────────┘
             │                             │
             │                    ┌────────┴────────┐
             │                    │                 │
             │               ┌─────────┐       ┌─────────┐
             │               │ db_web1 │       │ db_web2 │
             │               └─────────┘       └─────────┘
             │
    ┌────────┴────────┐
    │                 │
    ▼                 ▼
┌─────────────┐  ┌─────────────┐
│  Domain A   │  │  Domain B   │
│ (forlizz)   │  │ (smartagri) │
└──────┬──────┘  └──────┬──────┘
       │                │
       ▼                ▼
┌─────────────┐  ┌─────────────┐
│ forlizz_app │  │  web1_app   │
│  (Static)   │  │  (Laravel)  │
└─────────────┘  └─────────────┘
```

**Struktur Folder di Server:**
```text
/opt/docker-apps/
├── shared/          → Infrastruktur utama gabungan
│   ├── docker-compose.yml (Postgres, Nginx Proxy, MQTT)
│   ├── nginx.conf
│   ├── mosquitto.conf
│   └── passwd
├── forlizz/         → Project 1 (Static HTML)
│   ├── Dockerfile
│   ├── docker-compose.yml
│   └── src/
└── web1/            → Project 2 (Laravel + Reverb + Vite + Queue)
    ├── Dockerfile
    ├── docker-compose.yml
    ├── nginx.conf
    ├── supervisord.ini
    └── src/
```

---

## 🏗️ SETUP AWAL (Sekali Saja)

```bash
# Login ke VPS
ssh root@YOUR_IP

# Install Docker
apt update && apt upgrade -y
curl -fsSL https://get.docker.com -o get-docker.sh && sh get-docker.sh
apt install docker-compose-plugin -y

# Buat struktur & network
mkdir -p /opt/docker-apps/shared
docker network create webapps
```

---

## ⚙️ SETUP INFRASTRUKTUR SHARED (Postgres, Nginx Proxy, MQTT)

Masuk ke folder `shared`:
```bash
cd /opt/docker-apps/shared
```

### 1. Buat File Konfigurasi

**a. nginx.conf**
```bash
nano nginx.conf
```
```nginx
events {
    worker_connections 1024;
}

http {
    # Logging
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    # Gzip compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript;

    # Nanti konfigurasi domain ditambahkan di sini...
}
```

**b. mosquitto.conf**
```bash
nano mosquitto.conf
```
```
listener 1883
listener 9001
protocol websockets

# Authentication
allow_anonymous false
password_file /mosquitto/config/passwd

# Persistence
persistence true
persistence_location /mosquitto/data/

# Logging
log_dest file /mosquitto/log/mosquitto.log
log_dest stdout
```

**c. File passwd (Buat Kosong Dulu)**
```bash
touch passwd
```

**d. docker-compose.yml**
```bash
nano docker-compose.yml
```
```yaml
services:
  postgres:
    image: postgres:15-alpine
    container_name: shared_postgres
    restart: always
    environment:
      POSTGRES_USER: webadmin
      POSTGRES_PASSWORD: YOUR_PASSWORD
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - webapps

  mqtt:
    image: eclipse-mosquitto:2
    container_name: mqtt_broker
    restart: always
    ports:
      - "1883:1883"
      - "9001:9001"
    volumes:
      - ./mosquitto.conf:/mosquitto/config/mosquitto.conf
      - ./passwd:/mosquitto/config/passwd
      - ./mqtt-data:/mosquitto/data
      - ./mqtt-log:/mosquitto/log
    networks:
      - webapps

  nginx-proxy:
    image: nginx:alpine
    container_name: nginx_proxy
    restart: always
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      - webapps

volumes:
  postgres_data:

networks:
  webapps:
    external: true
```

### 2. Generate Password MQTT Broker
Gunakan command ini untuk mengisi file `passwd` (ganti `rahasia` dengan password Anda):
```bash
docker run --rm -v "/opt/docker-apps/shared/passwd:/passwd" eclipse-mosquitto:2 mosquitto_passwd -b /passwd admin rahasia
```

### 3. Jalankan Shared Infrastructure
```bash
docker compose up -d
```

### 4. Buka Port Firewall (UFW)
```bash
ufw allow 80/tcp     # HTTP (Nginx)
ufw allow 443/tcp    # HTTPS
ufw allow 1883/tcp   # MQTT
ufw allow 9001/tcp   # MQTT WebSockets (opsional)
ufw allow 8080/tcp   # Reverb WebSocket (opsional, ganti jika port beda)
ufw enable
```

---

## 🌍 SETUP PROJECT BARU (Laravel)

Setiap kali menambah website baru, ikuti panduan ini.

### 1. Buat Database
```bash
docker exec -it shared_postgres psql -U webadmin -c "CREATE DATABASE db_namaproject;"
```

### 2. Buat Folder & File Konfigurasi
```bash
mkdir -p /opt/docker-apps/namaproject/src
cd /opt/docker-apps/namaproject
```

**a. Dockerfile (Support Node/Vite)**
```bash
nano Dockerfile
```
```dockerfile
FROM php:8.2-fpm-alpine

RUN apk add --no-cache nginx supervisor libpng-dev libzip-dev postgresql-dev nodejs npm \
    && docker-php-ext-install pdo pdo_pgsql zip gd bcmath

COPY --from=composer:latest /usr/bin/composer /usr/bin/composer
COPY nginx.conf /etc/nginx/http.d/default.conf
COPY supervisord.ini /etc/supervisor.d/supervisord.ini

WORKDIR /var/www/html
COPY src/ /var/www/html/
RUN rm -rf vendor composer.lock && composer install --optimize-autoloader --no-dev --no-interaction \
    && npm install && npm run build
RUN chown -R www-data:www-data /var/www/html \
    && chmod -R 755 /var/www/html/storage /var/www/html/bootstrap/cache

EXPOSE 80 8080
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]
```

**b. supervisord.ini (Support Queue & Reverb)**
```bash
nano supervisord.ini
```
```ini
[supervisord]
nodaemon=true

[program:php-fpm]
command=/usr/local/sbin/php-fpm
autostart=true
autorestart=true

[program:nginx]
command=/usr/sbin/nginx -g "daemon off;"
autostart=true
autorestart=true

[program:mqtt-listener]
command=php /var/www/html/artisan mqtt:listen
autostart=true
autorestart=true
stdout_logfile=/var/www/html/storage/logs/mqtt.log
stderr_logfile=/var/www/html/storage/logs/mqtt-error.log

[program:queue-worker]
command=php /var/www/html/artisan queue:work --tries=3 --timeout=90
autostart=true
autorestart=true
stdout_logfile=/var/www/html/storage/logs/queue.log
stderr_logfile=/var/www/html/storage/logs/queue-error.log

[program:reverb]
command=php /var/www/html/artisan reverb:start --host=0.0.0.0 --port=8080
autostart=true
autorestart=true
stdout_logfile=/var/www/html/storage/logs/reverb.log
stderr_logfile=/var/www/html/storage/logs/reverb-error.log
```

**c. nginx.conf**
```bash
nano nginx.conf
```
```nginx
server {
    listen 80;
    root /var/www/html/public;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

**d. docker-compose.yml**
```bash
nano docker-compose.yml
```
```yaml
services:
  app:
    build: .
    container_name: namaproject_app
    restart: always
    environment:
      APP_ENV: production
      APP_DEBUG: "false"
      APP_URL: https://domain.com
      DB_CONNECTION: pgsql
      DB_HOST: shared_postgres
      DB_PORT: 5432
      DB_DATABASE: db_namaproject
      DB_USERNAME: webadmin
      DB_PASSWORD: YOUR_PASSWORD
    ports:
      - "8080:8080" # Ubah host port jika >1 project pakai reverb
    volumes:
      - ./src/storage:/var/www/html/storage
    networks:
      - webapps

networks:
  webapps:
    external: true
```

### 3. Clone Source Code
```bash
cd src
git clone https://github.com/USERNAME/REPO.git .
```

Siapkan `.env`:
```bash
cp .env.example .env
nano .env
```
*(Pastikan DB_HOST=shared_postgres, DB_DATABASE=db_namaproject, serta konfigurasi REVERB/VITE_REVERB disesuaikan)*

### 4. Build & Start Project
```bash
cd /opt/docker-apps/namaproject
docker compose up -d --build
```

### 5. Konfigurasi Awal Laravel
```bash
docker exec namaproject_app php artisan key:generate
docker exec namaproject_app php artisan migrate --force
docker exec namaproject_app php artisan storage:link
```

### 6. Daftarkan Domain ke Nginx Proxy
```bash
nano /opt/docker-apps/shared/nginx.conf
```
Tambahkan di dalam blok `http {}`:
```nginx
    server {
        listen 80;
        server_name domainanda.com www.domainanda.com;
        
        location / {
            proxy_pass http://namaproject_app:80;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
```
Restart proxy:
```bash
cd /opt/docker-apps/shared
docker compose restart nginx-proxy
```

---

## 🌐 SETUP CLOUDFLARE
1. Login Cloudflare → Pilih domain
2. **DNS** → Tambah A record: `@` → `IP_VPS_ANDA`
3. **SSL/TLS** → Set ke **"Flexible"**

---

# 🔧 Commands Berguna

```bash
# Lihat semua container
docker ps

# Lihat logs
docker logs NAMA_PROJECT_app

# Rebuild setelah update
cd /opt/docker-apps/NAMA_PROJECT && docker compose up -d --build

# Laravel commands
docker exec NAMA_PROJECT_app php artisan cache:clear
docker exec NAMA_PROJECT_app php artisan migrate

# Hapus project
cd /opt/docker-apps/NAMA_PROJECT && docker compose down
rm -rf /opt/docker-apps/NAMA_PROJECT
```

---

# 📋 Checklist Tambah Project Baru

- [ ] Buat folder `/opt/docker-apps/NAMA_PROJECT`
- [ ] Buat database (jika Laravel)
- [ ] Buat file Docker (Dockerfile, docker-compose.yml, dll)
- [ ] Clone/upload source code ke `src/`
- [ ] Build & start container
- [ ] Setup Laravel (key, migrate, storage:link)
- [ ] Tambah domain di nginx.conf
- [ ] Restart nginx proxy
- [ ] Setup DNS di Cloudflare

---

# 🔌 PostgreSQL Remote Access (Opsional)

Jika ingin akses database dari server/device lain.

## 1. Update docker-compose.yml

```bash
nano /opt/docker-apps/shared/docker-compose.yml
```

Tambahkan `ports` di service `postgres`:
```yaml
  postgres:
    image: postgres:15-alpine
    container_name: shared_postgres
    restart: always
    environment:
      POSTGRES_USER: webadmin
      POSTGRES_PASSWORD: YOUR_PASSWORD
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"    # Tambahkan ini untuk remote access
    networks:
      - webapps
```

```bash
cd /opt/docker-apps/shared && docker compose up -d
```

## 2. Whitelist IP dengan Firewall

```bash
# Izinkan hanya IP tertentu
ufw allow from 182.8.225.79 to any port 5432    # Contoh IP 1
ufw allow from 103.xxx.xxx.xxx to any port 5432 # Contoh IP 2

# Reload firewall
ufw reload

# Cek status
ufw status
```

## 3. Koneksi dari Device Lain

```
Host: YOUR_VPS_IP
Port: 5432
Database: db_namaproject
Username: webadmin
Password: YOUR_PASSWORD
```

---

# 🆘 Troubleshooting

## Error: Cloudflare 521 "Web server is down"

**Penyebab:** Cloudflare tidak bisa connect ke server.

**Solusi:**
```bash
# Cek container running
docker ps

# Cek nginx proxy logs
docker logs nginx_proxy --tail 20

# Test akses lokal
curl -I http://localhost

# Cek firewall
ufw status
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 22/tcp
ufw enable
```

---

## Error: "host not found in upstream" di Nginx

**Penyebab:** Container yang di-reference di nginx.conf belum running.

**Solusi:**
1. Cek nama container di docker-compose.yml (`container_name`)
2. Pastikan container running: `docker ps`
3. Update nginx.conf dengan nama container yang benar
4. Restart nginx: `cd /opt/docker-apps/shared && docker compose restart nginx-proxy`

---

## Error: 500 Internal Server Error

**Penyebab umum:**

### 1. APP_KEY kosong
```bash
docker exec CONTAINER_NAME php artisan key:generate --force
```

### 2. .env salah (DB_HOST masih 127.0.0.1)
Pastikan `DB_HOST=shared_postgres`.

### 3. Database belum dibuat
```bash
docker exec -it shared_postgres psql -U webadmin -c "CREATE DATABASE db_NAME;"
docker exec CONTAINER_NAME php artisan migrate --force
```

### 4. Permission storage folder
```bash
docker exec CONTAINER_NAME chmod -R 777 storage
docker exec CONTAINER_NAME chmod -R 777 bootstrap/cache
```

---

## Error: "composer.json not found" saat build

**Penyebab:** Git clone membuat subfolder tambahan.

**Solusi:**
```bash
# Cek isi folder src
ls -la /opt/docker-apps/PROJECT/src/

# Jika ada subfolder, pindahkan isinya
mv /opt/docker-apps/PROJECT/src/SUBFOLDER/* /opt/docker-apps/PROJECT/src/
rm -rf /opt/docker-apps/PROJECT/src/SUBFOLDER

# Atau clone dengan benar (pakai titik di akhir)
cd /opt/docker-apps/PROJECT/src
rm -rf *
git clone https://github.com/USER/REPO.git .
```

---

## Error: vendor folder corrupt

**Penyebab:** vendor folder dari development ikut ter-upload.

**Solusi:**
Pastikan `Dockerfile` menggunakan perintah ini saat build:
```dockerfile
RUN rm -rf vendor composer.lock && composer install --optimize-autoloader --no-dev --no-interaction
```
Lalu lakukan rebuild:
```bash
docker compose up -d --build
```
