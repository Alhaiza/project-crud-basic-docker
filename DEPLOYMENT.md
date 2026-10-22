# Panduan Komprehensif Deployment & DevOps: crud-docker

Dokumen ini menjelaskan secara mendalam bagaimana modul orkestrasi **crud-docker** bekerja untuk mengintegrasikan **crud-fe** (Nuxt 4), **crud-be** (Laravel 12), dan basis data MySQL 8.0, serta panduan praktis untuk deployment di komputer lokal maupun server produksi Linux/VPS.

---

## 1. Arsitektur & Cara Kerja Deployment

Modul `crud-docker` mengadopsi pola arsitektur **Unified Ingress Gateway Pattern** menggunakan **Docker Compose** dan **Nginx Reverse Proxy**.

### Diagram Arsitektur & Topologi Jaringan

```mermaid
flowchart TD
    User["Peramban Klien / Publik\n(http://domain.com atau http://localhost)"] -->|Port 80 / 443| Ingress["Nginx Ingress Reverse Proxy\n(crud-nginx:80)"]

    subgraph Docker Network: crud-network [Jaringan Privat Terisolasi: crud-network]
        Ingress -->|1. URI /api/*\nvia FastCGI:9000| Backend["Backend API: Laravel 12\n(PHP 8.2-FPM Alpine)"]
        Ingress -->|2. URI /* (Non-API)\nvia HTTP Proxy:3000| Frontend["Frontend UI: Nuxt 4\n(Standalone Nitro Node 22)"]
        Backend -->|3. Query SQL PDO\nPort 3306| Database[("Database: MySQL 8.0\nVolume: crud_db_data")]
    end

    Dev["Aplikasi GUI Host (DBeaver / DataGrip)"] -.->|Port 3307 External| Database
```

### 5 Tahapan Siklus Hidup (Lifecycle) Deployment

1. **Perakitan Multi-Stage Build:**
   - **Frontend Nuxt 4 (`../crud-fe/Dockerfile`):** Stage builder mengompilasi Vite dan menghasilkan folder `.output`. Seluruh `node_modules` dibuang di stage runner, menghasilkan kontainer ringan (~27 MiB RAM).
   - **Backend Laravel 12 (`../crud-be/Dockerfile`):** Stage vendor Composer mengunduh paket produksi (`--no-dev`), lalu disalin ke runtime PHP 8.2-FPM Alpine dengan OPcache aktif (~32 MiB RAM).

2. **Orkestrasi Booting & Healthcheck (`docker-compose.yml`):**
   - MySQL boot terlebih dahulu dan dipantau via `mysqladmin ping`.
   - Backend ditahan sampai MySQL `healthy`, lalu skrip `entrypoint.sh` menjalankan TCP socket probe (`nc -z`), auto-generate `APP_KEY`, pembersihan residu cache Windows, optimasi rute, dan migrasi tabel (`php artisan migrate --force`).
   - Frontend dan Nginx Ingress aktif setelah backend siap.

3. **Jaringan Internal Terisolasi (`crud-network`):**
   - Seluruh kontainer berkomunikasi lewat DNS internal (`database:3306`, `backend:9000`, `frontend:3000`).
   - Port backend dan frontend tidak diekspos ke publik (aman dari port scan).

4. **Routing Satu Pintu & Zero CORS (Nginx Ingress):**
   - Port 80 Nginx membedah URI: prefix `/api/*` dialihkan ke FastCGI PHP-FPM, sedangkan rute lainnya `/*` dialihkan ke server Nitro Nuxt 4.
   - Karena berjalan di satu origin yang sama, browser tidak mengalami masalah CORS dan cookie Sanctum aman.

5. **Persistensi Data (Docker Volumes):**
   - `crud_db_data`: Menjaga database MySQL tetap aman saat kontainer dimatikan/di-rebuild.
   - `crud_be_storage`: Menjaga file upload, log, dan sesi Laravel tetap utuh.

---

## 2. Pemetaan Berkas di Dalam `crud-docker/`

