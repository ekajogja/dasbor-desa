# Dasbor Desa

Dasbor Desa adalah sistem **visualisasi dan monitoring** berbasis web yang dirancang untuk melengkapi (bukan menggantikan) sistem administrasi desa yang sudah ada seperti SISKEUDES, Prodeskel, dan aplikasi wajib lainnya.

## Proposisi Nilai

**Masalah yang Dipecahkan:**
1. Data tersebar di banyak aplikasi (SISKEUDES, Prodeskel, SIMDABUMAS, spreadsheet manual)
2. Sulit mendapatkan gambaran cepat tentang kinerja desa secara keseluruhan
3. Laporan untuk atasan atau publik memerlukan waktu lama untuk disusun
4. Tidak ada cara mudah untuk mengukur kualitas pelayanan dan transparansi

**Solusi:**
- Dashboard terpusat yang mengagregasi data dari berbagai sumber
- Visualisasi otomatis untuk laporan rutin
- Perhitungan indikator kinerja secara real-time
- Akses publik untuk transparansi tanpa membuka data sensitif

**Yang TIDAK Dilakukan Sistem Ini:**
- Menggantikan SISKEUDES (tetap gunakan SISKEUDES untuk keuangan)
- Menggantikan sistem kependudukan (tetap gunakan Prodeskel/e-Desa)
- Menambah beban input data manual yang berlebihan

---

## Prinsip Desain

### 1. Integrasi, Bukan Duplikasi
Sistem ini dirancang untuk **mengimpor** data dari aplikasi existing:
- APBDes dari SISKEUDES (format Excel standar)
- Data layanan dari SIMDABUMAS (template CSV)
- Data agregat penduduk dari Prodeskel (tanpa data individu)

### 2. Single Instance per Desa
- Satu desa, satu deployment (tidak multi-tenant)
- Isolasi penuh antar desa
- Cocok untuk deployment di server lokal atau shared hosting murah
- Tidak ada ketergantungan pada cloud vendor tertentu

### 3. Offline-Capable
- Operator dapat ekspor template Excel, isi offline, lalu impor kembali
- PWA untuk akses dashboard publik tanpa koneksi
- Deployment dapat di server lokal (Raspberry Pi) untuk desa tanpa internet stabil

### 4. Minimal Training
- Interface familiar (mirip Excel/Google Sheets untuk tabel data)
- Video tutorial 5-10 menit per modul
- Form sederhana dengan validasi yang jelas

---

## Fitur Utama

### A. Dashboard Publik (Tanpa Login)
Transparansi untuk warga:
- Ringkasan APBDes dan realisasi per bidang
- Progres kegiatan pembangunan saat ini
- Statistik layanan publik (jumlah surat, waktu rata-rata)
- Indikator kinerja: Pelayanan, Transparansi, Partisipasi, Ketahanan

### B. Dashboard Privat (Operator/Admin)
Monitoring operasional:
- Deteksi anomali (realisasi melebihi pagu, layanan terlambat)
- Analitik cashflow dan deviasi anggaran
- Daftar tugas mendesak (permohonan layanan pending)
- Audit log semua perubahan data

### C. Modul Input dan Impor
- Form manual untuk data kecil
- Impor Excel batch untuk data besar (APBDes, realisasi)
- Validasi otomatis sesuai Permendagri 20/2018
- Soft delete dan version history untuk data kritis

### D. Ekspor Laporan
Template siap pakai untuk:
- LPJ Kepala Desa
- Laporan realisasi APBDes (triwulan/semester)
- Format SIPEDE (Sistem Pemerintahan Desa)
- Laporan capaian SDGs Desa

---

## Arsitektur Teknis

### Stack Teknologi
- **Backend**: Django 5.x + PostgreSQL
- **Frontend**: Bootstrap 5 + HTMX (interaktivitas tanpa reload)
- **Grafik**: Chart.js
- **Peta**: Leaflet.js
- **Deployment**: Docker Compose atau shared hosting cPanel

### Model Data Inti

#### 1. Statistik Penduduk (Agregat Bulanan)
Sumber: impor dari Prodeskel/e-Desa
```
- periode (bulan/tahun)
- jumlah_kk, jumlah_jiwa
- usia_produktif, lansia, balita
- penerima_bantuan_sosial
- partisipan_musrenbang
```

#### 2. APBDes
Sumber: impor dari SISKEUDES
```
- tahun, kode_bidang, kode_kegiatan
- uraian, pagu, realisasi
- mengikuti struktur Permendagri 20/2018
```

