# Tugas Mandiri Modul 4 – Query Processing

**Mata Kuliah:** SD3203 – Teknologi Basis Data
**DBMS / Tools:** PostgreSQL + DBeaver

---

## 1. Soal

Pegawai toko ingin melihat data retail yang berisi:

- id pelanggan
- nama pelanggan
- kota
- tanggal
- total harga

Buatlah kuerinya dengan metode eksekusi **Nested Loop Join** dan analisa eksekusinya.

---

## 2. Analisis Kebutuhan Data

Sebelum menulis kueri, tentukan dulu kolom yang diminta berasal dari tabel mana.

| Kolom yang diminta | Tabel asal |
|---|---|
| id pelanggan | `pelanggan.id` |
| nama pelanggan | `pelanggan.nama` |
| kota | `pelanggan.kota` |
| tanggal | `transaksi.tanggal` |
| total harga | `transaksi.total_harga` |

**Kesimpulan:** hanya dua tabel yang dibutuhkan, yaitu `pelanggan` dan `transaksi`. Tabel `produk` **tidak perlu** di-join karena tidak ada kolom produk yang diminta.

**Kunci join:** `transaksi.id_pelanggan` (foreign key) = `pelanggan.id` (primary key).

Dalam notasi aljabar relasional:

```
π id, nama, kota, tanggal, total_harga ( pelanggan ⨝ pelanggan.id = transaksi.id_pelanggan  transaksi )
```

---

## 3. Langkah Pengerjaan

### Langkah 1 – Buat Skema

```sql
CREATE SCHEMA retail;
SET search_path = retail;
```

> `SET search_path` membuat tabel yang dibuat/dipanggil setelahnya otomatis berada di skema `retail`. Kalau tidak dijalankan, tulis nama tabel lengkap, misalnya `retail.pelanggan`.

### Langkah 2 – Buat Tabel

Tabel `pelanggan` dan `produk` dibuat lebih dulu karena tabel `transaksi` mereferensikan keduanya lewat foreign key.

```sql
CREATE TABLE pelanggan (
    id            SERIAL PRIMARY KEY,
    nama          VARCHAR(50) NOT NULL,
    kota          VARCHAR(50),
    jenis_kelamin VARCHAR(1) CHECK (jenis_kelamin IN ('L', 'P'))
);

CREATE TABLE produk (
    id          SERIAL PRIMARY KEY,
    nama_produk VARCHAR(100) NOT NULL,
    kategori    VARCHAR(50),
    harga       BIGINT
);

CREATE TABLE transaksi (
    id           SERIAL PRIMARY KEY,
    id_pelanggan INT REFERENCES pelanggan(id),
    id_produk    INT REFERENCES produk(id),
    tanggal      DATE,
    jumlah       INT,
    total_harga  BIGINT,
    FOREIGN KEY (id_pelanggan) REFERENCES pelanggan(id),
    FOREIGN KEY (id_produk)    REFERENCES produk(id)
);
```

> Tipe `BIGINT` dipakai untuk `harga` dan `total_harga` agar aman untuk nilai besar (data ini mencapai 100.000.000). `INT` sebenarnya masih cukup untuk data soal, tetapi `BIGINT` lebih aman jika nilai transaksi bertambah.

### Langkah 3 – Isi Data

