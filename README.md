# WAA Student Journey Dashboard

Dashboard analyst internal untuk memantau progres siswa Work Abroad Academy (WAA) — kehadiran, nilai ujian, status job offer (JO), feedback/NPS, dan persiapan kerja (interview, document review, learning plan). Single-file HTML, tidak butuh backend atau build step.

**File:** `waa_student_dashboard.html`
**Audiens:** tim Acops/SSO & ops WAA yang maintain dashboard ini.

---

## 1. Cara Kerja Singkat

```
Airtable (source of truth)
   → BigQuery replica (replica_airtable_us)
   → dbt models (staging + mart, schoters_dbt_us)
   → Google Sheets (publish as CSV)
   → Dashboard ini (fetch CSV langsung dari browser)
```

Dashboard ini **tidak query BigQuery secara langsung**. Ia membaca 3 Google Sheets yang sudah di-publish sebagai CSV (lihat `URLS` di bagian atas `<script>`), yang masing-masing diisi dari hasil query dbt mart:

| Key di `URLS` | Isi | Granularitas |
|---|---|---|
| `regional` | Ringkasan per regional (kota/kantor) | 1 baris = 1 regional × program |
| `batch` | Ringkasan per batch | 1 baris = 1 batch |
| `student` | Data lengkap per siswa (Student Journey) | 1 baris = 1 siswa |

Karena CSV publish Google Sheets tidak mengizinkan CORS, fetch dilakukan lewat fallback 3 layer (`PROXIES`): coba langsung dulu, lalu `corsproxy.io`, lalu `allorigins.win`. Kalau ketiganya gagal, tab itu akan kosong dengan pesan error di Console — ini paling sering terjadi kalau salah satu proxy publik down, bukan masalah di data.

**Refresh data**: tombol "Refresh" di pojok kanan atas, atau reload halaman. Tidak ada auto-refresh berkala.

---

## 2. Struktur Tab

### Regional
Ringkasan agregat per regional — total siswa, total batch, rata-rata kehadiran/nilai, distribusi tier. Tidak ada panel detail (klik baris tidak membuka apa-apa di tab ini).

### Batch
Ringkasan per batch — bisa diklik untuk membuka panel detail batch (nilai rata-rata per jenis ujian, jumlah partisipan, NPS, jumlah JO).

### Student
Tab paling detail. Tabel utama menampilkan 1 baris per siswa dengan kolom: nama, batch, program, regional, kehadiran, nilai, tier, NPS, jumlah issue, status JO, **jumlah 1on1 Interview, jumlah Document Review, status Learning Plan**. Klik baris manapun membuka panel detail ("Student Passport") di sisi kanan.

**Panel detail Student** menampilkan:
- Ringkasan kehadiran & nilai
- Status Job Offer + riwayat lengkap (kalau ada)
- **Persiapan Kerja**: jumlah 1on1 Interview, jumlah Document Review, dan tombol **"Tampilkan dokumen"** untuk Learning Plan (lihat §4)
- Riwayat NPS, rating rendah, complaint (collapsible)
- Breakdown nilai per jenis ujian (Ausbildung atau Tokutei, tergantung program siswa)
- Logbook sesi yang sudah diikuti

---

## 3. Filter & Pencarian

Sidebar kiri berubah sesuai tab aktif (lihat `buildFilters()`). Semua filter bersifat **client-side** — tidak ada query ulang ke server, jadi instan tapi terbatas oleh berapa banyak data yang sudah ter-fetch.

- Search box, dropdown Program/Regional/Batch (searchable combobox via `<datalist>`), filter Tier, filter Status JO, filter Status Kelulusan, filter Event/Logbook.
- Dropdown **Batch otomatis ikut filter Program** yang aktif — pilih Tokutei dulu, baru dropdown Batch cuma nampilin batch Tokutei.
- Tabel Student dibatasi **2000 baris pertama** (`LIMIT` di `renderStudent()`) untuk performa. Kalau total data lebih dari itu, gunakan filter untuk mempersempit.

---

## 4. Catatan Penting: Learning Plan (Link Dokumen)

Ini bagian yang paling rawan kalau ada perubahan struktur data di Airtable/BigQuery — baca sebelum mengubah apapun yang berhubungan dengan dokumen.

**Riwayat keputusan (penting untuk konteks):**
1. Awalnya `learning_plan_url` diambil dari field attachment Airtable (`generate_learning_plan`). **Ini gagal** — attachment URL Airtable (domain `airtableusercontent.com`) adalah signed URL yang **expire dalam ~2 jam**. Karena sync BigQuery cuma sehari sekali, URL yang tersimpan sudah pasti expired sebelum sempat dibuka siapapun.
2. Solusinya: pindah sumber ke kolom **`link_docs__from__3__student_profiling_`** — field lookup ke Google Drive yang linknya **permanen** (selama file tidak dihapus/sharing tidak dicabut). Field ini `ARRAY<STRING>`, kita ambil elemen **terakhir** kalau ada lebih dari satu (lihat CTE `learning_plan_cte` di §6).

**Kenapa link-nya perlu dikonversi, bukan dipakai mentah:**
Link asli dari kolom itu berformat `https://drive.google.com/uc?id=FILE_ID&export=download` — kalau dipakai langsung sebagai `href`/`src`, browser akan **langsung download**, bukan preview. Fungsi `driveEmbedUrl(url)` di JS mengekstrak `FILE_ID` dari URL (support format `uc?id=` maupun `file/d/.../view`) dan merakit ulang jadi `https://drive.google.com/file/d/FILE_ID/preview` — format ini yang aman untuk ditampilkan sebagai viewer, baik dibuka di tab baru maupun di-embed iframe.

