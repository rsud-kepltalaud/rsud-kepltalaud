# Sistem Informasi RSUD Talaud

Portal satu-halaman (SPA) dengan rute berbasis hash, frontend mandiri di `index.html`, dan backend Google Apps Script di `Code.gs`. Data dinamis hanya berasal dari backend yang dikonfigurasi; portal tidak membuat atau menampilkan data pasien contoh.

## Isi folder

- `index.html` — antarmuka responsif, seluruh CSS/JavaScript, navigasi SPA, formulir publik, dan panel administrator.
- `Code.gs` — API JSON dan integrasi Google Sheets.

## Prasyarat

- Akun Google yang berwenang mengelola spreadsheet dan Apps Script.
- Spreadsheet khusus untuk aplikasi ini. Jangan memakai spreadsheet yang dapat diakses publik.
- Browser modern; koneksi ke CDN Bootstrap, Bootstrap Icons, dan Google Fonts diperlukan untuk aset visual.

## 1. Membuat Google Spreadsheet

1. Buat spreadsheet baru dan simpan di Google Drive dengan akses **Restricted**.
2. Salin ID dari URL, yaitu bagian di antara `/d/` dan `/edit`.
3. Buka **Extensions → Apps Script** dari spreadsheet tersebut.
4. Ganti isi `Code.gs` dengan file `Code.gs` dari folder ini, lalu simpan.
5. Di Apps Script, buka **Project Settings → Script properties**, lalu tambahkan:
   - Property: `SPREADSHEET_ID`
   - Value: ID spreadsheet yang disalin pada langkah 2.
6. Pilih dan jalankan `setupDatabase()` dari editor Apps Script. Berikan izin Google yang diminta. Fungsi ini membuat sheet dan baris header yang belum ada. Fungsi tidak mengosongkan sheet berisi data; header yang tidak cocok menyebabkan error eksplisit.
7. Pada sheet `Admin`, tambahkan satu baris untuk setiap email Google administrator:
   - `email`: alamat akun Google persis.
   - `role`: `admin`.
   - `active`: `true`.
   - `createdAt`: tanggal penambahan.
   Jangan menambahkan akun yang tidak berwenang. Jangan pernah mengisi kredensial admin di frontend.

Sheet `Pasien`, `Pendaftaran`, dan `Pengaduan` berisi data pribadi; jangan membagikan sheet tersebut ke publik. Formulir publik sengaja tidak meminta NIK, diagnosis, atau nomor rekam medis.

`setupDatabase()` juga menyiapkan sheet `InformasiPelayanan` dengan kolom `id`, `nama_pelayanan`, `slug`, `deskripsi`, `konten`, `persyaratan`, `prosedur`, `jadwal`, `status`, `created_at`, dan `updated_at`. Sepuluh slug yang didukung adalah `rawat-jalan`, `rawat-inap`, `igd`, `laboratorium`, `radiologi`, `farmasi`, `operasi`, `persalinan`, `rehabilitasi-medik`, dan `ambulans`. Administrator dapat menambah, mengedit, mengaktifkan/nonaktifkan, atau menghapus konten dari menu **Administrator → Informasi Pelayanan**. Hanya record berstatus `aktif` yang dikirim ke halaman publik. Lengkapi hanya dengan informasi yang telah diverifikasi dan disahkan.

Halaman layanan menggunakan URL hash `#/pelayanan/<slug>`, sehingga dapat dibagikan dan dibuka langsung di GitHub Pages. Rute lama `#/rawat-jalan` dan seterusnya tetap didukung.

## 2. Deployment Google Apps Script

1. Pilih **Deploy → New deployment → Web app**.
2. Pilih identitas eksekusi setelah menguji perilaku `Session.getActiveUser().getEmail()` pada jenis akun dan kebijakan Google Workspace organisasi Anda.
3. Batasi akses deployment sesuai kebijakan organisasi. Otorisasi admin pada kode selalu memeriksa email Google terhadap sheet `Admin` dan menolak akses jika identitas tidak dapat diverifikasi.
4. Jangan menganggap deployment **Execute as me / Anyone** sebagai autentikasi administrator: pada beberapa konfigurasi Google, email pengguna anonim tidak tersedia. Kode gagal-aman dan menolak operasi admin dalam kondisi tersebut.
5. Deploy, salin URL Web App berakhiran `/exec`, lalu uji endpoint `?action=getDoctorList`.

