# Runbook: Sistem Data Warehouse PT. XYZ

**Tujuan:** Dokumen ini merinci prosedur operasional standar (SOP) untuk menjalankan, memonitor, dan memelihara infrastruktur Data Warehouse (DWH) PT. XYZ yang berjalan di lingkungan Docker Compose.

## 1. Sistem & Arsitektur

### 1.1. Deskripsi
Sistem ini dirancang untuk mengekstrak data dari sistem operasional (OLTP) ke Data Warehouse (DWH) untuk analisis Business Intelligence. Seluruh proses orkestrasi ETL (Extract, Transform, Load) dikelola oleh **Apache Airflow**.

### 1.2. Arsitektur Komponen (Services)
Sistem ini terdiri dari beberapa layanan Docker yang saling bergantung:

* **`sqlserver` (ptxyz_sqlserver):** Database OLTP utama (SQL Server 2022) yang menjadi sumber data.
* **`postgres` (ptxyz_postgres):** Database metadata untuk Apache Airflow.
* **`redis` (ptxyz_redis):** *Message broker* untuk antrian tugas (Celery) Airflow.
* **`airflow-webserver`:** Antarmuka (UI) Apache Airflow.
* **`airflow-scheduler`:** Penjadwal utama yang memicu eksekusi DAG (pipeline).
* **`airflow-worker`:** *Worker* yang menjalankan tugas-tugas ETL.
* **Layanan Analitik:** `superset`, `jupyter`, `grafana`, `metabase`.
* **`db_init`:** Layanan *one-time* untuk inisialisasi skema/data awal (jika ada).

---

## 2. Prosedur Operasional Harian

Prosedur ini mencakup cara mengelola siklus hidup tumpukan aplikasi (stack).

### 2.1. Prasyarat
Sebelum menjalankan sistem, pastikan hal berikut:
1.  **Docker** dan **Docker Compose** terinstal di server host.
2.  File `.env` sudah ada dan terisi dengan benar (terutama `MSSQL_SA_PASSWORD`, `POSTGRES_PASSWORD`, dll.).

### 2.2. Menjalankan Sistem
Untuk memulai semua layanan (termasuk Airflow, database, dll.) dalam mode *detached* (background):

```bash
docker compose up -d

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

# Desain OLTP Proyek Perancangan Data Warehouse Industri Pertambangan PT. ΧΥΖ

**Operational Environment (OLTP)**
1.  Business Strategy
2.  Business Process
    ↓
3.  OLTP System
    (Departemen, Karyawan, Produksi, Pengiriman, Penjualan, dil)
4.  ETL Process
    Extract-Transform-Load

**Informational Environment (OLAP)**
5.  Data Warehouse / Data Mart
6.  Data Analytics & Decision Making

Sistem OLTP (Online Transaction Processing) pada Industri Pertambangan PT. XYZ berfungsi sebagai sistem operasional utama yang menangani transaksi harian perusahaan pertambangan.

Sistem ini menjadi sumber data awal yang nantinya akan diekstraksi dan diolah ke dalam sistem Data Warehouse (DWH) untuk keperluan analisis dan pelaporan manajemen.

Desain OLTP ini disusun agar dapat menangani proses pencatatan aktivitas produksi, pengiriman hasil tambang, transaksi penjualan, serta pengelolaan data karyawan dan pelanggan secara efisien dan konsisten.

Sistem OLTP dirancang dengan model relasional (Relational Database Model) menggunakan normalisasi hingga 3NF (Third Normal Form) untuk menghindari duplikasi data dan menjaga integritas relasi antar entitas.

```dbml
// =============================================
// OLTP PT. XYZ - Sistem Operasional Industri Pertambangan
// Format: dbdiagram.io (.dbml)
// =============================================

// Master data
Table departemen {
  id_departemen int [pk, increment]
  nama_departemen varchar(100)
}

Table karyawan {
  id_karyawan int [pk, increment]
  nama_karyawan varchar(100)
  jabatan varchar(50)
  id_departemen int [ref: > departemen.id_departemen]
}

Table lokasi_tambang {
  id_tambang int [pk, increment]
  nama_tambang varchar(100)
  lokasi varchar(100)
  jenis_tambang varchar(50)
}

Table alat_berat {
  id_alat int [pk, increment]
  nama_alat varchar(100)
  jenis_alat varchar(50)
  kapasitas decimal(10,2)
  status varchar(20)
}

// Transaksi utama
Table produksi {
  id_produksi int [pk, increment]
  id_tambang int [ref: > lokasi_tambang.id_tambang]
  id_karyawan int [ref: > karyawan.id_karyawan]
  id_alat int [ref: > alat_berat.id_alat]
  tanggal_produksi date
  jumlah_produksi decimal(12,2)
}

