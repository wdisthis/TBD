# LAPORAN PRAKTIKUM TEKNOLOGI BASIS DATA
## Modul 3 – Index Tuning pada Basis Data XYZ Mart

| | |
|---|---|
| **Kelompok** | 4 |
| **Kelas** | *(isi: RA/RB/RC)* |
| **Tanggal Praktikum** | *(isi: dd-mm-yyyy)* |
| **Judul Praktikum** | Modul 3 – Index Tuning |
| **Dosen PJ Praktikum** | *(isi nama dosen)* |
| **Program Studi** | Sains Data, Fakultas Sains, Institut Teknologi Sumatera (2026) |

**Anggota Kelompok**

| No | Nama | NIM |
|---|---|---|
| 1 | Yollanda Agustina | 124450024 |
| 2 | Fitra | 124450102 |
| 3 | Manuel Frederika | 124450039 |
| 4 | Cerine Sihotang | 124450049 |
| 5 | Rifky Henry Ferdianto | 124450115 |

---

## Daftar Isi
1. [Tujuan](#1-tujuan)
2. [Deskripsi Masalah](#2-deskripsi-masalah)
3. [Dataset dan Struktur Awal](#3-dataset-dan-struktur-awal)
4. [Alat dan Persiapan](#4-alat-dan-persiapan)
5. [Metode Umum Pengerjaan](#5-metode-umum-pengerjaan)
6. [Penerapan Index Tuning (Case A–F)](#6-penerapan-index-tuning)
7. [Verifikasi Seluruh Index](#7-verifikasi-seluruh-index)
8. [Ringkasan Hasil](#8-ringkasan-hasil)
9. [Catatan dan Keterbatasan](#9-catatan-dan-keterbatasan)
10. [Kesimpulan](#10-kesimpulan)

---

## 1. Tujuan
1. Mengidentifikasi query yang lambat akibat *full table scan* pada basis data XYZ Mart.
2. Merancang dan membuat index yang sesuai dengan pola query (equality, prefix `LIKE`, range, join, dan covering).
3. Membandingkan rencana eksekusi (`EXPLAIN`) sebelum dan sesudah index dibuat.
4. Menginterpretasikan dampak index terhadap performa query.

## 2. Deskripsi Masalah
XYZ Mart adalah perusahaan ritel yang menjual produk elektronik, furniture, pakaian, makanan, hingga perlengkapan olahraga. Seiring bertambahnya transaksi, eksekusi query untuk pencarian produk dan analisis penjualan menjadi lambat. Tim bertindak sebagai **Database Administrator (DBA)** yang ditugaskan mengoptimalkan kinerja basis data melalui **index tuning**.

Tanpa index, MySQL melakukan *full table scan* (membaca seluruh baris), sehingga waktu eksekusi naik linear (O(n)) seiring pertumbuhan data.

## 3. Dataset dan Struktur Awal
Database: `xyz_mart`, dengan 7 kategori produk: Elektronik, Furniture, Kecantikan, Makanan, Olahraga, Otomotif, Pakaian.

| Tabel | Jumlah Baris | Kolom |
|---|---|---|
| `products` | 200 | `product_id`, `product_name`, `category`, `price`, `stock` |
| `transactions` | 500 | `transaction_id`, `product_id`, `quantity`, `transaction_date` |

Relasi: `transactions.product_id` → `products.product_id`.

## 4. Alat dan Persiapan
- MySQL / MariaDB beserta klien (MySQL Workbench, DBeaver, atau CLI).
- Dataset dalam bentuk file CSV (diunduh dari tautan Google Drive pada soal): `products.csv` (200 baris) dan `transactions.csv` (500 baris). *(Sesuaikan nama file dengan file yang Anda miliki.)*

**Langkah persiapan**
1. Unduh file CSV dari tautan yang diberikan pada lembar tugas dan simpan di laptop pada folder `C:\FITRA\Project\KULYAH\TBD\Tugas-Kelompok-Modul-3\`.
2. Buat database dan pilih database tersebut:
   ```sql
   CREATE DATABASE IF NOT EXISTS xyz_mart;
   USE xyz_mart;
   ```
3. Buat struktur tabel (DDL awal, tanpa index tambahan selain primary key):
   ```sql
   CREATE TABLE products (
       product_id   INT PRIMARY KEY,
       product_name VARCHAR(100),
       category     VARCHAR(50),
       price        INT,
       stock        INT
   );

   CREATE TABLE transactions (
       transaction_id   INT PRIMARY KEY,
       product_id       INT,
       quantity         INT,
       transaction_date DATE,
       FOREIGN KEY (product_id) REFERENCES products(product_id)
   );
   ```
   > Tipe data di atas adalah perkiraan berdasarkan isi kolom (harga berupa bilangan bulat besar, tanggal berformat `YYYY-MM-DD`). Sesuaikan jika CSV Anda berbeda.
4. Impor data CSV ke tabel. Pilih salah satu cara:

   **Cara 1 – Table Data Import Wizard (MySQL Workbench):** klik kanan tabel → *Table Data Import Wizard* → pilih file CSV → *Use existing table* → cocokkan kolom → *Next* sampai selesai. Impor `products` terlebih dahulu, baru `transactions`.

   **Cara 2 – `LOAD DATA LOCAL INFILE` (CLI/SQL):**
   ```sql
   SET GLOBAL local_infile = 1;

   LOAD DATA LOCAL INFILE 'C:/FITRA/Project/KULYAH/TBD/Tugas-Kelompok-Modul-3/products.csv'
   INTO TABLE products
   FIELDS TERMINATED BY ',' ENCLOSED BY '"'
   LINES TERMINATED BY '\n'
   IGNORE 1 ROWS;

   LOAD DATA LOCAL INFILE 'C:/FITRA/Project/KULYAH/TBD/Tugas-Kelompok-Modul-3/transactions.csv'
   INTO TABLE transactions
   FIELDS TERMINATED BY ',' ENCLOSED BY '"'
   LINES TERMINATED BY '\n'
   IGNORE 1 ROWS;
   ```
   > Gunakan tanda garis miring `/` pada path Windows. Jika file CSV memakai pemisah `;` atau akhir baris `\r\n`, sesuaikan `FIELDS TERMINATED BY` dan `LINES TERMINATED BY`. Klien harus dijalankan dengan opsi `--local-infile=1` bila memakai CLI.
5. Pastikan struktur dan jumlah data sesuai:
   ```sql
   DESCRIBE products;
   DESCRIBE transactions;
   SELECT COUNT(*) FROM products;       -- 200
   SELECT COUNT(*) FROM transactions;   -- 500
   ```
   > 📸 *Tangkapan layar: struktur tabel dan jumlah baris*

## 5. Metode Umum Pengerjaan
Setiap case dikerjakan dengan alur yang sama:

1. **Jalankan query** dan catat hasil (contoh 5 baris pertama).
2. **`EXPLAIN` sebelum index** → catat `type`, `key`, dan `rows`.
3. **Buat index** dengan `CREATE INDEX`.
4. **`EXPLAIN` sesudah index** → bandingkan `type`, `key`, `rows`, dan `Extra`.
5. **Jalankan ulang query** → pastikan hasil tetap konsisten.
6. **Interpretasi** perbedaan rencana eksekusi.

Cara membaca kolom `EXPLAIN`:

| Kolom | Arti |
|---|---|
| `type = ALL` | Full table scan |
| `type = ref` | Pencarian equality lewat index non-unik |
| `type = range` | Pemindaian rentang pada index |
| `key` | Index yang benar-benar dipakai |
| `rows` | Estimasi jumlah baris yang diperiksa |
| `Extra` | Informasi tambahan (`Using filesort`, `Using index`, dst.) |

---

## 6. Penerapan Index Tuning

### CASE A – Pencarian Produk berdasarkan Kategori
**Kebutuhan:** pencarian produk per kategori sering dilakukan (filter equality).

**Query**
```sql
SELECT product_id, product_name, price
FROM products
WHERE category = 'Elektronik';
```

**Langkah**
1. Cek rencana eksekusi sebelum index:
   ```sql
   EXPLAIN SELECT product_id, product_name, price
   FROM products WHERE category = 'Elektronik';
   ```
   **Hasil sebelum:** `type: ALL`, `rows: 200` (full table scan).
2. Buat index:
   ```sql
   CREATE INDEX idx_category ON products(category);
   ```
3. Jalankan `EXPLAIN` ulang.
   **Hasil sesudah:** `type: ref`, `rows: 26` (akses lewat index).

**Contoh hasil query (5 baris pertama, sebelum = sesudah)**

| product_id | product_name | price |
|---|---|---|
| 7 | Televisi 7 | 15669378 |
| 13 | Meja 13 | 18677447 |
| 14 | AC 14 | 11297265 |
| 19 | Sofa 19 | 13511490 |
| 24 | Jam Tangan 24 | 8544792 |

> 📸 *Tangkapan layar: EXPLAIN sebelum, CREATE INDEX, EXPLAIN sesudah, hasil query*

**Pembahasan:** Index pada `category` mengubah pemeriksaan 200 baris menjadi sekitar 26 baris (kategori Elektronik), karena MySQL langsung menuju entri index yang cocok.

---

### CASE B – Pencarian Produk dengan Awalan Nama (`LIKE`)
**Kebutuhan:** pencarian nama produk dengan pola prefix (`'Laptop%'`).

**Query**
```sql
SELECT product_id, product_name, category
FROM products
WHERE product_name LIKE 'Laptop%';
```

**Langkah**
1. `EXPLAIN` sebelum index → `type: ALL`, `rows: 200`.
2. Buat index:
   ```sql
   CREATE INDEX idx_product_name ON products(product_name);
   ```
3. `EXPLAIN` sesudah index → `type: range` (range scan pada index).

**Contoh hasil query (sebelum index, urutan baris sesuai tabel)**

| product_id | product_name | category |
|---|---|---|
| 16 | Laptop 16 | Otomotif |
| 25 | Laptop 25 | Otomotif |
| 26 | Laptop 26 | Olahraga |
| 38 | Laptop 38 | Olahraga |
| 135 | Laptop 135 | Kecantikan |

**Contoh hasil query (sesudah index, urutan mengikuti index nama)**

| product_id | product_name | category |
|---|---|---|
| 135 | Laptop 135 | Kecantikan |
| 150 | Laptop 150 | Olahraga |
| 16 | Laptop 16 | Otomotif |
| 160 | Laptop 160 | Makanan |
| 167 | Laptop 167 | Makanan |

> 📸 *Tangkapan layar: EXPLAIN sebelum dan sesudah, hasil query*

**Pembahasan:** Pola `LIKE 'Laptop%'` (wildcard hanya di belakang) dapat memakai index B-Tree sebagai range scan. Isi data sama, tetapi urutan baris berbeda karena setelah index dipakai hasil terbaca berurutan secara leksikografis (`Laptop 135` < `Laptop 16`). Tanpa `ORDER BY`, urutan hasil memang tidak dijamin.

> ⚠️ Pola `LIKE '%Laptop'` atau `'%Laptop%'` **tidak** dapat memanfaatkan index ini.

---

### CASE C – Filter dan Pengurutan Berdasarkan Rentang Harga
**Kebutuhan:** menampilkan 10 produk dengan harga tertinggi dalam rentang tertentu.

**Query**
```sql
SELECT product_id, product_name, price
FROM products
WHERE price BETWEEN 1000000 AND 5000000
ORDER BY price DESC
LIMIT 10;
```

**Langkah**
1. `EXPLAIN` sebelum index → `type: ALL`, `rows: 200`, disertai *filesort*.
2. Buat index:
   ```sql
   CREATE INDEX idx_price ON products(price);
   ```
3. `EXPLAIN` sesudah index → `type: range`, jumlah baris berkurang, kemungkinan *filesort* hilang.

**Contoh hasil query (sebelum = sesudah)**

| product_id | product_name | price |
|---|---|---|
| 124 | Printer 124 | 4786603 |
| 160 | Laptop 160 | 4763427 |
| 129 | Kursi 129 | 4627760 |
| 165 | Sepatu 165 | 4614021 |
| 175 | AC 175 | 4431861 |

> 📸 *Tangkapan layar: EXPLAIN sebelum dan sesudah, hasil query*

**Pembahasan:** Index pada `price` memungkinkan *range scan* dan karena index sudah terurut, `ORDER BY price DESC` dapat dipenuhi dengan membaca index secara terbalik tanpa operasi sort tambahan; `LIMIT 10` membuat pembacaan berhenti lebih awal.

---

### CASE D – Analisis Transaksi Berdasarkan Periode Tanggal
**Kebutuhan:** analisis transaksi pada periode Juni–Desember 2023.

**Query**
```sql
SELECT transaction_id, product_id, quantity, transaction_date
FROM transactions
WHERE transaction_date BETWEEN '2023-06-01' AND '2023-12-31'
LIMIT 10;
```

**Langkah**
1. `EXPLAIN` sebelum index → `type: ALL` (full table scan pada tabel `transactions`).
2. Buat index:
   ```sql
   CREATE INDEX idx_transaction_date ON transactions(transaction_date);
   ```
3. `EXPLAIN` sesudah index → `type: range`.

**Contoh hasil query (sebelum index)**

| transaction_id | product_id | quantity | transaction_date |
|---|---|---|---|
| 2 | 173 | 5 | 2023-11-05 |
| 6 | 119 | 1 | 2023-06-24 |
| 7 | 173 | 9 | 2023-11-14 |
| 8 | 88 | 2 | 2023-07-11 |
| 9 | 87 | 8 | 2023-06-13 |

**Contoh hasil query (sesudah index, terurut tanggal)**

| transaction_id | product_id | quantity | transaction_date |
|---|---|---|---|
| 72 | 42 | 10 | 2023-06-01 |
| 87 | 191 | 1 | 2023-06-01 |
| 255 | 104 | 2 | 2023-06-02 |
| 73 | 159 | 6 | 2023-06-03 |
| 42 | 156 | 6 | 2023-06-05 |

> 📸 *Tangkapan layar: EXPLAIN sebelum dan sesudah, hasil query*

**Pembahasan:** Index pada `transaction_date` mengubah pemindaian seluruh tabel (O(n)) menjadi pencarian rentang pada struktur B-Tree (O(log n) untuk menemukan batas awal, lalu membaca berurutan). Perbedaan urutan baris pada sampel terjadi karena `LIMIT` tanpa `ORDER BY` mengambil baris sesuai jalur akses yang dipilih optimizer.

---

### CASE E – Analisis JOIN: Pendapatan per Kategori
**Kebutuhan:** menghitung total pendapatan (`quantity × price`) per kategori dengan menggabungkan `transactions` dan `products`.

**Query**
```sql
SELECT p.category, SUM(t.quantity * p.price) AS total_revenue
FROM transactions t
JOIN products p ON t.product_id = p.product_id
GROUP BY p.category;
```

**Langkah**
1. `EXPLAIN` sebelum index → full scan pada tabel (nested loop join tanpa index pendukung di `transactions.product_id`).
2. Buat index:
   ```sql
   CREATE INDEX idx_tx_product ON transactions(product_id);
   CREATE INDEX idx_products_category_price ON products(category, price);
   ```
3. `EXPLAIN` sesudah index → join dapat memakai index pada `product_id`.

**Contoh hasil query (sebelum = sesudah)**

| category | total_revenue |
|---|---|
| Elektronik | 4337240032 |
| Furniture | 2556460242 |
| Kecantikan | 4025000355 |
| Makanan | 2176176714 |
| Olahraga | 5947356132 |

> 📸 *Tangkapan layar: EXPLAIN sebelum dan sesudah, hasil query*

**Pembahasan:** Index pada foreign key `transactions.product_id` mempercepat pencarian pasangan baris saat join. Index komposit `(category, price)` ditujukan untuk membantu pengelompokan berdasarkan kategori sekaligus menyediakan kolom `price` langsung dari index.

---

### CASE F – Top-N Produk Terlaris (Covering Index)
**Kebutuhan:** 10 produk dengan jumlah terjual terbanyak.

**Query**
```sql
SELECT t.product_id, p.product_name, SUM(t.quantity) AS total_sold
FROM transactions t
JOIN products p ON t.product_id = p.product_id
GROUP BY t.product_id
ORDER BY total_sold DESC
LIMIT 10;
```

**Langkah**
1. `EXPLAIN` sebelum index → full scan, `GROUP BY` memakai tabel sementara dan sort.
2. Buat covering index:
   ```sql
   CREATE INDEX idx_tx_product_qty ON transactions(product_id, quantity);
   ```
3. `EXPLAIN` sesudah index → kolom `product_id` dan `quantity` dibaca langsung dari index (`Extra: Using index`), tanpa mengakses baris tabel.

**Contoh hasil query (sebelum = sesudah)**

| product_id | product_name | total_sold |
|---|---|---|
| 34 | Sepatu 34 | 52 |
| 124 | Printer 124 | 43 |
| 149 | Helm 149 | 38 |
| 53 | Smartphone 53 | 37 |
| 87 | Televisi 87 | 36 |

> 📸 *Tangkapan layar: EXPLAIN sebelum dan sesudah, hasil query*

**Pembahasan:** Karena seluruh kolom `transactions` yang dibutuhkan (`product_id`, `quantity`) ada di dalam index, MySQL tidak perlu membaca baris tabel (*index-only scan*). Data juga sudah terkelompok per `product_id`, sehingga agregasi lebih efisien.

---

## 7. Verifikasi Seluruh Index
```sql
SHOW INDEXES FROM products;
SHOW INDEXES FROM transactions;
```
> 📸 *Tangkapan layar: daftar index pada kedua tabel*

| No | Index | Tabel (Kolom) | Tujuan |
|---|---|---|---|
| 1 | `idx_category` | products (category) | Filter equality |
| 2 | `idx_product_name` | products (product_name) | Pencarian prefix `LIKE` |
| 3 | `idx_price` | products (price) | Range query + sorting |
| 4 | `idx_transaction_date` | transactions (transaction_date) | Query periode waktu |
| 5 | `idx_tx_product` | transactions (product_id) | Percepatan JOIN (foreign key) |
| 6 | `idx_products_category_price` | products (category, price) | Komposit untuk join + agregasi |
| 7 | `idx_tx_product_qty` | transactions (product_id, quantity) | Covering index agregasi |

## 8. Ringkasan Hasil

| Case | Jenis Optimasi | Sebelum | Sesudah |
|---|---|---|---|
| A | Equality filter | `ALL`, 200 baris | `ref`, 26 baris |
| B | Prefix `LIKE` | `ALL`, 200 baris | `range` |
| C | Range + `ORDER BY` | `ALL` + filesort | `range` (potensi tanpa filesort) |
| D | Range tanggal | `ALL` | `range` |
| E | JOIN + `GROUP BY` | Full scan + nested loop | Join memakai index `product_id` |
| F | Covering index | Full scan + sort | Index-only untuk `product_id`, `quantity` |

## 9. Catatan dan Keterbatasan
Hal-hal berikut sebaiknya diperhatikan saat melengkapi dan mengumpulkan laporan:

1. **Nilai `rows` pada Case D, E, F.** Dokumen sumber mencatat `rows: 200` pada sebelum-index di Case D–F, padahal tabel `transactions` berisi 500 baris. Nilai ini perlu disesuaikan dengan hasil `EXPLAIN` sebenarnya (kemungkinan salinan dari Case A–C).
2. **Hasil `EXPLAIN` sesudah index** pada beberapa case hanya tertulis "ref atau range". Sebaiknya diganti dengan nilai aktual (`type`, `key`, `rows`, `Extra`) dari tangkapan layar masing-masing.
3. **Ukuran data kecil (200 dan 500 baris).** Optimizer MySQL kadang tetap memilih full scan karena lebih murah pada tabel sekecil ini, sehingga perbedaan waktu eksekusi belum signifikan. Manfaat index terlihat jelas ketika data bertambah besar.
4. **Case C:** penghilangan *filesort* hanya terjadi jika optimizer memilih `idx_price`; periksa kolom `Extra` pada `EXPLAIN`.
5. **Case D:** query di file SQL memakai `LIMIT 10`, sedangkan versi `EXPLAIN` tidak. Samakan agar perbandingan adil.
6. **Case E:** join berdasarkan `p.product_id` biasanya sudah memakai primary key `products`; peran `idx_products_category_price` perlu dikonfirmasi lewat `EXPLAIN` (apakah benar dipakai pada kolom `key`).
7. **Klaim "waktu respons turun 50–80%"** pada file SQL belum didukung pengukuran. Untuk mendukungnya, gunakan `EXPLAIN ANALYZE` (MySQL 8.0.18+) atau ukur waktu eksekusi sebelum dan sesudah index.
8. Pada sampel hasil, entri `175 AC 1754431861` pada dokumen sumber tampaknya `AC 175` dengan harga `4431861`; sudah dirapikan di laporan ini.
9. **Trade-off index:** setiap index menambah ruang penyimpanan dan memperlambat `INSERT`/`UPDATE`/`DELETE`. Index `idx_tx_product` bersifat redundan terhadap `idx_tx_product_qty` (awalan kolom sama); pertimbangkan menghapus salah satu pada lingkungan produksi.

## 10. Kesimpulan
Index tuning diterapkan pada enam kasus query di basis data XYZ Mart dengan tujuh index. Setiap index dirancang untuk pola query tertentu:

1. `idx_category` mempercepat filter berdasarkan kategori.
2. `idx_product_name` mengoptimalkan pencarian prefix `LIKE`.
3. `idx_price` mempercepat query rentang harga dengan pengurutan.
4. `idx_transaction_date` efisien untuk analisis periode.
5. `idx_tx_product` dan `idx_products_category_price` membantu join dan agregasi.
6. `idx_tx_product_qty` menjadi covering index yang menghindari akses tabel.

Secara umum, index mengubah akses dari *full table scan* (O(n)) menjadi *index scan* (O(log n)), dan manfaatnya semakin besar seiring pertumbuhan data. Pemilihan index harus disesuaikan dengan pola query nyata serta mempertimbangkan biaya penulisan data dan penyimpanan.

---

## Lampiran
- **File SQL:** `index_tuning_xyzmart.sql` (seluruh DDL dan query optimasi)
- **Pengumpulan:** `.pdf` + `.sql` dalam satu berkas `nama_kelompok_judul modul.zip`
