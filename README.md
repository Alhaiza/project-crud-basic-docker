# CRUD Docker Orchestration (`crud-docker`)

Modul orkestrasi kontainer terpadu untuk stack fullstack monorepo:

- **`crud-fe`**: Nuxt 4 Frontend (Node 22 Nitro)
- **`crud-be`**: Laravel 12 Backend REST API (PHP 8.2-FPM)
- **`database`**: MySQL 8.0 Community Server
- **`nginx`**: Ingress Gateway & Reverse Proxy (Port 80)

## Menjalankan Cepat

```bash
# 1. Pindah ke direktori ini
cd crud-docker

# 2. Siapkan file .env
cp .env.example .env

# 3. Jalankan seluruh stack
docker compose up -d --build

# 4. Periksa status
docker compose ps
```

- **Frontend:** [http://localhost/](http://localhost/)
- **Backend API:** [http://localhost/api/products](http://localhost/api/products)
- **Database GUI (DBeaver):** `localhost:3307` (user: `root`, pass: `root`)

Untuk panduan deployment lengkap ke VPS dan arsitektur teknis, baca [DEPLOYMENT.md](file:///d:/Project/Project/mono-repo-crud-basic/crud-docker/DEPLOYMENT.md).
