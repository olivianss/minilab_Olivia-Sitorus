# Minilab Big Data

Panduan setup lokal untuk minilab big data menggunakan **Python**, **PostgreSQL**, dan **MinIO** sebagai object storage. Dokumen ini menjelaskan langkah-langkah konfigurasi lingkungan pengembangan, mulai dari instalasi dependensi hingga proses ingestion data ke datalake.

---

## Prasyarat

Pastikan sudah terinstal di sistem kamu:

| Tools | Versi Minimum | 
|---|---|
| Python | 3.8+ | 
| Docker Desktop | Terbaru | 
---

## Langkah Setup

### Langkah 1  Cek Python

Buka terminal, cek apakah Python sudah ada:

```bash
python --version
```
> Pastikan output menampilkan **Python 3.8** ke atas.

---

### Langkah 2 Buat Folder Proyek & Virtual Environment

```bash
mkdir minilab-bigdata
cd minilab-bigdata
python -m venv venv
```

Aktifkan virtual environment:

```bash
# Windows
venv\Scripts\activate
```

> Setelah aktif, prefix `(venv)` akan muncul di awal baris terminal.

---

### Langkah 3 Install Library Python

Pastikan venv sudah aktif, lalu jalankan:

```bash
pip install pandas sqlalchemy psycopg2-binary openpyxl minio jupyter
```

**Fungsi masing-masing library:**

| Library | Kegunaan |
|---|---|
| `pandas` | Manipulasi data — membaca CSV/Excel menjadi DataFrame |
| `sqlalchemy` | Koneksi ke PostgreSQL via Python |
| `psycopg2-binary` | Driver/adapter PostgreSQL untuk Python |
| `openpyxl` | Membaca dan menulis file Excel (`.xlsx`) |
| `minio` | Client Python untuk MinIO object storage |
| `jupyter` | Notebook interaktif untuk eksplorasi data |

---

### Langkah 4  Buat File `docker-compose.yml`

Buat file `docker-compose.yml` di dalam folder `minilab-bigdata`:

```yaml
services:
  postgres:
    image: postgres:15
    container_name: minilab-postgres
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: admin123
      POSTGRES_DB: minilab_db
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  minio:
    image: minio/minio
    container_name: minilab-minio
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin123
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio_data:/data
    command: server /data --console-address ":9001"

volumes:
  postgres_data:
  minio_data:
```

**Kredensial layanan:**

| Layanan | Username | Password | Port |
|---|---|---|---|
| PostgreSQL | `admin` | `admin123` | `5432` |
| MinIO API | `minioadmin` | `minioadmin123` | `9000` |
| MinIO Console | `minioadmin` | `minioadmin123` | `9001` |

---

### Langkah 5  Jalankan Docker Compose

Pastikan Docker Desktop sudah berjalan, lalu:

```bash
# Jalankan semua container di background
docker compose up -d

# Verifikasi — harus ada 2 container dengan status "Up"
docker ps
```

### Langkah 6  Akses MinIO & Buat Bucket

1. Buka browser, akses: **http://localhost:9001**
2. Login dengan:
   - **Username:** `minioadmin`
   - **Password:** `minioadmin123`
3. Klik **"Create a Bucket"**
4. Beri nama bucket: **`minilab-datalake`**
5. Klik **"Create Bucket"**

> Bucket ini akan menjadi tempat penyimpanan semua file data mentah (raw layer datalake).

---

### Langkah 7 — Buat Tabel PostgreSQL

#### Buat tabel
with engine.connect() as conn:
    conn.execute(sqlalchemy.text("""
    CREATE TABLE IF NOT EXISTS anggota (
        anggota_id   INT PRIMARY KEY,
        nama         VARCHAR(100),
        kota         VARCHAR(100),
        umur         INT,
        tanggal_daftar DATE
    );

    INSERT INTO anggota VALUES
    (1,  'Alfian',   'Surabaya',   22, '2024-01-05'),
    (2,  'Brenda',   'Jakarta',    25, '2024-02-10'),
    (3,  'Cahyo',    'Bandung',    28, '2024-01-15'),
    (4,  'Dinda',    'Malang',     21, '2024-03-01'),
    (5,  'Erwin',    'Semarang',   30, '2024-02-20'),
    (6,  'Fauzia',   'Yogyakarta', 24, '2024-01-28'),
    (7,  'Gilang',   'Medan',      27, '2024-03-10'),
    (8,  'Hesti',    'Makassar',   23, '2024-02-05'),
    (9,  'Ilham',    'Depok',      31, '2023-12-20'),
    (10, 'Jasmine',  'Bekasi',     26, '2024-01-18'),
    (11, 'Kevin',    'Surabaya',   29, '2024-03-22'),
    (12, 'Laras',    'Jakarta',    20, '2024-02-14'),
    (13, 'Mahesa',   'Bandung',    33, '2023-11-30'),
    (14, 'Nabila',   'Malang',     22, '2024-04-01'),
    (15, 'Oscar',    'Semarang',   28, '2024-01-10'),
    (16, 'Prita',    'Yogyakarta', 25, '2024-03-18'),
    (17, 'Qori',     'Medan',      32, '2023-12-10'),
    (18, 'Rendi',    'Makassar',   24, '2024-02-22'),
    (19, 'Siska',    'Jakarta',    27, '2023-11-15'),
    (20, 'Taufik',   'Surabaya',   35, '2024-01-25')
    ON CONFLICT (anggota_id) DO NOTHING;


### Langkah 8 Script Ingestion ke MinIO

from minio import Minio
from minio.error import S3Error
import pandas as pd
import sqlalchemy
import io

#### Koneksi MinIO
```
client = Minio(
    "localhost:9000",
    access_key="minioadmin",
    secret_key="minioadmin123",
    secure=False
)
```
#### Fungsi upload DataFrame ke MinIO
```
def upload_to_minio(df, path):
    csv_bytes = df.to_csv(index=False).encode('utf-8')
    client.put_object(
        bucket,
        path,
        data=io.BytesIO(csv_bytes),
        length=len(csv_bytes),
        content_type="text/csv"
    )
    print(f"✅ Berhasil upload: {path} ({len(df)} baris)")
```
#### Koneksi PostgreSQL
```
engine = sqlalchemy.create_engine('postgresql://admin:admin123@localhost:5432/minilab_db')
```

#### Ingestion dari RDBMS
```
df_pinjam  = pd.read_sql("SELECT * FROM peminjaman", engine)

upload_to_minio(df_pinjam,  "raw/rdbms/peminjaman.csv")
```

#### Ingestion dari CSV
```
df_csv = pd.read_csv("data_buku.csv")
upload_to_minio(df_csv, "raw/csv/data_buku.csv")
```
#### Ingestion dari XLSX
```
df_xlsx = pd.read_excel("data_anggota.xlsx")
upload_to_minio(df_xlsx, "raw/xlsx/data_anggota.csv")
```