Table pelanggan {
  id_pelanggan int [pk, increment]
  nama_pelanggan varchar(100)
  alamat varchar(200)
  kontak varchar(50)
}

Table pengiriman {
  id_pengiriman int [pk, increment]
  id_produksi int [ref: > produksi.id_produksi]
  id_pelanggan int [ref: > pelanggan.id_pelanggan]
  tanggal_pengiriman date
  jumlah_dikirim decimal(12,2)
  tujuan varchar(200)
  status varchar(50)
}

Table penjualan {
  id_penjualan int [pk, increment]
  id_pengiriman int [ref: > pengiriman.id_pengiriman]
  tanggal_penjualan date
  jumlah_terjual decimal(12,2)
  harga_per_ton decimal(12,2)
  total_pendapatan decimal(15,2)
}

# Entitas dan Relasi Utama

## 1. Entitas Utama

| Entitas         | Deskripsi                                                                 |
|-----------------|---------------------------------------------------------------------------|
| departemen      | Menyimpan data unit kerja di dalam perusahaan                             |
| karyawan        | Menyimpan data seluruh karyawan yang terlibat dalam proses produksi, administrasi, dan pengiriman |
| lokasi_tambang  | Menyimpan informasi lokasi area pertambangan tempat proses produksi dilakukan |
| produksi        | Menyimpan data aktivitas produksi hasil tambang                           |
| pelanggan       | Menyimpan data pelanggan atau pihak pembeli hasil tambang                |
| penjualan       | Menyimpan data transaksi penjualan hasil tambang ke pelanggan            |
| pengiriman      | Menyimpan data pengiriman barang hasil tambang ke pelanggan, termasuk tanggal dan lokasi tujuan |
| alat_berat      | Menyimpan data alat berat yang digunakan dalam proses produksi           |

---

## 2. Tabel Deskripsi

### 2.1 Departemen

| Key | Kolom           | Tipe Data     | Deskripsi                    |
|-----|----------------|--------------|-------------------------------|
| PK  | id_departemen  | INT          | Identitas unik departemen     |
|     | nama_departemen| VARCHAR(100) | Nama unit kerja di perusahaan |

### 2.2 Karyawan

| Key | Kolom        | Tipe Data     | Deskripsi                        |
|-----|--------------|--------------|----------------------------------|
| PK  | id_karyawan  | INT          | Identitas unik karyawan          |
|     | nama_karyawan| VARCHAR(100) | Nama lengkap karyawan            |
|     | jabatan      | VARCHAR(50)  | Jabatan atau posisi kerja        |
| FK  | id_departemen| INT          | Relasi ke tabel departemen       |

### 2.3 Lokasi Tambang

| Key | Kolom        | Tipe Data     | Deskripsi                        |
|-----|--------------|--------------|----------------------------------|
| PK  | id_tambang   | INT          | Identitas unik tambang           |
|     | nama_tambang | VARCHAR(100) | Nama site atau wilayah tambang   |
|     | lokasi       | VARCHAR(100) | Alamat atau koordinat lokasi     |
|     | jenis_tambang| VARCHAR(50)  | Jenis hasil tambang              |

### 2.4 Alat Berat

| Key | Kolom      | Tipe Data     | Deskripsi                           |
|-----|------------|--------------|-------------------------------------|
| PK  | id_alat    | INT          | Identitas unik alat berat            |
|     | nama_alat  | VARCHAR(100) | Nama unit alat berat                 |
|     | jenis_alat | VARCHAR(50)  | Tipe alat (excavator, dump truck, dsb.) |
|     | kapasitas  | DECIMAL(10,2)| Kapasitas angkut atau muat           |
|     | status     | VARCHAR(20)  | Status operasional (aktif, perawatan, rusak) |

### 2.5 Produksi

| Key | Kolom           | Tipe Data     | Deskripsi                        |
|-----|-----------------|--------------|----------------------------------|
| PK  | id_produksi     | INT          | Identitas unik aktivitas produksi|
| FK  | id_tambang      | INT          | Relasi ke lokasi_tambang         |
| FK  | id_karyawan     | INT          | Relasi ke karyawan               |
| FK  | id_alat         | INT          | Relasi ke alat_berat             |
|     | tanggal_produksi| DATE         | Tanggal kegiatan produksi        |
|     | jumlah_produksi | DECIMAL(12,2)| Volume hasil produksi (ton)      |

### 2.6 Pengiriman

