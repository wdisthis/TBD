# 📘 Bahan Belajar UTS Teknologi Basis Data (Modul 3–7)

> Disusun dari slide Pertemuan 3, 4, 5, 6, 7. Fokus ke kisi-kisi:
> **M3** Performance Tuning (soal beranak) · **M4** Schema Tuning/Denormalisasi (soal beranak) · **M5** Indexing B-Tree dll · **M6** Query Processing · **M7** SQL → Aljabar Relasional (soal beranak) + Query Equivalen.
>
> Cara pakai: baca **Cheat Sheet** (Bagian 0) dulu, lalu tiap modul, lalu kerjakan **Latihan Soal** (Bagian 8) tanpa melihat jawaban.

---

## 0. CHEAT SHEET (BACA 10 MENIT SEBELUM UJIAN)

| Topik | Yang wajib diingat |
|---|---|
| Bottleneck | 80% waktu di 20% kode. Cari bottleneck dulu, baru eliminasi. Tuning komponen non-bottleneck = hampir tak berpengaruh |
| 3 level tuning | Hardware · Parameter DBMS · Desain (skema, indeks, transaksi) |
| Disk | 1 disk ≈ <100 I/O acak/detik; 1 transaksi = 2 I/O → 1 disk = **50 tps**; butuh n tps → **n/50 disk** |
| 5-minute rule | Halaman acak diakses **> 1x per 5 menit** → taruh di memori |
| 1-minute rule | Data **sekuensial** diakses ≥ 1x per **1 menit** → taruh di memori |
| RAID | RAID 1 = **r + 2w** (write-heavy/OLTP) · RAID 5 = **r + 4w** (read-heavy/DW) |
| Rata-rata throughput | **Harmonic mean**: n / (1/t1 + 1/t2 + … + 1/tn). Bukan rata-rata biasa |
| Materialized View | Hasil query disimpan fisik. Immediate vs Deferred maintenance |
| Set orientation | 1 query `GROUP BY` menggantikan banyak query per baris |
| Lock contention | Kurangi dengan MVCC dan Degree-Two Consistency (cursor stability) |
| TPC | TPC-C (OLTP, standar utama), TPC-H (ad hoc, 22 query, **MV dilarang**), TPC-R (reporting, MV boleh), TPC-W (web), TPC-A/B (usang) |
| Schema tuning | Horizontal split (baris) · Vertical split (kolom) · Denormalisasi (redundant col, derived col, collapsing table, duplicating table) |
| Jaga integritas denormalisasi | **Trigger** (paling baik, tapi mahal) · Application logic (transaksi) · Batch reconciliation |
| Index | Clustered = 1 per tabel, ubah urutan fisik · Non-clustered = banyak, struktur terpisah |
| Dense vs Sparse | Dense: 1 entri per record · Sparse: 1 entri per blok, cari entri terbesar **< K** lalu scan |
| Index suppression | `WHERE total + 3 = 20` ❌ → `WHERE total = 17` ✅ (jangan ada fungsi/hitungan di kolom berindeks) |
| Fase query | Parsing & translation → Optimization → Evaluation (Execution + Fetching) |
| Simbol aljabar | σ select · π project · × kartesian · ⨝ join · ∪ · ∩ · − · ← assign · ρ rename |
| Optimasi heuristik | **Seleksi sedini mungkin, proyeksi sedini mungkin, join selambat mungkin**. Ganti σ(×) jadi ⨝ |
| Set difference | **TIDAK komutatif** (E1 − E2 ≠ E2 − E1) |
| Cost | Disk I/O (blok) jadi komponen terbesar. Rumus nested loop: `nr·bs + br` |

---

# MODUL 3 — DATABASE PERFORMANCE & PERFORMANCE TUNING

## 3.1 Konsep dasar

- **Performance tuning** = menyesuaikan parameter + memilih desain agar kinerja meningkat. Caranya: **identifikasi bottleneck → eliminasi**.
- Tujuan performa: naikkan **throughput** (jumlah data/transaksi yang diproses per waktu) dan minimalkan **kontensi** (rebutan sumber daya).
- "Time is Money": mempercepat sistem = mempercepat pekerjaan karyawan & target perusahaan.

### Faktor yang mempengaruhi performa sistem
1. **Hardware**: disk space, RAM, network, CPU
2. Setting Operating System
3. Parameter Database Server
4. **Database design** (skema, ER diagram, indexing)
5. Indeks pada tabel
6. Pernyataan SQL

### Isu performa database
- Kecepatan hardware baca/tulis data.
- Faktor pengambilan data: **cache** (ukuran objek), **metode akses** (table scan/index), **proses saling bertabrakan**, proses tidak langsung (rollback, archiving).

### Tiga level tuning
| Level | Contoh |
|---|---|
| Hardware | Jumlah disk, RAM, RAID |
| Parameter DBMS | Ukuran buffer, cache level, background process |
| Desain database tingkat tinggi | Skema, indeks, transaksi |

### Parameter yang bisa di-tuning
Hardware · Schema · Indices · Materialized views · Transactions

## 3.2 Bottleneck

- Kinerja biasanya dibatasi oleh **satu atau beberapa komponen** = **bottleneck** (analogi jalan 3 lajur menyempit jadi 1 lajur).
- Bisa di **hardware** (disk sibuk, CPU idle menunggu) atau **software**.
- Contoh: 80% waktu habis di loop kecil, 20% sisanya kode lain → loop kecil = bottleneck.
- Memperbaiki komponen non-bottleneck → pengaruh kecil ke keseluruhan.
- Langkah: **identifikasi bottleneck → eliminasi dengan meningkatkan performa komponen penyebab**. Setelah satu bottleneck hilang, bisa muncul bottleneck baru di komponen lain.
- **Identifikasi**: transaksi meminta layanan ke server process, disk, CPU, lock. Tiap layanan membentuk **antrian**. Transaksi kecil sering menghabiskan waktu **menunggu di antrian (terutama disk I/O)**, bukan mengeksekusi kode.

## 3.3 Hardware Tuning

### Hitungan disk (WAJIB BISA)
- Waktu akses disk konvensional ≈ **10 ms**, kecepatan transfer 25–100 MB/s.
- Satu operasi I/O ≈ 4 KB. Satu disk **< 100 I/O acak/detik**.
- Jika 1 transaksi = 2 I/O → **1 disk = maks 50 transaksi/detik**.
- Butuh **100 tps** → perlu **2 disk** (100/50). Secara umum butuh **n tps → n/50 disk** (data dipartisi/di-stripe ke n/50 disk).

### Strategi menaikkan throughput transaksi
1. Tambah jumlah disk (disk partitioning / **striping**)
2. Kurangi jumlah operasi I/O per transaksi
3. Gunakan **memori** untuk data yang sering diakses

### Five-Minute Rule
- Pertanyaan: data mana yang disimpan di memori?
- Jika halaman diakses **n kali/detik**, biaya disk yang "dihemat" = `n × (price-per-disk-drive / accesses-per-second-per-disk)`.
- Biaya menyimpan halaman di memori = `price-per-MB-of-memory / pages-per-MB-of-memory`.
- **Break-even point** = nilai n saat kedua biaya sama. Jika akses lebih sering dari itu, beli memori lebih hemat.
- **Aturan 5 menit:** halaman yang diakses acak dan dipakai **lebih sering dari 1x per 5 menit** harus disimpan di memori.
  - **Hot data** (lebih sering dari 1x/5 menit) → taruh di memori
  - **Cold data** (lebih jarang) → biarkan di disk

### One-Minute Rule
- Untuk data **sekuensial**, lebih banyak halaman terbaca per detik (asumsi baca 1 MB sekaligus).
- **Aturan 1 menit:** data yang diakses sekuensial **sekali atau lebih dalam 1 menit** harus disimpan di memori.
- Harga disk & memori berubah banyak, tapi **rasionya tidak banyak berubah**, jadi tetap "5 menit dan 1 menit", bukan "1 jam atau 1 detik".

### Pilihan RAID
RAID = menggabungkan beberapa hard disk/SSD jadi satu kesatuan logis untuk meningkatkan kinerja penyimpanan.

| | RAID 1 (Mirroring) | RAID 5 |
|---|---|---|
| Cara kerja | Data ditulis ke **2 disk** (copy identik) | Data dibagi ke beberapa disk dengan **parity tersebar** |
| Read | Bisa lebih cepat (baca dari salah satu disk) | Baik |
| Write | **2 operasi I/O** | **4 operasi I/O** |
| Beban kerja | **r + 2w** | **r + 4w** |
| Pakai saat | **Write-intensive** (OLTP, keuangan, e-commerce); konsistensi & ketersediaan krusial | **Read-intensive** (data warehouse, analisis, backup); data sangat besar & efisiensi kapasitas penting |

> Cara bandingkan: hitung total I/O. Contoh r=100, w=50 → RAID 1 = 100+2·50 = **200 I/O**; RAID 5 = 100+4·50 = **300 I/O**. Karena write banyak, RAID 1 lebih efisien.

## 3.4 Tuning Desain Database

1. **Tuning skema**
   - **Partisi vertikal**: isolasi data yang paling sering diakses. Contoh: `account(account-number, branch-name, balance)` → `(account-number, branch-name)` dan `(account-number, balance)`. branch-name tidak diproses bila tidak dibutuhkan.
   - **Denormalisasi**: gabung tabel untuk mempercepat query. Kekurangan: butuh storage lebih + beban programmer menjaga konsistensi.
2. **Materialized View**
3. **Index**: tujuan mempercepat SELECT, JOIN, WHERE. Tuning index = membuat indeks yang tepat, menghapus indeks berlebih, memilih jenis tepat, clustered vs non-clustered, analisis beban kerja (workload).

