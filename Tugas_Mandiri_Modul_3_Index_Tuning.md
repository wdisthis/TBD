# LEMBAR KERJA PRAKTIKUM TEKNOLOGI BASIS DATA

| | |
|---|---|
| **Nama mahasiswa** | (isi nama) |
| **NIM** | (isi NIM) |
| **Kelas** | (RA / RB / RC) |
| **Tanggal Praktikum** | (dd-mm-yyyy) |
| **Judul Praktikum** | Modul 3 – Index Tunning |
| **Dosen PJ Praktikum** | (isi nama dosen) |

---

# TUGAS MANDIRI: Index Tuning pada Database ABC Retail

## 0. Studi Kasus

Sebagai DBA di ABC Retail, tugasnya adalah melakukan *index tuning* pada database `praktikum_index_tuning` agar query yang sering digunakan berjalan lebih cepat.

**Tabel Products**

| product_id | product_name | category   | price      | stock |
|-----------:|--------------|------------|-----------:|------:|
| 1          | Laptop       | Elektronik | 12.000.000 | 10    |
| 2          | Printer      | Elektronik | 2.500.000  | 20    |
| 3          | Meja         | Furniture  | 1.500.000  | 15    |
| 4          | Kursi        | Furniture  | 700.000    | 30    |

**Tabel Transactions**

| transaction_id | product_id | quantity | transaction_date |
|---------------:|-----------:|---------:|------------------|
| 1              | 1          | 2        | 2023-02-15       |
| 2              | 2          | 1        | 2023-02-20       |
| 3              | 3          | 3        | 2023-03-10       |
| 4              | 4          | 4        | 2023-04-25       |

### Query yang diasumsikan sering dipakai

| Kode | Kebutuhan bisnis | Query |
|------|------------------|-------|
| Q1 | Cari produk berdasarkan kategori | `WHERE category = 'Elektronik'` |
| Q2 | Laporan transaksi per produk | `JOIN` Products dan Transactions |
| Q3 | Laporan transaksi per rentang tanggal | `WHERE transaction_date BETWEEN ...` |
| Q4 | Produk satu kategori diurutkan harga | `WHERE category = ... ORDER BY price DESC` |
| Q5 | Total kuantitas terjual per produk | `GROUP BY product_id` + `SUM(quantity)` |

> **Catatan pengumpulan:** kumpulkan dalam bentuk `nim_nama_modul_kelas.pdf` dan file `.sql` ke Google Classroom. Untuk setiap langkah di bawah, tempel **screenshot kode + hasil** pada kolom yang disediakan.

---

## 1. Persiapan Database

### Langkah 1.1 – Jalankan Apache dan MySQL

Buka XAMPP Control Panel, klik **Start** pada Apache dan MySQL sampai statusnya *Running* (hijau). Lalu buka `http://localhost/phpmyadmin`.

📸 *Screenshot: XAMPP Control Panel + phpMyAdmin*

### Langkah 1.2 – Buat database

```sql
CREATE DATABASE praktikum_index_tuning;
USE praktikum_index_tuning;
```

**Fungsi kode:** `CREATE DATABASE` membuat database baru, `USE` memilih database tersebut sebagai database aktif.

### Langkah 1.3 – Buat tabel

```sql
CREATE TABLE Products (
    product_id   INT PRIMARY KEY AUTO_INCREMENT,
    product_name VARCHAR(100),
    category     VARCHAR(50),
    price        DECIMAL(12, 2),
    stock        INT
);

CREATE TABLE Transactions (
    transaction_id   INT PRIMARY KEY AUTO_INCREMENT,
    product_id       INT,
    quantity         INT,
    transaction_date DATE
);
```

**Fungsi kode:** `PRIMARY KEY` otomatis membuat indeks unik (PRIMARY) pada `product_id` dan `transaction_id`.

> **Catatan:** `FOREIGN KEY` sengaja tidak dipasang pada `Transactions.product_id`. Pada InnoDB, foreign key otomatis membuat indeks pada kolomnya, sehingga perbandingan "sebelum dan sesudah indeks" tidak akan terlihat jelas.

