# Arsitektur Dasbor Desa

Dokumen ini menjelaskan arsitektur teknis sistem Dasbor Desa dengan fokus pada integrasi dengan ekosistem aplikasi desa yang sudah ada dan implementasi praktis di lapangan.

---

## 1. Konteks Ekosistem

### Aplikasi yang Sudah Ada di Desa

| Aplikasi | Fungsi | Status | Interaksi dengan Dasbor Desa |
|----------|---------|---------|-------------------------------|
| SISKEUDES | Keuangan desa (APBDes, SPJ) | **Wajib** | Dasbor mengimpor laporan realisasi Excel |
| Prodeskel/e-Desa | Administrasi kependudukan | **Wajib** | Dasbor mengimpor agregat statistik penduduk |
| SIMDABUMAS | Surat keterangan | Opsional | Dasbor mengimpor log layanan CSV |
| SIMBADA | Inventaris aset | Opsional | Dasbor mengimpor daftar aset Excel |
| SIPEDE | Pelaporan ke Kemendagri | **Wajib** | Dasbor menyediakan ekspor format SIPEDE |
| Spreadsheet Manual | Tracking kegiatan, dll | Umum | Dasbor menggantikan dengan database terstruktur |

### Posisi Dasbor Desa

```mermaid
graph TD
    "SISKEUDES (Keuangan)" --> "Laporan Excel Realisasi"
    "Laporan Excel Realisasi" --> "DASBOR DESA (Agregasi & Visualisasi)"
    "Prodeskel (Kependudukan)" --> "DASBOR DESA (Agregasi & Visualisasi)"
    "SIMDABUMAS (Layanan)" --> "DASBOR DESA (Agregasi & Visualisasi)"
    "DASBOR DESA (Agregasi & Visualisasi)" --> "Dashboard Publik"
    "DASBOR DESA (Agregasi & Visualisasi)" --> "Laporan Triwulan"
```

**Prinsip**: Dasbor Desa adalah layer visualisasi dan analitik, bukan sistem sumber data primer.

---

## 2. Arsitektur Aplikasi

### Stack Teknologi

```mermaid
flowchart TD
    subgraph Client ["Browser"]
        A["Chrome/Firefox/Safari - Desktop & Mobile"]
    end

    A -- "HTTPS" --> FL

    subgraph FL ["Frontend Layer"]
        direction TB
        B1["Bootstrap 5 (CSS Framework)"]
        B2["HTMX (Dynamic Interactivity)"]
        B3["Chart.js (Grafik)"]
        B4["Leaflet.js (Peta)"]
        B5["Django Template Language"]
    end

    FL -- "Template Rendering" --> BL

    subgraph BL ["Backend Layer"]
        direction TB
        C1["Django 5.x (Python Web Framework)"]
        C2["Django ORM (Database Abstraction)"]
        C3["Django REST Framework (API - Optional)"]
        C4["Celery (Background Tasks - Optional)"]
        C5["django-simple-history (Audit Trail)"]
    end

    BL -- "SQL Queries" --> DL

    subgraph DL ["Database Layer"]
        direction TB
        D1["PostgreSQL 14+ (Primary)"]
        D2["SQLite (Development Only)"]
    end
```

### Deployment Options

**Option A: Shared Hosting (Murah, Mudah)**
- cPanel dengan Python support
- PostgreSQL database
- Cost: ~Rp 200.000/tahun
- Cocok untuk: Desa dengan koneksi internet stabil

**Option B: VPS (Fleksibel)**
- Ubuntu 22.04 LTS
- Nginx + Gunicorn
- PostgreSQL
- Cost: ~Rp 50.000/bulan
- Cocok untuk: Desa yang ingin kontrol penuh

**Option C: Server Lokal (Offline-Capable)**
- Raspberry Pi 4 (4GB RAM)
- Ubuntu Server + Docker
- PostgreSQL
- Cost: ~Rp 1.500.000 (one-time hardware)
- Cocok untuk: Desa tanpa internet stabil

---

## 3. Model Data

### ERD (Entity Relationship Diagram)

```mermaid
erDiagram
    StatistikPenduduk {
        date periode PK
        int jumlah_kk
        int jumlah_jiwa
        int laki_laki
        int perempuan
        int usia_0_5
        int usia_6_17
        int usia_18_59
        int usia_60_plus
        int penerima_bpnt
        int penerima_pkh
        int penerima_bst
        int partisipan_musrenbang
    }

    KodeRekening {
        string kode PK
        int level
        string nama
        boolean is_active
    }

    APBDes {
        int tahun
        string uraian
        decimal pagu
        decimal realisasi
        string sumber_dana
        boolean is_perubahan
    }

    Kegiatan {
        string nama
        string bidang
        text deskripsi
        string lokasi
        decimal latitude
        decimal longitude
        decimal nilai_anggaran
        date tanggal_mulai
        date target_selesai
        date tanggal_selesai_aktual
        int progres_persen
        string penanggung_jawab
        string status
    }

    FotoProgres {
        string file_path
        date tanggal_upload
        int progres_saat_foto
        text keterangan
    }

    TransaksiRealisasi {
        date tanggal
        text uraian
        decimal jumlah
        string bukti_dokumen
        string status_verifikasi
        text catatan_verifikasi
    }

    LayananPublik {
        string jenis_layanan
        string pemohon_nama
        string pemohon_nik
        datetime tanggal_masuk
        datetime tanggal_selesai
        int waktu_proses_hari
        string status
        text catatan
        int rating_kepuasan
    }

    AsetDesa {
        string kode_aset PK
        string nama_aset
        string kategori
        int tahun_perolehan
        decimal nilai_perolehan
        string kondisi
        string lokasi
        decimal latitude
        decimal longitude
        string foto
    }

    User {
        string username PK
        string password
        string email
        string first_name
        string last_name
        boolean is_active
        boolean is_staff
    }

    AuditLog {
        datetime history_date
        string history_type
        int record_id
    }

    KodeRekening ||--o{ KodeRekening : "parent"
    KodeRekening ||--o{ APBDes : "dianggarkan_dalam"
    APBDes ||--o{ TransaksiRealisasi : "memiliki"
    APBDes ||--o| Kegiatan : "mendanai"
    Kegiatan ||--o{ FotoProgres : "memiliki"
    Kegiatan ||--o{ TransaksiRealisasi : "mencatat"
    User ||--o{ FotoProgres : "unggah_oleh"
    User ||--o{ TransaksiRealisasi : "input_oleh"
    User ||--o{ TransaksiRealisasi : "verifikasi_oleh"
    User ||--o{ LayananPublik : "petugas_handle"
    User ||--o{ AuditLog : "aksi_oleh"
```

