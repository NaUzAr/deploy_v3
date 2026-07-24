# 🐳 PANDUAN LENGKAP: Deploy Laravel ke VPS dari Nol

Panduan ini dibuat agar **siapa pun** (bahkan pemula) bisa men-deploy project Laravel ke VPS Hostinger dari awal.
Semua command tinggal **copy-paste**. Bagian yang perlu diganti ditandai dengan `⚠️ GANTI`.

---

## 📋 Penjelasan Singkat

**Apa yang akan kita bangun?**

```text
┌──────────────────────────────────────────────────────────────┐
│                        VPS SERVER                            │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ SHARED (folder: /opt/docker-apps/shared/)            │    │
│  │                                                      │    │
│  │  🌐 Nginx Proxy    → Menerima request dari internet  │    │
│  │  📡 MQTT Broker    → Menerima data dari sensor IoT   │    │
│  │  🐘 PostgreSQL     → Database (opsional, bisa skip)  │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │ PROJECT (folder: /opt/docker-apps/swaratani/)        │    │
│  │                                                      │    │
│  │  🟢 PHP-FPM        → Menjalankan Laravel             │    │
│  │  🔧 Queue Worker   → Menjalankan background jobs     │    │
│  │  📡 MQTT Listener  → Mendengar data sensor           │    │
│  │  🔌 Reverb         → WebSocket realtime (port 8080)  │    │
│  │  🎨 Vite           → Build asset CSS/JS saat deploy  │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

**Struktur folder yang akan kita buat di server:**
```text
/opt/docker-apps/
├── shared/                  ← Infrastruktur (Nginx, MQTT, Postgres)
│   ├── docker-compose.yml
│   ├── nginx.conf
│   ├── mosquitto.conf
│   └── passwd
│
└── swaratani/               ← Project Laravel
    ├── docker-compose.yml
    ├── Dockerfile
    ├── nginx.conf
    ├── supervisord.ini
    └── src/                 ← Source code Laravel (dari git clone)
```

Ada **2 file `nginx.conf`** yang berbeda fungsi:
- `shared/nginx.conf` → **Proxy utama** yang menerima request dari internet dan meneruskannya ke project. **Di sinilah domain `swaratani.id` ditulis.**
- `swaratani/nginx.conf` → **Nginx internal** di dalam container project. Hanya bertugas menjalankan file PHP. **Tidak perlu diubah.**

---

# ═══════════════════════════════════════════
# LANGKAH 1: MASUK KE SERVER & INSTALL DOCKER
# ═══════════════════════════════════════════

> ℹ️ Langkah ini hanya dilakukan **sekali** saat pertama kali setup VPS baru.
> Jika Docker sudah terinstall, langsung lompat ke **Langkah 2**.

### 1.1 Login ke VPS via SSH

```bash
ssh root@IP_VPS_ANDA
```
> ⚠️ GANTI `IP_VPS_ANDA` dengan IP VPS Anda (contoh: `203.194.115.76`).

### 1.2 Update Sistem

```bash
apt update && apt upgrade -y
```

### 1.3 Install Docker

```bash
curl -fsSL https://get.docker.com -o get-docker.sh && sh get-docker.sh
```

### 1.4 Install Docker Compose Plugin

```bash
apt install docker-compose-plugin -y
```

### 1.5 Verifikasi Docker Terinstall

```bash
docker --version
docker compose version
```
> ✅ Jika muncul versi Docker dan Docker Compose, berarti berhasil.

### 1.6 Buat Docker Network

```bash
docker network create webapps
```
> Network ini digunakan agar semua container bisa saling berkomunikasi.

---

# ═══════════════════════════════════════════
# LANGKAH 2: SETUP INFRASTRUKTUR SHARED
# ═══════════════════════════════════════════

> ℹ️ Folder `shared/` berisi 3 service yang dipakai bersama oleh semua project:
> **Nginx Proxy**, **MQTT Broker**, dan **PostgreSQL**.

### 2.1 Buat Folder

```bash
mkdir -p /opt/docker-apps/shared
cd /opt/docker-apps/shared
```

### 2.2 Buat File `docker-compose.yml`

```bash
nano docker-compose.yml
```

**Paste seluruh isi ini:**
```yaml
services:
  postgres:
    image: postgres:15-alpine
    container_name: shared_postgres
    restart: always
    environment:
      POSTGRES_USER: webadmin
      POSTGRES_PASSWORD: GANTI_PASSWORD_POSTGRES
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