**Kenapa preview-nya dibuat sebagai toggle (klik untuk muncul), bukan auto-embed:**
- Hindari memuat puluhan/ratusan iframe sekaligus kalau suatu saat panel detail menampilkan banyak dokumen.
- **Bug yang sempat terjadi dan sudah diperbaiki**: iframe sempat dipasang dengan atribut `loading="lazy"`. Chrome punya heuristik yang menganggap iframe sebagai "hidden" kalau elemen itu baru saja muncul di DOM (mis. dalam overlay yang baru dibuka) — akibatnya `src` tidak langsung di-fetch dan terbaca `undefined` dari luar. **Solusi**: iframe dibuat dinamis lewat `document.createElement` di fungsi `toggleDocEmbed()`, ukuran width/height di-set dulu, `src` di-assign terakhir setelah elemen sudah terpasang penuh di DOM — dan **tanpa** `loading="lazy"`. Jangan tambahkan kembali atribut itu ke iframe ini.

**Syarat agar dokumen bisa di-preview:**
File di Google Drive harus di-share sebagai **"Anyone with the link" (Viewer)**. Kalau file masih private, iframe akan gagal load / muncul halaman "minta akses" — ini di luar kendali kode dashboard, harus dicek manual di Drive.

---

## 5. Konvensi Kode (penting sebelum edit)

- **CSV parser** (`parseCSV`) scan karakter demi karakter, bukan `split('\n')` naif — supaya field yang mengandung newline/koma di dalam tanda kutip (misal `kualitatif_feedback`, `list_sesi`) tidak korup. Jangan diganti dengan regex split sederhana.
- **Header CSV** dinormalisasi otomatis ke `snake_case` (`toLowerCase().replace(/\s+/g,'_')`). Jadi nama kolom di Sheets bisa pakai spasi/kapital bebas — yang penting konsisten dengan nama field yang dipakai di JS (`r.nama_kolom`).
- **Field gabungan riwayat** (`detail_job_apply_history`, `detail_nps_history`, `detail_complaint`, `detail_rating_rendah`, `jo_status_pairs`) memakai delimiter custom dari hasil query dbt: antar-record dipisah `~~~`, antar-field dalam satu record dipisah `||`. Parser ada di `parseDetailList()` — kalau format delimiter di SQL berubah, fungsi ini wajib disesuaikan.
- **Variabel CSS** (`--ink`, `--stamp`, `--gold`, dst di `:root`) dipakai konsisten antara dashboard ini dan `waa_leaderboard.html` (tema "travel document" — navy, krem, aksen stempel merah-oranye). Kalau ganti palet, sebaiknya ganti di kedua file biar tetap satu tone.
- **Tidak ada dependency eksternal** selain Google Fonts (Fraunces, Inter, IBM Plex Mono) dan 2 CORS proxy publik untuk fetch CSV. Tidak ada framework JS — vanilla `innerHTML` rendering.

---

## 6. Referensi Singkat: Sumber SQL (dbt mart)

CSV `student` di-generate dari satu query besar yang menggabungkan beberapa base Airtable (Progress Student, Class Record, Job Apply, NPS/Inbound, Session Rating) via `searching_id`. Beberapa CTE kunci yang relevan untuk kolom-kolom baru di dashboard ini:

- **`job_apply_agg`** — agregat riwayat job apply per siswa, termasuk `detail_job_apply_history` (string gabungan semua riwayat apply, dipakai untuk render kartu riwayat JO).
- **`learning_plan_cte`** — ambil 1 link Drive terakhir dari `link_docs__from__3__student_profiling_` (base `appfahxliwrm7qrfm`), hasilnya jadi `learning_plan_url`.
- **`interview_docrev_cte`** — ambil `jumlah_1on1_interview` dan `jumlah_docrev` dari base `appazoma85wc2rntm` (kolom asli: `_1on1_interview_from_reguler`, `_docrev_from_reguler`, tipe `INTEGER`).

Detail lengkap query ada di dbt project (folder mart), bukan disertakan di sini supaya README ini tidak cepat basi setiap kali query berubah. Kalau perlu lihat versi SQL lengkap saat ini, cek riwayat percakapan dengan Claude atau langsung ke dbt repo.

---

## 7. Known Issues / Hal yang Perlu Diperhatikan

- **Limit 2000 baris** di tabel Student — kalau total siswa aktif sudah mendekati angka itu, pertimbangkan menaikkan `LIMIT` atau menambah pagination asli (saat ini belum ada pagination, cuma "tampilkan semua vs sembunyikan").
- **3 CORS proxy publik** (`corsproxy.io`, `allorigins.win`) di luar kendali kita — kalau salah satu down secara permanen, tinggal hapus dari array `PROXIES` atau ganti dengan proxy lain.
- **Cache browser**: tidak ada cache eksplisit di dashboard ini (beda dengan `waa_leaderboard.html` yang punya `localStorage` cache 30 menit). Setiap reload = fetch ulang CSV. Kalau performa jadi masalah, bisa dipertimbangkan menambah cache serupa.
- **Sharing Google Drive** untuk Learning Plan harus dicek manual per file — dashboard tidak bisa mendeteksi atau memperingatkan kalau sebuah file masih private.

---

## 8. Deployment

Dashboard ini di-deploy sebagai static file di GitHub Pages (`vanseann.github.io`). Tidak ada build step — edit langsung file `.html`, commit, push, dan GitHub Pages akan serve versi terbaru (biasanya butuh beberapa menit + hard refresh browser untuk lihat perubahan, karena cache CDN GitHub Pages).
