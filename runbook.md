# 🚩 Runbook: Setup Lokal [DW_Project_Kelompok22]

Dokumen ini adalah panduan langkah-demi-langkah (SOP) untuk menginstal, mengkonfigurasi, dan menjalankan proyek `DW_Project_Kelompok22` di lingkungan pengembangan lokal menggunakan Docker.

**Terakhir Diperbarui:** 22 Oktober 2025

---

## 1. Prasyarat (Prerequisites)

Pastikan sistem Anda memenuhi persyaratan minimum berikut sebelum memulai:

- **Docker Engine:** Versi `20.10+`
- **Docker Compose:** Versi `V2+`
- **RAM:** Minimal 8 GB
- **Disk:** Minimal 20 GB ruang kosong

---

## 2. Instalasi dan Konfigurasi

Ikuti langkah-langkah ini secara berurutan untuk menyiapkan file proyek.

### Langkah 2.1: Clone Proyek dan Siapkan Environment

1.  Clone repositori proyek (ganti `[URL_REPO_ANDA]` dengan URL Git Anda):
    ```bash
    git clone [URL_REPO_ANDA]/DW_Project_Kelompok22.git
    ```
2.  Masuk ke direktori proyek:
    ```bash
    cd DW_Project_Kelompok22
    ```
3.  Salin file konfigurasi _environment_ dari contoh:
    ```bash
    cp env.example .env
    ```

### Langkah 2.2: Konfigurasi File `.env`

Buka file `.env` yang baru Anda salin dan atur _password_ untuk SQL Server.

```ini
# Atur password ini
MSSQL_SA_PASSWORD=PTXYZSecure123!

# ... (variabel lain mungkin ada di sini)
```

### Langkah 2.3: Modifikasi `docker-compose.yml`

Buka file `docker-compose.yml` dan lakukan 3 perubahan berikut:

1.  **Tambahkan `healthcheck`** pada layanan `superset`:
    ```yaml
    services:
      superset:
        # ... (konfigurasi superset lainnya)
        healthcheck:
          test: ["CMD-SHELL", "curl -f http://localhost:8088/health"]
          interval: 30s
          timeout: 30s
          retries: 5
          start_period: 300s
    ```
2.  **Pastikan `sqlserver`** menggunakan _environment variable_ untuk password:
    ```yaml
    services:
      sqlserver:
        # ...
        environment:
          - ACCEPT_EULA=Y
          - MSSQL_SA_PASSWORD=${MSSQL_SA_PASSWORD} # <-- Pastikan ini ada
    ```
3.  **Tambahkan `healthcheck`** pada layanan `sqlserver`:
    ```yaml
    services:
      sqlserver:
        # ... (environment di atas)
        healthcheck:
          # CATATAN: Menggunakan ${MSSQL_SA_PASSWORD} agar konsisten dengan .env
          test: ["CMD-SHELL", "/opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P '${MSSQL_SA_PASSWORD}' -Q 'SELECT 1' -C -N"]
          interval: 30s
          timeout: 10s
          retries: 10
          start_period: 60s
    ```

### Langkah 2.4: Pastikan Isi `requirements.txt`

Pastikan file `requirements.txt` (yang digunakan oleh layanan Python/ETL Anda) berisi _library_ berikut:

```txt
pandas
pyodbc
python-dotenv
sqlalchemy
```

### Langkah 2.5: Modifikasi Kode ETL `standaloneetl.py`

Buka file `standaloneetl.py` dan lakukan 2 modifikasi kode penting:

1.  **Perbarui Path CSV di `extract_and_load_to_staging`:**
    Ganti _hardcoded path_ untuk membaca CSV.

    - _Temukan dan ganti baris seperti ini:_
      ```python
      # equipment_df = pd.read_csv('data/raw/Dataset/dataset_alat_berat_dw.csv')
      ```
    - _Dengan kode ini (biasanya di dekat bagian `import` atau di dalam fungsi):_

      ```python
      import os

      # Tentukan base path
      BASE_DATA_PATH = os.path.join('data', 'raw', 'Dataset')

      # Buat path lengkap
      equipment_path = os.path.join(BASE_DATA_PATH, 'dataset_alat_berat_dw.csv')
      production_path = os.path.join(BASE_DATA_PATH, 'dataset_production.csv')
      transaction_path = os.path.join(BASE_DATA_PATH, 'dataset_transaksi.csv')

      # Pastikan Anda menggunakan variabel ini saat membaca CSV
      # cth: equipment_df = pd.read_csv(equipment_path)
      ```

2.  **Ganti Fungsi `get_sql_connection()`:**
    Hapus fungsi `get_sql_connection()` yang lama dan ganti **sepenuhnya** dengan versi `pyodbc` yang lebih aman di bawah ini:

    ```python
    import pyodbc
    import os
    import logging
    from dotenv import load_dotenv

    def get_sql_connection():
        """Membuat koneksi SQL Server menggunakan pyodbc"""
        try:
            # Mengambil password dari environment variable
            load_dotenv()
            db_password = os.getenv('MSSQL_SA_PASSWORD')
            if not db_password:
                raise ValueError("Password SQL Server tidak ditemukan di environment variable MSSQL_SA_PASSWORD")

            conn_str = (
                r'DRIVER={ODBC Driver 18 for SQL Server};'
                r'SERVER=sqlserver,1433;'  # 'sqlserver' adalah nama layanan di docker-compose
                r'DATABASE=PTXYZ_DataWarehouse;'
                r'UID=sa;'
                r'PWD=' + db_password + ';'
                r'TrustServerCertificate=yes;'
            )

            logging.info("Mencoba terhubung ke SQL Server menggunakan pyodbc...")
            conn = pyodbc.connect(conn_str)
            logging.info("Koneksi ke SQL Server berhasil!")
            return conn
        except Exception as e:
            logging.error(f"Error terhubung ke SQL Server dengan pyodbc: {str(e)}")
            raise
    ```

### Langkah 2.6: Verifikasi File `.env`

Sebelum menjalankan, pastikan file `.env` sudah benar-benar ada dan bukan `env.example`.

```bash
# Perintah ini akan menampilkan detail file .env jika ada
ls -la .env
```

---

## 3. Menjalankan Aplikasi

Setelah semua konfigurasi di atas selesai, jalankan aplikasi dari terminal Anda.

1.  **(Opsional tapi Direkomendasikan) Bersihkan Docker:**
    Untuk memastikan tidak ada konflik, hapus semua kontainer dan _volume_ yang ada:

    ```bash
    docker system prune -af --volumes
    ```

2.  **Bangun dan Jalankan Layanan:**
    Perintah ini akan membangun _image_ dan menjalankan semua layanan di latar belakang (`-d`):
    ```bash
    docker compose up -d
    ```

---

## 4. Verifikasi Status

1.  **Tunggu Inisialisasi:**
    Beri waktu **5-10 menit** agar semua layanan (terutama `sqlserver` dan `superset`) selesai melakukan inisialisasi internal.

2.  **Periksa Status Kontainer:**
    Jalankan perintah berikut untuk melihat status semua layanan:

    ```bash
    docker compose ps
    ```

3.  **Hasil yang Diharapkan:**
    Anda seharusnya melihat semua layanan dalam status `running` atau `healthy`. Jika statusnya `starting` atau `unhealthy`, tunggu beberapa saat lagi atau periksa log dengan `docker compose logs [NAMA_LAYANAN]`.