```sql
INSERT INTO pelanggan (nama, kota, jenis_kelamin) VALUES
('Adit',   'Jakarta',    'L'),
('Budi',   'Bandung',    'L'),
('Citra',  'Surabaya',   'P'),
('Dewi',   'Yogyakarta', 'P'),
('Eka',    'Medan',      'L'),
('Faisal', 'Palembang',  'L'),
('Gita',   'Semarang',   'P'),
('Hadi',   'Makassar',   'L'),
('Indah',  'Malang',     'P'),
('Joko',   'Bali',       'L');

INSERT INTO produk (nama_produk, kategori, harga) VALUES
('Laptop Asus',       'Elektronik',      15000000),
('Smartphone iPhone', 'Elektronik',      18000000),
('Kulkas Samsung',    'Peralatan Rumah',  5000000),
('TV LG',             'Elektronik',       8000000),
('Headphone Sony',    'Aksesoris',        2000000),
('Sepatu Nike',       'Fashion',          1300000),
('Tas Louis Vuitton', 'Fashion',          7000000),
('Jam Tangan Rolex',  'Aksesoris',       50000000),
('Kamera Canon',      'Fotografi',       12000000),
('Buku Data Science', 'Edukasi',           500000);

INSERT INTO transaksi (id_pelanggan, id_produk, tanggal, jumlah, total_harga) VALUES
(1, 1,  '2024-03-01', 1, 15000000),
(1, 2,  '2024-03-02', 1, 18000000),
(2, 3,  '2024-03-03', 2, 10000000),
(3, 4,  '2024-03-04', 1,  8000000),
(4, 5,  '2024-03-05', 3,  6000000),
(5, 6,  '2024-03-06', 1,  1300000),
(6, 7,  '2024-03-07', 1,  7000000),
(7, 8,  '2024-03-08', 1, 50000000),
(8, 9,  '2024-03-09', 1, 12000000),
(9, 10, '2024-03-10', 2,  1000000),
(1, 5,  '2024-03-11', 1,  2000000),
(2, 7,  '2024-03-12', 2, 14000000),
(3, 9,  '2024-03-13', 1, 12000000),
(4, 6,  '2024-03-14', 2,  2600000),
(5, 8,  '2024-03-15', 1, 50000000),
(6, 2,  '2024-03-16', 1, 18000000),
(7, 3,  '2024-03-17', 1,  5000000),
(8, 4,  '2024-03-18', 1,  8000000),
(9, 1,  '2024-03-19', 1, 15000000),
(10,10, '2024-03-20', 3,  1500000),
(1, 2,  '2024-03-21', 2, 36000000),
(2, 5,  '2024-03-22', 1,  2000000),
(3, 8,  '2024-03-23', 1, 50000000),
(4, 9,  '2024-03-24', 2, 24000000),
(5, 6,  '2024-03-25', 1,  1300000),
(6, 7,  '2024-03-26', 1,  7000000),
(7, 8,  '2024-03-27', 2,100000000),
(8, 10, '2024-03-28', 1,   500000),
(9, 1,  '2024-03-29', 1, 15000000),
(10,3,  '2024-03-30', 2, 10000000);
```

> Urutan `id` pada `pelanggan`/`produk` mengikuti `SERIAL` (1–10), sama persis dengan tabel di soal. Format tanggal `3/1/2024` pada soal diasumsikan **1 Maret 2024** (`2024-03-01`), dan seterusnya berurutan sampai 30 Maret.

Cek jumlah data:

```sql
SELECT 'pelanggan' AS tabel, COUNT(*) FROM pelanggan
UNION ALL SELECT 'produk',    COUNT(*) FROM produk
UNION ALL SELECT 'transaksi', COUNT(*) FROM transaksi;
```

Hasil yang diharapkan: pelanggan = 10, produk = 10, transaksi = 30.

### Langkah 4 – Nonaktifkan Hash Join dan Merge Join

Data ini sangat kecil, sehingga PostgreSQL cenderung memilih **Hash Join** (cost paling rendah). Supaya yang dipakai **Nested Loop Join**, dua metode lain dimatikan sementara, sama seperti di modul:

```sql
SET enable_hashjoin  = OFF;
SET enable_mergejoin = OFF;
```

> `SET` hanya berlaku pada sesi/koneksi yang sedang aktif. Jalankan di editor yang sama dengan kueri berikutnya.

### Langkah 5 – Kueri Nested Loop Join

```sql
EXPLAIN ANALYZE
SELECT  p.id          AS id_pelanggan,
        p.nama        AS nama_pelanggan,
        p.kota,
        t.tanggal,
        t.total_harga
FROM    pelanggan p
JOIN    transaksi t ON t.id_pelanggan = p.id;
```

- `EXPLAIN ANALYZE` menjalankan kueri sungguhan lalu menampilkan **query plan** beserta waktu nyata.
- Untuk melihat datanya saja, hapus kata `EXPLAIN ANALYZE`.

### Langkah 6 – Lihat Hasil Data (tanpa EXPLAIN)

```sql
SELECT  p.id AS id_pelanggan, p.nama AS nama_pelanggan, p.kota,
        t.tanggal, t.total_harga
FROM    pelanggan p
JOIN    transaksi t ON t.id_pelanggan = p.id
ORDER BY t.id;
```