### Langkah 1.4 – Isi data

```sql
INSERT INTO Products (product_name, category, price, stock) VALUES
('Laptop',  'Elektronik', 12000000, 10),
('Printer', 'Elektronik',  2500000, 20),
('Meja',    'Furniture',   1500000, 15),
('Kursi',   'Furniture',    700000, 30);

INSERT INTO Transactions (product_id, quantity, transaction_date) VALUES
(1, 2, '2023-02-15'),
(2, 1, '2023-02-20'),
(3, 3, '2023-03-10'),
(4, 4, '2023-04-25');
```

### Langkah 1.5 – Verifikasi data dan indeks awal

```sql
SELECT * FROM Products;
SELECT * FROM Transactions;

SHOW INDEX FROM Products;
SHOW INDEX FROM Transactions;
```

**Analisis:** pada kondisi awal, hanya ada indeks `PRIMARY` pada masing-masing tabel. Kolom `category`, `price`, `product_id` (di Transactions), dan `transaction_date` **belum memiliki indeks**, sehingga query yang memfilter kolom-kolom tersebut berpotensi melakukan *full table scan*.

📸 *Screenshot: hasil SELECT dan SHOW INDEX*

---

## 2. Analisis Query SEBELUM Indeks (Baseline)

Aktifkan *profiling* agar waktu eksekusi tiap query dapat dicatat:

```sql
SET profiling = 1;
```

### Q1 – Filter kategori

```sql
SELECT * FROM Products WHERE category = 'Elektronik';
EXPLAIN SELECT * FROM Products WHERE category = 'Elektronik';
```

### Q2 – JOIN produk dan transaksi

```sql
SELECT p.product_name, t.quantity, t.transaction_date
FROM Transactions t
JOIN Products p ON t.product_id = p.product_id;

EXPLAIN SELECT p.product_name, t.quantity, t.transaction_date
FROM Transactions t
JOIN Products p ON t.product_id = p.product_id;
```

### Q3 – Rentang tanggal

```sql
SELECT * FROM Transactions
WHERE transaction_date BETWEEN '2023-02-01' AND '2023-03-31';

EXPLAIN SELECT * FROM Transactions
WHERE transaction_date BETWEEN '2023-02-01' AND '2023-03-31';
```

### Q4 – Filter kategori + urut harga

```sql
SELECT * FROM Products
WHERE category = 'Elektronik'
ORDER BY price DESC;

EXPLAIN SELECT * FROM Products
WHERE category = 'Elektronik'
ORDER BY price DESC;
```

### Q5 – Agregasi per produk

```sql
SELECT product_id, SUM(quantity) AS total_qty
FROM Transactions
GROUP BY product_id;

EXPLAIN SELECT product_id, SUM(quantity) AS total_qty
FROM Transactions
GROUP BY product_id;
```

### Catat waktu eksekusi baseline

```sql
SHOW PROFILES;
```

### Cara membaca hasil EXPLAIN

| Kolom | Arti |
|-------|------|
| `type` | Jenis akses: `ALL` = full table scan, `ref` / `range` / `index` = memakai indeks |
| `possible_keys` | Indeks yang *mungkin* dipakai |
| `key` | Indeks yang *benar-benar* dipakai |
| `rows` | Perkiraan jumlah baris yang dipindai |
| `Extra` | Informasi tambahan, mis. `Using where`, `Using filesort`, `Using index` |

### Hasil baseline yang diharapkan

| Query | type | key | rows | Extra |
|-------|------|-----|-----:|-------|
| Q1 | ALL | NULL | 4 | Using where |
| Q2 | ALL (Transactions) + eq_ref (Products, via PRIMARY) | NULL / PRIMARY | 4 / 1 | – |
| Q3 | ALL | NULL | 4 | Using where |
| Q4 | ALL | NULL | 4 | Using where; **Using filesort** |
| Q5 | ALL | NULL | 4 | Using temporary |

**Analisis baseline:**

