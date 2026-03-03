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