### Materialized View (MV)
- Salinan **fisik** hasil query (join/agregasi) yang disimpan di database.
- Cocok untuk query agregat (SUM, COUNT, AVG) atau join tabel besar yang sering dijalankan.
- **Kekurangan:** butuh storage tambahan, butuh maintenance.
- **Immediate maintenance**: setiap update tabel asli, MV langsung diperbarui.
- **Deferred maintenance**: MV diperbarui periodik (harian) atau saat diminta. Menghindari inkonsistensi akibat kesalahan program update.
- Pertimbangan memilih MV: membantu satu tipe transaksi bisa merugikan transaksi lain; tergantung biaya; user sering tak tahu biaya sebenarnya; memilih manual itu lama & membosankan → ada **materialized view selection wizard**.

Contoh (dari slide):

```sql
-- Query biasa (tanpa MV)
SELECT branch_name,
       EXTRACT(MONTH FROM tanggal) AS bulan,
       SUM(jumlah) AS total_transaksi
FROM transaksi
GROUP BY branch_name, EXTRACT(MONTH FROM tanggal);

-- Materialized View
CREATE MATERIALIZED VIEW mv_transaksi_bulanan AS
SELECT branch_name,
       EXTRACT(MONTH FROM tanggal) AS bulan,
       SUM(jumlah) AS total_transaksi
FROM transaksi
GROUP BY branch_name, EXTRACT(MONTH FROM tanggal);
```

## 3.5 Tuning Transaksi

Dua pendekatan dasar:
1. **Meningkatkan Set Orientation**: manipulasi data dalam bentuk kumpulan (set), bukan baris per baris. Satu operasi memproses banyak data.
2. **Mengurangi Lock Contention**: kontensi terjadi saat beberapa transaksi mengakses sumber daya yang sama (tabel/baris) bersamaan → harus menunggu kunci (lock).
3. **Query rewriting** untuk meningkatkan performa.

### Mengurangi overhead komunikasi & pengolahan query
- Gabungkan beberapa query SQL/ODBC/JDBC terpisah menjadi satu **set-oriented query**.
- Gunakan **Stored Procedure**: tidak perlu parse & optimasi berulang → lebih cepat.

```sql
-- Stored procedure (dari slide)
CREATE PROCEDURE CalculateTotalSalary()
BEGIN
  SELECT department_id, SUM(salary)
  FROM employees
  GROUP BY department_id;
END;
```

### Set-oriented query (INI SOAL DI AKHIR SLIDE)

```sql
-- TANPA set orientation (satu per satu)
SELECT SUM(salary) FROM employees WHERE department_id = 1;
SELECT SUM(salary) FROM employees WHERE department_id = 2;
-- ... diulang untuk tiap departemen

-- DENGAN set orientation
SELECT department_id, SUM(salary)
FROM employees
GROUP BY department_id;
```

### Mengurangi Lock Contention
1. **MVCC (Multi-Version Concurrency Control)**: DB menyimpan beberapa versi data yang sama sehingga pembaca tidak memblokir penulis. Contoh: **Oracle snapshots** yang mendukung multi-version 2PL.
2. **Degree-Two Consistency (Cursor Stability)** untuk long transaction: baris yang dibaca cursor dikunci sementara, **dilepas saat cursor pindah**. Lock hanya sementara, bukan sepanjang transaksi.

### Contoh transaksi (konsep ACID/state)
Transfer $50 dari A ke B:

```
Ti: read(A); A := A − 50; write(A); read(B); B := B + 50; write(B).
```

State diagram: `active → partially committed → committed`; `active/partially committed → failed → aborted`.

Jadwal (schedule) T1 & T2: serial (T1 lalu T2 atau sebaliknya) dan concurrent (interleaved) yang tetap serializable.

### Jenis pemrosesan transaksi
- **Offline (batch)**: daily/weekly processing
- **Online (real time)**: **OLTP** dan **OLAP**

## 3.6 Performance Simulation
- Simulasi memakai **model antrian** untuk memprediksi bottleneck dan mengevaluasi dampak tuning **tanpa akses langsung ke sistem asli**.
- Model menyederhanakan detail (mis. service time diperkirakan dengan **rata-rata**, contoh waktu baca disk rata-rata).
- Output: **throughput rata-rata** (tugas selesai per periode waktu) dan **waktu respons rata-rata** (awal sampai selesai).
- Parameter yang bisa diubah: jumlah disk, memori, algoritma, dll.

## 3.7 Performance Benchmarks

- **Benchmark** = kumpulan tugas/beban kerja/query standar untuk mengukur dan membandingkan kinerja.
- **Tujuan:** membandingkan sistem (Oracle vs PostgreSQL vs MySQL); mengukur hasil tuning; menentukan kapasitas sistem sebelum deployment; mengidentifikasi bottleneck.
- Yang diukur: **throughput (tps)**, **waktu respons**, **ketersediaan / mean time to failure**.

### Rata-rata throughput → HARMONIC MEAN
Contoh slide: tipe A = 99 tps, tipe B = 1 tps.
- Salah: (99+1)/2 = 50 tps.
- Benar: satu transaksi tiap jenis butuh 1/99 + 1/1 ≈ **1,01 detik** → 2 transaksi per 1,01 detik = **≈ 1,98 tps**.
- Rumus: **n / (1/t1 + 1/t2 + … + 1/tn)**, dengan ti = throughput jenis i.
- Rumus berlaku jika transaksi dijalankan **bergantian**. Jika bersamaan, throughput bisa lebih rendah karena **interferensi** (lock contention, rebutan sumber daya).

### Kelas aplikasi database
1. **OLTP**: butuh konkurensi tinggi + teknik cepat untuk commit.
2. **Decision support / OLAP**: butuh algoritma evaluasi query & optimasi query yang baik.
- Contoh: Teradata → decision support; Oracle → menyeimbangkan keduanya (snapshot untuk read-only transaction panjang).

### Benchmark suites (TPC = Transaction Processing Council)
| Benchmark | Isi |
|---|---|
| **TPC-A** | Aplikasi teller bank **dengan komunikasi**; OLTP sederhana; sudah usang |
| **TPC-B** | Seperti TPC-A **tanpa komunikasi**; usang |
| **TPC-C** | OLTP kompleks: sistem inventaris (pemesanan, pembayaran, cek stok, pengiriman). **Standar utama OLTP hingga kini** |
| **TPC-D** | Decision support kompleks; digantikan TPC-H dan TPC-R |
| **TPC-H** | (H = ad hoc) 22 query, tekanan agregasi; **MV dilarang**; indeks hanya di PK dan FK |
| **TPC-R** | (R = reporting) seperti TPC-H tapi **tanpa batasan MV dan indeks** |
| **TPC-W** | (W = Web) toko buku web end-to-end, halaman statis + dinamis |

### Pengukuran kinerja TPC
- **Transaksi per detik** dengan batas waktu respons tertentu.
- **Transaksi per detik per dolar** (biaya kepemilikan sistem).
- Ukuran database harus **naik** seiring naiknya tps (mencerminkan dunia nyata).
- Hasil benchmark **wajib diaudit pihak eksternal** → tidak dimanipulasi vendor.
- TPC-H & TPC-R punya dua tes: **Power test** (query & update berurutan, ambil rata-rata waktu) dan **Throughput test** (query & update bersamaan; beberapa stream paralel + satu stream update paralel).

## 3.8 Rangkuman Modul 3 (versi slide)
Hardware (memory/RAM, buffer I/O, CPU, storage) · Skema (fragmentasi, denormalisasi) · Transaksi (konkurensi, serializability) · File (metode akses: indeks, sekuensial, hash, B-tree).

---

# MODUL 4 — SCHEMA TUNING

## 4.1 Mengapa schema tuning?
- Tuning level 3 (desain DB) mencakup skema, indeks, transaksi.
- **Identify schema tuning:** (a) tentukan area yang perlu diperbaiki (normalisasi, denormalisasi, indeks, relasi antar tabel); (b) identifikasi masalah desain (redundansi data, indeks tak efisien, relasi tak optimal, tipe data tak optimal); (c) temukan solusi.

## 4.2 Database Design
- Setelah ER diagram dan skema → diperoleh **Conceptual schema** dan **External schema**.
- **Conceptual schema**: representasi abstrak seluruh struktur data tanpa detail fisik.
- **External schema**: pandangan spesifik data untuk user/aplikasi tertentu.
- Langkah lanjut: **indexing & clustering**, lalu perbaiki conceptual & external schema untuk performa.

## 4.3 Memahami beban kerja (Workload)
- **Query**: query mana paling penting & paling sering dipanggil; tabel mana diakses; atribut apa diambil; atribut dalam `WHERE`/`JOIN` dan **selektivitasnya**.
- **Update**: update mana yang sering; atribut dalam kondisi `WHERE`/`JOIN`.
- **Performance standard**: standar performa yang harus dicapai.

**Keputusan yang diambil:** indeks apa (clustered? hash/tree? dynamic/static? dense/sparse?), ubah conceptual schema?, alternatif lain (normalisasi/denormalisasi)?

## 4.4 Tuning Conceptual Schema
Pemilihan tidak hanya berdasar redundansi, juga **beban kerja**. Opsi:
- **Horizontal decomposition** (bagi tabel per baris)
- **Vertical decomposition** (bagi tabel per kolom)
- Pilih **3NF daripada BCNF** (tergantung beban kerja)
- **Denormalisasi** / tambah kolom
- Jika perubahan dilakukan saat DB sudah dipakai = **schema evolution** (bisa disamarkan dengan **view**).

Ringkasan cara tuning skema:
1. Membagi tabel (horizontal/vertical) → bisa menambah kompleksitas
2. Denormalisasi (tambah kolom redundan, atribut turunan, gabung tabel, salin tabel)
3. Kumpulkan data sejenis ke satu disk untuk data yang sering diakses

## 4.5 Horizontal Splitting (Partitioning)
- Membagi tabel berdasarkan **baris**.
- Dipilih jika: tabel sangat besar (ukuran kecil → baca lebih sedikit) dan pemisahan sesuai **kategori alami** pada data.
- Data disimpan di bagian/media berbeda untuk mempermudah eksekusi.
- **Contoh slide:** tabel produk dibagi per key: **A–G shard** (ARC1, BRK8, BRK9) dan **H–Z shard** (HOS8, WGT4, WGT6). Semua kolom sama, barisnya berbeda.