- Q1, Q3, Q4, Q5 memakai `type = ALL` karena kolom yang difilter/dikelompokkan belum diindeks, sehingga seluruh baris dibaca.
- Q2: tabel `Transactions` dipindai penuh, lalu tiap baris dicocokkan ke `Products` lewat PRIMARY KEY. Sisi `Transactions.product_id` belum punya indeks.
- Q4 menampilkan `Using filesort`: hasil harus diurutkan secara terpisah karena tidak ada indeks yang menyimpan urutan `price`.
- Q5 menampilkan `Using temporary`: tabel sementara dibuat untuk proses `GROUP BY`.

📸 *Screenshot: hasil EXPLAIN Q1–Q5 dan SHOW PROFILES (sebelum indeks)*

> Hasil di tabel adalah gambaran umum. Isi dengan hasil EXPLAIN dan waktu eksekusi aktual dari komputer masing-masing.

---

## 3. Membuat Indeks

### Langkah 3.1 – Indeks untuk filter kategori (Q1)

```sql
CREATE INDEX idx_products_category ON Products(category);
```

**Alasan:** `category` sering muncul di klausa `WHERE`.

### Langkah 3.2 – Indeks untuk JOIN (Q2)

```sql
CREATE INDEX idx_trx_product_id ON Transactions(product_id);
```

**Alasan:** `product_id` di `Transactions` adalah kolom penghubung JOIN, jadi harus diindeks agar pencarian baris yang cocok tidak memindai seluruh tabel.

### Langkah 3.3 – Indeks untuk rentang tanggal (Q3)

```sql
CREATE INDEX idx_trx_date ON Transactions(transaction_date);
```

**Alasan:** indeks berurutan (B-Tree) sangat efektif untuk kondisi rentang seperti `BETWEEN`, `>`, `<`.

### Langkah 3.4 – Indeks komposit untuk filter + urut (Q4)

```sql
CREATE INDEX idx_products_cat_price ON Products(category, price);
```

**Alasan:** kolom pertama (`category`) melayani `WHERE`, kolom kedua (`price`) sudah terurut di dalam tiap kategori, sehingga `ORDER BY price` tidak perlu *filesort*. Urutan kolom penting: kolom yang difilter dengan `=` diletakkan lebih dulu.

### Langkah 3.5 – Indeks covering untuk agregasi (Q5)

```sql
CREATE INDEX idx_trx_product_qty ON Transactions(product_id, quantity);
```

**Alasan:** `GROUP BY product_id` dan `SUM(quantity)` hanya membutuhkan dua kolom itu. Seluruh data dapat dibaca dari indeks tanpa menyentuh tabel (*covering index*).

### Langkah 3.6 – Verifikasi indeks

```sql
SHOW INDEX FROM Products;
SHOW INDEX FROM Transactions;
```

**Hasil yang diharapkan:**

| Tabel | Indeks |
|-------|--------|
| Products | PRIMARY, idx_products_category, idx_products_cat_price |
| Transactions | PRIMARY, idx_trx_product_id, idx_trx_date, idx_trx_product_qty |

📸 *Screenshot: hasil SHOW INDEX*

---

## 4. Analisis Query SESUDAH Indeks

Perbarui statistik tabel agar optimizer punya informasi terbaru:

```sql
ANALYZE TABLE Products, Transactions;
```

Jalankan ulang Q1–Q5 beserta `EXPLAIN` (kode sama seperti Bagian 2), lalu `SHOW PROFILES;`.

### Hasil yang diharapkan (pada data besar, lihat Bagian 5)

| Query | type | key yang dipakai | Extra |
|-------|------|------------------|-------|
| Q1 | ref | idx_products_category | – |
| Q2 | ref / eq_ref | idx_trx_product_id, PRIMARY | – |
| Q3 | range | idx_trx_date | Using index condition |
| Q4 | ref | idx_products_cat_price | **tanpa** Using filesort |
| Q5 | index | idx_trx_product_qty | Using index |

### Mengapa pada 4 baris hasilnya bisa tetap `ALL`?