> ⚠️ GANTI `GANTI_PASSWORD_POSTGRES` dengan password database Anda.
> Simpan file: tekan `Ctrl+X`, lalu `Y`, lalu `Enter`.

---

### 2.3 Buat File `mosquitto.conf`

```bash
nano mosquitto.conf
```

**Paste seluruh isi ini:**
```
listener 1883
listener 9001
protocol websockets
allow_anonymous false
password_file /mosquitto/config/passwd
persistence true
persistence_location /mosquitto/data/
log_dest file /mosquitto/log/mosquitto.log
log_dest stdout
```

> Simpan file: tekan `Ctrl+X`, lalu `Y`, lalu `Enter`.

---

### 2.4 Buat File Password MQTT (Kosong Dulu)

```bash
touch passwd
```

---

### 2.5 Buat File `nginx.conf` (Proxy Utama)

> ⚠️ **INI ADALAH FILE PROXY UTAMA.** Di sinilah Anda mendaftarkan domain `swaratani.id`.

```bash
nano nginx.conf
```

**Paste seluruh isi ini:**
```nginx
events {
    worker_connections 1024;
}

http {
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    gzip on;
    gzip_types text/plain text/css application/json application/javascript;

    # ===== DOMAIN: swaratani.id =====
    server {
        listen 80;
        server_name swaratani.id www.swaratani.id;

        location / {
            proxy_pass http://swaratani_app:80;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

    # ===== TEMPLATE: Copy block ini untuk menambah domain baru =====
    # server {
    #     listen 80;
    #     server_name DOMAIN_BARU.com www.DOMAIN_BARU.com;
    #
    #     location / {
    #         proxy_pass http://NAMA_CONTAINER_app:80;
    #         proxy_set_header Host $host;
    #         proxy_set_header X-Real-IP $remote_addr;
    #         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    #         proxy_set_header X-Forwarded-Proto $scheme;
    #     }
    # }
}
```

> Simpan file: tekan `Ctrl+X`, lalu `Y`, lalu `Enter`.

---

### 2.6 Generate Password MQTT

```bash
docker run --rm -v "/opt/docker-apps/shared/passwd:/passwd" eclipse-mosquitto:2 mosquitto_passwd -b /passwd admin GANTI_PASSWORD_MQTT
```

> ⚠️ GANTI `GANTI_PASSWORD_MQTT` dengan password MQTT yang Anda inginkan.
> Perintah ini membuat user `admin` dengan password yang Anda tentukan.

---

### 2.7 Jalankan Semua Infrastruktur

```bash
cd /opt/docker-apps/shared
docker compose up -d
```

### 2.8 Verifikasi Semua Container Menyala

```bash
docker ps
```

> ✅ Harus muncul 3 container: `shared_postgres`, `mqtt_broker`, `nginx_proxy` dengan status `Up`.

---

# ═══════════════════════════════════════════
# LANGKAH 3: DEPLOY PROJECT SWARATANI
# ═══════════════════════════════════════════

### 3.1 Buat Folder Project

```bash
mkdir -p /opt/docker-apps/swaratani/src
cd /opt/docker-apps/swaratani
```

---

### 3.2 Buat File `Dockerfile`

```bash
nano Dockerfile
```