Cuplikan hasil (30 baris total):

| id_pelanggan | nama_pelanggan | kota | tanggal | total_harga |
|---:|---|---|---|---:|
| 1 | Adit | Jakarta | 2024-03-01 | 15000000 |
| 1 | Adit | Jakarta | 2024-03-02 | 18000000 |
| 2 | Budi | Bandung | 2024-03-03 | 10000000 |
| 3 | Citra | Surabaya | 2024-03-04 | 8000000 |
| 4 | Dewi | Yogyakarta | 2024-03-05 | 6000000 |
| 5 | Eka | Medan | 2024-03-06 | 1300000 |
| … | … | … | … | … |

Jumlah baris = **30**, sama dengan jumlah transaksi, karena setiap transaksi pasti punya satu pelanggan (relasi *many-to-one*).

---

## 4. Analisis Eksekusi (Query Plan)

Berikut hasil `EXPLAIN ANALYZE` yang dijalankan langsung di PostgreSQL 16. Angka *cost* dan *time* di komputer Anda bisa sedikit berbeda, tetapi **struktur plan-nya sama**.

### 4.1 Hasil Utama – Nested Loop Join

```
Nested Loop  (cost=0.00..6.56 rows=30 width=29) (actual time=0.013..0.035 rows=30 loops=1)
  Join Filter: (p.id = t.id_pelanggan)
  Rows Removed by Join Filter: 126
  ->  Seq Scan on transaksi t  (cost=0.00..1.30 rows=30 width=16) (actual time=0.003..0.005 rows=30 loops=1)
  ->  Materialize  (cost=0.00..1.15 rows=10 width=17) (actual time=0.000..0.001 rows=5 loops=30)
        ->  Seq Scan on pelanggan p  (cost=0.00..1.10 rows=10 width=17) (actual time=0.003..0.004 rows=10 loops=1)
Planning Time: 0.244 ms
Execution Time: 0.070 ms
```

### 4.2 Membaca Plan Baris demi Baris

| Baris plan | Arti |
|---|---|
| `Nested Loop` | Metode join yang dipakai. Terbukti PostgreSQL memakai **nested loop join** setelah hash join dan merge join dimatikan. |
| `Join Filter: (p.id = t.id_pelanggan)` | Kondisi pencocokan yang dicek untuk setiap pasangan baris. |
| `Rows Removed by Join Filter: 126` | Ada 126 pasangan yang dibandingkan tetapi **tidak cocok** lalu dibuang. |
| `Seq Scan on transaksi t` (tabel luar / *outer*) | Tabel `transaksi` dibaca sekali dari awal sampai akhir, menghasilkan 30 baris (`loops=1`). |
| `Materialize` → `Seq Scan on pelanggan p` (tabel dalam / *inner*) | Tabel `pelanggan` dibaca **sekali** (10 baris), disimpan di memori, lalu dipakai ulang. |
| `loops=30` pada `Materialize` | Bagian *inner* diulang **30 kali**, satu kali untuk setiap baris tabel luar. |
| `rows=30` pada `Nested Loop` | Hasil akhir 30 baris, sesuai jumlah transaksi. |
| `cost=0.00..6.56` | Estimasi biaya (startup..total) dalam satuan internal PostgreSQL, bukan milidetik. |
| `Planning Time` / `Execution Time` | Waktu menyusun rencana dan waktu mengeksekusi (ms). |

### 4.3 Cara Kerja Nested Loop pada Kasus Ini

Pseudocode:

```
untuk setiap baris t di transaksi:              -- outer loop (30 iterasi)
    untuk setiap baris p di pelanggan:          -- inner loop (maks. 10 baris)
        jika p.id = t.id_pelanggan:
            keluarkan (p.id, p.nama, p.kota, t.tanggal, t.total_harga)
            berhenti (lanjut ke baris transaksi berikutnya)
```