Pada tabel yang sangat kecil (4 baris), optimizer MySQL/MariaDB sering **tetap memilih full table scan** meskipun indeks tersedia. Membaca 4 baris dalam satu halaman data lebih murah daripada membuka struktur indeks lalu melompat ke tabel. Ini sejalan dengan teori di modul bahwa pada **tabel kecil, indeks tidak memberi manfaat signifikan**. Selisih waktu eksekusi juga hanya sekitar 0,0005 detik sehingga tidak bermakna.

Untuk membuktikan manfaat indeks, lakukan salah satu atau kedua langkah berikut.

**a) Paksa penggunaan indeks:**

```sql
EXPLAIN SELECT * FROM Products FORCE INDEX (idx_products_category)
WHERE category = 'Elektronik';
```

**b) Gunakan data lebih besar (Bagian 5).**

📸 *Screenshot: hasil EXPLAIN Q1–Q5 dan SHOW PROFILES (sesudah indeks)*

---

## 5. Pembuktian dengan Data Besar (Opsional, Direkomendasikan)

XAMPP memakai MariaDB yang menyediakan engine `seq` untuk membangkitkan angka berurutan. Data di bawah mensimulasikan database ABC Retail yang sebenarnya.

```sql
-- Tambah 100.000 produk
INSERT INTO Products (product_name, category, price, stock)
SELECT CONCAT('Produk-', seq),
       ELT(1 + seq MOD 5, 'Elektronik', 'Furniture', 'Fashion', 'Olahraga', 'Dapur'),
       1000 * (1 + seq MOD 10000),
       seq MOD 100
FROM seq_1_to_100000;

-- Tambah 300.000 transaksi
INSERT INTO Transactions (product_id, quantity, transaction_date)
SELECT 1 + seq MOD 100000,
       1 + seq MOD 5,
       DATE_ADD('2023-01-01', INTERVAL seq MOD 365 DAY)
FROM seq_1_to_300000;

ANALYZE TABLE Products, Transactions;
```

### Perbandingan before/after

Untuk membandingkan, **hapus indeks** lalu ukur, kemudian **buat ulang** dan ukur lagi:

```sql
-- BEFORE: hapus indeks buatan sendiri
DROP INDEX idx_products_category  ON Products;
DROP INDEX idx_products_cat_price ON Products;
DROP INDEX idx_trx_product_id     ON Transactions;
DROP INDEX idx_trx_date           ON Transactions;
DROP INDEX idx_trx_product_qty    ON Transactions;

SET profiling = 1;
SELECT COUNT(*) FROM Products WHERE category = 'Elektronik' AND price > 9000000;
SELECT COUNT(*) FROM Transactions
WHERE transaction_date BETWEEN '2023-02-01' AND '2023-02-28';
EXPLAIN SELECT COUNT(*) FROM Products WHERE category = 'Elektronik' AND price > 9000000;
EXPLAIN SELECT COUNT(*) FROM Transactions
WHERE transaction_date BETWEEN '2023-02-01' AND '2023-02-28';

-- AFTER: buat ulang indeks (Langkah 3.1 - 3.5), lalu jalankan query yang sama
SHOW PROFILES;
```

Alternatif yang lebih presisi di MariaDB: gunakan `ANALYZE SELECT ...` yang menampilkan waktu dan jumlah baris **aktual**.

### Tabel catatan hasil (isi sendiri)

| Query | Waktu sebelum indeks | Waktu sesudah indeks | Rows sebelum | Rows sesudah | type sebelum → sesudah |
|-------|---------------------:|---------------------:|-------------:|-------------:|------------------------|
| Q1 | … s | … s | … | … | ALL → ref |
| Q2 | … s | … s | … | … | ALL → ref |
| Q3 | … s | … s | … | … | ALL → range |
| Q4 | … s | … s | … | … | ALL → ref (tanpa filesort) |
| Q5 | … s | … s | … | … | ALL → index |