**Paste seluruh isi ini:**
```dockerfile
FROM php:8.3-fpm-alpine

RUN apk add --no-cache nginx supervisor libpng-dev libzip-dev postgresql-dev nodejs npm \
    && docker-php-ext-install pdo pdo_pgsql zip gd bcmath pcntl

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

> Simpan file: tekan `Ctrl+X`, lalu `Y`, lalu `Enter`.
> ℹ️ File ini **tidak perlu diubah**. Sudah include: PHP, Nginx, Node.js, npm, Composer.

---

### 3.3 Buat File `supervisord.ini`

> ℹ️ File ini mengatur proses-proses yang berjalan otomatis di dalam container:
> PHP-FPM, Nginx, MQTT Listener, Queue Worker, dan Reverb.

```bash
nano supervisord.ini
```

**Paste seluruh isi ini:**
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

> Simpan file: tekan `Ctrl+X`, lalu `Y`, lalu `Enter`.
> ℹ️ File ini **tidak perlu diubah**.

---

### 3.4 Buat File `nginx.conf` (Internal Container)

> ⚠️ **INI BUKAN proxy utama!** Ini adalah Nginx internal di dalam container project.
> File ini **TIDAK PERLU nama domain**. Biarkan apa adanya.

```bash
nano nginx.conf
```

**Paste seluruh isi ini:**
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

> Simpan file: tekan `Ctrl+X`, lalu `Y`, lalu `Enter`.
> ℹ️ File ini **tidak perlu diubah**.

---

### 3.5 Buat File `docker-compose.yml`

```bash
nano docker-compose.yml
```

**Paste seluruh isi ini:**
```yaml
services:
  app:
    build: .
    container_name: swaratani_app
    restart: always
    environment:
      APP_ENV: production
      APP_DEBUG: "false"
      APP_URL: https://swaratani.id
      DB_CONNECTION: pgsql
      DB_HOST: GANTI_HOST_DATABASE
      DB_PORT: 5432
      DB_DATABASE: GANTI_NAMA_DATABASE
      DB_USERNAME: GANTI_USERNAME_DATABASE
      DB_PASSWORD: GANTI_PASSWORD_DATABASE
    ports:
      - "8080:8080"
    volumes:
      - ./src/storage:/var/www/html/storage
    networks:
      - webapps

networks:
  webapps:
    external: true
```

> ⚠️ GANTI bagian berikut sesuai koneksi database Anda:
> | Placeholder                | Ganti Dengan                         | Contoh                     |
> |----------------------------|--------------------------------------|----------------------------|
> | `GANTI_HOST_DATABASE`      | IP/hostname server database Anda     | `shared_postgres` atau `10.0.0.5` |
> | `GANTI_NAMA_DATABASE`      | Nama database yang sudah dibuat      | `db_swaratani`             |
> | `GANTI_USERNAME_DATABASE`  | Username database                    | `webadmin`                 |
> | `GANTI_PASSWORD_DATABASE`  | Password database                    | `supersecret123`           |
>
> ℹ️ Jika database ada di **server ini juga** (Postgres Docker), gunakan `DB_HOST: shared_postgres`.
> ℹ️ Jika database ada di **server lain**, gunakan IP server database tersebut.
>
> Simpan file: tekan `Ctrl+X`, lalu `Y`, lalu `Enter`.

---

### 3.6 Clone Source Code Laravel

```bash
cd /opt/docker-apps/swaratani/src
git clone https://github.com/USERNAME/REPO.git .
```

> ⚠️ GANTI `https://github.com/USERNAME/REPO.git` dengan URL repository GitHub Anda.
> ⚠️ Jangan lupa tanda **titik (`.`)** di akhir! Agar file masuk langsung tanpa subfolder.

---

### 3.7 Siapkan File `.env`

```bash
cp .env.example .env
nano .env
```

**Hapus semua isi `.env`, lalu paste seluruh isi ini:**

```env
APP_NAME=Swaratani
APP_ENV=production
APP_KEY=
APP_DEBUG=false
APP_URL=https://swaratani.id

APP_LOCALE=en
APP_FALLBACK_LOCALE=en
APP_FAKER_LOCALE=en_US

APP_MAINTENANCE_DRIVER=file

BCRYPT_ROUNDS=12

LOG_CHANNEL=stack
LOG_STACK=single
LOG_DEPRECATIONS_CHANNEL=null
LOG_LEVEL=error

DB_CONNECTION=pgsql
DB_HOST=GANTI_HOST_DATABASE
DB_PORT=5432
DB_DATABASE=GANTI_NAMA_DATABASE
DB_USERNAME=GANTI_USERNAME_DATABASE
DB_PASSWORD=GANTI_PASSWORD_DATABASE

SESSION_DRIVER=file
SESSION_LIFETIME=120
SESSION_ENCRYPT=false
SESSION_PATH=/
SESSION_DOMAIN=null

BROADCAST_CONNECTION=reverb
FILESYSTEM_DISK=local
QUEUE_CONNECTION=database

CACHE_STORE=database

REDIS_CLIENT=phpredis
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

MAIL_MAILER=resend
MAIL_FROM_ADDRESS="noreply@swaratani.id"
MAIL_FROM_NAME="Swaratani IoT"

RESEND_API_KEY=GANTI_RESEND_API_KEY

AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=
AWS_USE_PATH_STYLE_ENDPOINT=false

VITE_APP_NAME="${APP_NAME}"

# MQTT Configuration
MQTT_HOST=GANTI_HOST_MQTT
MQTT_PORT=1883
MQTT_USERNAME=GANTI_USERNAME_MQTT
MQTT_PASSWORD=GANTI_PASSWORD_MQTT

# Reverb (WebSocket) Configuration
REVERB_APP_ID=12345
REVERB_APP_KEY=swatani_key
REVERB_APP_SECRET=swatani_secret
REVERB_HOST="127.0.0.1"
REVERB_PORT=8080
REVERB_SCHEME=http

VITE_REVERB_APP_KEY="${REVERB_APP_KEY}"
VITE_REVERB_HOST="swaratani.id"
VITE_REVERB_PORT=8080
VITE_REVERB_SCHEME="https"
```