Jika tanpa penghentian dini, jumlah perbandingan maksimum adalah **30 × 10 = 300** pasangan. Namun `pelanggan.id` adalah primary key (nilainya unik), sehingga PostgreSQL tahu bahwa begitu satu pelanggan yang cocok ditemukan, pencarian untuk transaksi tersebut boleh dihentikan. Itulah sebabnya hanya 126 pasangan tidak cocok yang tercatat (`Rows Removed by Join Filter: 126`), bukan 270. Pada Variasi B di bawah, tabel dalam adalah `transaksi` (tidak unik pada `id_pelanggan`), sehingga setiap baris luar harus memindai seluruh 30 baris dan angka 270 muncul.

Karena tiap transaksi hanya cocok dengan **satu** pelanggan, hasil join tetap 30 baris.



---

## 5. Variasi Plan yang Mungkin Muncul

Bentuk plan dapat berbeda tergantung kondisi statistik tabel. Ini normal dan penting dipahami.

### Variasi A – Statistik belum diperbarui (Index Nested Loop)

Jika kueri dijalankan tepat setelah `INSERT` (sebelum `ANALYZE`/autovacuum berjalan), PostgreSQL masih memakai estimasi default sehingga plan-nya seperti ini:

```
Nested Loop  (cost=0.15..311.50 rows=1450 width=252) (actual time=0.020..0.064 rows=30 loops=1)
  ->  Seq Scan on transaksi t  (cost=0.00..24.50 rows=1450 width=16) (actual time=0.004..0.007 rows=30 loops=1)
  ->  Index Scan using pelanggan_pkey on pelanggan p  (cost=0.15..0.20 rows=1 width=240) (actual time=0.001..0.001 rows=1 loops=30)
        Index Cond: (id = t.id_pelanggan)
Planning Time: 0.125 ms
Execution Time: 0.089 ms
```

Perbedaan penting:

- Tabel dalam dibaca dengan **Index Scan** pada `pelanggan_pkey`, yaitu jenis **Index Nested Loop Join**.
- Estimasi `rows=1450` jauh dari kenyataan `rows=30`, karena statistik belum ada.
- Plan ini mirip contoh di modul (Gambar 9), di mana tabel dalam memakai `Index Scan using dokter_pkey`.

Agar estimasi akurat, jalankan:

```sql
ANALYZE pelanggan;
ANALYZE transaksi;
```

Setelah itu, estimasi menjadi `rows=30` dan plan berubah menjadi seperti Bagian 4.1.

### Variasi B – Nested Loop murni (brute force)

Jika index scan dan materialize juga dimatikan, terlihat nested loop paling dasar:

```sql
SET enable_indexscan = OFF;
SET enable_bitmapscan = OFF;
SET enable_material  = OFF;
```

```
Nested Loop  (cost=0.00..17.85 rows=30 width=29) (actual time=0.006..0.042 rows=30 loops=1)
  Join Filter: (p.id = t.id_pelanggan)
  Rows Removed by Join Filter: 270
  ->  Seq Scan on pelanggan p  (cost=0.00..1.10 rows=10 width=17) (actual time=0.002..0.003 rows=10 loops=1)
  ->  Seq Scan on transaksi t  (cost=0.00..1.30 rows=30 width=16) (actual time=0.000..0.002 rows=30 loops=10)
Planning Time: 0.056 ms
Execution Time: 0.051 ms
```

Di sini terlihat jelas sifat *brute force*:

- Tabel luar = `pelanggan` (10 baris), tabel dalam = `transaksi` dibaca ulang `loops=10` kali.
- Total perbandingan = 10 × 30 = **300**. Yang cocok 30, yang dibuang `270`.

### Perbandingan Ketiga Plan

| Aspek | Utama (4.1) | Variasi A | Variasi B |
|---|---|---|---|
| Tabel luar (outer) | `transaksi` | `transaksi` | `pelanggan` |
| Akses tabel dalam | Seq Scan + Materialize | Index Scan (PK) | Seq Scan berulang |
| Loop tabel dalam | 30× (dari memori) | 30× (lewat indeks) | 10× (baca ulang tabel) |
| Pasangan dibuang | 126 | – (langsung via indeks) | 270 |
| Estimasi cost total | 6.56 | 311.50 | 17.85 |
| Execution time | 0.070 ms | 0.089 ms | 0.051 ms |