**Analisis yang diharapkan:** pada ratusan ribu baris, `rows` turun drastis (dari seluruh tabel menjadi sebagian kecil), `type` berubah dari `ALL` menjadi `ref`/`range`/`index`, `Using filesort` hilang pada Q4, dan waktu eksekusi turun signifikan.

---

## 6. Menghapus Indeks yang Tidak Diperlukan

### Evaluasi indeks

| Indeks | Dipakai oleh | Keputusan |
|--------|--------------|-----------|
| idx_products_category | Q1 | **Redundan** (lihat di bawah) |
| idx_products_cat_price | Q1, Q4 | Pertahankan |
| idx_trx_product_id | Q2 | **Redundan** (lihat di bawah) |
| idx_trx_date | Q3 | Pertahankan |
| idx_trx_product_qty | Q2, Q5 | Pertahankan |

**Analisis redundansi:** indeks komposit `(category, price)` sudah dapat melayani query yang hanya memfilter `category` (prinsip *leftmost prefix*), sehingga `idx_products_category` tidak diperlukan. Begitu juga `idx_trx_product_qty (product_id, quantity)` sudah melayani kebutuhan JOIN pada `product_id`, sehingga `idx_trx_product_id` berlebihan. Indeks berlebihan hanya memperlambat `INSERT`/`UPDATE`/`DELETE` dan memakan ruang penyimpanan.

### Langkah 6.1 – Hapus indeks redundan

```sql
DROP INDEX idx_products_category ON Products;
DROP INDEX idx_trx_product_id    ON Transactions;
```

**Fungsi kode:** `DROP INDEX <nama_indeks> ON <nama_tabel>` menghapus indeks dari tabel tertentu.

### Langkah 6.2 – Verifikasi penghapusan

```sql
SHOW INDEX FROM Products;
SHOW INDEX FROM Transactions;
```

### Langkah 6.3 – Pastikan query tetap memakai indeks

```sql
EXPLAIN SELECT * FROM Products WHERE category = 'Elektronik';
EXPLAIN SELECT p.product_name, t.quantity
FROM Transactions t JOIN Products p ON t.product_id = p.product_id;
```

**Hasil yang diharapkan:** `key` berubah menjadi `idx_products_cat_price` (Q1) dan `idx_trx_product_qty` (Q2). Artinya performa tetap terjaga dengan jumlah indeks yang lebih sedikit.

📸 *Screenshot: SHOW INDEX dan EXPLAIN setelah penghapusan indeks*

---

## 7. Kesimpulan

1. **Indeks mempercepat query** pada kolom yang dipakai di `WHERE`, `JOIN`, `ORDER BY`, dan `GROUP BY`. Hal ini terlihat dari `type` yang berubah dari `ALL` menjadi `ref`/`range`/`index` dan kolom `rows` yang menurun.
2. **`EXPLAIN` adalah alat utama index tuning**: lihat `type`, `possible_keys`, `key`, `rows`, dan `Extra` untuk memastikan indeks benar-benar dipakai.
3. **Indeks komposit** (`category, price`) dapat menghilangkan `Using filesort`, dan **covering index** (`product_id, quantity`) dapat menjawab query langsung dari indeks.
4. **Pada tabel kecil**, optimizer bisa tetap memilih full table scan karena lebih murah. Manfaat indeks baru jelas pada data besar.
5. **Jangan membuat indeks berlebihan.** Indeks redundan memperlambat operasi write dan memboroskan penyimpanan. Evaluasi dan hapus indeks yang tidak dipakai.
6. **Untuk kolom berkardinalitas rendah** seperti `category` (hanya beberapa nilai unik), indeks tunggal kurang efektif. Lebih baik digabung dengan kolom lain dalam indeks komposit.

---

## Lampiran: Kumpulan Semua Query (untuk file `.sql`)

