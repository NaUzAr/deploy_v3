# 🚀 Panduan Setup & Clone Web Multi-Project (Local Windows)

Panduan ini berisi cara mensimulasikan lingkungan server (VPS) yang memiliki banyak website, tetapi langsung di dalam komputer Windows Anda (di drive `D:\Dev\SWATANI\setup_tutorial`).

---

## 🛠️ BAGIAN 1: YANG HANYA DILAKUKAN 1x SEUMUR HIDUP (Infrastruktur Dasar)
Bagian ini hanya perlu dijalankan sekali untuk menyalakan database, MQTT, dan Nginx secara global. Pastikan Docker Desktop sudah menyala di Windows Anda.

1. Buka Terminal/PowerShell di Windows.
2. Buat jaringan docker terlebih dahulu:
   ```powershell
   docker network create webapps
   ```
3. Jalankan **PostgreSQL Shared**:
   ```powershell
   cd D:\Dev\SWATANI\setup_tutorial\shared\postgres
   docker compose up -d
   ```
4. Jalankan **MQTT Broker**:
   ```powershell
   cd D:\Dev\SWATANI\setup_tutorial\shared\mqtt
   docker compose up -d
   ```
5. Jalankan **Nginx Proxy**:
   ```powershell
   cd D:\Dev\SWATANI\setup_tutorial\shared\nginx-proxy
   docker compose up -d
   ```

Sekarang infrastruktur dasar Anda sudah berjalan!

---

## 🌍 BAGIAN 2: SETIAP KALI INGIN CLONE/MENAMBAH WEB BARU

Setiap kali Anda ingin men-deploy website baru (misal Anda punya web clone `web_baru`), ikuti panduan berikut.

### Langkah 1: Buat Folder Project
Buat folder baru di dalam `projects/`, misal `web_baru`.
```powershell
mkdir D:\Dev\SWATANI\setup_tutorial\projects\web_baru\src
```

### Langkah 2: Buat Database (Jika Pakai Laravel)
```powershell
docker exec -it shared_postgres psql -U webadmin -c "CREATE DATABASE db_web_baru;"
```

### Langkah 3: Gunakan Template dari Folder `templates/`
*Copy* semua file dari `templates/laravel/` (seperti `docker-compose.yml`, `Dockerfile`, `nginx.conf`, `supervisord.ini`) dan *Paste* ke dalam folder `D:\Dev\SWATANI\setup_tutorial\projects\web_baru\`.

**Yang Perlu Diubah di Template `docker-compose.yml` (Laravel):**
- Buka file `docker-compose.yml` di folder `web_baru`.
- Ganti `container_name: NAMA_PROJECT_app` menjadi `container_name: web_baru_app`.
- Ganti `DB_DATABASE: db_NAMA_PROJECT` menjadi `DB_DATABASE: db_web_baru`.
- Masukkan password Anda di `DB_PASSWORD: secretpassword`.

### Langkah 4: Masukkan Source Code (Clone)
Buka terminal di dalam folder `src` dan clone repository Anda:
```powershell
cd D:\Dev\SWATANI\setup_tutorial\projects\web_baru\src
git clone https://github.com/USERNAME/REPO_ANDA.git .
```
*(Jangan lupa titik `.` di belakang agar *file* langsung masuk tanpa sub-folder)*.

### Langkah 5: Siapkan File Konfigurasi Laravel (.env)
Ini adalah langkah paling krusial. Copy `.env.example` menjadi `.env`.
```powershell
cp .env.example .env
```
Buka file `.env` dengan editor teks, dan pastikan konfigurasi *database* mengarah ke Postgres Docker:
```env
DB_CONNECTION=pgsql
DB_HOST=shared_postgres
DB_PORT=5432
DB_DATABASE=db_web_baru
DB_USERNAME=webadmin
DB_PASSWORD=secretpassword
```

### Langkah 6: Nyalakan Docker Project Baru
Kembali ke folder `web_baru` (di mana ada `docker-compose.yml`), lalu jalankan:
```powershell
cd D:\Dev\SWATANI\setup_tutorial\projects\web_baru
docker compose up -d --build
```

### Langkah 7: Setup Laravel Pertama Kali
Jalankan perintah ini hanya saat pertama kali clone:
```powershell
docker exec web_baru_app php artisan key:generate
docker exec web_baru_app php artisan migrate --force
docker exec web_baru_app php artisan storage:link
```

### Langkah 8: Daftarkan Domain Lokal ke Nginx Proxy
Agar Anda bisa mengaksesnya di browser dengan nama cantik (misal `swatani.local`), edit konfigurasi Nginx Proxy:
1. Buka file `D:\Dev\SWATANI\setup_tutorial\shared\nginx-proxy\nginx.conf`.
2. Tambahkan atau edit block `server {}` seperti ini:
   ```nginx
   server {
       listen 80;
       server_name webbaru.local;
       
       location / {
           proxy_pass http://web_baru_app:80;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }
   }
   ```
3. Restart Nginx Proxy:
   ```powershell
   cd D:\Dev\SWATANI\setup_tutorial\shared\nginx-proxy
   docker compose restart
   ```
4. Tambahkan `127.0.0.1 webbaru.local` ke file `C:\Windows\System32\drivers\etc\hosts` Anda.

**SELESAI!** Anda bisa membuka `http://webbaru.local` di browser Anda! Ulangi Langkah 1-8 ini untuk setiap website baru.
