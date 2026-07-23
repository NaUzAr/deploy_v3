# 🚨 SOP: MIGRASI & DEPLOYMENT SERVER BARU (DARI NOL)

SOP ini dibuat khusus untuk memandu _developer_ atau tim melakukan reset (wipe out) pada VPS lama dan melakukan _deployment_ ulang menggunakan infrastruktur `shared/` terbaru yang mendukung **Laravel Reverb, Queue, Vite, dan MQTT**.

⚠️ **PENTING**: Semua skrip dan file konfigurasi (_copy-paste_) merujuk pada file panduan utama: `cara_with_mqtt.md`. SOP ini hanya sebagai pedoman langkah kerja (urutan eksekusi).

---

## FASE 1: PERSIAPAN SERVER (Pilih salah satu)

**Opsi A: Jika Menggunakan Server Baru (Fresh Install OS)**
Jika Anda baru saja me-rebuild OS atau membeli VPS baru, Anda wajib meng-_install_ Docker terlebih dahulu:
1. Masuk ke VPS via SSH (`ssh root@IP_VPS`).
2. _Update_ sistem dan _install_ Docker:
   ```bash
   apt update && apt upgrade -y
   curl -fsSL https://get.docker.com -o get-docker.sh && sh get-docker.sh
   apt install docker-compose-plugin -y
   ```

**Opsi B: Jika Memakai Server Lama (Wipe Out)**
Langkah ini akan menghapus total konfigurasi Docker lama yang berantakan agar VPS kembali bersih.
1. Masuk ke VPS via SSH (`ssh root@IP_VPS`).
2. Pastikan sudah ada **Snapshot** atau **Backup** di panel VPS sebelum memulai eksekusi.
3. Matikan dan hapus semua _container_ Docker lama:
   ```bash
   docker stop $(docker ps -a -q)
   docker rm $(docker ps -a -q)
   ```
4. Hapus seluruh folder _project_ dan konfigurasi lama:
   ```bash
   rm -rf /opt/docker-apps/*
   ```

---

## FASE 2: SETUP INFRASTRUKTUR UTAMA (SHARED)
Langkah ini membangun ulang **Nginx Proxy**, **MQTT Broker**, dan **Postgres** dalam satu tempat (folder `shared/`) yang rapi.

1. Buat folder dan jaringan docker baru:
   ```bash
   mkdir -p /opt/docker-apps/shared
   cd /opt/docker-apps/shared
   docker network create webapps
   ```
2. Buka tutorial **`cara_with_mqtt.md`** di repositori, lalu lakukan _copy-paste_ isi file untuk membuat:
   - `nginx.conf`
   - `mosquitto.conf`
   - `docker-compose.yml`
   - Buat file `passwd` (awalnya kosong) dengan perintah `touch passwd`
3. _Generate_ password untuk MQTT Broker (Ganti `rahasia` dengan password yang diinginkan):
   ```bash
   docker run --rm -v "/opt/docker-apps/shared/passwd:/passwd" eclipse-mosquitto:2 mosquitto_passwd -b /passwd admin rahasia
   ```
4. Jalankan infrastruktur:
   ```bash
   docker compose up -d
   ```

---

## FASE 3: DEPLOY PROJECT (Contoh: Swaratani Web)
Fase ini untuk meng-_clone_ _source code_ dan menjalankan project Laravel.

1. Buat folder _project_:
   ```bash
   mkdir -p /opt/docker-apps/swaratani_web/src
   cd /opt/docker-apps/swaratani_web
   ```
2. Sama seperti Fase 2, _copy-paste_ file _template project_ dari **`cara_with_mqtt.md`**:
   - `Dockerfile` (sudah termasuk perintah NodeJS/Vite)
   - `supervisord.ini` (sudah termasuk Queue & Reverb)
   - `nginx.conf`
   - `docker-compose.yml` (mengekspos port `8080:8080` untuk Reverb)
3. _Clone source code_ dan atur `.env`:
   ```bash
   cd src
   git clone https://github.com/USERNAME/REPO_SWARATANI.git .
   cp .env.example .env
   nano .env
   ```
   > 💡 **Penting**: Karena _database_ berada di server eksternal, pastikan `DB_HOST`, `DB_USERNAME`, dan `DB_PASSWORD` di `.env` diisi dengan kredensial akses database eksternal tersebut. (Bukan `shared_postgres`).
4. _Build_ dan jalankan aplikasi:
   ```bash
   cd /opt/docker-apps/swaratani_web
   docker compose up -d --build
   ```
5. _Setup_ awal Laravel:
   ```bash
   docker exec swaratani_web_app php artisan key:generate
   docker exec swaratani_web_app php artisan storage:link
   ```
   _(Perintah `php artisan migrate` bisa dilewati jika database eksternal Anda sudah ada tabelnya)._

---

## FASE 4: MENGHUBUNGKAN DOMAIN KE NGINX PROXY
Agar `swaratani_web` bisa diakses publik lewat nama domain atau IP.

1. Daftarkan domain di file konfigurasi _proxy_ utama:
   ```bash
   nano /opt/docker-apps/shared/nginx.conf
   ```
2. Tambahkan _block_ kode `server { ... }` (ambil contoh dari tutorial `cara_with_mqtt.md`) dan pastikan `proxy_pass` mengarah ke nama container: `http://swaratani_web_app:80`.
3. _Restart_ Nginx Proxy:
   ```bash
   cd /opt/docker-apps/shared
   docker compose restart nginx-proxy
   ```

VPS sudah siap dan project web berhasil di-_deploy_ dengan struktur baru secara utuh!