> Selisih waktu hanya sepersekian milidetik dan bisa berubah di tiap eksekusi. Pada data sekecil ini, perbedaan waktu **tidak bermakna**, jadi perbandingan sebaiknya fokus pada *struktur plan* dan *jumlah operasi*.

---

## 6. Perbandingan dengan Hash Join (Pilihan Default PostgreSQL)

Tanpa mematikan hash join, PostgreSQL memilih plan berikut:

```
Hash Join  (cost=16.52..44.89 rows=1450 width=252) (actual time=0.067..0.079 rows=30 loops=1)
  Hash Cond: (t.id_pelanggan = p.id)
  ->  Seq Scan on transaksi t  (cost=0.00..24.50 rows=1450 width=16) (actual time=0.014..0.017 rows=30 loops=1)
  ->  Hash  (cost=12.90..12.90 rows=290 width=240) (actual time=0.026..0.026 rows=10 loops=1)
        Buckets: 1024  Batches: 1  Memory Usage: 9kB
        ->  Seq Scan on pelanggan p  (cost=0.00..12.90 rows=290 width=240) (actual time=0.010..0.012 rows=10 loops=1)
Planning Time: 0.693 ms
Execution Time: 0.133 ms
```

Hash Join membuat *hash table* dari tabel kecil (`pelanggan`) lalu mencari kecocokan tiap baris `transaksi` secara langsung, tanpa perulangan bersarang. Untuk data besar, cara ini umumnya jauh lebih efisien dibanding nested loop murni.

---

## 7. Kesimpulan

1. Kebutuhan data hanya melibatkan tabel **`pelanggan`** dan **`transaksi`**, di-join dengan `transaksi.id_pelanggan = pelanggan.id`. Tabel `produk` tidak diperlukan.
2. Setelah `enable_hashjoin` dan `enable_mergejoin` dimatikan, PostgreSQL memakai **Nested Loop Join**. Tabel luar (`transaksi`, 30 baris) dibaca sekali, lalu untuk setiap barisnya tabel dalam (`pelanggan`, 10 baris) dicocokkan (`loops=30`).
3. Hasil join berjumlah **30 baris**, sesuai jumlah transaksi.
4. Bentuk plan bergantung pada **statistik tabel** dan **ketersediaan indeks**. Tanpa statistik, PostgreSQL bisa memakai Index Nested Loop lewat `pelanggan_pkey`. Dengan statistik akurat, ia memakai Seq Scan + Materialize pada tabel kecil.
5. Nested Loop cocok untuk **tabel kecil** atau saat tabel dalam punya **indeks** pada kolom join. Untuk tabel besar tanpa indeks, biayanya mendekati O(n × m) sehingga Hash Join atau Merge Join lebih baik.
6. Perbedaan waktu eksekusi pada data sekecil ini sangat tipis, sehingga keputusan metode join sebaiknya didasarkan pada ukuran data dan indeks, bukan hanya waktu eksekusi.

---

## 8. Kembalikan Pengaturan Semula

Sesuai petunjuk modul, aktifkan kembali metode join yang tadi dimatikan:

```sql
SET enable_hashjoin  = ON;
SET enable_mergejoin = ON;
-- jika Variasi B dijalankan:
SET enable_indexscan  = ON;
SET enable_bitmapscan = ON;
SET enable_material   = ON;
```

Atau cukup `RESET ALL;` untuk mengembalikan semua pengaturan sesi ke bawaan.

---

## Lampiran – Kueri Lengkap (Siap Salin)

```sql
-- 1. Setup
CREATE SCHEMA retail;
SET search_path = retail;
-- (jalankan CREATE TABLE dan INSERT pada Langkah 2 dan 3)

-- 2. Perbarui statistik
ANALYZE pelanggan;
ANALYZE transaksi;

-- 3. Paksa nested loop join
SET enable_hashjoin  = OFF;
SET enable_mergejoin = OFF;

-- 4. Analisa eksekusi
EXPLAIN ANALYZE
SELECT  p.id          AS id_pelanggan,
        p.nama        AS nama_pelanggan,
        p.kota,
        t.tanggal,
        t.total_harga
FROM    pelanggan p
JOIN    transaksi t ON t.id_pelanggan = p.id;

-- 5. Kembalikan pengaturan
SET enable_hashjoin  = ON;
SET enable_mergejoin = ON;
```