### Detail Model

#### 1. StatistikPenduduk
**Sumber Data**: Impor bulanan dari Prodeskel/e-Desa

| Field | Type | Constraint | Deskripsi |
|-------|------|-----------|-----------|
| id | AutoField | PK | Primary key |
| periode | DateField | UNIQUE | Bulan statistik (tanggal 1) |
| jumlah_kk | IntegerField | NOT NULL | Jumlah Kepala Keluarga |
| jumlah_jiwa | IntegerField | NOT NULL | Total jiwa |
| laki_laki | IntegerField | NOT NULL | Jumlah penduduk laki-laki |
| perempuan | IntegerField | NOT NULL | Jumlah penduduk perempuan |
| usia_0_5 | IntegerField | DEFAULT 0 | Balita |
| usia_6_17 | IntegerField | DEFAULT 0 | Usia sekolah |
| usia_18_59 | IntegerField | DEFAULT 0 | Usia produktif |
| usia_60_plus | IntegerField | DEFAULT 0 | Lansia |
| penerima_bpnt | IntegerField | DEFAULT 0 | Penerima Bantuan Pangan |
| penerima_pkh | IntegerField | DEFAULT 0 | Penerima PKH |
| penerima_bst | IntegerField | DEFAULT 0 | Penerima BST |
| partisipan_musrenbang | IntegerField | DEFAULT 0 | Peserta Musrenbang |
| created_at | DateTimeField | AUTO | Waktu record dibuat |
| updated_at | DateTimeField | AUTO | Waktu terakhir update |

**Catatan**: Tidak ada data individu, hanya agregat.

#### 2. KodeRekening
**Sumber Data**: Fixture standar dari Permendagri 20/2018

| Field | Type | Constraint | Deskripsi |
|-------|------|-----------|-----------|
| id | AutoField | PK | Primary key |
| kode | CharField(20) | UNIQUE | Format: 1.2.3 (Bidang.Kegiatan.Jenis) |
| level | IntegerField | NOT NULL | 1=Bidang, 2=Kegiatan, 3=Jenis Belanja |
| parent_id | ForeignKey | NULL | Self-referencing untuk hierarki |
| nama | CharField(255) | NOT NULL | Nama pos anggaran |
| is_active | BooleanField | DEFAULT True | Aktif/nonaktif |

**Fixture Awal**:
```
Bidang (Level 1):
01 - Penyelenggaraan Pemerintahan Desa
02 - Pelaksanaan Pembangunan Desa
03 - Pembinaan Kemasyarakatan
04 - Pemberdayaan Masyarakat
05 - Penanggulangan Bencana, Darurat, dan Mendesak

Jenis Belanja (Level 3):
51 - Belanja Pegawai
52 - Belanja Barang dan Jasa
53 - Belanja Modal
```

#### 3. APBDes
**Sumber Data**: Impor dari SISKEUDES

| Field | Type | Constraint | Deskripsi |
|-------|------|-----------|-----------|
| id | AutoField | PK | Primary key |
| tahun | IntegerField | NOT NULL | Tahun anggaran |
| kode_rekening_id | ForeignKey | NOT NULL | Relasi ke KodeRekening |
| uraian | TextField | NOT NULL | Deskripsi detail |
| pagu | DecimalField(15,2) | NOT NULL | Anggaran yang dialokasikan |
| realisasi | DecimalField(15,2) | DEFAULT 0 | Total realisasi sampai saat ini |
| sumber_dana | CharField(50) | NULL | ADD, DD, PADes, dll |
| is_perubahan | BooleanField | DEFAULT False | APBDes atau Perubahan APBDes |
| tanggal_perubahan | DateField | NULL | Jika ada perubahan |
| created_at | DateTimeField | AUTO | Waktu record dibuat |
| updated_at | DateTimeField | AUTO | Waktu terakhir update |

**Validasi**:
- `realisasi <= pagu`
- `tahun >= 2020` (asumsi sistem mulai dipakai)

#### 4. Kegiatan
**Sumber Data**: Input manual atau impor

| Field | Type | Constraint | Deskripsi |
|-------|------|-----------|-----------|
| id | AutoField | PK | Primary key |
| apbdes_id | ForeignKey | NULL | Relasi ke APBDes (jika ada) |
| nama | CharField(255) | NOT NULL | Nama kegiatan |
| bidang | CharField(100) | NOT NULL | Bidang kegiatan |
| deskripsi | TextField | NULL | Deskripsi lengkap |
| lokasi | CharField(255) | NULL | Lokasi fisik kegiatan |
| latitude | DecimalField(9,6) | NULL | Koordinat untuk peta |
| longitude | DecimalField(9,6) | NULL | Koordinat untuk peta |
| nilai_anggaran | DecimalField(15,2) | NOT NULL | Total anggaran kegiatan |
| tanggal_mulai | DateField | NOT NULL | Mulai pelaksanaan |
| target_selesai | DateField | NOT NULL | Target penyelesaian |
| tanggal_selesai_aktual | DateField | NULL | Actual completion |
| progres_persen | IntegerField | DEFAULT 0 | 0-100 |
| penanggung_jawab | CharField(100) | NOT NULL | Nama penanggung jawab |
| status | CharField(20) | NOT NULL | Choices: perencanaan, berjalan, selesai, terhenti |
| created_at | DateTimeField | AUTO | Waktu record dibuat |
| updated_at | DateTimeField | AUTO | Waktu terakhir update |

#### 5. FotoProgres
**Sumber Data**: Upload manual