```sql
-- 1. Persiapan
CREATE DATABASE praktikum_index_tuning;
USE praktikum_index_tuning;

CREATE TABLE Products (
    product_id   INT PRIMARY KEY AUTO_INCREMENT,
    product_name VARCHAR(100),
    category     VARCHAR(50),
    price        DECIMAL(12, 2),
    stock        INT
);

CREATE TABLE Transactions (
    transaction_id   INT PRIMARY KEY AUTO_INCREMENT,
    product_id       INT,
    quantity         INT,
    transaction_date DATE
);

INSERT INTO Products (product_name, category, price, stock) VALUES
('Laptop',  'Elektronik', 12000000, 10),
('Printer', 'Elektronik',  2500000, 20),
('Meja',    'Furniture',   1500000, 15),
('Kursi',   'Furniture',    700000, 30);

INSERT INTO Transactions (product_id, quantity, transaction_date) VALUES
(1, 2, '2023-02-15'),
(2, 1, '2023-02-20'),
(3, 3, '2023-03-10'),
(4, 4, '2023-04-25');

SHOW INDEX FROM Products;
SHOW INDEX FROM Transactions;

-- 2. Baseline
SET profiling = 1;
EXPLAIN SELECT * FROM Products WHERE category = 'Elektronik';
EXPLAIN SELECT p.product_name, t.quantity, t.transaction_date
        FROM Transactions t JOIN Products p ON t.product_id = p.product_id;
EXPLAIN SELECT * FROM Transactions
        WHERE transaction_date BETWEEN '2023-02-01' AND '2023-03-31';
EXPLAIN SELECT * FROM Products WHERE category = 'Elektronik' ORDER BY price DESC;
EXPLAIN SELECT product_id, SUM(quantity) AS total_qty FROM Transactions GROUP BY product_id;
SHOW PROFILES;

-- 3. Buat indeks
CREATE INDEX idx_products_category  ON Products(category);
CREATE INDEX idx_trx_product_id     ON Transactions(product_id);
CREATE INDEX idx_trx_date           ON Transactions(transaction_date);
CREATE INDEX idx_products_cat_price ON Products(category, price);
CREATE INDEX idx_trx_product_qty    ON Transactions(product_id, quantity);
SHOW INDEX FROM Products;
SHOW INDEX FROM Transactions;

-- 4. Setelah indeks
ANALYZE TABLE Products, Transactions;
EXPLAIN SELECT * FROM Products WHERE category = 'Elektronik';
EXPLAIN SELECT p.product_name, t.quantity, t.transaction_date
        FROM Transactions t JOIN Products p ON t.product_id = p.product_id;
EXPLAIN SELECT * FROM Transactions
        WHERE transaction_date BETWEEN '2023-02-01' AND '2023-03-31';
EXPLAIN SELECT * FROM Products WHERE category = 'Elektronik' ORDER BY price DESC;
EXPLAIN SELECT product_id, SUM(quantity) AS total_qty FROM Transactions GROUP BY product_id;
EXPLAIN SELECT * FROM Products FORCE INDEX (idx_products_category) WHERE category = 'Elektronik';
SHOW PROFILES;

-- 5. Data besar (opsional)
INSERT INTO Products (product_name, category, price, stock)
SELECT CONCAT('Produk-', seq),
       ELT(1 + seq MOD 5, 'Elektronik', 'Furniture', 'Fashion', 'Olahraga', 'Dapur'),
       1000 * (1 + seq MOD 10000),
       seq MOD 100
FROM seq_1_to_100000;

INSERT INTO Transactions (product_id, quantity, transaction_date)
SELECT 1 + seq MOD 100000,
       1 + seq MOD 5,
       DATE_ADD('2023-01-01', INTERVAL seq MOD 365 DAY)
FROM seq_1_to_300000;

ANALYZE TABLE Products, Transactions;

-- 6. Hapus indeks redundan
DROP INDEX idx_products_category ON Products;
DROP INDEX idx_trx_product_id    ON Transactions;
SHOW INDEX FROM Products;
SHOW INDEX FROM Transactions;
EXPLAIN SELECT * FROM Products WHERE category = 'Elektronik';
EXPLAIN SELECT p.product_name, t.quantity
        FROM Transactions t JOIN Products p ON t.product_id = p.product_id;
```