> ⚠️ GANTI bagian berikut:
>
> | Placeholder                | Keterangan                                    | Contoh                     |
> |----------------------------|-----------------------------------------------|----------------------------|
> | `GANTI_HOST_DATABASE`      | IP server database Anda                       | `203.194.115.76` atau `shared_postgres` |
> | `GANTI_NAMA_DATABASE`      | Nama database yang sudah dibuat               | `db_web1`                  |
> | `GANTI_USERNAME_DATABASE`  | Username database                             | `webadmin`                 |
> | `GANTI_PASSWORD_DATABASE`  | Password database                             | `Rizal1234`                |
> | `GANTI_HOST_MQTT`          | IP/nama container MQTT Broker                 | `mqtt_broker` atau `76.13.21.230` |
> | `GANTI_USERNAME_MQTT`      | Username MQTT yang sudah dibuat               | `iot`                      |
> | `GANTI_PASSWORD_MQTT`      | Password MQTT yang sudah dibuat               | `smartgh`                  |
> | `GANTI_RESEND_API_KEY`     | API key dari akun Resend.com Anda             | `re_RDP5z6b4_xxxx`        |
>
> ℹ️ Jika database ada di **server VPS ini** (Postgres Docker), gunakan `DB_HOST=shared_postgres`.
> ℹ️ Jika database ada di **server lain**, gunakan IP server database tersebut.
> ℹ️ Jika MQTT broker ada di **server VPS ini**, gunakan `MQTT_HOST=mqtt_broker`.
> ℹ️ Jika MQTT broker ada di **server lain**, gunakan IP server MQTT tersebut.
>
> Simpan file: tekan `Ctrl+X`, lalu `Y`, lalu `Enter`.

---

### 3.8 Build & Jalankan Project

```bash
cd /opt/docker-apps/swaratani
docker compose up -d --build
```

> ⏳ Proses build pertama kali bisa memakan waktu **5-15 menit** (download image, install composer, npm install, npm run build).

### 3.9 Cek Apakah Container Sudah Menyala

```bash
docker ps
```

> ✅ Harus muncul container `swaratani_app` dengan status `Up`.
> ❌ Jika tidak muncul atau status `Exited`, cek log error:
> ```bash
> docker logs swaratani_app --tail 50
> ```

---

### 3.10 Setup Awal Laravel (Hanya Sekali)

```bash
docker exec swaratani_app php artisan key:generate
docker exec swaratani_app php artisan storage:link
```

> ℹ️ Jika database **belum punya tabel** (database kosong), jalankan juga:
> ```bash
> docker exec swaratani_app php artisan migrate --force
> ```

---

# ═══════════════════════════════════════════
# LANGKAH 4: BUKA PORT FIREWALL
# ═══════════════════════════════════════════

```bash
ufw allow 22/tcp      # SSH (WAJIB! Agar tidak terkunci dari server)
ufw allow 80/tcp      # HTTP
ufw allow 443/tcp     # HTTPS
ufw allow 1883/tcp    # MQTT
ufw allow 9001/tcp    # MQTT WebSocket
ufw allow 8080/tcp    # Reverb WebSocket
ufw enable
```

> ⚠️ Saat diminta konfirmasi, ketik `y` lalu `Enter`.

---

# ═══════════════════════════════════════════
# LANGKAH 5: SETUP DNS DI CLOUDFLARE
# ═══════════════════════════════════════════