| Field | Type | Constraint | Deskripsi |
|-------|------|-----------|-----------|
| id | AutoField | PK | Primary key |
| kegiatan_id | ForeignKey | NOT NULL | Relasi ke Kegiatan |
| file_path | CharField(255) | NOT NULL | Path file di media/ |
| tanggal_upload | DateField | NOT NULL | Tanggal foto diambil |
| progres_saat_foto | IntegerField | NOT NULL | Progres saat foto (%) |
| keterangan | TextField | NULL | Deskripsi foto |
| uploaded_by | ForeignKey | NOT NULL | User yang upload |

#### 6. TransaksiRealisasi
**Sumber Data**: Impor dari SISKEUDES atau input manual

| Field | Type | Constraint | Deskripsi |
|-------|------|-----------|-----------|
| id | AutoField | PK | Primary key |
| apbdes_id | ForeignKey | NOT NULL | Relasi ke APBDes |
| kegiatan_id | ForeignKey | NULL | Relasi ke Kegiatan (opsional) |
| tanggal | DateField | NOT NULL | Tanggal transaksi |
| uraian | TextField | NOT NULL | Deskripsi pengeluaran |
| jumlah | DecimalField(15,2) | NOT NULL | Nominal transaksi |
| bukti_dokumen | FileField | NULL | Upload scan kwitansi/SPJ |
| status_verifikasi | CharField(20) | DEFAULT pending | Choices: pending, verified, rejected |
| catatan_verifikasi | TextField | NULL | Catatan dari verifikator |
| created_by | ForeignKey | NOT NULL | User yang input |
| verified_by | ForeignKey | NULL | User yang verifikasi |
| created_at | DateTimeField | AUTO | Waktu record dibuat |

**Trigger**: Setelah transaksi verified, update `realisasi` di tabel APBDes.

#### 7. LayananPublik
**Sumber Data**: Impor dari SIMDABUMAS atau input manual

| Field | Type | Constraint | Deskripsi |
|-------|------|-----------|-----------|
| id | AutoField | PK | Primary key |
| jenis_layanan | CharField(100) | NOT NULL | KTP, KK, Surat Keterangan, dll |
| pemohon_nama | CharField(100) | NOT NULL | Nama pemohon |
| pemohon_nik | CharField(16) | NULL | NIK (opsional untuk agregat) |
| tanggal_masuk | DateTimeField | NOT NULL | Waktu permohonan masuk |
| tanggal_selesai | DateTimeField | NULL | Waktu layanan selesai |
| waktu_proses_hari | IntegerField | NULL | Kalkulasi otomatis (hari kerja) |
| status | CharField(20) | NOT NULL | Choices: pending, proses, selesai, batal |
| petugas | ForeignKey | NOT NULL | User operator yang handle |
| catatan | TextField | NULL | Catatan internal |
| rating_kepuasan | IntegerField | NULL | 1-5 (opsional) |
| created_at | DateTimeField | AUTO | Waktu record dibuat |
| updated_at | DateTimeField | AUTO | Waktu terakhir update |

**Standar Waktu Proses** (untuk kalkulasi indeks):
- KTP/KK: 3 hari kerja
- Surat Keterangan: 1 hari kerja
- Legalisasi: 2 hari kerja

#### 8. AsetDesa
**Sumber Data**: Impor dari SIMBADA atau input manual

| Field | Type | Constraint | Deskripsi |
|-------|------|-----------|-----------|
| id | AutoField | PK | Primary key |
| kode_aset | CharField(50) | UNIQUE | Kode inventaris |
| nama_aset | CharField(255) | NOT NULL | Nama barang |
| kategori | CharField(50) | NOT NULL | Choices: Tanah, Bangunan, Kendaraan, Alat, dll |
| tahun_perolehan | IntegerField | NOT NULL | Tahun pembelian/hibah |
| nilai_perolehan | DecimalField(15,2) | NOT NULL | Harga beli |
| kondisi | CharField(20) | NOT NULL | Choices: baik, rusak_ringan, rusak_berat |
| lokasi | CharField(255) | NULL | Lokasi fisik aset |
| latitude | DecimalField(9,6) | NULL | Untuk aset properti |
| longitude | DecimalField(9,6) | NULL | Untuk aset properti |
| foto | FileField | NULL | Foto aset |
| created_at | DateTimeField | AUTO | Waktu record dibuat |
| updated_at | DateTimeField | AUTO | Waktu terakhir update |

#### 9. User (Django Built-in, Extended)
**Sumber Data**: Setup manual oleh admin

| Field | Type | Constraint | Deskripsi |
|-------|------|-----------|-----------|
| id | AutoField | PK | Django default |
| username | CharField | UNIQUE | Login username |
| password | CharField | NOT NULL | Hashed password |
| email | EmailField | NULL | Email (opsional) |
| first_name | CharField | NULL | Nama depan |
| last_name | CharField | NULL | Nama belakang |
| is_active | BooleanField | DEFAULT True | Akun aktif |
| is_staff | BooleanField | DEFAULT False | Akses admin panel |
| groups | ManyToMany | NULL | Role grouping |

**Groups (Roles)**:
- `Admin`: Full access
- `Operator Keuangan`: APBDes, Transaksi, Kegiatan
- `Operator Layanan`: LayananPublik
- `Operator Umum`: Aset, Statistik Penduduk
- `Viewer`: Read-only access

#### 10. AuditLog (via django-simple-history)
**Sumber Data**: Otomatis dari sistem

Setiap model kritikal (APBDes, Transaksi, Kegiatan) memiliki tracking history otomatis.

| Field | Type | Deskripsi |
|-------|------|-----------|
| history_id | AutoField | PK untuk record history |
| history_date | DateTimeField | Kapan perubahan terjadi |
| history_type | CharField | +: created, ~: updated, -: deleted |
| history_user | ForeignKey | User yang melakukan aksi |
| [all fields from original model] | Various | Snapshot nilai saat itu |

---

## 4. Sitemap dan Navigation

### A. Area Publik (No Login Required)

```mermaid
graph TD
    A["Landing Page (/)\n- Hero section\n- Link ke Dashboard Publik\n- Info kontak desa"]
    B["Dashboard Publik (/publik/)\n- Ringkasan APBDes tahun ini\n- 4 Indikator Visioner\n- Tabel kegiatan berjalan\n- Grafik realisasi per bulan\n- Statistik layanan"]
    C["Detail Kegiatan (/publik/kegiatan/:id)\n- Info lengkap kegiatan\n- Foto progres timeline\n- Peta lokasi"]

    A --> B
    B --> C
```