| Berkas | Peran Utama | Alasan & Cara Kerjanya |
| :--- | :--- | :--- |
| **`docker-compose.yml`** | Master Conductor | Mengorkestrasikan 4 layanan dengan referensi build context `../crud-be` dan `../crud-fe`. |
| **`nginx/default.conf`** | Ingress Router | Menyatukan port publik ke port 80, memisahkan lalu lintas `/api/*` (FastCGI) dan `/*` (Nuxt Nitro). |
| **`.env.example`** | Template Kredensial | Berkas contoh konfigurasi lingkungan untuk port publik, kredensial database, dan encryption key. |
| **`.env`** | Konfigurasi Aktif | Variabel lingkungan yang aktif dibaca oleh `docker-compose.yml`. |
| **`project-context.md`** | Context Memory | Dokumentasi internal untuk AI Agent & DevOps engineer. |

---

## 3. Panduan Deploy di Komputer Lokal (Pengujian)

### Langkah 1: Masuk ke Folder `crud-docker`

```bash
cd crud-docker
```

### Langkah 2: Siapkan File `.env`

```bash
cp .env.example .env
```

*(Nilai default sudah diatur untuk port HTTP 80 dan port MySQL eksternal 3307).*

### Langkah 3: Build & Nyalakan Cluster

```bash
docker compose up -d --build
```

### Langkah 4: Verifikasi Status Kontainer

```bash
docker compose ps
```

Pastikan keempat kontainer berstatus `Up` (dan database `healthy`):

- `crud-database` (port 3307 -> 3306)
- `crud-backend` (port 9000 internal)
- `crud-frontend` (port 3000 internal)
- `crud-nginx` (port 80 publik)

### Langkah 5: Akses Aplikasi

- **Aplikasi Web Frontend:** [http://localhost/](http://localhost/)
- **REST API Backend:** [http://localhost/api/products](http://localhost/api/products)
- **GUI Database (DBeaver):** Host `localhost`, Port `3307`, User `root`, Password `root`.

### Menghentikan Cluster

```bash
docker compose down
```

---

## 4. Panduan Deploy ke Server Linux / VPS Produksi

### Langkah 1: Hubungkan ke VPS & Pasang Docker

```bash
ssh root@ip_server_anda

# Update paket sistem
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git ufw

# Pasang Docker resmi
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo apt install -y docker-compose-plugin

# Konfigurasi firewall
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
```

### Langkah 2: Clone Repositori Monorepo

```bash
git clone https://github.com/username/mono-repo-crud-basic.git /var/www/app-crud
cd /var/www/app-crud/crud-docker
```

### Langkah 3: Konfigurasi Kredensial Produksi (`.env`)

```bash
cp .env.example .env
nano .env
```

*Ganti `DB_ROOT_PASSWORD` dan `DB_PASSWORD` dengan kombinasi karakter acak yang kuat.*

### Langkah 4: Bangun & Jalankan Kontainer

```bash
docker compose up -d --build
docker compose ps
docker compose logs -f backend
```

### Langkah 5: Konfigurasi HTTPS / SSL Otomatis via Certbot

1. Arahkan domain (misal `app.domainanda.com`) ke IP server.

2. Pasang Certbot:

   ```bash
   sudo apt install -y certbot
   ```

3. Hentikan Nginx sejenak:

   ```bash
   docker compose stop nginx
   ```

4. Generate sertifikat:

   ```bash
   sudo certbot certonly --standalone -d app.domainanda.com
   ```

5. Buka `docker-compose.yml`, ekspos port 443 dan mount volume sertifikat `/etc/letsencrypt:/etc/letsencrypt:ro`.

6. Tambahkan blok SSL di `nginx/default.conf` lalu hidupkan kembali:

   ```bash
   docker compose up -d nginx
   ```

---

## 5. Runbook Pemeliharaan & Troubleshooting

```bash
# Masuk ke direktori crud-docker
cd crud-docker

# Memperbarui aplikasi setelah ada git push baru di server
git pull origin main
docker compose up -d --build

# Menjalankan artisan di kontainer backend
docker compose exec backend php artisan db:seed
docker compose exec backend php artisan optimize:clear

# Backup database ke file SQL
docker compose exec database mysqldump -u root -p${DB_ROOT_PASSWORD} db_project_mono_repo_crud > backup_$(date +%Y%m%d).sql

# Monitor penggunaan RAM
docker stats --no-stream
```