#### 3. Transaksi Realisasi
Sumber: impor dari SISKEUDES atau input manual
```
- tanggal, kegiatan, uraian, jumlah
- bukti_dokumen (upload scan)
- status_verifikasi
```

#### 4. Kegiatan Pembangunan
Sumber: input manual atau impor
```
- nama, bidang, lokasi, nilai_anggaran
- tanggal_mulai, target_selesai
- progres_persen, foto_progres
- penanggung_jawab
```

#### 5. Layanan Publik
Sumber: impor dari SIMDABUMAS atau input manual
```
- jenis_layanan, tanggal_masuk
- tanggal_selesai, waktu_proses (hari)
- status, petugas
```

#### 6. Aset Desa
Sumber: impor dari SIMBADA atau input manual
```
- nama_aset, kategori
- tahun_perolehan, nilai_perolehan
- kondisi, lokasi
```

#### 7. Audit Log
Otomatis dari sistem
```
- user, aksi, tabel, record_id
- nilai_lama, nilai_baru
- timestamp
```

### Indikator Visioner (Kalkulasi Otomatis)

#### Indeks Pelayanan
Formula:
```
(Jumlah layanan selesai tepat waktu / Total layanan) × 40% +
(1 - (Rata-rata waktu proses / Target waktu standar)) × 30% +
(Pengaduan terselesaikan / Total pengaduan) × 30%

Skala: 0-100
```

#### Indeks Transparansi
Formula:
```
(Frekuensi publikasi laporan / 4 per tahun) × 40% +
(Dokumen APBDes lengkap / Total dokumen wajib) × 30% +
(Update data terakhir < 30 hari) × 30%

Skala: 0-100
```

#### Indeks Partisipasi
Formula:
```
(Jumlah peserta Musrenbang / Target peserta) × 50% +
(Usulan warga yang direalisasi / Total usulan) × 30% +
(Kehadiran di rapat RT/RW / Total rapat) × 20%

Skala: 0-100
```

#### Indeks Ketahanan
Formula:
```
(Serapan anggaran pembangunan / Pagu pembangunan) × 40% +
(Kegiatan selesai tepat waktu / Total kegiatan) × 30% +
(1 - (Nilai aset rusak / Total nilai aset)) × 30%

Skala: 0-100
```

Semua formula dapat dikonfigurasi ulang melalui file JSON tanpa mengubah kode.

---

## Alur Kerja Ideal

### Setup Awal (Sekali)
1. Install sistem di server lokal atau hosting
2. Konfigurasi profil desa (nama, alamat, logo)
3. Impor data historis APBDes dari SISKEUDES
4. Impor data agregat penduduk dari Prodeskel
5. Setup user dan role (Admin, Operator Keuangan, Operator Layanan)

### Operasional Rutin

**Mingguan:**
- Input/impor data layanan publik yang baru masuk
- Update progres kegiatan pembangunan (foto, persentase)

**Bulanan:**
- Impor realisasi APBDes dari SISKEUDES
- Update statistik penduduk (agregat)
- Review anomali yang terdeteksi sistem

**Triwulan/Semester:**
- Ekspor laporan untuk atasan (Camat, Bupati)
- Review indikator kinerja
- Publikasi dashboard ke warga (screenshot atau PDF)

**Tahunan:**
- Input APBDes tahun baru
- Arsip data tahun lalu
- Evaluasi target indikator

---

## Status Proyek

**Proyek independen tanpa afiliasi lembaga atau perusahaan. Dikembangkan sendiri tanpa timeline atau komitmen deliverable.**

Saat ini masih berupa konsep dan dokumentasi. Implementasi kode akan dimulai bertahap sesuai ketersediaan waktu.

### Apa yang Ada Sekarang
- Spesifikasi arsitektur
- Model data dan relasi
- Dokumentasi teknis

### Yang Akan Dikerjakan (tanpa urutan pasti)
- Setup project Django dasar
- Model database dan migrasi
- Dashboard publik sederhana
- Form input manual
- Impor Excel SISKEUDES
- Ekspor laporan

**Tidak ada jaminan kapan fitur akan selesai. Kontribusi komunitas sangat menentukan kecepatan pengembangan.**

---

## Kontribusi

Saya mengembangkan ini sendiri tanpa tim atau pendanaan. Saya sangat butuh masukan untuk memastikan sistem ini tidak jadi software yang ngawur dan malah bikin ribet perangkat desa.