### B. Area Privat (Login Required)

```mermaid
graph TD
    Login["Login (/login/)\n- Username/password\n- Remember me"]
    DashP["Dashboard Privat (/dashboard/)"]

    Login --> DashP

    subgraph Sidebar ["Navigasi Sidebar"]
        DashP --> DM["Data Master"]
        DashP --> ID["Input Data"]
        DashP --> LP["Laporan"]
        DashP --> AN["Analitik"]
        DashP --> AD["Administrasi (Admin Only)"]

        DM --> DM1["Statistik Penduduk"]
        DM --> DM2["APBDes"]
        DM --> DM3["Kegiatan"]
        DM --> DM4["Layanan"]
        DM --> DM5["Aset"]

        ID --> ID1["Impor APBDes (Excel)"]
        ID --> ID2["Impor Layanan (CSV)"]
        ID --> ID3["Input Kegiatan (Form)"]
        ID --> ID4["Input Transaksi (Form)"]

        LP --> LP1["Ekspor LPJ (Word)"]
        LP --> LP2["Ekspor Realisasi (Excel)"]
        LP --> LP3["Format SIPEDE (Excel)"]

        AN --> AN1["Cashflow Analysis"]
        AN --> AN2["Deteksi Anomali"]
        AN --> AN3["Indikator Trend"]

        AD --> AD1["Manajemen User"]
        AD --> AD2["Audit Log"]
        AD --> AD3["Pengaturan Sistem"]
    end
```

### C. URL Structure

```
/                           → Landing page publik
/publik/                    → Dashboard publik
/publik/kegiatan/:id/       → Detail kegiatan publik
/login/                     → Halaman login
/logout/                    → Logout action

/dashboard/                 → Dashboard privat (overview)

/data/penduduk/             → Tabel statistik penduduk
/data/apbdes/               → Tabel APBDes
/data/kegiatan/             → Tabel kegiatan (list view)
/data/kegiatan/:id/         → Detail kegiatan (edit)
/data/layanan/              → Tabel layanan
/data/aset/                 → Tabel aset

/input/apbdes/              → Form impor APBDes Excel
/input/layanan/             → Form impor layanan CSV
/input/kegiatan/create/     → Form create kegiatan baru
/input/transaksi/create/    → Form create transaksi

/laporan/lpj/               → Generate LPJ Word
/laporan/realisasi/         → Generate laporan realisasi Excel
/laporan/sipede/            → Generate format SIPEDE

/analitik/cashflow/         → Grafik cashflow
/analitik/anomali/          → Deteksi anomali realisasi
/analitik/indikator/        → Trend indikator dari waktu ke waktu

/admin/users/               → CRUD user (admin only)
/admin/audit/               → View audit log (admin only)
/admin/settings/            → Konfigurasi sistem

/profil/                    → Edit profil user sendiri
/api/                       → API endpoints (optional, untuk integrasi)
```

---

## 5. User Interface Patterns

### Prinsip Desain UI
1. **Familiar**: Mirip Excel/Google Sheets untuk tabel data
2. **Minimal Klik**: Maksimal 3 klik untuk aksi umum
3. **Responsive**: Mobile-friendly untuk operator di lapangan
4. **Loading States**: Indikator jelas saat proses impor/ekspor
5. **Validasi Real-time**: Feedback langsung saat input form

### Komponen Reusable

#### 1. Data Table Component
```html
<!-- Pattern untuk semua tabel data -->
<div class="card">
  <div class="card-header">
    <h5>{{ table_title }}</h5>
    <div class="actions">
      <button class="btn btn-sm btn-primary" hx-get="/{{ model }}/create/">Tambah</button>
      <button class="btn btn-sm btn-success" hx-get="/{{ model }}/import/">Impor Excel</button>
      <button class="btn btn-sm btn-secondary" hx-get="/{{ model }}/export/">Ekspor Excel</button>
    </div>
  </div>
  <div class="card-body">
    <!-- Filter & Search -->
    <div class="filters" hx-get="/{{ model }}/" hx-trigger="change" hx-target="#table-content">
      <input type="text" name="search" placeholder="Cari...">
      <select name="filter_tahun">...</select>
      <select name="filter_status">...</select>
    </div>
    
    <!-- Table -->
    <div id="table-content">
      <table class="table table-hover">
        <thead>...</thead>
        <tbody>...</tbody>
      </table>
      <!-- Pagination -->
      <nav>...</nav>
    </div>
  </div>
</div>
```

#### 2. Form Component dengan HTMX
```html
<!-- Form dengan auto-save draft -->
<form hx-post="/{{ model }}/save/" hx-trigger="submit">
  <div class="row">
    <div class="col-md-6">
      <label>{{ field_label }}</label>
      <input type="text" name="{{ field_name }}" 
             hx-post="/validate/" hx-trigger="blur"
             hx-target="#{{ field_name }}-error">
      <div id="{{ field_name }}-error" class="invalid-feedback"></div>
    </div>
  </div>
  <button type="submit" class="btn btn-primary">Simpan</button>
</form>
```

#### 3. Chart Component
```html
<!-- Reusable chart container -->
<div class="card">
  <div class="card-header">
    <h5>{{ chart_title }}</h5>
    <select id="chart-period" hx-get="/analitik/{{ chart_type }}/" hx-target="#chart-canvas">
      <option value="bulan">Bulan Ini</option>
      <option value="triwulan">Triwulan Ini</option>
      <option value="tahun">Tahun Ini</option>
    </select>
  </div>
  <div class="card-body">
    <canvas id="chart-canvas"></canvas>
  </div>
</div>
```

### Interaction Patterns

#### Impor Excel (APBDes dari SISKEUDES)
```
1. User klik "Impor APBDes"
2. Modal terbuka dengan:
   - Link download template Excel
   - File upload field
   - Preview tabel hasil parsing (HTMX)
3. User upload file
4. Sistem parsing, tampilkan preview:
   - Jumlah baris valid
   - Jumlah error (dengan detail)
5. User klik "Konfirmasi Impor"
6. Background task proses impor
7. Notifikasi sukses + redirect ke tabel APBDes
```