## 4.6 Vertical Splitting
- Membagi tabel berdasarkan **kolom**: pisahkan data **sering diakses** dari data **jarang diakses**.
- Menjaga desain relasi tetap sesuai normalisasi.
- Lakukan jika: beberapa kolom diakses lebih sering dari lainnya; tabel besar.
- **Contoh slide:** (Key, Name, Description, Stock, Price, LastOrdered) → tabel 1 (Key, Name, Description, Price) dan tabel 2 (Key, Stock, LastOrdered). **Key (PK) ada di kedua tabel** agar bisa digabung kembali.
- **Contoh bank:** `account(account-number, branch-name, balance)` → `account-branch(account-number, branch-name)` + `account-balance(account-number, balance)`. Keduanya BCNF, tapi performa bisa berbeda tergantung query.

| | Horizontal | Vertical |
|---|---|---|
| Dibagi berdasarkan | Baris | Kolom |
| Struktur kolom | Sama di semua partisi | Berbeda, PK diulang |
| Cocok jika | Tabel sangat besar, ada kategori alami | Sebagian kolom lebih sering diakses |

## 4.7 DENORMALISASI (INTI SOAL BERANAK)

### Definisi
Kebalikan normalisasi: **menggabungkan tabel atau menambah redundansi** data untuk meningkatkan kinerja pada kondisi tertentu. Risiko: **redundansi dan potensi inkonsistensi**.

### Tujuan
- Mengurangi kebutuhan **JOIN**
- Mengurangi **foreign key**
- Meningkatkan **kecepatan pembacaan** data
- Meningkatkan efisiensi query tertentu yang sangat sering dijalankan

### Kapan dipakai
- Banyak **query baca**, jarang update
- Butuh akses cepat dalam jumlah besar (**data warehousing / OLAP**)
- Redundansi bukan masalah besar & aplikasi bisa mengelola konsistensi
- Kinerja query lebih diutamakan daripada konsistensi ketat

### Pertimbangan sebelum denormalisasi (checklist slide)
1. Transaksi kritikal apa & respons time yang diharapkan?
2. Seberapa sering transaksi dijalankan?
3. Tabel/kolom apa yang dipakai & berapa baris diakses?
4. Komposisi select/insert/update/delete?
5. Urutan sortir yang biasa dipakai?
6. Ekspektasi konkurensi?
7. Ukuran tabel yang paling sering diakses?
8. Ada proses ringkasan data (summary)?
9. Di mana data disimpan secara fisik?

### Teknik denormalisasi

| Teknik | Penjelasan | Contoh dari slide | Kelebihan | Kekurangan |
|---|---|---|---|---|
| **Adding redundant columns** | Salin kolom dari tabel lain untuk menghapus JOIN yang sering | Tambah kolom `au_lastname` di `titleauthor`, tidak perlu join ke `authors` | Hilang JOIN | Harus sinkronisasi kolom baru; storage lebih |
| **Adding derived columns** | Kolom hasil hitung/agregasi dari kolom lain (di tabel sama atau lain) | Kolom `total_sales` di tabel `title` | Hilangkan JOIN dan hitung agregat berulang | Storage lebih; maintenance lebih |
| **Collapsing tables** | Gabung dua tabel jadi satu | Gabung dua tabel relasi **one-to-one** | Hilangkan JOIN, eksekusi lebih cepat | Hilang pemisahan konseptual data. **Syarat: hubungan 1:1** |
| **Duplicating tables** | Salin subset data yang sering diakses kelompok user tertentu ke tabel terpisah | Subset data untuk satu departemen | Kurangi **kontensi**, akses lebih cepat untuk user tertentu | Harus kelola redundansi |

### Mengelola data yang didenormalisasi (agar tetap konsisten)
1. **Triggers**: update data turunan/duplikat setiap kali data dasar berubah. **Terbaik dari sisi integritas**, tapi bisa **mahal secara performa**.
2. **Application logic**: gunakan **transaksi** di tiap aplikasi yang update data denormalisasi agar perubahan **atomic**. Syarat integritas harus terdokumentasi dan diketahui semua developer.
3. **Batch reconciliation**: jalankan berkala untuk menyamakan kembali data. Dipakai jika **konsistensi 100% tidak wajib setiap saat**.

## 4.8 Normalisasi (1NF → 2NF → 3NF) — contoh Faktur TeraSoft

**Data awal (faktur 035, 30 Juni 2021, kepada Fulan, Jl. ABC 7, HP 0812345):**
10 keping DVD17 "DVD blank" @2000 = 20000; 2 buah KBD09 "Keyboard" @35000 = 70000; Total 90000.

### 1NF — hilangkan atribut multi-nilai/berulang
Satu baris per item barang. Tabel jadi 2 baris (DVD17 dan KBD09), data faktur (Tgl, Nama, Alamat, NoHP, NoFaktur, Total) **diulang** di tiap baris.
- **PK gabungan: (KodeBarang, NoFaktur)**
- **Ketergantungan fungsional penuh** (butuh seluruh PK): `(KodeBarang, NoFaktur) → Jumlah, HargaSatuan`
- **Ketergantungan fungsional parsial** (hanya tergantung sebagian PK):
  - `NoFaktur → Tgl, Nama, Alamat, NoHP`
  - `KodeBarang → Satuan, NamaBarang`

### 2NF — hilangkan ketergantungan parsial
Pecah jadi 3 tabel:
- **Barang** (**KodeBarang**, Satuan, NamaBarang)
- **FakturPelanggan** (**NoFaktur**, Tgl, Nama, Alamat, NoHP)
- **DetailFaktur** (**KodeBarang, NoFaktur**, Jumlah, HargaSatuan)

### 3NF — hilangkan ketergantungan transitif
Di FakturPelanggan: `NoFaktur → NoHP` dan `NoHP → Nama, Alamat` = **transitif**. Pecah:
- **Faktur** (**NoFaktur**, Tgl, NoHP[FK])
- **Pelanggan** (**NoHP**, Nama, Alamat)

> Hasil akhir 3NF: Barang, Faktur, Pelanggan, DetailFaktur.
> **Denormalisasi** kasus ini: gabung kembali Faktur+Pelanggan jadi FakturPelanggan jika query selalu butuh nama/alamat pelanggan bersama faktur (hemat JOIN, tapi Nama/Alamat redundan).

---

# MODUL 5 — INDEXING & QUERY TUNING

## 5.1 Tiga tahap proses query pada DBMS
1. **Parsing** (pemeriksaan & analisis query)
2. **Execution** (eksekusi)
3. **Fetching** (pengambilan hasil)

### SQL Parsing Phase
- Memecah query SQL jadi komponen kecil → ubah ke struktur data → optimasi tanpa mengubah hasil.
- Alur: **Syntax check → Naming (semantic) check → Access right check → Decompose & analyze → Generate access plan → Store access plan di SQL cache**.
- Cek di **shared pool**: jika query sudah ada (sudah pernah di-optimasi & dieksekusi), **hard parsing dilewati**. Jika tidak ada, parser kirim ke **Optimizer**.
- Contoh error:
  - **Syntax check**: `SELECT * FORM dim;` → ERROR 1064 syntax error.
  - **Semantic/naming check**: `SELECT * FROM din;` → ERROR 1146 table doesn't exist.
- **Shared pool** = area memori yang menyimpan informasi query SQL yang sudah dieksekusi (istilah Oracle).

### Query Optimizer
- Komponen DBMS yang menganalisis SQL dan memilih cara **paling efisien** (execution plan optimal).
- Mengumpulkan **statistik** dari data dictionary (jumlah kolom, baris, indeks, ukuran tabel), menyusun ulang Join/Where, dan **partition pruning** (abaikan partisi yang tak relevan).
- Membuat beberapa **query plan** (tiap plan mencakup **path/metode** dan **cost/biaya**). Plan terbaik dikirim ke **execution engine**.
- **Access plan** = rencana hasil optimizer yang menggambarkan bagaimana query dieksekusi secara fisik.

### SQL Execution Phase
Menjalankan access plan: baca disk (I/O), join, filter. **Urutan logis eksekusi klausa SQL:**

`FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT/OFFSET`

> Hafalan: **F-J-W-G-H-S-D-O-L**. Ini sering jadi soal! Perhatikan: `SELECT` dieksekusi **setelah** `WHERE` dan `HAVING`, itu sebabnya alias SELECT tidak bisa dipakai di WHERE.

### SQL Fetching Phase
Database mengirim hasil (sudah difilter/dikelompokkan/diurutkan) ke user/aplikasi. Biasanya memakai **tabel sementara** untuk menampung data sementara.

## 5.2 Indeks

- **Indeks** = struktur data untuk mempercepat pencarian, sorting, agregasi, join. Berisi **index key + pointer**.
- **Index key**: nilai kolom yang diindeks. **Pointer**: alamat/referensi ke baris data di tabel.
- File indeks **lebih kecil** dari file data asli.
- Faktor evaluasi indeks: **access time, insertion time, deletion time, space overhead**.

### Dua tipe utama
| | Ordered index | Hash index |
|---|---|---|
| Cara kerja | Search key tersimpan **terurut** | Fungsi hash memetakan key ke **bucket** |
| Bagus untuk | Pencarian **rentang** (range) & sorting | **Pencocokan tepat** (`=`), misal `employee_id` |

### Jenis ordered index
- **Clustering (primary) index**: file diurutkan sekuensial sesuai search key. Search key tidak harus primary key.
- **Secondary (non-cluster) index**: search key berbeda dari urutan sekuensial file. Tidak mengubah urutan fisik.
- **Index-sequential file**: file terurut sekuensial berdasarkan search key + indeks.
- **Dense index**: **setiap record** punya entri indeks.
- **Sparse index**: hanya **sebagian** nilai key punya entri. Cara cari K: **temukan entri indeks dengan nilai search key terbesar yang < K** (atau ≤ K), lalu **scan file berurutan** mulai dari record yang ditunjuk.

| | Dense | Sparse |
|---|---|---|
| Entri | Satu per record | Satu per blok/sebagian |
| Ukuran | Lebih besar | Lebih kecil |
| Cari | Langsung | Cari entri terdekat lalu scan |
| Syarat | Bisa pada file apa pun | File harus **terurut** (clustering) |

