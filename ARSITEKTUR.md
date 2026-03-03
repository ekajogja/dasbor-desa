# Arsitektur & Perancangan Dasbor Desa

Dokumen ini merinci arsitektur antarmuka, sitemap, dan alur pengguna untuk proyek **Dasbor Desa**. Fokus utama adalah menyediakan sistem monitoring yang bersih, modular, dan mudah diimplementasikan menggunakan Django.

## 1. Peta Situs (Sitemap)

Struktur navigasi Dasbor Desa dibagi menjadi dua area utama: **Publik** dan **Terbatas (Privat)**.

### A. Area Publik (Tanpa Login)
- **Landing Page (`index.html`)**: Gerbang utama, informasi umum, dan tautan ke modul mockup.
- **Beranda Publik (`dashboard_publik.html`)**: Ringkasan indikator desa (Pelayanan, Transparansi, Partisipasi, Ketahanan), grafik realisasi APBDes, progres kegiatan berjalan, statistik layanan bulanan, dan ringkasan aset.
- **Detail Informasi (`detail_data.html`)**: Tampilan rincian dari satu entri data publik (misal: detail satu kegiatan).

### B. Area Terbatas (Setelah Login)
- **Halaman Login (`login.html`)**: Otentikasi pengguna (Admin/Operator).
- **Beranda Privat (`dashboard_privat.html`)**: Ringkasan operasional (permohonan layanan, lapor kegiatan), analitik cashflow, deviasi anggaran, distribusi waktu layanan, dan audit log terbaru.
- **Modul Data (Tabel & Tabs) (`tabel_data.html`)**:
    - Tab 1: Data Penduduk
    - Tab 2: Anggaran (APBDes)
    - Tab 3: Transaksi Realisasi (Detail pengeluaran)
    - Tab 4: Progres Kegiatan
    - Tab 5: Layanan Publik
    - Tab 6: Aset Desa
    - Tab 7: Audit Log (Hanya Admin)
- **Modul Input (Form & Tabs) (`form_input.html`)**:
    - Tab 1: Form APBDes & Kegiatan
    - Tab 2: Form Realisasi & Layanan
    - Tab 3: Form Aset & Pendukung
- **Administrasi Pengguna (`admin_user.html`)**: Manajemen akun operator desa dan hak akses.
- **Pengaturan Profil (`profil.html`)**: Update informasi akun sendiri.

### C. Halaman Pendukung
- **Detail Data (`detail_data.html`)**: Tampilan rincian dari satu entri data.
- **Halaman Error (`404.html`)**: Tampilan jika halaman tidak ditemukan.

---

## 2. Alur Pengguna (User Flow)

### Alur Publik
1. Pengunjung masuk ke **Landing Page**.
2. Memilih **Beranda Publik** untuk melihat statistik desa secara umum.
3. Dapat mengklik grafik atau tabel untuk melihat detail (read-only).

### Alur Operator/Admin (Privat)
1. User melakukan **Login**.
2. Diarahkan ke **Beranda Privat** (Dashboard Internal) untuk melihat anomali atau tugas mendesak.
3. Menggunakan sidebar untuk navigasi ke **Tabel Data** (untuk monitoring detail) atau **Form Input** (untuk entri data baru).
4. Mengisi data melalui **Form Berjenjang (Tabs)** yang divalidasi sistem.
5. Data yang diinput akan secara otomatis mengupdate agregasi di **Beranda Publik**.
6. Admin dapat mengakses menu **Administrasi User** untuk mengelola hak akses dan melihat **Audit Log**.

---

## 3. Strategi Template Django

Mockup ini dirancang agar mudah diintegrasikan ke dalam sistem templating Django (`Django Template Language`).

### Struktur Modular
- `base.html`: Kerangka utama (HTML, Head, Body, Scripts).
- `_navbar.html`: Komponen navigasi atas (termasuk profil user).
- `_sidebar.html`: Komponen navigasi samping (dinamis berdasarkan role).
- `_footer.html`: Informasi hak cipta dan versi.

### Penggunaan Blocks
Setiap halaman mockup menggunakan `{% block content %}` sebagai penampung isi utama, sehingga konsisten dengan cara kerja Django.

### Interaktivitas (HTMX)
Untuk implementasi akhir di Django, interaksi antar tab atau update data parsial akan menggunakan **HTMX** guna memberikan pengalaman pengguna yang mulus tanpa reload halaman penuh (meskipun pada mockup ini masih menggunakan reload/static links demi kompatibilitas GitHub Pages).

---

## 4. Teknologi UI (Stack Mockup)
- **Framework CSS**: Bootstrap 5 (Ringan, familiar, dokumentasi lengkap).
- **Ikon**: Bootstrap Icons.
- **Grafik**: Chart.js (Mudah digunakan untuk representasi data numerik).
- **Peta**: Leaflet.js (Peta interaktif Desa Tiudan).
- **Interaksi Ringan**: Bootstrap Native JS (Modals, Tabs, Tooltips).