#### Input Kegiatan dengan Progres
```
1. User klik "Tambah Kegiatan"
2. Form multi-step:
   Step 1: Info dasar (nama, bidang, lokasi)
   Step 2: Anggaran dan timeline
   Step 3: Upload foto awal (opsional)
3. Setiap step ada validasi sebelum lanjut
4. Review semua data di step terakhir
5. Submit → Create record + redirect ke detail
```

#### Update Progres Kegiatan (Quick Action)
```
1. Dari tabel kegiatan, klik ikon "Update Progres"
2. Modal kecil terbuka:
   - Slider untuk % progres (0-100)
   - Upload foto (opsional)
   - Textarea keterangan
3. Submit → Update via HTMX (no page reload)
4. Row di tabel update progres bar
```

---

## 6. Integrasi dengan Sistem Existing

### A. Impor APBDes dari SISKEUDES

**Format Excel SISKEUDES (Output Laporan Realisasi)**:
```
| Kode Rekening | Uraian            | Anggaran   | Realisasi  | Sisa       |
|---------------|-------------------|------------|------------|------------|
| 1.1.1.01      | Penghasilan Tetap | 100000000  | 80000000   | 20000000   |
| 1.1.2.01      | ATK Kantor        | 5000000    | 4500000    | 500000     |
```

**Script Impor** (`management/commands/import_siskeudes.py`):
```python
import openpyxl
from django.core.management.base import BaseCommand
from apps.keuangan.models import APBDes, KodeRekening

class Command(BaseCommand):
    def handle(self, *args, **options):
        file_path = options['file']
        tahun = options['tahun']
        
        wb = openpyxl.load_workbook(file_path)
        sheet = wb.active
        
        for row in sheet.iter_rows(min_row=2, values_only=True):
            kode, uraian, pagu, realisasi, _ = row
            
            # Cari atau buat kode rekening
            kode_rek, created = KodeRekening.objects.get_or_create(
                kode=kode,
                defaults={'nama': uraian}
            )
            
            # Update atau create APBDes
            APBDes.objects.update_or_create(
                tahun=tahun,
                kode_rekening=kode_rek,
                defaults={
                    'uraian': uraian,
                    'pagu': pagu,
                    'realisasi': realisasi
                }
            )
```

**Penggunaan**:
```bash
python manage.py import_siskeudes --file=/path/to/laporan.xlsx --tahun=2024
```

### B. Impor Layanan dari SIMDABUMAS

**Format CSV**:
```csv
jenis_layanan,pemohon_nama,tanggal_masuk,tanggal_selesai,status
KTP,Ahmad Susanto,2024-01-10 09:00,2024-01-12 14:00,selesai
Surat Keterangan Usaha,Siti Aminah,2024-01-11 10:30,,proses
```

**Script Impor**:
```python
import csv
from datetime import datetime
from apps.layanan.models import LayananPublik

def import_layanan_csv(file_path, petugas_default):
    with open(file_path, 'r') as f:
        reader = csv.DictReader(f)
        for row in reader:
            LayananPublik.objects.create(
                jenis_layanan=row['jenis_layanan'],
                pemohon_nama=row['pemohon_nama'],
                tanggal_masuk=datetime.fromisoformat(row['tanggal_masuk']),
                tanggal_selesai=datetime.fromisoformat(row['tanggal_selesai']) if row['tanggal_selesai'] else None,
                status=row['status'],
                petugas=petugas_default
            )
```

### C. Sinkronisasi Statistik Penduduk dari Prodeskel

**Template Excel untuk Impor**:
```
| Periode    | Jumlah KK | Jumlah Jiwa | Laki-laki | Perempuan | Usia 0-5 | ... |
|------------|-----------|-------------|-----------|-----------|----------|-----|
| 2024-01-01 | 450       | 1580        | 790       | 790       | 120      | ... |
```

**Atau via API (jika Prodeskel memiliki endpoint)**:
```python
import requests
from apps.penduduk.models import StatistikPenduduk

def sync_statistik_from_prodeskel():
    response = requests.get('http://prodeskel.local/api/statistik')
    data = response.json()
    
    StatistikPenduduk.objects.update_or_create(
        periode=data['periode'],
        defaults={
            'jumlah_kk': data['jumlah_kk'],
            'jumlah_jiwa': data['jumlah_jiwa'],
            # ... field lainnya
        }
    )
```

---

## 7. Kalkulasi Indikator Visioner

### Management Command Terjadwal

**File**: `management/commands/calculate_indicators.py`