**Penting untuk deployment publik:** dukungan CORS dan ketersediaan identitas Google pada Apps Script dapat bergantung pada konfigurasi akun/deployment. Uji permintaan GET dan POST dari origin GitHub Pages serta verifikasi `loginAdmin` sebelum menerima data sungguhan. Bila browser menolak CORS, jangan menonaktifkan validasi atau autentikasi; gunakan gateway/proxy server terkelola yang menambahkan CORS dan mempertahankan otorisasi server-side.

## 3. Menghubungkan frontend ke API

Di awal skrip `index.html`, ganti:

```js
const API_URL = "";
```

dengan URL Web App resmi berakhiran `/exec`, misalnya:

```js
const API_URL = "https://script.google.com/macros/s/DEPLOYMENT_ID/exec";
```

Jangan menaruh secret, token, password, atau ID spreadsheet pada frontend. URL deployment bukan pengganti kontrol akses. Jika URL kosong atau tidak valid, aplikasi menampilkan pemberitahuan bahwa server belum dikonfigurasi dan tidak mengirimkan data formulir.

### Aksi API yang tersedia

- GET `getDoctorList`, `getDoctorSchedule`, dan `getData` untuk data publik yang telah dipublikasikan.
- POST `createAppointment` dan `submitComplaint` untuk membuat permohonan/tiket.
- POST `checkAppointment` dan `getComplaintStatus` menerima pasangan ID dan nomor kontak di body (bukan URL), lalu hanya mengembalikan status—bukan identitas atau isi pesan.
- GET `loginAdmin`, `getDashboardStatistics`; POST `insertData`, `updateData`, `deleteData` memerlukan email yang aktif pada sheet `Admin`.

Endpoint data pasien, pendaftaran, pengaduan, administrator, dan log tidak tersedia untuk akses publik. CRUD dibatasi pada Dokter, JadwalDokter, Poliklinik, Berita, Galeri, dan InformasiPelayanan. Konten publik layanan hanya membaca status aktif. `updateData` dan `deleteData` tetap memerlukan otorisasi administrator.

## 4. Upload ke GitHub dan aktifkan GitHub Pages

1. Buat repository GitHub baru dan unggah seluruh isi folder `rsud-talaud/` ke root repository (agar halaman utama berada di root sebagai `index.html`).
2. Di **Settings → Pages**, pilih sumber publikasi yang sesuai, biasanya branch `main` dan folder `/ (root)`.
3. Simpan, lalu tunggu URL Pages diterbitkan.
4. Buka URL HTTPS Pages dan uji seluruh rute, formulir, lookup privat, serta akses administrator.
5. Sebelum penggunaan nyata, pastikan domain Pages yang benar diizinkan oleh kebijakan organisasi dan konfigurasi Apps Script.

## 5. Catatan sebelum penggunaan nyata

- Konten profil, jadwal, fasilitas, alamat, berita, serta daftar dokter harus diverifikasi dan diisi oleh pengelola rumah sakit. Nilai kosong sengaja tidak diganti dengan statistik atau identitas rekaan.
- Pendaftaran digital baru merupakan permohonan untuk diverifikasi petugas, bukan konfirmasi janji atau antrean final.
- Status publik hanya menampilkan status. Nomor registrasi/tiket dan nomor kontak harus dijaga oleh pengguna.
- Spreadsheet bukan pengganti sistem rekam medis atau kontrol akses tingkat produksi. Tinjau kebijakan privasi, retensi/penghapusan data, kontrol akses Google Workspace, audit, backup, pembatasan penyalahgunaan, dan ketentuan perlindungan data sebelum mengoperasikan layanan sungguhan.
- Lakukan uji CORS, otorisasi, pembatasan akses sheet, validasi input, dan skenario beban dalam lingkungan non-produksi terlebih dahulu.