### Data sparsity / cardinality
- Jumlah nilai **berbeda** pada kolom. Sparsity tinggi = banyak nilai berbeda (cocok diindeks). Rendah = sedikit nilai berbeda.
- Jenis index: **Hash, B-tree, Bitmap**.

## 5.3 B-TREE / B+-TREE (WAJIB PAHAM)

- B-tree adalah indeks **paling umum**, cocok untuk berbagai query terutama **range search**. Seimbang (balanced): semua leaf pada kedalaman sama. Bertingkat hingga **n level**.
- **Pada B+-tree (yang dipakai DBMS):** semua nilai key ada di **leaf**; leaf terhubung berurutan (linked list) → range scan cepat. Internal node hanya menyimpan key pemandu.
- **Aturan umum (order n = maks n pointer, maks n−1 key per node):**
  - Leaf: minimal ⌈(n−1)/2⌉ key
  - Internal: minimal ⌈n/2⌉ pointer anak
  - Root: minimal 2 anak (jika bukan leaf)
- **Insert:** cari leaf → sisipkan → jika **overflow**, **split** node jadi dua, naikkan key tengah (leaf: *copy up*; internal: *push up*) ke parent. Split bisa merambat ke atas sampai root (tinggi pohon +1).
- **Delete:** hapus dari leaf → jika **underflow**, **redistribute** (pinjam dari sibling) atau **merge** dengan sibling; bisa merambat ke atas.
- **Pencarian:** dari root turun satu jalur sampai leaf → biaya ≈ **tinggi pohon (+1 untuk baca data)** → O(log n).

### Contoh insert B+-tree (maks 3 key per node, n = 4)
Sisipkan berurutan: **10, 20, 30, 40, 50**

1. Insert 10, 20, 30 → satu leaf: `[10 | 20 | 30]`
2. Insert 40 → overflow → split: `[10 | 20]` dan `[30 | 40]`, parent (root) berisi **30**
   ```
            [30]
           /    \
     [10 20]    [30 40]
   ```
3. Insert 50 → masuk leaf kanan: `[30 | 40 | 50]` (belum overflow)
   ```
            [30]
           /    \
     [10 20]    [30 40 50]
   ```
> Jika keluar soal dengan angka lain, ikuti pola yang sama: urutkan, overflow → split → copy up key pertama leaf kanan ke parent.

### Hash vs B-tree
- **Hash**: sangat cepat untuk `=`, **tidak** bisa range/sort.
- **B-tree**: bagus untuk `=`, `<`, `>`, `BETWEEN`, `ORDER BY`, `MIN/MAX`.

### Bitmap index
Untuk kolom dengan **nilai terbatas** (cardinality rendah), mis. **Female/Male**, status. Satu bitmap per nilai.

## 5.4 Clustered vs Non-Clustered

| | Clustered | Non-clustered |
|---|---|---|
| Urutan fisik data | **Diubah** mengikuti indeks | **Tidak** berubah |
| Jumlah per tabel | **Hanya 1** | **Banyak** (contoh SQL Server 2005: sampai 249) |
| Leaf node | **Data page** itu sendiri | **Pointer/referensi** ke baris data |
| Struktur | Data = indeks | Struktur **terpisah** |
| Tingkat perantara | Lebih sedikit | Lebih banyak |
| Cocok untuk | Pengelompokan, pengurutan, range, kolom pencarian utama | Mengambil satu/sejumlah kecil data |

- `SELECT` tanpa `ORDER BY` pada tabel ber-clustered index → hasil sesuai urutan clustered index.
- Disarankan **setiap tabel punya clustered index**, di kolom yang dipakai cari/kelompok/urut.
- Non-clustered: lebih kecil dari tabel dasar → I/O lebih sedikit; diperbarui seiring perubahan tabel.

## 5.5 INDEX TUNING

### Kapan MEMBUAT indeks ✅
- Kolom di klausa **WHERE / HAVING** sebagai kriteria pencarian
- Kolom di **GROUP BY / ORDER BY**
- Fungsi **MAX / MIN** pada kolom
- Kolom **Primary key** (unique index) dan **Foreign key** (sering untuk join)
- Kolom dengan **banyak nilai unik** (high cardinality, mis. ID)
- Kolom untuk **JOIN** (bahkan jika sudah ada PK/FK, buat indeks tambahan bila perlu)
- Kolom untuk filter nilai/rentang yang sering diakses (mis. `emp_SSN = ...`, `sale_date BETWEEN ...`)

Contoh slide:
```sql
SELECT customer_id, COUNT(*)
FROM Orders
WHERE status = 'Active'
GROUP BY customer_id
ORDER BY order_date;
-- indeks pada: status, customer_id, order_date
```
Contoh join:
```sql
SELECT DISTINCT t.title
FROM pubs..titles AS t
JOIN pubs..sales  AS s ON t.title_id = s.title_id;
-- indeks pada sales.title_id
```

### Kapan MENGHINDARI indeks ❌
- **Tabel kecil**
- Kolom yang filter-nya mengembalikan **persentase tinggi** baris (selektivitas rendah / low sparsity)
- Tabel dengan **batch update besar & sering**
- Kolom dengan **banyak NULL**
- Kolom yang **sering dimanipulasi/diubah** (maintenance indeks besar)
- **Terlalu banyak indeks** (tiap indeks = overhead untuk INSERT/UPDATE/DELETE)

### Index Suppression (indeks tidak terpakai karena kondisi tidak efektif)
```sql
SELECT * FROM mytable WHERE total + 3 = 20;   -- SALAH (indeks pada total tidak dipakai)
SELECT * FROM mytable WHERE total = 17;       -- BENAR
```
Juga: `WHERE YEAR(joining_date) = 2022` ❌ → `WHERE joining_date >= '2022-01-01' AND joining_date < '2023-01-01'` ✅. Dan `LIKE '%john'` ❌ vs `LIKE 'john%'` ✅.

### Tips lain
- **Rebuild indeks secara berkala**, kumpulkan **statistik**.
- Batasi indeks pada kolom yang mengembalikan **sedikit baris**.
- **Bitmap** untuk kolom nilai terbatas.
- **Index tuning wizard**: analisis histori workload dan rekomendasikan indeks (SQL Server: Tools → Wizards → Management → Index Tuning Wizard; pakai **SQL Profiler** untuk workload realistis).

### Fill Factor
- Persentase ruang halaman indeks/tabel yang diisi data. Default SQL Server ≈ **100% penuh** → memicu **page split** dan **fragmentasi**.
- Set **< 100%** → ada ruang kosong untuk insert/update/delete tanpa langsung page split.

### Berapa banyak indeks? (DSS vs OLTP)
- Indeks non-cluster mempercepat SELECT tapi menambah **overhead INSERT/UPDATE/DELETE** → cari **keseimbangan baca vs tulis**.
- **DSS (Decision Support System)**: mengumpulkan & menganalisis data, laporan komprehensif, rekomendasi → **banyak indeks** boleh (baca dominan).
- **OLTP**: **indeks lebih sedikit** (PK, FK, beberapa indeks selektif) agar tulis tidak terbebani. Contoh: perbankan dengan transaksi sering insert & update saldo.

## 5.6 Optimizer Choice & Hints
- **Rule-based optimizer**: aturan & poin preset; tiap operasi punya **biaya tetap**.
- **Cost-based optimizer**: algoritma berdasarkan **statistik** objek; menjumlahkan biaya **processing + I/O + resource** jadi total cost.
- **Optimizer hints**: instruksi khusus dalam SQL untuk **menggantikan perilaku default optimizer** (karena optimizer tak selalu memilih terbaik; statistik bisa sudah tidak relevan).

---

# MODUL 6 — QUERY PROCESSING

## 6.1 Tiga langkah
1. **Parsing & translation**: cek syntax & grammar, validasi relasi; ubah SQL ke bentuk **aljabar relasional / struktur (tree, graph)**.
2. **Optimization**: buat **evaluation plan** optimal (cost rendah).
3. **Evaluation**: pilih plan terbaik & eksekusi → hasil query.

## 6.2 Aljabar Relasional (bahasa prosedural)

| Operator | Simbol | Fungsi | Contoh SQL → RA |
|---|---|---|---|
| Select | σ | Pilih **baris** yang memenuhi predikat | `SELECT * FROM instructor WHERE dept_name='Physics'` → σ dept_name="Physics" (instructor) |
| Project | π | Pilih **kolom**; duplikat dihapus (relasi = himpunan) | `SELECT id, name, salary FROM instructor` → π ID, name, salary (instructor) |
| Cartesian product | × | Kombinasi semua pasangan tuple | `SELECT * FROM instructor, teaches` → instructor × teaches |
| Join | ⨝ | σθ (r × s) | `... JOIN teaches ON instructor.id = teaches.id` → instructor ⨝ instructor.id=teaches.id teaches |
| Union | ∪ | Gabung 2 relasi | `... UNION ...` |
| Intersection | ∩ | Tuple di kedua relasi | `... INTERSECT ...` |
| Difference | − | Ada di r, tidak di s | `... MINUS/EXCEPT ...` |
| Assignment | ← | Simpan ekspresi sementara | Physics ← σ dept_name="Physics" (instructor) |
| Rename | ρ | Ganti nama relasi/atribut | `SELECT * FROM countries nation` → ρ nation (countries) |

- **Syarat ∪, ∩, −**: **arity sama** (jumlah atribut sama) dan **domain atribut kompatibel**.
- Karena atribut `ID` ada di dua relasi pada ×, dibedakan dengan prefix: `instructor.ID`, `teaches.ID`.
- Contoh rename dengan project: `SELECT country_id AS id, country_name AS name FROM countries nation` → π id, name (ρ nation (id, name, region_id) (countries)).

### Contoh union/intersect/difference (course taught Fall 2017 / Spring 2018)
```
Union      : π course_id (σ semester="Fall" ∧ year=2017 (section))  ∪  π course_id (σ semester="Spring" ∧ year=2018 (section))
Intersect  : ...  ∩  ...
Difference : ...  −  ...
```
Contoh assignment: `Physics ← σ dept_name="Physics"(instructor)`; `Music ← σ dept_name="Music"(instructor)`; hasil: `Physics ∪ Music`.