```python
from django.core.management.base import BaseCommand
from apps.indikator.models import IndikatorBulanan
from apps.layanan.models import LayananPublik
from apps.keuangan.models import APBDes
from django.db.models import Avg, Count, Q
from datetime import datetime, timedelta

class Command(BaseCommand):
    def handle(self, *args, **options):
        periode = datetime.now().date().replace(day=1)
        
        # Hitung Indeks Pelayanan
        indeks_pelayanan = self.calculate_pelayanan(periode)
        
        # Hitung Indeks Transparansi
        indeks_transparansi = self.calculate_transparansi(periode)
        
        # Hitung Indeks Partisipasi
        indeks_partisipasi = self.calculate_partisipasi(periode)
        
        # Hitung Indeks Ketahanan
        indeks_ketahanan = self.calculate_ketahanan(periode)
        
        # Simpan ke database
        IndikatorBulanan.objects.update_or_create(
            periode=periode,
            defaults={
                'indeks_pelayanan': indeks_pelayanan,
                'indeks_transparansi': indeks_transparansi,
                'indeks_partisipasi': indeks_partisipasi,
                'indeks_ketahanan': indeks_ketahanan
            }
        )
    
    def calculate_pelayanan(self, periode):
        # Query layanan bulan ini
        layanan = LayananPublik.objects.filter(
            tanggal_masuk__month=periode.month,
            tanggal_masuk__year=periode.year
        )
        
        total = layanan.count()
        if total == 0:
            return 0
        
        # Komponen 1: Ketepatan waktu (40%)
        tepat_waktu = layanan.filter(
            status='selesai',
            waktu_proses_hari__lte=3  # Standar 3 hari
        ).count()
        skor_tepat_waktu = (tepat_waktu / total) * 40
        
        # Komponen 2: Kecepatan rata-rata (30%)
        rata_proses = layanan.filter(
            status='selesai'
        ).aggregate(Avg('waktu_proses_hari'))['waktu_proses_hari__avg'] or 0
        
        standar_waktu = 3
        if rata_proses <= standar_waktu:
            skor_kecepatan = 30
        else:
            skor_kecepatan = max(0, 30 * (1 - ((rata_proses - standar_waktu) / standar_waktu)))
        
        # Komponen 3: Penyelesaian pengaduan (30%)
        # Asumsi: ada model Pengaduan terpisah
        skor_pengaduan = 30  # Simplified, implementasi sebenarnya perlu query Pengaduan
        
        return round(skor_tepat_waktu + skor_kecepatan + skor_pengaduan, 2)
    
    def calculate_transparansi(self, periode):
        # Komponen 1: Frekuensi publikasi (40%)
        # Cek berapa kali data di-update bulan ini
        from apps.keuangan.models import APBDes
        updates_count = APBDes.history.filter(
            history_date__month=periode.month,
            history_date__year=periode.year
        ).count()
        
        # Target: minimal 4 update per bulan (1 per minggu)
        skor_publikasi = min(40, (updates_count / 4) * 40)
        
        # Komponen 2: Kelengkapan dokumen (30%)
        # Cek apakah semua kegiatan memiliki foto progres
        from apps.kegiatan.models import Kegiatan, FotoProgres
        kegiatan_aktif = Kegiatan.objects.filter(
            status__in=['berjalan', 'selesai']
        )
        kegiatan_dengan_foto = kegiatan_aktif.filter(
            fotoprogres__isnull=False
        ).distinct()
        
        if kegiatan_aktif.count() > 0:
            skor_kelengkapan = (kegiatan_dengan_foto.count() / kegiatan_aktif.count()) * 30
        else:
            skor_kelengkapan = 30
        
        # Komponen 3: Kebaruan data (30%)
        # Cek apakah ada update dalam 30 hari terakhir
        last_update = APBDes.objects.latest('updated_at').updated_at
        days_since_update = (datetime.now() - last_update).days
        
        if days_since_update <= 7:
            skor_kebaruan = 30
        elif days_since_update <= 30:
            skor_kebaruan = 20
        else:
            skor_kebaruan = 10
        
        return round(skor_publikasi + skor_kelengkapan + skor_kebaruan, 2)
    
    def calculate_partisipasi(self, periode):
        from apps.penduduk.models import StatistikPenduduk
        
        # Ambil data statistik bulan ini
        stat = StatistikPenduduk.objects.filter(periode=periode).first()
        if not stat:
            return 0
        
        # Komponen 1: Kehadiran Musrenbang (50%)
        # Target: minimal 10% dari jumlah KK
        target_partisipan = stat.jumlah_kk * 0.1
        if stat.partisipan_musrenbang >= target_partisipan:
            skor_musrenbang = 50
        else:
            skor_musrenbang = (stat.partisipan_musrenbang / target_partisipan) * 50
        
        # Komponen 2: Usulan yang direalisasi (30%)
        # Simplified: assume 70% realisasi
        skor_usulan = 21  # 0.7 * 30
        
        # Komponen 3: Kehadiran rapat RT/RW (20%)
        # Simplified: assume 80% kehadiran
        skor_rapat = 16  # 0.8 * 20
        
        return round(skor_musrenbang + skor_usulan + skor_rapat, 2)
    
    def calculate_ketahanan(self, periode):
        from apps.keuangan.models import APBDes
        from apps.kegiatan.models import Kegiatan
        from apps.aset.models import AsetDesa
        
        # Komponen 1: Serapan anggaran pembangunan (40%)
        apbdes_pembangunan = APBDes.objects.filter(
            tahun=periode.year,
            kode_rekening__kode__startswith='02'  # Bidang Pembangunan
        )
        total_pagu = sum([a.pagu for a in apbdes_pembangunan])
        total_realisasi = sum([a.realisasi for a in apbdes_pembangunan])
        
        if total_pagu > 0:
            persen_serapan = (total_realisasi / total_pagu)
            # Ideal: 80-90% serapan
            if 0.8 <= persen_serapan <= 0.9:
                skor_serapan = 40
            else:
                skor_serapan = max(0, 40 - abs(persen_serapan - 0.85) * 100)
        else:
            skor_serapan = 0
        
        # Komponen 2: Ketepatan waktu kegiatan (30%)
        kegiatan_selesai = Kegiatan.objects.filter(
            status='selesai',
            tanggal_selesai_aktual__lte=F('target_selesai')
        ).count()
        total_kegiatan_selesai = Kegiatan.objects.filter(status='selesai').count()
        
        if total_kegiatan_selesai > 0:
            skor_ketepatan = (kegiatan_selesai / total_kegiatan_selesai) * 30
        else:
            skor_ketepatan = 30
        
        # Komponen 3: Kondisi aset (30%)
        total_aset = AsetDesa.objects.count()
        aset_baik = AsetDesa.objects.filter(kondisi='baik').count()
        
        if total_aset > 0:
            skor_aset = (aset_baik / total_aset) * 30
        else:
            skor_aset = 30
        
        return round(skor_serapan + skor_ketepatan + skor_aset, 2)
```

**Jalankan via Cron Job** (setiap tanggal 1):
```bash
0 0 1 * * cd /path/to/project && python manage.py calculate_indicators
```

---

## 8. Ekspor Laporan

### Template Laporan LPJ (Word)

**File**: `templates/laporan/lpj_template.docx`