| Key | Kolom          | Tipe Data     | Deskripsi                          |
|-----|----------------|--------------|------------------------------------|
| PK  | id_pengiriman  | INT          | Identitas unik pengiriman          |
| FK  | id_produksi    | INT          | Relasi ke produksi                 |
| FK  | id_pelanggan   | INT          | Relasi ke pelanggan                |
|     | tanggal_pengiriman | DATE     | Tanggal pengiriman dilakukan       |
|     | jumlah_dikirim | DECIMAL(12,2)| Volume hasil tambang yang dikirim |
|     | tujuan         | VARCHAR(200) | Lokasi atau pelabuhan tujuan       |
|     | status         | VARCHAR(50)  | Status pengiriman (dikirim, diterima, tertunda) |

### 2.7 Pelanggan

| Key | Kolom          | Tipe Data     | Deskripsi                        |
|-----|----------------|--------------|----------------------------------|
| PK  | id_pelanggan   | INT          | Identitas unik pelanggan         |
|     | nama_pelanggan | VARCHAR(100) | Nama perusahaan atau pembeli     |
|     | alamat         | VARCHAR(200) | Alamat pelanggan                 |
|     | kontak         | VARCHAR(50)  | Nomor atau email kontak          |

### 2.8 Penjualan

| Key | Kolom            | Tipe Data     | Deskripsi                           |
|-----|-----------------|--------------|-------------------------------------|
| PK  | id_penjualan    | INT          | Identitas unik penjualan            |
| FK  | id_pengiriman   | INT          | Relasi ke pengiriman                |
|     | tanggal_penjualan| DATE        | Tanggal transaksi penjualan         |
|     | jumlah_terjual  | DECIMAL(12,2)| Volume hasil tambang yang dijual    |
|     | harga_per_ton   | DECIMAL(12,2)| Harga per ton produk tambang        |
|     | total_pendapatan| DECIMAL(15,2)| Nilai total transaksi               |

---

## 3. Relasi Antar Tabel

| Relasi                     | Jenis Hubungan | Keterangan                                      |
|-----------------------------|----------------|------------------------------------------------|
| karyawan → departemen       | Many to One    | Setiap karyawan berasal dari satu departemen  |
| produksi → lokasi_tambang   | Many to One    | Setiap produksi terjadi di satu lokasi tambang|
| produksi → karyawan         | Many to One    | Setiap kegiatan produksi dikerjakan oleh satu karyawan |
| produksi → alat_berat       | Many to One    | Setiap produksi menggunakan satu alat berat   |
| pengiriman → produksi       | Many to One    | Pengiriman didasarkan pada satu hasil produksi|
| pengiriman → pelanggan      | Many to One    | Pengiriman ditujukan pada satu pelanggan      |
| penjualan → pengiriman      | Many to One    | Setiap penjualan berasal dari satu pengiriman |

# Scheduling

Proses Extract, Transform, Load (ETL) pada proyek ini dijalankan menggunakan **Apache Airflow** sebagai engine orkestrasi pipeline. Airflow berfungsi untuk mengatur alur kerja (**workflow orchestration**) dan menjadwalkan eksekusi setiap tahapan ETL secara otomatis. Seluruh sistem dijalankan melalui layanan `airflow-scheduler` yang telah terintegrasi di dalam berkas `docker-compose.yml`, sehingga seluruh komponen seperti Airflow, database, dan message broker dapat berjalan bersamaan hanya dengan satu perintah.

Pipeline ETL dijadwalkan untuk berjalan **satu kali setiap hari pada pukul 02.00 WIB**. Penjadwalan harian ini dipilih karena frekuensi pembaruan data operasional tidak terlalu tinggi, sehingga pembaruan harian dianggap optimal untuk kebutuhan analisis data warehouse.

### Alasan penentuan jadwal pukul 02.00 WIB

1. **Ketersediaan Data Lengkap**  
   Seluruh data operasional harian telah terkumpul dan divalidasi pada malam hari, sehingga proses ETL dapat memproses data yang sudah lengkap dan konsisten.

2. **Minim Gangguan Sistem**  
   Waktu dini hari merupakan periode off-peak dimana aktivitas pengguna dan transaksi sistem relatif rendah. Hal ini meminimalkan risiko benturan dengan proses operasional utama.

3. **Efisiensi Analitik**  
   Pembaruan data setiap hari sudah mencukupi untuk mendukung kebutuhan analisis dan pelaporan yang dilakukan secara harian tanpa membebani sumber daya sistem.

### Recovery Strategy

Untuk menjaga keandalan proses ETL, diterapkan strategi recovery apabila terjadi kegagalan selama eksekusi pipeline:

- Airflow secara otomatis melakukan **retry** pada task yang gagal sebanyak tiga kali dengan jeda waktu sekitar lima menit di antara setiap percobaan.
- Jika setelah batas retry proses tetap gagal, sistem akan menandai task sebagai **failed** dan mengirimkan notifikasi otomatis kepada tim pengembang.