## 6.3 Query Equivalen
Banyak cara menulis query dalam RA; **tidak identik tapi ekivalen** (hasil sama di database apa pun).

- Contoh 1: σ dept="Physics" ∧ salary>90000 (instructor) ≡ σ dept="Physics" (σ salary>90000 (instructor))
- Contoh 2: σ dept="Physics" (instructor ⨝ teaches) ≡ (σ dept="Physics" (instructor)) ⨝ teaches ← **lebih efisien** (seleksi sebelum join).

Contoh skema kuliah (slide 20):
```
Mhs(nim, nama_mhs, tgl_lahir, alamat, kota)
Kuliah(kode_kul, nama_kul, sks, semester)
Nilai(nim, kode_kul, indeks_nilai)

SELECT kode_kul, sks FROM kuliah WHERE semester < 3;
Benar : π kode_kul,sks ( σ semester<3 (kuliah) )
Salah : σ semester<3 ( π kode_kul,sks (kuliah) )   ← atribut semester sudah hilang setelah π!
```
> ⚠️ Slide dosen memakai simbol δ untuk seleksi (σ). Di ujian, tulis **σ**. Dan jangan pernah menaruh σ di luar π jika kolom filter sudah dibuang oleh π.

## 6.4 Estimasi biaya query (statistik katalog)

| Simbol | Arti |
|---|---|
| **nr** | Jumlah record pada tabel r |
| **br** | Jumlah blok yang menampung tabel r |
| **sr** | Ukuran tiap record (byte) |
| **fr** | Blocking factor = jumlah record per blok |
| **V(A, r)** | Jumlah nilai **unik** atribut A di r |
| **SC(A, r)** | Rata-rata jumlah record yang memenuhi kondisi sama dengan pada A |

**Rumus penting:**
- `br = ⌈ nr / fr ⌉`
- `fr = ⌊ ukuran blok / sr ⌋`
- `SC(A, r) = nr / V(A, r)` (untuk atribut non-key; untuk key SC = 1)

Biaya diukur dari **sumber daya**: akses disk (terbesar), waktu CPU, waktu komunikasi.

## 6.5 Algoritma operasi SELEKSI (file scan)

| Kode | Algoritma | Biaya (blok transfer) |
|---|---|---|
| **A1** | Sequential (linear) search | `br` (kalau key equality, rata-rata `br/2`) |
| **A2** | Binary search (file terurut) | `⌈log2(br)⌉` (+ `⌈SC/fr⌉ − 1` bila banyak match) |
| **A3** | Primary index, key, equality | `HTi + 1` |
| **A4** | Primary index, non-key, equality | `HTi + ⌈SC/fr⌉` |
| **A5** | Secondary index, equality | key: `HTi + 1` · non-key: `HTi + SC` |
| **A6** | Primary index, comparison (`<, >, ≤, ≥`) | `HTi + br/2` (kira-kira) |
| **A7** | Secondary index, comparison | `HTi + nr/2` (kira-kira, bisa buruk) |

HTi = tinggi (height) tree indeks.

> Cara ingat: **A1–A2 tanpa indeks · A3–A4 primary index · A5 secondary · A6–A7 untuk perbandingan (range)**.
> Secondary index untuk non-key bisa memakan `SC` I/O acak → bisa **lebih buruk dari linear scan** bila SC besar.

## 6.6 Operasi JOIN — contoh perhitungan dari slide

**Mhs ⨝ Nilai** dengan:
- nmhs = 1200, fmhs = 20 → **bmhs = 1200/20 = 60 blok**
- nnilai = 10.000, fnilai = 50 → **bnilai = 10.000/50 = 200 blok**
- V(kode_kul, nilai) = 50, V(nim, nilai) = 1000

**Hitung biaya (nested loop):**

| Strategi | Rumus | Hasil |
|---|---|---|
| Nested-loop **tuple**, Mhs outer | `nmhs · bnilai + bmhs` = 1200·200 + 60 | **240.060** blok |
| Nested-loop tuple, Nilai outer | `nnilai · bmhs + bnilai` = 10000·60 + 200 | **600.200** blok |
| **Block** nested-loop, Mhs outer | `bmhs · bnilai + bmhs` = 60·200 + 60 | **12.060** blok |
| Block nested-loop, Nilai outer | `bnilai · bmhs + bnilai` = 200·60 + 200 | **12.200** blok |

> Kesimpulan: **block nested-loop** jauh lebih murah; pilih **relasi lebih kecil sebagai outer** (Mhs, 60 blok).
> Jika ada indeks pada `nilai.nim`: SC(nim, nilai) = 10000/1000 = **10 record per mahasiswa** → indexed nested-loop: `bmhs + nmhs · (HTi + 10)` (kira-kira).
> Ukuran hasil join: `nmhs ⨝ nilai` → tiap record nilai cocok dengan 1 mahasiswa (nim = FK) → **10.000 record**.

## 6.7 Operasi lain
- **Eliminasi duplikat**: **urutkan dulu**, record sama jadi berdekatan → mudah dihapus.
- **Proyeksi**: bisa menghasilkan duplikat → hapus duplikat; **jika atribut proyeksi mengandung key, tidak ada duplikat** → tak perlu eliminasi.

## 6.8 Evaluasi ekspresi
Mengevaluasi query bagian per bagian:
- **Materialisasi**: hasil tiap operasi disimpan di **tabel temporer** untuk level berikutnya. Ada biaya tulis/baca temporer.
- **Pipeline**: ganti tabel temporer dengan **pipeline operasi (buffer)** → **hilangkan biaya baca/tulis tabel temporer**.

## 6.9 Transformasi ekspresi relasional
- Query bisa ditulis dengan banyak cara dengan **biaya berbeda** → cari bentuk ekuivalen paling murah.
- **Tugas optimizer:** (1) buat ekspresi logis sama dengan awal, (2) hitung biaya alternatif, (3) ulangi.
- Aturan ekivalensi memastikan **jumlah atribut dan jumlah record hasil sama**.
- Prinsip: seleksi sedini mungkin; join dengan urutan optimal (kecilkan hasil temporer); proyeksi sedini mungkin (sisakan atribut yang diminta + yang perlu join berikutnya).

### Contoh transformasi (slide 29 & 34)
Tampilkan nama_mhs yang mengambil mata kuliah SBD, dengan indeks nilai = A:

```
Awal       : π nama_mhs ( σ nama_kul='SBD' ∧ indeks_nilai='A' ( Kuliah ⨝ (Nilai ⨝ Mhs) ) )
Aturan 6   : ... ( (Kuliah ⨝ Nilai) ⨝ Mhs )                      -- asosiatif join
Aturan 7   : π nama_mhs ( ( σ nama_kul='SBD' ∧ indeks_nilai='A' (Kuliah ⨝ Nilai) ) ⨝ Mhs )   -- σ didorong ke bawah
Aturan 1   : π nama_mhs ( ( σ nama_kul='SBD'(Kuliah) ⨝ σ indeks_nilai='A'(Nilai) ) ⨝ Mhs )   -- σ dipisah per relasi
Aturan 8   : π nama_mhs ( ( π nim ( σ nama_kul='SBD'(Kuliah) ⨝ σ indeks_nilai='A'(Nilai) ) ) ⨝ Mhs ) -- proyeksi sedini mungkin (sisakan nim)
```

## 6.10 Struktur DBMS untuk pemrosesan query
| Komponen | Fungsi |
|---|---|
| **File Manager** | Alokasi disk & struktur data |
| **Buffer Manager** | Transfer data disk ↔ memori utama |
| **Query Parser** | Terjemahkan query ke bahasa mesin |
| **Strategy Selector** | Transformasi query ke bentuk efisien & tentukan strategi eksekusi |
| **Authorization/Integrity Manager** | Batasan integritas & otoritas akses user |
| **Recovery Manager** | Menjaga konsistensi saat kegagalan |
| **Concurrency Controller** | Menjamin interaksi konkuren tanpa konflik |

---

# MODUL 7 — QUERY OPTIMIZATION (SQL → ALJABAR + EKIVALENSI)

## 7.1 Dua cara evaluasi alternatif
1. **Equivalent expressions** (ekspresi sederajat): bentuk berbeda, hasil logis sama, performa beda.
2. **Different algorithms** tiap operasi:
   - Selection (σ) → **Linear scan vs Index scan**
   - Join (⨝) → **Nested loop vs Sort-merge vs Hash join**
   - Projection (π) → **Filter vs Sorting** untuk DISTINCT
   - Aggregation → **Sorting vs Hash vs Index-based**

Contoh equivalent (dari slide):
```sql
-- Awal
SELECT * FROM pelanggan, transaksi
WHERE pelanggan.id = transaksi.pelanggan_id AND transaksi.tanggal > '2024-01-01';

-- Optimasi: filter transaksi dulu, baru join
SELECT * FROM pelanggan
JOIN (SELECT * FROM transaksi WHERE tanggal > '2024-01-01') AS transaksi_filtered
ON pelanggan.id = transaksi_filtered.pelanggan_id;
```

## 7.2 Evaluation Plan
- Menentukan **algoritma tiap operasi** + **koordinasi eksekusi**:
  - **Pipelining**: hasil langsung dikirim ke operasi berikut **tanpa** simpan sementara.
  - **Materialization**: tiap operasi **simpan hasil** dulu sebelum lanjut.

## 7.3 Cost-Based Query Optimization
- Pilih strategi dengan **biaya terendah** (selisih bisa detik vs berhari-hari).
- Langkah: (1) hasilkan ekspresi setara logis, (2) buat alternatif query plan, (3) pilih cost terendah.
- Faktor estimasi: **statistik relasi** (jumlah tuple, jumlah nilai unik), **estimasi ukuran hasil intermediate**, **rumus biaya algoritma**.