Struktur:
```
LAPORAN PERTANGGUNGJAWABAN
KEPALA DESA {{ nama_desa }}
TAHUN ANGGARAN {{ tahun }}

I. PENDAPATAN DESA
   A. Pendapatan Asli Desa: Rp {{ pad }}
   B. Transfer: Rp {{ transfer }}
   C. Lain-lain: Rp {{ lainnya }}
   TOTAL PENDAPATAN: Rp {{ total_pendapatan }}

II. BELANJA DESA
   A. Bidang Penyelenggaraan Pemerintahan: Rp {{ belanja_01 }}
   B. Bidang Pembangunan: Rp {{ belanja_02 }}
   ...
   TOTAL BELANJA: Rp {{ total_belanja }}

III. REALISASI KEGIATAN
   [Tabel kegiatan dengan progres]

IV. PENUTUP
   [Auto-generated text]
```

**Script Generate**:
```python
from docx import Document
from apps.keuangan.models import APBDes
from apps.kegiatan.models import Kegiatan

def generate_lpj(tahun):
    doc = Document('templates/laporan/lpj_template.docx')
    
    # Agregasi data APBDes
    apbdes_data = APBDes.objects.filter(tahun=tahun)
    belanja_per_bidang = {}
    for bidang in ['01', '02', '03', '04', '05']:
        total = sum([
            a.realisasi for a in apbdes_data 
            if a.kode_rekening.kode.startswith(bidang)
        ])
        belanja_per_bidang[f'belanja_{bidang}'] = total
    
    # Replace placeholders
    context = {
        'nama_desa': 'Desa Contoh',
        'tahun': tahun,
        'total_pendapatan': sum([...]),
        'total_belanja': sum(belanja_per_bidang.values()),
        **belanja_per_bidang
    }
    
    for paragraph in doc.paragraphs:
        for key, value in context.items():
            if f'{{{{ {key} }}}}' in paragraph.text:
                paragraph.text = paragraph.text.replace(f'{{{{ {key} }}}}', str(value))
    
    # Tambah tabel kegiatan
    table = doc.add_table(rows=1, cols=4)
    table.style = 'Table Grid'
    hdr_cells = table.rows[0].cells
    hdr_cells[0].text = 'Kegiatan'
    hdr_cells[1].text = 'Anggaran'
    hdr_cells[2].text = 'Realisasi'
    hdr_cells[3].text = 'Progres'
    
    kegiatan_list = Kegiatan.objects.filter(
        apbdes__tahun=tahun
    )
    for k in kegiatan_list:
        row_cells = table.add_row().cells
        row_cells[0].text = k.nama
        row_cells[1].text = f'Rp {k.nilai_anggaran:,.0f}'
        row_cells[2].text = f'Rp {k.apbdes.realisasi:,.0f}'
        row_cells[3].text = f'{k.progres_persen}%'
    
    output_path = f'/tmp/LPJ_{tahun}.docx'
    doc.save(output_path)
    return output_path
```

---

## 9. Security dan Performance

### A. Security Checklist

