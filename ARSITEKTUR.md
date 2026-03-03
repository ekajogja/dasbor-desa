# Arsitektur & Perancangan Dasbor Desa

Dokumen ini merinci arsitektur antarmuka, sitemap, dan alur pengguna untuk proyek **Dasbor Desa**. Fokus utama adalah menyediakan sistem monitoring yang bersih, modular, dan mudah diimplementasikan menggunakan Django.

## 1. Peta Situs (Sitemap)

Struktur navigasi Dasbor Desa dibagi menjadi dua area utama: **Publik** dan **Terbatas (Privat)**.

### A. Area Publik (Tanpa Login)
- **Landing Page (`index.html`)**: Gerbang utama, informasi umum, dan tautan ke modul mockup.
- **Beranda Publik (`dashboard_publik.html`)**: Ringkasan indikator desa, grafik APBDes, dan peta lokasi desa.
- **Detail Informasi**: Halaman statis untuk informasi transparansi lainnya.

### B. Area Terbatas (Setelah Login)
- **Halaman Login (`login.html`)**: Otentikasi pengguna.
- **Beranda Privat (`dashboard_privat.html`)**: Ringkasan operasional dan akses cepat ke fungsi manajemen.
- **Modul Data (Tabel & Tabs) (`tabel_data.html`)**:
    - Tab 1: Data Penduduk
    - Tab 2: Anggaran (APBDes)
    - Tab 3: Progres Kegiatan
- **Modul Input (Form & Tabs) (`form_input.html`)**:
    - Tab 1: Informasi Dasar
    - Tab 2: Detail Anggaran
    - Tab 3: Lampiran Dokumen
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
2. Diarahkan ke **Beranda Privat** (Dashboard Internal).
3. Menggunakan sidebar untuk navigasi ke **Tabel Data** atau **Form Input**.
4. Mengisi data melalui **Form Berjenjang (Tabs)** untuk memecah form yang panjang.
5. Admin dapat mengakses menu **Administrasi User** untuk mengelola tim operator.

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