### Komponen biaya
1. **Disk I/O** (terbesar; baca index lebih cepat dari full table scan)
2. **Storage cost** (ruang data sementara)
3. **Computation cost** (hitung, sort, filter; hash join > nested loop jika data besar)
4. **Memory usage cost** (kalau memori kurang → pakai disk, lambat)
5. **Communication cost** (sistem terdistribusi)

### Contoh cost-based (slide)
```sql
SELECT E.Lname, E.Address
FROM EMPLOYEE E, DEPARTMENT D
WHERE E.Dno = D.Dnumber AND D.Dname = 'HR';
```
- **Strategi 1 – Full table scan**: scan seluruh EMPLOYEE dan DEPARTMENT lalu join → biaya tinggi.
- **Strategi 2 – Index-based join**: pakai indeks `D.Dname` untuk cari 'HR' dulu, lalu ambil EMPLOYEE yang relevan saja → biaya lebih rendah.

## 7.4 Dua metode optimasi
| | Heuristic (logical) | Cost-based (physical) |
|---|---|---|
| Dasar | Aturan & sifat operator aljabar, query tree | Estimasi biaya fisik |
| Mencoba semua jalur? | **Tidak** | Mengevaluasi berbagai strategi |
| Hasil | Struktur query tree yang diperbaiki | Plan dengan cost terendah |

### Query tree
- **Leaf node** = tabel dasar (base relations)
- **Intermediate node** = hasil sementara dari σ, π, ⨝
- **Root node** = hasil akhir

## 7.5 11 ATURAN TRANSFORMASI (HAFALKAN!)

Notasi: c = kondisi, L = daftar atribut, E = ekspresi/relasi, θ = kondisi join.

| # | Aturan | Rumus |
|---|---|---|
| 1 | **Cascade σ** (pecah seleksi konjungtif) | σc1∧c2 (E) = σc1 (σc2 (E)) |
| 2 | **Komutatif σ** | σc1 (σc2 (E)) = σc2 (σc1 (E)) |
| 3 | **Cascade π** (hanya proyeksi terakhir) | πL1 (πL2 (… πLn (E))) = πL1 (E) |
| 4 | **σ dan π bertukar** (jika c hanya memakai atribut di L) | πL (σc (E)) = σc (πL (E)) |
| 5 | **Komutatif θ-join / ×** | E1 ⨝θ E2 = E2 ⨝θ E1 ; E1 × E2 = E2 × E1 |
| 6 | **σ dengan θ-join**: (a) c1 hanya atribut R | σc1 (R ⨝ S) = (σc1 (R)) ⨝ S |
|   | (b) c1 hanya atribut R, c2 hanya atribut S | σc1∧c2 (R ⨝ S) = (σc1 (R)) ⨝ (σc2 (S)) |
| 7 | **π dengan θ-join** | πL1∪L2 (R ⨝θ S) = (πL1 (R)) ⨝θ (πL2 (S)) — atribut join harus ikut dipertahankan |
| 8 | **Komutatif himpunan** (∪, ∩; **− TIDAK**) | R ∪ S = S ∪ R ; R ∩ S = S ∩ R ; R − S ≠ S − R |
| 9 | **Asosiatif** θ-join, ×, ∪, ∩ | (R ⨝ S) ⨝ T = R ⨝ (S ⨝ T) |
| 10 | **σ dengan operasi himpunan** | σc (R ∪ S) = σc (R) ∪ σc (S) (juga ∩, −) |
| 11 | **π dengan ∪** | πL (R ∪ S) = πL (R) ∪ πL (S) |

> Catatan: di versi slide M6 penomoran sedikit berbeda (daftar rule), tapi isinya sama. Kalau soal menyebut "aturan no. X", cek konteks soal dan sebutkan **nama aturannya** juga supaya aman.

### Bentuk Equivalence (versi kuliah M7)
- **σθ (E1 × E2) = E1 ⨝θ E2** — kartesius + seleksi → theta join (hindari m×n pasangan).
- **σθ1 (E1 ⨝θ2 E2) = E1 ⨝θ1∧θ2 E2** — gabungkan kondisi seleksi ke kondisi join.
- **E1 ⨝θ E2 = E2 ⨝θ E1** — komutatif (beda performa tergantung outer/inner loop).
- **(E1 ⨝ E2) ⨝ E3 = E1 ⨝ (E2 ⨝ E3)** — asosiatif natural join. Theta join: `(E1 ⨝θ1 E2) ⨝θ2∧θ3 E3 = E1 ⨝θ1∧θ3 (E2 ⨝θ2 E3)`, dengan **θ2 hanya atribut E2 dan E3**.
- **σθ1∧θ2 (E1 ⨝θ E2) = (σθ1 (E1)) ⨝θ (σθ2 (E2))** — θ1 hanya atribut E1, θ2 hanya atribut E2.
- **πL1∪L2 (E1 ⨝θ E2) = (πL1 (E1)) ⨝θ (πL2 (E2))** — jika θ memakai atribut dari L1 ∪ L2. Jika θ memakai atribut lain **L3**: `πL1∪L2 (E1 ⨝θ E2) = πL1∪L2 ( (πL1∪L3 (E1)) ⨝θ (πL2∪L4 (E2)) )`.
- **∪ dan ∩ komutatif dan asosiatif.**

## 7.6 Strategi Heuristik (LANGKAH BAKU)

**Prinsip utama:** *kecilkan hasil antara sebelum operasi mahal (JOIN).*

**Algoritma lengkap:**
1. Pecah seleksi majemuk jadi seleksi tunggal (**Rule 1**).
2. Dorong seleksi **turun serendah mungkin** di tree (**Rule 2, 4, 6, 10**).
3. Atur ulang **urutan join** (Rule 9): leaf dengan seleksi paling membatasi dieksekusi dulu; hindari kartesius.
4. **Ganti × + σ menjadi ⨝** bila memungkinkan; jika tidak, lakukan σ segera setelah ×.
5. Dorong **proyeksi** turun (**Rule 3, 4, 7, 11**): sisakan atribut yang diminta **dan** atribut untuk join selanjutnya.
6. Identifikasi sub-tree yang bisa di-**pipeline**.

Ringkas: **σ lebih awal · π lebih awal · ⨝ lebih akhir.**

## 7.7 Langkah mengubah SQL → Aljabar Relasional (UNTUK SOAL BERANAK)

**Template jawaban 5 langkah** (tulis berurutan di lembar jawab!):
1. **Identifikasi relasi** di `FROM` → leaf.
2. Gabung relasi memakai **×** (atau ⨝ jika ada kondisi join).
3. Terapkan **σ** untuk kondisi `WHERE`.
4. Terapkan **π** untuk atribut `SELECT`.
5. (Opsional) Gambar **query tree**, lalu **optimasi heuristik**: dorong σ ke bawah, π ke bawah, ganti × jadi ⨝.

### Contoh A (skema kuliah, jawaban lengkap)

```
Mhs(nim, nama_mhs, tgl_lahir, alamat, kota)
Kuliah(kode_kul, nama_kul, sks, semester)
Nilai(nim, kode_kul, indeks_nilai)
```

**Soal:** Tampilkan nama mahasiswa yang mendapat nilai 'A' pada mata kuliah 'SBD'.

**SQL:**
```sql
SELECT m.nama_mhs
FROM Mhs m, Nilai n, Kuliah k
WHERE m.nim = n.nim
  AND n.kode_kul = k.kode_kul
  AND k.nama_kul = 'SBD'
  AND n.indeks_nilai = 'A';
```

**(a) Aljabar relasional awal (belum optimal):**
```
π nama_mhs ( σ m.nim=n.nim ∧ n.kode_kul=k.kode_kul ∧ nama_kul='SBD' ∧ indeks_nilai='A' ( Mhs × Nilai × Kuliah ) )
```

**(b) Query tree awal (digambar dari bawah ke atas):**
```
              π nama_mhs                      ← root
                  |
        σ (semua kondisi AND)
                  |
                  ×
                / \
               ×   Kuliah                     ← leaf
              / \
           Mhs   Nilai                        ← leaf
```

**(c) Optimasi heuristik (langkah + aturan yang dipakai):**
1. **Rule 1** — pecah σ konjungtif jadi σ tunggal.
2. **Rule 6/10** — dorong σ turun: `σ nama_kul='SBD' (Kuliah)` dan `σ indeks_nilai='A' (Nilai)` langsung di atas leaf.
3. **σ + ×  → ⨝** — kondisi `m.nim=n.nim` dan `n.kode_kul=k.kode_kul` menjadikan × sebagai **join**.
4. **Rule 9** — atur urutan join: kecil dulu (hasil seleksi Kuliah & Nilai), baru Mhs.
5. **Rule 3, 4, 7** — dorong π: `π kode_kul (σ nama_kul='SBD' (Kuliah))`, `π nim, kode_kul (σ indeks_nilai='A' (Nilai))`, `π nim, nama_mhs (Mhs)`.

**Hasil optimal:**
```
π nama_mhs (
   ( π nim, nama_mhs (Mhs) )
   ⨝ nim
   ( π nim ( ( π nim, kode_kul (σ indeks_nilai='A' (Nilai)) )
             ⨝ kode_kul
             ( π kode_kul (σ nama_kul='SBD' (Kuliah)) ) ) )
)
```

**Query tree optimal:**
```
                        π nama_mhs
                            |
                          ⨝ nim
                         /     \
          π nim,nama_mhs        π nim
                |                  |
              Mhs              ⨝ kode_kul
                              /          \
              π nim,kode_kul              π kode_kul
                    |                         |
          σ indeks_nilai='A'          σ nama_kul='SBD'
                    |                         |
                  Nilai                     Kuliah
```

**Kenapa lebih efisien?** Join menerima data yang sudah jauh lebih sedikit baris **dan** kolom → hasil antara kecil → I/O dan memori turun, tidak ada kartesius.

### Contoh B (SQL → RA sederhana, banyak keluar)