- [ ] HTTPS wajib di production (Let's Encrypt)
- [ ] CSRF protection (Django default)
- [ ] SQL injection prevention (Django ORM)
- [ ] XSS protection (Django template escaping)
- [ ] File upload validation (extension, size, virus scan)
- [ ] Rate limiting untuk login (django-ratelimit)
- [ ] Password policy (minimal 8 karakter, kombinasi)
- [ ] Session timeout (30 menit idle)
- [ ] Audit log untuk aksi sensitif
- [ ] Backup database otomatis (daily)

### B. Performance Optimization

- [ ] Database indexing untuk field yang sering di-query
- [ ] Pagination untuk tabel besar (50 rows per page)
- [ ] Caching untuk dashboard publik (Redis, 1 jam)
- [ ] Lazy loading untuk grafik (load on scroll)
- [ ] Compress static files (gzip)
- [ ] CDN untuk Bootstrap, Chart.js (fallback lokal)
- [ ] Background task untuk impor besar (Celery)

### C. Backup Strategy

**Daily Backup** (Cron):
```bash
#!/bin/bash
# backup_db.sh
DATE=$(date +%Y%m%d)
pg_dump dasbor_desa > /backups/db_$DATE.sql
find /backups -name "db_*.sql" -mtime +30 -delete
```

**Media Files Backup**:
```bash
rsync -avz /path/to/media/ /backups/media_$DATE/
```

---

## 10. Deployment Checklist

### Minimal Requirements

**Hardware**:
- CPU: 2 core
- RAM: 2GB
- Storage: 20GB (10GB untuk database, 10GB untuk media files)

**Software**:
- OS: Ubuntu 22.04 LTS
- Python: 3.11+
- PostgreSQL: 14+
- Nginx: 1.18+

### Environment Variables

```bash
# .env.production
DEBUG=False
SECRET_KEY=<random-50-char-string>
ALLOWED_HOSTS=dasbordesa.id,www.dasbordesa.id

DATABASE_URL=postgresql://user:pass@localhost:5432/dasbor_desa

# Desa-specific config
DESA_NAMA=Desa Tiudan
DESA_KODE=35.01.01.2001
DESA_ALAMAT=Jl. Raya Tiudan No. 1
DESA_KECAMATAN=Gondang
DESA_KABUPATEN=Mojokerto
DESA_PROVINSI=Jawa Timur

# Email (optional, untuk notifikasi)
EMAIL_HOST=smtp.gmail.com
EMAIL_PORT=587
EMAIL_USE_TLS=True
EMAIL_HOST_USER=desatiudan@gmail.com
EMAIL_HOST_PASSWORD=<app-password>

# Storage
MEDIA_ROOT=/var/www/dasbor-desa/media
STATIC_ROOT=/var/www/dasbor-desa/static
```

### Deployment Steps (Ubuntu VPS)

```bash
# 1. Update system
sudo apt update && sudo apt upgrade -y

# 2. Install dependencies
sudo apt install python3.11 python3.11-venv postgresql nginx -y

# 3. Setup database
sudo -u postgres createuser dasbor_user
sudo -u postgres createdb dasbor_desa -O dasbor_user
sudo -u postgres psql -c "ALTER USER dasbor_user PASSWORD 'strong_password';"

# 4. Clone repository
cd /var/www
git clone https://github.com/org/dasbor-desa.git
cd dasbor-desa

# 5. Setup Python environment
python3.11 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# 6. Configure environment
cp .env.example .env
nano .env  # Edit sesuai konfigurasi

# 7. Run migrations
python manage.py migrate
python manage.py loaddata fixtures/kode_rekening.json
python manage.py createsuperuser

# 8. Collect static files
python manage.py collectstatic --noinput

# 9. Setup Gunicorn
sudo nano /etc/systemd/system/dasbor-desa.service
```

**File**: `/etc/systemd/system/dasbor-desa.service`
```ini
[Unit]
Description=Dasbor Desa Gunicorn
After=network.target

[Service]
User=www-data
Group=www-data
WorkingDirectory=/var/www/dasbor-desa
Environment="PATH=/var/www/dasbor-desa/venv/bin"
ExecStart=/var/www/dasbor-desa/venv/bin/gunicorn --workers 3 --bind 127.0.0.1:8000 config.wsgi:application

[Install]
WantedBy=multi-user.target
```

```bash
# 10. Start Gunicorn
sudo systemctl start dasbor-desa
sudo systemctl enable dasbor-desa

# 11. Configure Nginx
sudo nano /etc/nginx/sites-available/dasbor-desa
```

**File**: `/etc/nginx/sites-available/dasbor-desa`
```nginx
server {
    listen 80;
    server_name dasbordesa.id www.dasbordesa.id;

    location /static/ {
        alias /var/www/dasbor-desa/static/;
    }

    location /media/ {
        alias /var/www/dasbor-desa/media/;
    }

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

```bash
# 12. Enable site
sudo ln -s /etc/nginx/sites-available/dasbor-desa /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx

# 13. Setup SSL (Let's Encrypt)
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d dasbordesa.id -d www.dasbordesa.id

# 14. Setup cron jobs
crontab -e
```

**Cron jobs**:
```cron
# Kalkulasi indikator (setiap tanggal 1, jam 01:00)
0 1 1 * * cd /var/www/dasbor-desa && /var/www/dasbor-desa/venv/bin/python manage.py calculate_indicators

# Backup database (setiap hari jam 02:00)
0 2 * * * /var/www/dasbor-desa/scripts/backup_db.sh
```

---

## 11. Testing dan Validasi (Belum Dilakukan)

Berikut area yang perlu ditest sebelum sistem ini bisa dianggap layak pakai:

### A. Validasi Data
- [ ] Test impor Excel SISKEUDES dengan data real (minimal 3 desa berbeda)
- [ ] Validasi formula indikator dengan praktisi tata kelola desa
- [ ] Cross-check hasil kalkulasi dengan perhitungan manual

### B. Usability Testing
- [ ] Test dengan operator desa yang actual (bukan asumsi developer)
- [ ] Ukur waktu yang dibutuhkan untuk task umum (input kegiatan, update progres)
- [ ] Identifikasi pain points dalam workflow

### C. Security Testing
- [ ] Penetration testing dasar (OWASP Top 10)
- [ ] Test file upload vulnerabilities
- [ ] Test SQL injection (meski Django ORM harusnya aman)
- [ ] Test authorization (apakah role benar-benar enforce?)

### D. Performance Testing
- [ ] Load testing dengan data 5 tahun APBDes
- [ ] Test response time dashboard dengan 100+ kegiatan
- [ ] Test impor Excel dengan file 1000+ baris

### E. Deployment Testing
- [ ] Test di shared hosting murah (bukan VPS)
- [ ] Test di koneksi internet lambat (3G)
- [ ] Test di browser lama (IE11, jika masih ada yang pakai)

**Status saat ini: Semua belum dilakukan.**

---

## 12. Catatan untuk Developer

### Ini Bukan Proyek Enterprise

Dokumentasi ini mungkin terlihat lengkap, tapi **jangan tertipu**. Ini proyek personal yang dikembangkan dengan pemahaman terbatas tentang:
- Regulasi keuangan desa yang sebenarnya
- Workflow operasional perangkat desa di lapangan
- Best practice Django untuk aplikasi pemerintahan

### Yang Harus Anda Tahu

**Asumsi yang Mungkin Salah:**
- Formula indikator visioner dibuat berdasarkan logika umum, bukan riset mendalam
- Struktur kode rekening mengikuti Permendagri 20/2018, tapi bisa saja ada update terbaru
- Workflow impor/ekspor diasumsikan dari dokumentasi SISKEUDES online, belum ditest dengan data real

**Area yang Butuh Validasi:**
- Apakah benar SISKEUDES mengeluarkan laporan Excel dengan format seperti yang diasumsikan?
- Apakah Prodeskel/e-Desa punya API atau minimal ekspor CSV yang bisa dipakai?
- Apakah indikator yang dihitung benar-benar berguna atau cuma angka kosong?

**Kontribusi yang Sangat Dibutuhkan:**
- Testing dengan data real dari desa
- Code review untuk security holes
- Validasi formula indikator dengan ahli tata kelola
- Simplifikasi arsitektur jika terlalu kompleks

### Quick Start untuk Developer

Jika Anda ingin kontribusi:

```bash
# Clone repo
git clone https://github.com/your-username/dasbor-desa.git
cd dasbor-desa

# Setup virtual env
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Copy env example
cp .env.example .env

# Run migrations
python manage.py migrate

# Load sample data (jika ada)
python manage.py loaddata fixtures/sample_data.json

# Run dev server
python manage.py runserver
```

Silakan buka issue jika ada yang tidak jelas atau tidak masuk akal.

---

## Kesimpulan

Dokumentasi arsitektur ini adalah **rencana ideal**, bukan realita yang sudah terbangun. Banyak bagian yang masih berupa asumsi dan perlu divalidasi dengan:
- Data real dari desa
- Feedback dari perangkat desa yang akan pakai sistem ini
- Code review dari developer berpengalaman
- Security audit

**Disclaimer:**
- Belum ada testing di production
- Belum ada user real yang mencoba
- Formula indikator bisa jadi tidak relevan
- Integrasi dengan sistem existing bisa jadi lebih sulit dari perkiraan
- Deployment guide mungkin ada yang missed

Jika Anda menemukan hal yang tidak masuk akal, **tolong buka issue**. Kritik konstruktif jauh lebih berharga daripada silent readers.

Dokumentasi ini akan terus diperbaiki berdasarkan pembelajaran dari implementasi nyata dan feedback komunitas. Saat ini statusnya: **work in progress, highly experimental**.