**Yang Paling Saya Butuhkan:**

**Dari Praktisi Desa:**
- Tolong koreksi kalau ada asumsi saya yang salah atau tidak realistis
- Validasi apakah fitur yang saya rencanakan benar-benar dibutuhkan
- Kasih tahu kalau ada regulasi yang saya lewatkan
- Sharing pengalaman: sistem seperti apa yang benar-benar membantu vs yang cuma jadi beban

**Dari Developer:**
- Code review (kalau nanti sudah ada kode)
- Security audit (ini sistem pemerintahan, harus aman)
- Saran arsitektur yang lebih baik
- Kontribusi kode kalau ada waktu

**Dari Akademisi:**
- Kritik metodologi indikator (apakah formula saya masuk akal?)
- Referensi penelitian tentang tata kelola desa digital
- Validasi asumsi-asumsi yang saya buat

Gunakan GitHub Issues untuk diskusi. Semua feedback, sekecil apapun, sangat berarti.

---

## Pertanyaan yang Mungkin Muncul

**Q: Apakah sistem ini sudah bisa dipakai?**
A: Belum. Saat ini baru dokumentasi dan desain. Implementasi kode belum dimulai. Jangan berharap bisa langsung deploy.

**Q: Kapan selesai?**
A: Tidak tahu. Ini proyek sampingan tanpa deadline. Bisa bulan, bisa tahun, bisa juga tidak selesai kalau ternyata konsepnya salah.

**Q: Ada dukungan teknis atau garansi?**
A: Tidak ada. Ini proyek hobi open source. Pakai dengan risiko sendiri. Kalau ada bug atau masalah, bisa report di Issues tapi tidak ada jaminan akan diperbaiki kapan.

**Q: Kenapa tidak pakai aplikasi yang sudah ada saja?**
A: Karena kebanyakan SaaS berbayar atau tidak open source. Proyek ini eksperimen untuk bikin alternatif yang gratis dan transparan. Tapi kalau SaaS yang ada sudah bagus dan terjangkau, ya pakai itu saja.

**Q: Apakah ini akan terus dikembangkan?**
A: Tergantung. Kalau ada feedback bahwa ini berguna dan ada kontribusi dari komunitas, akan dilanjutkan. Kalau ternyata tidak ada yang butuh atau konsepnya keliru, ya stop.

**Q: Boleh fork atau pakai untuk project lain?**
A: Silakan. Lisensi MIT. Tapi tolong validate dulu konsepnya, jangan asal pakai mentah-mentah. Sistem pemerintahan bukan mainan.

---

## Batasan dan Disclaimer

- Sistem ini **tidak menggantikan** aplikasi wajib pemerintah (SISKEUDES, Prodeskel)
- **Tidak menyimpan** data kependudukan individu (hanya agregat)
- **Tidak mengatur** proses bisnis desa (hanya monitoring)
- **Bukan sistem akuntansi** (gunakan SISKEUDES untuk pembukuan resmi)
- Fokus pada **transparansi dan monitoring**, bukan administrasi lengkap

---

## Catatan Teknis

**Repository ini belum berisi kode implementasi.** Yang ada saat ini:
- Dokumentasi arsitektur
- Spesifikasi model data
- Mockup UI

Ketika kode sudah mulai dikembangkan, instruksi instalasi akan ditambahkan di sini.

---

## Kontak

- **GitHub Issues**: Bug report, feature request, atau pertanyaan
- **GitHub Discussions**: Diskusi dan sharing pengalaman

Response bisa lambat karena ini proyek sampingan. No promises.

---

## Lisensi

MIT License - Bebas digunakan, dimodifikasi, dan didistribusikan untuk kepentingan publik.

---

**Catatan Penting:**

Ini adalah proyek eksperimental yang dikembangkan oleh individu berdasarkan pemahaman terbatas tentang tata kelola desa. Bukan produk jadi, bukan produk komersial, dan sangat mungkin ada hal-hal yang salah atau tidak praktis.

**Gunakan dengan risiko sendiri.** Jika Anda menemukan masalah atau punya saran perbaikan, silakan buka issue di GitHub. Kontribusi dalam bentuk apapun sangat dihargai.

Jika ternyata proyek ini tidak berguna atau malah menyulitkan, tolong beritahu saya agar bisa dihentikan atau diubah arah pengembangannya. Tujuan utama adalah membantu, bukan menambah masalah.
