# Dasbor Desa

Dasbor Desa adalah proyek open source untuk membangun sistem monitoring dan transparansi desa berbasis indikator, dengan pendekatan semi-publik dan arsitektur yang sederhana, terpisah per desa (single instance).

Proyek ini berfokus pada layer monitoring dan visualisasi strategis, bukan sistem administrasi kependudukan penuh. Sistem dirancang agar dapat berdiri sendiri maupun berjalan berdampingan dengan sistem informasi desa lain.

---

## Visi

Menyediakan fondasi teknis yang bersih, modular, dan mudah direplikasi untuk membangun dashboard desa yang:

* Real-time (berbasis input langsung ke sistem)
* Terstruktur (menggunakan database relasional)
* Semi-publik (sesuai klasifikasi informasi)
* Siap dikembangkan lintas desa tanpa arsitektur multi-tenant yang kompleks

---

## Prinsip Arsitektur

* Satu desa, satu instance (isolasi penuh per deployment)
* Satu codebase utama
* Konfigurasi berbasis environment variable
* Customisasi melalui konfigurasi atau feature flag, bukan fork permanen
* Database relasional (PostgreSQL)
* Backend-first (Django)

Tidak ada dependensi pada SaaS tertentu. Integrasi eksternal bersifat opsional.

---

## Ruang Lingkup Sistem

### 1. Dashboard Indikator Standar

Mengakomodasi kebutuhan pelaporan formal dan monitoring dasar:

* Ringkasan APBDes
* Serapan anggaran
* Progres kegiatan
* Statistik layanan
* Ringkasan aset

Fokus: konsistensi data dan keterbacaan lintas level pemerintahan.

---

### 2. Dashboard Indikator Visioner

Layer strategis untuk mengukur kualitas tata kelola:

* Indeks Pelayanan Desa
* Indeks Transparansi
* Indeks Partisipasi
* Indeks Ketahanan

Indikator dihitung dari data operasional yang sama, tanpa duplikasi sumber.

---

## Model Akses

### Publik

* Ringkasan indikator agregat
* Visualisasi non-sensitif

### Terbatas (Login)

* Detail data
* Manajemen input
* Dokumen pendukung
* Audit log

Kontrol akses berbasis role dan group.

---

## Fitur Inti

* Form input terstruktur dengan validasi
* Table view dengan filter, sorting, pagination
* Dashboard berbasis grafik
* Audit trail perubahan data
* Impor dari Excel (template terstandar)
* Ekspor ke Excel dan Word

---

## Struktur Repository (Usulan)

```
root/
 ├── backend/
 │    ├── config/
 │    ├── apps/
 │    ├── templates/
 │    ├── static/
 │    └── manage.py
 │
 ├── docs/
 ├── mockups/
 └── README.md
```

`mockups/` hanya berisi referensi desain statis untuk eksplorasi UI, tidak menjadi bagian dari runtime sistem.

---

## Spesifikasi Awal: Tabel, Form Input, dan Charts

### A. Layer Publik

#### 1. Tabel (Read-Only)

* `indikator_ringkasan_tahunan`

  * tahun
  * total_anggaran
  * total_realisasi
  * persentase_serapan
  * indeks_pelayanan
  * indeks_transparansi

* `kegiatan_berjalan`

  * nama_kegiatan
  * bidang
  * nilai_anggaran
  * progres_persen
  * status

* `statistik_layanan_bulanan`

  * bulan
  * jumlah_surat_diterbitkan
  * jumlah_permohonan_masuk
  * rata_rata_waktu_layanan

* `ringkasan_aset`

  * kategori
  * jumlah_unit
  * estimasi_nilai

#### 2. Charts (Agregat, Non-Sensitif)

* Line chart: Serapan anggaran per bulan
* Bar chart: Progres kegiatan per bidang
* Pie chart: Komposisi belanja (operasional vs pembangunan)
* Line chart: Tren jumlah layanan publik
* Radar chart: Indeks visioner (pelayanan, transparansi, partisipasi, ketahanan)

Semua grafik berbasis agregasi, tanpa menampilkan data individu.

---

### B. Layer Privat (Login Required)

#### 1. Tabel Operasional

* `apbdes`

  * id
  * tahun
  * bidang
  * sub_bidang
  * jenis_belanja
  * pagu
  * realisasi
  * updated_at

* `kegiatan`

  * id
  * nama
  * bidang
  * lokasi
  * nilai_anggaran
  * tanggal_mulai
  * tanggal_selesai
  * progres_persen
  * penanggung_jawab

* `transaksi_realisasi`

  * id
  * kegiatan_id
  * tanggal
  * uraian
  * jumlah
  * bukti_dokumen

* `layanan`

  * id
  * jenis_layanan
  * tanggal_permohonan
  * tanggal_selesai
  * status
  * petugas

* `aset`

  * id
  * nama_aset
  * kategori
  * tahun_perolehan
  * nilai_perolehan
  * kondisi

* `users`

  * id
  * nama
  * role
  * aktif

* `audit_log`

  * id
  * user_id
  * aksi
  * tabel
  * record_id
  * timestamp

#### 2. Form Input

* Form input APBDes (tahunan dan perubahan)
* Form input kegiatan baru
* Form update progres kegiatan
* Form input realisasi transaksi
* Form pencatatan layanan masuk
* Form penyelesaian layanan
* Form pencatatan aset
* Form manajemen user & role

Semua form memiliki:

* Validasi server-side
* Field wajib terstandar
* Timestamp otomatis
* Pencatatan audit log

#### 3. Charts Internal (Detail & Analitik)

* Cashflow bulanan (anggaran vs realisasi)
* Deviasi anggaran per kegiatan
* Distribusi waktu penyelesaian layanan
* Heatmap beban layanan per bulan
* Ranking kegiatan berdasarkan progres
* Deteksi anomali sederhana (lonjakan realisasi tidak wajar)

---

### C. Relasi Layer Publik dan Privat

* Layer publik hanya membaca hasil agregasi dari tabel operasional.
* Tidak ada duplikasi tabel khusus untuk publik.
* Perubahan data di layer privat langsung mempengaruhi agregasi publik.
* Indikator visioner dihitung melalui fungsi/management command terjadwal.

---

## Roadmap Teknis

### Phase 1

* Finalisasi model data inti
* Implementasi modul indikator standar
* Sistem autentikasi dan role

### Phase 2

* Implementasi indikator visioner
* Impor/ekspor data
* Hardening keamanan dan audit log

### Phase 3

* Optimasi performa
* Dokumentasi deployment per desa
* Standarisasi konfigurasi

---

## Kontribusi

Kontribusi diharapkan dalam bentuk:

* Perbaikan arsitektur
* Penyempurnaan model data
* Refactor kode
* Pengujian keamanan
* Penyusunan indikator berbasis data nyata

Gunakan issue dan pull request untuk diskusi teknis.

---

## Catatan

Proyek ini tidak berpretensi menjadi sistem nasional atau menggantikan sistem administrasi yang telah ada. Fokusnya adalah menyediakan fondasi dashboard monitoring desa yang bersih, modular, dan dapat dikembangkan lebih lanjut oleh komunitas.