1. Login ke [Cloudflare](https://dash.cloudflare.com).
2. Pilih domain **swaratani.id**.
3. Buka menu **DNS** → **Records**.
4. Tambahkan A Record:
   - **Type:** `A`
   - **Name:** `@`
   - **IPv4 Address:** `IP_VPS_ANDA`
   - **Proxy status:** ☁️ (Proxied)
5. Tambahkan A Record untuk `www`:
   - **Type:** `A`
   - **Name:** `www`
   - **IPv4 Address:** `IP_VPS_ANDA`
   - **Proxy status:** ☁️ (Proxied)
6. Buka menu **SSL/TLS** → Set ke **"Flexible"**.

> ⚠️ GANTI `IP_VPS_ANDA` dengan IP VPS Anda.

---

# ═══════════════════════════════════════════
# ✅ SELESAI! WEBSITE ANDA SUDAH LIVE!
# ═══════════════════════════════════════════

Buka browser dan akses: **https://swaratani.id**

---

# 🔧 COMMAND-COMMAND BERGUNA (Referensi)

### Lihat Container yang Sedang Berjalan
```bash
docker ps
```

### Lihat Log Error Project
```bash
docker logs swaratani_app --tail 50
```

### Rebuild Setelah Update Code (Manual)
```bash
cd /opt/docker-apps/swaratani/src
git pull

cd /opt/docker-apps/swaratani
docker compose up -d --build
```

### 🚀 Rebuild Setelah Update Code (Otomatis via Script)

Project ini sudah menyertakan file `deploy.sh` di dalam source code (`src/deploy.sh`).
Script ini akan otomatis: **pull code → rebuild docker → migrate → cache config → restart**.

**Setup pertama kali (sekali saja):**
```bash
# Copy script dari src ke folder project dan beri izin eksekusi
cp /opt/docker-apps/swaratani/src/deploy.sh /opt/docker-apps/swaratani/deploy.sh
chmod +x /opt/docker-apps/swaratani/deploy.sh
```

**Setiap kali ingin deploy update:**
```bash
cd /opt/docker-apps/swaratani
bash deploy.sh
```

> ℹ️ Script ini melakukan langkah-langkah berikut secara otomatis:
> 1. `git pull origin main` — Tarik code terbaru dari GitHub
> 2. `docker compose up -d --build` — Rebuild image & restart container
> 3. `php artisan migrate --force` — Jalankan migrasi database
> 4. `php artisan config:cache` — Cache konfigurasi
> 5. `php artisan route:cache` — Cache routing
> 6. `php artisan view:cache` — Cache view/template
> 7. `docker restart swaratani_app` — Restart container

**Isi file `deploy.sh`:**
```bash
#!/bin/bash
# ============================================
# Swaratani Deploy Script
# Jalankan di server: bash deploy.sh
# ============================================

set -e

PROJECT_DIR="/opt/docker-apps/swaratani"
CONTAINER="swaratani_app"

echo "🔄 Pulling latest code from GitHub..."
cd "$PROJECT_DIR/src"
git pull origin main

echo "🔨 Building & restarting Docker..."
cd "$PROJECT_DIR"
docker compose up -d --build

echo "⏳ Waiting for container to start..."
sleep 5

echo "🗄️ Running migrations..."
docker exec $CONTAINER php artisan migrate --force

echo "⚡ Caching config..."
docker exec $CONTAINER php artisan config:cache
docker exec $CONTAINER php artisan route:cache
docker exec $CONTAINER php artisan view:cache

echo "🔄 Restarting container..."
docker restart $CONTAINER

echo ""
echo "✅ Deploy selesai! Cek: https://swaratani.id"
echo ""
docker ps | grep $CONTAINER
```

### Jalankan Artisan Command
```bash
docker exec swaratani_app php artisan cache:clear
docker exec swaratani_app php artisan config:clear
docker exec swaratani_app php artisan migrate --force
```

### Restart Nginx Proxy (Setelah Edit Domain)
```bash
cd /opt/docker-apps/shared
docker compose restart nginx-proxy
```

### Cek Log MQTT
```bash
docker exec swaratani_app cat /var/www/html/storage/logs/mqtt.log
```

### Cek Log Reverb
```bash
docker exec swaratani_app cat /var/www/html/storage/logs/reverb.log
```

### Cek Log Queue Worker
```bash
docker exec swaratani_app cat /var/www/html/storage/logs/queue.log
```

### Matikan Project
```bash
cd /opt/docker-apps/swaratani
docker compose down
```

### Matikan Semua Service
```bash
cd /opt/docker-apps/shared && docker compose down
cd /opt/docker-apps/swaratani && docker compose down
```

---

# 🆘 TROUBLESHOOTING (Solusi Masalah)

### ❌ Website tidak bisa diakses (Cloudflare Error 521)
```bash
# 1. Cek apakah semua container menyala
docker ps

# 2. Cek log nginx proxy
docker logs nginx_proxy --tail 20

# 3. Test akses dari dalam server
curl -I http://localhost

# 4. Cek firewall
ufw status
```

### ❌ Error 500 Internal Server Error
```bash
# 1. Cek apakah APP_KEY sudah di-generate
docker exec swaratani_app php artisan key:generate --force

# 2. Clear cache
docker exec swaratani_app php artisan config:clear
docker exec swaratani_app php artisan cache:clear

# 3. Cek permission storage
docker exec swaratani_app chmod -R 777 storage
docker exec swaratani_app chmod -R 777 bootstrap/cache
```

### ❌ Error "host not found in upstream" di Nginx
Artinya container project belum menyala saat Nginx Proxy dinyalakan.
```bash
# 1. Pastikan container project running
docker ps

# 2. Nyalakan ulang project jika belum running
cd /opt/docker-apps/swaratani && docker compose up -d

# 3. Restart nginx proxy
cd /opt/docker-apps/shared && docker compose restart nginx-proxy
```

### ❌ Error "composer.json not found" saat build
Artinya `git clone` membuat subfolder tambahan. Pastikan clone pakai titik (`.`) di akhir:
```bash
cd /opt/docker-apps/swaratani/src
rm -rf *
git clone https://github.com/USERNAME/REPO.git .
```

---

# 📋 CHECKLIST DEPLOY (Untuk Verifikasi)

- [ ] Docker & Docker Compose terinstall
- [ ] Network `webapps` sudah dibuat
- [ ] Folder `shared/` berisi: `docker-compose.yml`, `nginx.conf`, `mosquitto.conf`, `passwd`
- [ ] Password MQTT sudah di-generate
- [ ] Container `shared_postgres`, `mqtt_broker`, `nginx_proxy` sudah `Up`
- [ ] Folder `swaratani/` berisi: `Dockerfile`, `docker-compose.yml`, `nginx.conf`, `supervisord.ini`
- [ ] Source code sudah di-clone ke `swaratani/src/`
- [ ] File `.env` sudah dikonfigurasi (DB, MQTT, Reverb)
- [ ] Container `swaratani_app` sudah `Up`
- [ ] `php artisan key:generate` sudah dijalankan
- [ ] `php artisan storage:link` sudah dijalankan
- [ ] Port firewall sudah dibuka (22, 80, 443, 1883, 8080)
- [ ] DNS Cloudflare sudah mengarah ke IP VPS
- [ ] Website bisa diakses di browser

---

# 🆕 PEMBARUAN TERBARU (Juli 2026)

Aplikasi Swaratani telah mendapatkan beberapa update terbaru yang juga sudah tercakup dalam konfigurasi deploy ini:
1. **Pembaruan Real-time via Laravel Reverb**: Sistem monitoring kini menggunakan WebSocket (port 8080) untuk pembaharuan data secara instan tanpa perlu reload atau polling (menghemat resource server).
2. **Optimistic UI & Pengaman Tombol**: Fitur kontrol relay kini memiliki *debounce* dan jeda 20 detik setelah diklik untuk mencegah spam dan tabrakan data (race condition).
3. **Penyeragaman Navbar**: Tampilan antarmuka atas (Navbar) telah diseragamkan untuk seluruh sistem (Beranda, Monitoring, Riwayat, Jadwal, Admin, dll) untuk meningkatkan user experience.
4. **Peningkatan Keamanan**: Data sensitif pada file konfigurasi lokal telah dihapus dari *tracking* Git untuk menjaga rahasia *credentials*.
5. **Perbaikan Bug Jadwal**: Telah memperbaiki perhitungan *offset* sektor irigasi pada tampilan kalender/jadwal, serta merapikan sistem edit perangkat admin.