| SQL | Aljabar relasional |
|---|---|
| `SELECT * FROM Kuliah WHERE sks = 3` | σ sks=3 (Kuliah) |
| `SELECT nama_mhs, kota FROM Mhs` | π nama_mhs, kota (Mhs) |
| `SELECT nama_mhs FROM Mhs WHERE kota='Bandar Lampung'` | π nama_mhs (σ kota='Bandar Lampung' (Mhs)) |
| `SELECT m.nama_mhs, n.indeks_nilai FROM Mhs m JOIN Nilai n ON m.nim=n.nim` | π nama_mhs, indeks_nilai (Mhs ⨝ Mhs.nim=Nilai.nim Nilai) |
| `SELECT kode_kul FROM Kuliah WHERE semester=1 UNION SELECT kode_kul FROM Kuliah WHERE semester=2` | π kode_kul (σ semester=1 (Kuliah)) ∪ π kode_kul (σ semester=2 (Kuliah)) |
| `... INTERSECT ...` | ... ∩ ... |
| `... EXCEPT/MINUS ...` | ... − ... |
| `SELECT DISTINCT kota FROM Mhs` | π kota (Mhs) (π otomatis hapus duplikat) |
| `SELECT country_id AS id FROM countries` | π id (ρ (id, …) (countries)) |

> **Agregasi/GROUP BY** tidak ada di RA dasar slide; jika keluar, pakai operator **γ** (grouping): `γ dept_id; SUM(salary) (employees)`. Sebutkan bahwa ini ekstensi RA.

## 7.8 Soal Query Equivalen — cara menjawab
Tulis: (1) kedua ekspresi, (2) aturan yang dipakai per langkah, (3) kesimpulan "ekivalen" + mana yang lebih efisien dan alasan.

**Contoh 1:** Apakah `σ dept='Physics' ∧ salary>90000 (instructor)` ekivalen dengan `σ dept='Physics' (σ salary>90000 (instructor))`?
→ **Ya**, Rule 1 (cascade σ). Hasil sama di semua database. Kedua urutan bisa ditukar (Rule 2); pilih seleksi paling selektif dulu.

**Contoh 2:** `σ dept='Physics' (instructor ⨝ teaches)` vs `(σ dept='Physics' (instructor)) ⨝ teaches`.
→ **Ekivalen** (Rule 6a, karena kondisi hanya memakai atribut `instructor`). Bentuk kedua **lebih efisien** karena join memproses lebih sedikit baris.

**Contoh 3 (perangkap):** `πL (σc (E))` boleh ditukar menjadi `σc (πL (E))`?
→ **Hanya jika** atribut di c ⊆ L (Rule 4). Contoh salah: `σ semester<3 (π kode_kul,sks (kuliah))` karena `semester` sudah dibuang.

**Contoh 4 (perangkap):** `R − S` vs `S − R`?
→ **Tidak ekivalen** (set difference tidak komutatif).

## 7.9 Tips SQL praktis (bagian akhir M7)
| Tips | Contoh |
|---|---|
| Pakai indeks dengan tepat | `CREATE INDEX idx_orders_customer_id ON orders(customer_id);` |
| Hindari `SELECT *` | Pilih kolom yang perlu saja |
| Batasi baris | `WHERE` + `LIMIT 50` |
| Jangan fungsi di kolom berindeks | `YEAR(joining_date)=2022` ❌ → rentang tanggal ✅ |
| Join tepat | Filter dulu; `INNER JOIN` bila baris tanpa pasangan tak perlu |
| Hindari **N+1 query** | 1 query list + N query per item (100 user = **101 query**) → ganti 1 `JOIN` |
| `EXISTS` vs `IN` | Hasil sama; `EXISTS` menyatakan maksud "cek keberadaan" lebih jelas; kinerja tergantung optimizer |
| Hindari wildcard di awal `LIKE` | `'%john'` ❌ vs `'john%'` ✅ |
| Gunakan **EXPLAIN** | Lihat execution plan, temukan bottleneck |
| `UNION ALL` vs `UNION` | `UNION ALL` tidak hapus duplikat → lebih cepat bila duplikat boleh |

**Kesimpulan M7:** seleksi sedini mungkin · kurangi kolom via proyeksi · hindari kartesius tak perlu · pilih urutan/pengelompokan join efisien · terapkan aturan ekivalensi.

---

# 8. LATIHAN SOAL + JAWABAN MODEL (SOAL BERANAK)

## 🅼3 — Soal beranak: Performance Tuning

**Soal 1.** (a) Jelaskan apa itu bottleneck. (b) Sebuah sistem OLTP memakai disk dengan <100 I/O acak/detik. Tiap transaksi memerlukan 2 I/O. Berapa tps maksimum satu disk? Jika target 400 tps, berapa disk? (c) Sebutkan 3 strategi menaikkan throughput. (d) Kapan RAID 1 dipilih?

**Jawaban:**
- (a) Bottleneck = komponen (hardware atau software) yang membatasi kinerja keseluruhan sistem; contoh 80% waktu di loop kecil. Tuning dimulai dengan mengidentifikasi lalu mengeliminasinya; memperbaiki komponen non-bottleneck tidak banyak berpengaruh.
- (b) 100 / 2 = **50 tps** per disk. Target 400 tps → 400 / 50 = **8 disk** (data dipartisi/di-stripe).
- (c) Tambah disk (striping), kurangi I/O per transaksi, simpan data yang sering diakses di memori (five-minute rule).
- (d) Write-intensive (OLTP, keuangan, e-commerce) dan saat ketersediaan/konsistensi krusial (RAID 1 = r + 2w, lebih murah untuk write daripada RAID 5 = r + 4w).

**Soal 2.** Workload: 200 read, 100 write per detik. Bandingkan RAID 1 dan RAID 5.
- RAID 1 = 200 + 2·100 = **400 I/O**
- RAID 5 = 200 + 4·100 = **600 I/O**
- → RAID 1 lebih efisien karena write banyak.

**Soal 3.** Sistem menjalankan tipe A = 99 tps dan tipe B = 1 tps. Berapa rata-rata throughput?
- Harmonic mean: 2 / (1/99 + 1/1) = 2 / 1,0101 = **≈ 1,98 tps** (bukan 50).

**Soal 4.** Berikan contoh query set orientation untuk menghitung penjualan per kategori; mengapa lebih efisien?

```sql
-- Tanpa set orientation: satu query per kategori (N query)
SELECT SUM(jumlah) FROM penjualan WHERE kategori = 'Elektronik';
SELECT SUM(jumlah) FROM penjualan WHERE kategori = 'Pakaian';
-- ... dst

-- Dengan set orientation: 1 query
SELECT kategori, SUM(jumlah) AS total_penjualan
FROM penjualan
GROUP BY kategori;
```
**Alasan efisien:** (1) tabel hanya dipindai **sekali**, bukan N kali; (2) **overhead komunikasi** client-server turun (1 round trip vs N); (3) query hanya di-parse & dioptimasi **sekali**; (4) optimizer bisa memakai algoritma set (hash/sort aggregation) dan indeks lebih efektif; (5) lock/transaksi lebih singkat → mengurangi **lock contention**; (6) otomatis mencakup kategori baru tanpa ubah kode.

**Soal 5.** Jelaskan perbedaan Immediate dan Deferred maintenance pada Materialized View, kapan masing-masing dipakai.
- **Immediate**: MV diperbarui langsung tiap tabel asli berubah → data selalu segar, tapi beban update lebih besar (cocok jika update sedikit dan butuh data real-time).
- **Deferred**: MV diperbarui periodik/saat diminta → beban update ringan, tapi data bisa basi (cocok untuk laporan harian/OLAP).

## 🅼4 — Soal beranak: Schema Tuning & Denormalisasi

**Soal 1.** Tabel `Pegawai(id, nama, alamat, foto, cv, gaji, id_dept)` sangat besar. Aplikasi 95% hanya membaca `id, nama, id_dept`. (a) Teknik apa yang tepat? (b) Gambarkan hasilnya. (c) Syarat/risikonya?

**Jawaban:**
- (a) **Vertical splitting** (sebagian kolom lebih sering diakses; tabel besar).
- (b) `Pegawai_Inti(id, nama, id_dept)` dan `Pegawai_Detail(id, alamat, foto, cv, gaji)`; **id (PK) ada di keduanya**.
- (c) Tetap sesuai normalisasi; query yang butuh kedua bagian perlu JOIN; tambah kompleksitas.

**Soal 2.** Tabel `Transaksi` 500 juta baris, hampir semua query per tahun. Teknik? → **Horizontal splitting/partitioning** per tahun (kategori alami); query hanya menyentuh partisi relevan (**partition pruning**); data bisa di media berbeda.

**Soal 3.** Sistem laporan penjualan sering join `Penjualan` dan `Produk` hanya untuk mengambil `nama_produk`, dan menghitung total penjualan per produk berulang.
- **Redundant column**: tambah `nama_produk` di `Penjualan` → hilangkan JOIN.
- **Derived column**: tambah `total_penjualan` di `Produk` → hilangkan agregasi berulang.
- **Risiko:** storage lebih, inkonsistensi bila `nama_produk` berubah atau penjualan baru tidak memperbarui total.
- **Solusi menjaga integritas:** **trigger** (terbaik integritas, tapi mahal), **application logic** dengan transaksi (atomic), **batch reconciliation** berkala (jika konsistensi 100% tidak wajib).

**Soal 4.** Kapan denormalisasi layak dan kapan tidak?
- Layak: banyak baca, jarang update; data warehouse/OLAP; butuh akses cepat; redundansi dapat dikelola.
- Tidak layak: banyak update/OLTP tulis-berat; konsistensi ketat; storage terbatas.

**Soal 5.** Collapsing tables: syarat & contoh? → Syarat **hubungan 1:1**. Contoh `Mahasiswa(nim, nama)` dan `Mahasiswa_Detail(nim, alamat, hp)` digabung menjadi satu tabel. Untung: hilang JOIN; rugi: hilang pemisahan konseptual.

**Soal 6.** Duplicating tables: contoh & kelebihan? → Salin subset (mis. data penjualan bulan berjalan) ke tabel khusus tim laporan → kurangi **kontensi**, akses lebih cepat; kekurangan: kelola redundansi/sinkronisasi.