---

# Progres Pengerjaan

Progres pengerjaan proyek perancangan **Data Warehouse PT. XYZ** dimulai dari fondasi OLTP yang kuat, dilanjutkan dengan otomatisasi pipeline ETL menggunakan Airflow, serta diakhiri dengan integrasi insight bisnis.  

### Alur progres pengerjaan:

1. **Analisis Kebutuhan dan Identifikasi Sumber Data**  
   - Memahami sistem operasional harian PT. XYZ.  
   - Mengidentifikasi sumber data transaksi harian dari sistem OLTP.

2. **Perancangan Skema OLTP**  
   - Database relasional dibuat hingga normalisasi bentuk ke-3 (3NF).  
   - Entitas utama: Departemen, Karyawan, Lokasi Tambang, Alat Berat, Produksi, Pengiriman, Pelanggan, Penjualan.  
   - Relasi disusun jelas, misal Many-to-One antara Produksi dan Lokasi Tambang, Penjualan dan Pengiriman.

3. **Penyusunan Model Data Warehouse**  
   - Struktur OLTP digunakan sebagai dasar membangun Data Warehouse.  
   - Menyiapkan dimensional model (fact dan dimension tables) untuk analisis multidimensi.

4. **Perancangan dan Implementasi Proses ETL**  
   - Implementasi pipeline ETL menggunakan **Apache Airflow**.  
   - Dijankan di lingkungan **Docker Compose**.  
   - Kredensial dan variabel dikelola melalui file `.env`.

5. **Scheduling dan Automasi Pipeline**  
   - ETL dijalankan setiap hari pukul 02:00 WIB.  
   - Alasan: ketersediaan data lengkap, minim gangguan sistem, efisiensi analitik.

6. **Integrasi dan Validasi Data**  
   - Pengujian integritas dan konsistensi data antara OLTP dan Data Warehouse.

7. **Analisis dan Visualisasi Insight BI**  
   - Analisis tren produksi, evaluasi produktivitas alat berat dan karyawan.  
   - Pemantauan total pendapatan, harga per ton, performa pelanggan, kecepatan rantai pasok.

8. **Pelaporan dan Evaluasi**  
   - Laporan manajerial dan dashboard analitik untuk mendukung **data-driven decision making**.

---

# Insight BI

Berdasarkan desain sistem OLTP, PT. XYZ memiliki fondasi data yang kaya untuk diolah menjadi insight strategis.  

- **Produksi & Operasional**:  
  Analisis tren volume (`jumlah_produksi`) per tanggal (`tanggal_produksi`) dan lokasi (`lokasi_tambang`). Evaluasi produktivitas karyawan dan alat berat (`kapasitas`, `status`).  

- **Penjualan & Pelanggan**:  
  Monitoring `total_pendapatan`, `harga_per_ton`, segmentasi pelanggan, dan peringkat pelanggan berdasarkan volume (`jumlah_terjual`) atau nilai transaksi.  

- **Logistik & Rantai Pasok**:  
  Memantau status pengiriman, durasi antara `tanggal_produksi` dan `tanggal_pengiriman`, serta perbandingan antara `jumlah_produksi` dan `jumlah_terjual`.

---

# Diskusi Perbandingan Arsitektur On-Premise vs Cloud-Based

| Aspek | On-Premise | Cloud-Based |
|-------|------------|-------------|
| Infrastruktur & Pengelolaan | Seluruh sumber daya dikelola internal; kontrol penuh | Mengandalkan layanan pihak ketiga (AWS/GCP/Azure); konfigurasi otomatis |
| Skalabilitas & Kinerja | Terbatas oleh kapasitas perangkat keras; perlu upgrade manual | Mendukung auto-scaling; efisien untuk big data |
| Biaya & Efisiensi | Investasi awal besar; biaya operasional bulanan stabil | Model pay-as-you-go; biaya meningkat jika data/frekuensi tinggi |
| Keamanan & Kepatuhan | Data lokal, kontrol penuh; cocok untuk data sensitif | Dikelola penyedia layanan; standar enkripsi tinggi |
| Implementasi & Pemeliharaan | Manual oleh tim internal; fleksibel tapi butuh keahlian | Relatif cepat; patch dan pembaruan dikelola penyedia |

**Kesimpulan:**  
- On-Premise: kontrol penuh, keamanan tinggi, stabilitas jangka panjang, tapi butuh sumber daya lebih.  
- Cloud-Based: skalabilitas tinggi, efisiensi biaya awal, mudah dikelola, tapi tergantung koneksi internet.