**Soal 7.** Normalisasi faktur ke 3NF → lihat Bagian 4.8 (Barang, Faktur, Pelanggan, DetailFaktur).

**Soal 8.** Mengapa memilih 3NF dibanding BCNF kadang lebih baik? → Pemilihan tergantung **beban kerja**: BCNF bisa mengorbankan dependency-preservation sehingga menambah JOIN/biaya pengecekan, 3NF menjaga dependensi dengan lebih sedikit dekomposisi.

## 🅼5 — Indexing

**Soal 1.** Bedakan clustered vs non-clustered. → Lihat tabel 5.4. Ringkas: clustered mengubah urutan fisik, hanya 1 per tabel, leaf = data page; non-clustered struktur terpisah, banyak, leaf = pointer.

**Soal 2.** Bedakan dense dan sparse index; bagaimana mencari K di sparse index? → Dense: 1 entri/record; sparse: entri hanya untuk sebagian. Cari entri dengan key terbesar < K → buka record yang ditunjuk → scan sekuensial sampai ketemu.

**Soal 3.** Untuk query berikut, kolom mana diindeks dan mengapa?
```sql
SELECT customer_id, COUNT(*) FROM Orders
WHERE status='Active' GROUP BY customer_id ORDER BY order_date;
```
→ `status` (WHERE), `customer_id` (GROUP BY), `order_date` (ORDER BY). Catatan: `status` bernilai sedikit → pertimbangkan **bitmap**/selektivitas; di OLTP cukup selektif saja.

**Soal 4.** Perbaiki query: `SELECT * FROM mytable WHERE total + 3 = 20;` → `WHERE total = 17` (index suppression; jangan hitung di kolom berindeks).

**Soal 5.** Kapan indeks dihindari? → Tabel kecil, filter mengembalikan banyak baris, batch update sering, banyak NULL, kolom sering berubah, terlalu banyak indeks.

**Soal 6.** Sisipkan 10, 20, 30, 40, 50 ke B+-tree (maks 3 key/node) → lihat 5.3.

**Soal 7.** Urutan eksekusi klausa SQL? → FROM, JOIN, WHERE, GROUP BY, HAVING, SELECT, DISTINCT, ORDER BY, LIMIT/OFFSET.

**Soal 8.** Rule-based vs cost-based optimizer? → Rule-based: aturan preset, biaya tetap per operasi. Cost-based: statistik objek, total cost = processing + I/O + resource.

**Soal 9.** Apa itu fill factor dan mengapa <100%? → Persentase pengisian halaman; <100% menyisakan ruang agar insert/update tidak memicu page split & fragmentasi.

**Soal 10.** OLTP vs DSS dalam jumlah indeks? → OLTP: sedikit (PK, FK, beberapa selektif) agar tulis cepat; DSS: banyak indeks (baca dominan).

## 🅼6 — Query Processing

**Soal 1.** Tiga tahap query processing. → Parsing & translation; Optimization; Evaluation.

**Soal 2.** Hitung biaya block nested-loop Mhs ⨝ Nilai (nmhs=1200, fmhs=20; nnilai=10000, fnilai=50) dengan Mhs sebagai outer.
- bmhs = 60, bnilai = 200 → `60·200 + 60 = 12.060` blok. Dengan Nilai outer = `200·60 + 200 = 12.200`.

**Soal 3.** Materialisasi vs pipeline? → Materialisasi simpan hasil di tabel temporer (ada biaya baca/tulis). Pipeline kirim hasil langsung ke operasi berikut via buffer → biaya temporer hilang.

**Soal 4.** Mengapa `σ semester<3 (π kode_kul,sks (kuliah))` salah? → Atribut `semester` sudah dibuang oleh π sebelum σ dijalankan; benar: `π kode_kul,sks (σ semester<3 (kuliah))`.

**Soal 5.** Algoritma selection A1–A7 dan kapan secondary index buruk? → Lihat 6.5; A7/A5 non-key dengan SC besar bisa lebih buruk daripada linear scan.

**Soal 6.** Sebutkan 5 komponen DBMS untuk pemrosesan query beserta fungsinya. → Lihat 6.10.

**Soal 7.** Eliminasi duplikat? → Sort dulu, duplikat jadi berdekatan, hapus. Tidak perlu jika atribut proyeksi mengandung key.

**Soal 8.** Statistik apa yang dipakai estimasi biaya? → nr, br, sr, fr, V(A,r), SC(A,r).

## 🅼7 — SQL → Aljabar Relasional (beranak) + Ekivalensi

**Soal 1.** Diberikan:
```
Employee(eid, nama, gaji, did)
Department(did, dname, lokasi)
Project(pid, pname, did)
```
**SQL:** Tampilkan nama pegawai yang bekerja di departemen 'HR' dengan gaji > 5.000.000.
```sql
SELECT e.nama
FROM Employee e, Department d
WHERE e.did = d.did AND d.dname = 'HR' AND e.gaji > 5000000;
```
- **(a) RA awal:** π nama ( σ e.did=d.did ∧ dname='HR' ∧ gaji>5000000 ( Employee × Department ) )
- **(b) Setelah ×+σ → ⨝ (Rule: σθ(E1×E2) = E1⨝θE2):** π nama ( σ dname='HR' ∧ gaji>5000000 ( Employee ⨝ Employee.did=Department.did Department ) )
- **(c) Dorong σ (Rule 1, 6b):** π nama ( σ gaji>5000000 (Employee) ⨝ did σ dname='HR' (Department) )
- **(d) Dorong π (Rule 7, atribut join `did` dipertahankan):** π nama ( π nama,did (σ gaji>5000000 (Employee)) ⨝ did π did (σ dname='HR' (Department)) )
- **Query tree:**
```
            π nama
               |
            ⨝ did
           /      \
 π nama,did        π did
      |               |
σ gaji>5000000   σ dname='HR'
      |               |
  Employee        Department
```
- **Mengapa lebih baik?** Filter dilakukan sebelum join sehingga baris & kolom yang di-join jauh lebih sedikit; tidak ada kartesius.

**Soal 2.** Tuliskan RA untuk "mahasiswa yang mengambil MK semester 1 tetapi tidak MK semester 2".
```
π kode_kul (σ semester=1 (Kuliah)) − π kode_kul (σ semester=2 (Kuliah))
```
Syarat: arity dan domain kompatibel. (Karena − tidak komutatif, urutan operand menentukan hasil.)

**Soal 3.** Buktikan ekivalensi `σ c1∧c2 (R ⨝ S)` dengan `(σ c1 (R)) ⨝ (σ c2 (S))`. Syarat? → Berlaku jika c1 hanya memakai atribut R dan c2 hanya memakai atribut S (Rule 6b). Alasan: seleksi per relasi tidak mempengaruhi pasangan join yang sah, hasil akhir sama tetapi join menerima input lebih kecil.

**Soal 4.** Sebutkan prinsip optimasi heuristik. → Seleksi sedini mungkin, proyeksi sedini mungkin, join selambat mungkin; hindari kartesius; pilih urutan join yang menghasilkan hasil antara terkecil.

**Soal 5.** Perbaiki query dengan prinsip optimasi M7:
- `SELECT * FROM products;` → `SELECT product_id, product_name, price FROM products;`
- `WHERE YEAR(joining_date)=2022` → `WHERE joining_date >= '2022-01-01' AND joining_date < '2023-01-01'`
- `LIKE '%john'` → `LIKE 'john%'` (bila memungkinkan)
- N+1 query → 1 query `JOIN`
- `UNION` → `UNION ALL` bila duplikat boleh

**Soal 6.** Bandingkan Full Table Scan vs Index-Based Join pada query `EMPLOYEE E, DEPARTMENT D WHERE E.Dno = D.Dnumber AND D.Dname='HR'`. → Full scan memindai kedua tabel (I/O tinggi). Index-based: indeks `Dname` temukan 'HR' dulu, lalu ambil hanya EMPLOYEE terkait (I/O jauh lebih rendah). Optimizer cost-based memilih yang biaya estimasinya terendah.

---

# 9. 🧠 TRIK MENJAWAB UNTUK NILAI MAKSIMAL

1. **Soal beranak = nilai per sub-soal.** Jawab **semua** sub-soal; tulis langkah, jangan loncat ke hasil.
2. **Definisi + contoh + alasan.** Untuk soal "jelaskan", tulis 1 kalimat definisi, 1 contoh konkret, 1 alasan manfaat/kekurangan.
3. **Selalu sebut trade-off.** Denormalisasi: cepat baca ↔ redundansi & maintenance. Indeks: cepat SELECT ↔ lambat INSERT/UPDATE/DELETE. MV: cepat query ↔ storage + maintenance.
4. **Soal SQL→RA:** tulis versi awal **dan** versi teroptimasi, sebutkan aturan yang dipakai, gambar query tree.
5. **Soal perhitungan:** tulis rumus, substitusi angka, hasil + satuan (tps, blok, disk).
6. **Perangkap klasik:**
   - Harmonic mean, bukan rata-rata biasa
   - Set difference tidak komutatif
   - σ/π tidak boleh ditukar jika atribut dibuang/dibutuhkan
   - `WHERE col + 3 = x` mematikan indeks
   - Clustered index hanya 1 per tabel
   - TPC-H **melarang** MV, TPC-R **membolehkan**
7. **Konsisten pakai σ** (bukan δ) untuk select.

---

## ✅ Checklist Terakhir Sebelum Tidur
- [ ] Hafal 11 aturan transformasi + contoh tiap aturan
- [ ] Bisa ubah 1 SQL 3 tabel jadi RA + query tree + optimasi
- [ ] Bisa hitung disk (n/50), RAID (r+2w vs r+4w), harmonic mean, cost nested-loop
- [ ] Bisa sebut 4 teknik denormalisasi + cara menjaga integritas (trigger/app logic/batch)
- [ ] Bisa insert ke B+-tree tanpa lupa split
- [ ] Hafal urutan eksekusi klausa SQL (F-J-W-G-H-S-D-O-L)
- [ ] Bedakan clustered/non-clustered, dense/sparse, hash/B-tree, rule-based/cost-based

*Semangat, Fitra! 💪*
