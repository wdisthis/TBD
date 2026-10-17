# KUIS

**Mata Kuliah: Teknologi Basis Data**

Sebuah perusahaan e-commerce memiliki sistem basis data untuk mengelola data pelanggan, produk, transaksi, dan detail transaksi. Seiring meningkatnya jumlah pengguna, sistem mengalami penurunan performa, terutama ketika pelanggan mencari produk, melihat riwayat transaksi, dan ketika bagian manajemen menghasilkan laporan penjualan bulanan.

Sistem saat ini menggunakan struktur tabel berikut.

| Nama Tabel | Jumlah Data | Atribut |
|---|---|---|
| Pelanggan | 500.000 baris | id_pelanggan (PK), nama, email, kota |
| Produk | 200.000 baris | id_produk (PK), nama_produk, kategori, harga, stok |
| Transaksi | 5.000.000 baris | id_transaksi (PK), id_pelanggan (FK), tanggal, status, total_bayar |
| Detail_Transaksi | 20.000.000 baris | id_detail (PK), id_transaksi (FK), id_produk (FK), jumlah, harga_satuan |

Permasalahan yang ditemukan adalah sebagai berikut:

- Tabel Transaksi dan Detail_Transaksi terus bertambah sehingga proses pencarian dan pelaporan semakin lambat.
- Pencarian pelanggan berdasarkan email sering dilakukan, tetapi belum menggunakan indeks khusus.
- Laporan penjualan hanya membutuhkan transaksi berstatus Selesai, tetapi query memproses banyak transaksi dengan status lain.
- Beberapa laporan menggabungkan empat tabel sekaligus sebelum melakukan penyaringan data.
- Atribut total_bayar pada tabel Transaksi dapat dihitung dari penjumlahan jumlah dikalikan harga_satuan pada detail transaksi, tetapi nilainya juga disimpan secara terpisah.
- Sistem harus tetap menghasilkan data yang benar meskipun optimasi dilakukan.

**Jawablah Pertanyaan Berikut:**

1. Jelaskan alasan perusahaan membutuhkan Database Management System (DBMS) untuk mengelola data e-commerce dibandingkan menyimpan data dalam berkas terpisah. Kaitkan jawaban dengan minimal tiga karakteristik atau fungsi DBMS.

2. Usulkan dua perubahan skema basis data yang dapat meningkatkan performa sistem. Untuk setiap perubahan, jelaskan struktur awal, struktur yang diusulkan, alasan perubahan dapat meningkatkan performa, serta dampaknya terhadap konsistensi data dan proses pembaruan.

3. Perusahaan ingin mempercepat laporan penjualan bulanan tanpa mengubah hasil laporan. Bandingkan penggunaan normalisasi dan denormalisasi untuk kebutuhan tersebut.

4. Diketahui query SQL yang digunakan untuk mencari riwayat transaksi pelanggan berikut.

   ```sql
   SELECT id_transaksi, tanggal, total_bayar
   FROM Transaksi
   WHERE id_pelanggan = 12580
     AND status = 'Selesai'
   ORDER BY tanggal DESC;
   ```

   Saat ini, tabel Transaksi hanya memiliki indeks pada id_transaksi sebagai primary key.
   Jelaskan mengapa menambahkan indeks pada seluruh kolom tabel bukan strategi yang selalu tepat.

5. Manajemen ingin memperoleh total nilai penjualan per kategori produk untuk transaksi berstatus Selesai selama Januari 2026.

   Query SQL awal yang digunakan adalah sebagai berikut.

   ```sql
   SELECT
     P.kategori,
     SUM(D.jumlah * D.harga_satuan) AS total_penjualan
   FROM Pelanggan C
   JOIN Transaksi T
     ON C.id_pelanggan = T.id_pelanggan
   JOIN Detail_Transaksi D
     ON T.id_transaksi = D.id_transaksi
   JOIN Produk P
     ON D.id_produk = P.id_produk
   WHERE T.status = 'Selesai'
     AND T.tanggal >= '2026-01-01'
     AND T.tanggal < '2026-02-01'
   GROUP BY P.kategori;
   ```

   a. Jelaskan bagaimana urutan pelaksanaan operasi dapat memengaruhi biaya pemrosesan. (6 poin)

   b. Tuliskan ekspresi aljabar relasional untuk query tersebut sebelum optimasi. Gunakan simbol seleksi, proyeksi, join, dan agregasi jika diperlukan. (6 poin)

   c. Optimalkan ekspresi aljabar relasionalnya.

---

# JAWABAN

## Jawaban Soal 1 — Alasan Perusahaan Membutuhkan DBMS

### Gambaran Masalah

Jika data pelanggan, produk, transaksi, dan detail transaksi disimpan dalam berkas terpisah (misalnya CSV, Excel, atau file teks), perusahaan harus menulis sendiri program untuk membaca, mencari, menggabungkan, dan memperbarui data. Dengan volume 500 ribu pelanggan, 200 ribu produk, 5 juta transaksi, dan 20 juta detail transaksi, pendekatan ini rawan lambat, tidak konsisten, dan tidak aman. **DBMS** (*Database Management System*) menyediakan fungsi-fungsi bawaan yang menyelesaikan masalah tersebut.

### Karakteristik / Fungsi DBMS yang Relevan

| No | Karakteristik / Fungsi DBMS | Masalah pada Berkas Terpisah | Solusi DBMS pada Kasus E-Commerce |
|---|---|---|---|
| 1 | **Pengurangan redundansi dan kontrol konsistensi** (data disimpan terpusat dan terstruktur melalui skema/normalisasi) | Data pelanggan (nama, email, kota) bisa tersalin di banyak berkas transaksi. Jika pelanggan mengganti email, ada berkas yang terbarui dan ada yang tidak, sehingga timbul inkonsistensi. | Data pelanggan disimpan **sekali** di tabel `Pelanggan` dan direferensikan lewat `id_pelanggan`. Perubahan email cukup dilakukan di satu tempat. |
| 2 | **Integritas data** (*constraint*: PRIMARY KEY, FOREIGN KEY, NOT NULL, CHECK, UNIQUE) | Tidak ada yang mencegah transaksi merujuk ke pelanggan yang tidak ada, `harga` bernilai negatif, atau `id_transaksi` ganda. | `FOREIGN KEY` memastikan `Transaksi.id_pelanggan` selalu valid, `CHECK (harga >= 0 AND stok >= 0)` menolak data tidak wajar, dan `UNIQUE (email)` mencegah email ganda. |
| 3 | **Kontrol konkurensi dan transaksi (ACID)** | Ribuan pelanggan memesan bersamaan. Dua pelanggan bisa membeli stok terakhir yang sama sehingga stok menjadi negatif, atau file rusak karena ditulis bersamaan. | DBMS memakai *locking*/MVCC (*Multi-Version Concurrency Control*, yang mengurangi *lock contention* karena pembaca dan penulis tidak saling menunggu) dan transaksi ACID. Pembuatan pesanan (insert `Transaksi`, insert `Detail_Transaksi`, kurangi `stok`) dijalankan **atomik**: semua berhasil atau semua dibatalkan (*rollback*). |
| 4 | **Bahasa query deklaratif (SQL) dan optimasi query** | Setiap pencarian atau laporan memerlukan program khusus yang memindai berkas dari awal sampai akhir (*full scan*), dan penggabungan antar-berkas harus dibuat manual. | SQL cukup menyatakan *apa* yang diinginkan. *Query optimizer* memilih rencana eksekusi terbaik (urutan join, penggunaan indeks) berdasarkan statistik data. |
| 5 | **Indeks dan struktur penyimpanan efisien** | Mencari satu email di 500.000 baris berarti membaca seluruh berkas. | DBMS menyediakan indeks (B-Tree, Hash) sehingga pencarian email menjadi `O(log n)`, bukan `O(n)`. |
| 6 | **Independensi data** | Jika format berkas berubah (misalnya menambah kolom), semua program yang membacanya harus ditulis ulang. | Skema dapat diubah (`ALTER TABLE`) tanpa mengubah banyak aplikasi karena aplikasi mengakses data lewat lapisan logis (SQL/view). |
| 7 | **Keamanan dan kontrol akses** | Siapa pun yang dapat membuka berkas dapat melihat seluruh data, termasuk data pribadi pelanggan. | DBMS memiliki autentikasi, otorisasi per pengguna/peran (`GRANT`/`REVOKE`), *view* yang menyembunyikan kolom sensitif, serta enkripsi. Bagian manajemen dapat membaca laporan, tetapi tidak dapat mengubah transaksi. |
| 8 | **Backup, pemulihan (*recovery*), dan *logging*** | Jika sistem mati saat menulis, berkas dapat rusak dan tidak ada cara mengembalikan keadaan konsisten. | DBMS memakai *write-ahead log* dan *checkpoint* sehingga data dapat dipulihkan ke kondisi konsisten terakhir setelah kegagalan. |
| 9 | **Berbagi data multi-pengguna** | Berkas sulit diakses serentak oleh aplikasi web, aplikasi kasir, dan bagian laporan. | Banyak aplikasi dan pengguna dapat mengakses basis data yang sama secara bersamaan dengan isolasi yang terjaga. |

### Kesimpulan Soal 1

Perusahaan membutuhkan DBMS karena sistem e-commerce menuntut **konsistensi** (data pelanggan/produk tunggal dan valid), **keandalan transaksi** (pesanan dan stok harus akurat meski diakses serentak), **kinerja** (indeks dan optimasi query untuk jutaan baris), dan **keamanan serta pemulihan data**. Seluruhnya sulit dan mahal dicapai dengan berkas terpisah. Masalah yang kini muncul pada sistem (lambatnya pencarian dan laporan) pun justru dapat ditangani dengan fitur DBMS seperti indeks, partisi, dan optimasi query, yang tidak tersedia pada penyimpanan berkas biasa.

---

## Jawaban Soal 2 — Dua Perubahan Skema untuk Meningkatkan Performa

Dua perubahan yang diusulkan:

1. **Partisi horizontal tabel `Transaksi` dan `Detail_Transaksi` berdasarkan `tanggal`** (mengatasi pertumbuhan tabel dan lambatnya pencarian/pelaporan).
2. **Tabel ringkasan penjualan (*summary table*) sebagai denormalisasi terkontrol** (mengatasi laporan bulanan yang menggabungkan empat tabel dan memproses banyak status).

> **Kaitan dengan materi (Pertemuan 3 dan 4):** Perubahan 1 adalah *Horizontal Splitting*, yang dipilih karena tabel sangat besar dan pemisahannya mengikuti kategori yang tersirat secara alami pada data (bulan/tanggal). Perubahan 2 adalah denormalisasi berupa *derived data* / tabel agregat (serupa *materialized view* dan *duplicating table*). Pengelolaan data denormalisasi mengikuti tiga cara pada materi: **trigger** (integritas terbaik, tetapi mahal), **application logic** dengan transaksi atomik, dan **batch reconciliation** (bila konsistensi 100% tidak diperlukan setiap saat). Pada *materialized view*, pemeliharaan dapat bersifat *immediate* (diperbarui setiap data asli berubah) atau *deferred* (diperbarui periodik).

---

### Perubahan 1 — Partisi Tabel Berdasarkan Rentang Tanggal

**a. Struktur awal**

```sql
Transaksi(id_transaksi PK, id_pelanggan FK, tanggal, status, total_bayar)        -- 5.000.000 baris, satu tabel besar
Detail_Transaksi(id_detail PK, id_transaksi FK, id_produk FK, jumlah, harga_satuan) -- 20.000.000 baris, satu tabel besar
```

Seluruh data historis dan data terbaru berada dalam satu struktur fisik. Setiap query laporan atau pencarian harus berhadapan dengan seluruh tabel (5 juta dan 20 juta baris) dan terus membesar.

**b. Struktur yang diusulkan**

`Transaksi` dipartisi menurut rentang bulan pada kolom `tanggal` (*range partitioning*). Contoh dalam sintaks PostgreSQL:

```sql
CREATE TABLE Transaksi (
    id_transaksi  BIGINT,
    id_pelanggan  INT NOT NULL,
    tanggal       DATE NOT NULL,
    status        VARCHAR(20) NOT NULL,
    total_bayar   NUMERIC(14,2) NOT NULL,
    PRIMARY KEY (id_transaksi, tanggal)          -- kolom partisi wajib ikut dalam PK
) PARTITION BY RANGE (tanggal);

CREATE TABLE Transaksi_2026_01 PARTITION OF Transaksi
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE Transaksi_2026_02 PARTITION OF Transaksi
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- dan seterusnya, satu partisi per bulan
```

`Detail_Transaksi` ikut dipartisi dengan menambahkan kolom `tanggal_transaksi` (salinan `Transaksi.tanggal`) sebagai kunci partisi, sehingga partisi detail sejajar dengan partisi transaksi:

```sql
Detail_Transaksi(id_detail, id_transaksi, tanggal_transaksi, id_produk, jumlah, harga_satuan)
PARTITION BY RANGE (tanggal_transaksi);
```

> Alternatif tanpa menambah kolom: partisi `Detail_Transaksi` secara *hash* pada `id_transaksi`. Namun *partition pruning* berdasarkan bulan tidak didapat, sehingga manfaatnya lebih kecil.

**c. Alasan perubahan meningkatkan performa**

- **Partition pruning**: query laporan Januari 2026 (`tanggal >= '2026-01-01' AND tanggal < '2026-02-01'`) hanya membaca partisi `Transaksi_2026_01` dan mengabaikan partisi lain. Dari 5 juta baris, yang dipindai hanya sekitar satu bulan data (kira-kira beberapa persen dari total).
- **Indeks lebih kecil per partisi**: indeks B-Tree di tiap partisi lebih dangkal dan lebih muat di memori (*buffer cache*), sehingga pencarian lebih cepat.
- **Pemeliharaan lebih ringan**: `VACUUM`, `ANALYZE`, dan *rebuild index* dilakukan per partisi. Data lama dapat diarsipkan atau dihapus dengan `DROP/DETACH PARTITION` secara instan, tanpa `DELETE` jutaan baris.
- **Peluang paralelisme**: partisi dapat dipindai paralel.
- **Pertumbuhan data terkendali**: data baru selalu masuk ke partisi terbaru yang kecil dan "panas", sedangkan data lama bersifat statis.

**d. Dampak terhadap konsistensi data dan proses pembaruan**

| Aspek | Dampak |
|---|---|
| Konsistensi hasil query | **Tidak berubah**. Partisi adalah perubahan struktur fisik, bukan logis. Hasil `SELECT` tetap sama. |
| Integritas referensial | Perlu perhatian. Kolom kunci partisi harus ikut dalam PK/UNIQUE, dan beberapa DBMS (misalnya MySQL) tidak mendukung FOREIGN KEY pada tabel berpartisi. Pada kasus itu integritas harus dijaga di lapisan aplikasi atau dengan *trigger*. |
| Redundansi baru | `tanggal_transaksi` di `Detail_Transaksi` menggandakan `Transaksi.tanggal`. Nilai ini harus selalu sama. Risiko inkonsistensi dihindari dengan FK komposit `(id_transaksi, tanggal_transaksi) → Transaksi(id_transaksi, tanggal)` atau *trigger* yang menyalin nilai saat INSERT. |
| Pembaruan (UPDATE) | Mengubah `tanggal` sebuah transaksi dapat memindahkan baris ke partisi lain (*row movement*), yang lebih mahal. Untungnya tanggal transaksi jarang diubah. UPDATE pada `status` tetap murah karena baris tetap di partisi yang sama. |
| INSERT | Tetap cepat karena diarahkan otomatis ke partisi terbaru. Partisi bulan baru harus dibuat sebelum bulan tiba (otomatis lewat `pg_partman`/*scheduled job*). |

---

### Perubahan 2 — Tabel Ringkasan Penjualan Bulanan per Kategori (Denormalisasi Terkontrol)

**a. Struktur awal**

Laporan penjualan dihitung langsung dari empat tabel normal:

```
Pelanggan (500 rb) ⋈ Transaksi (5 jt) ⋈ Detail_Transaksi (20 jt) ⋈ Produk (200 rb)
```

Setiap kali manajemen membuka laporan, DBMS harus menggabungkan dan mengagregasi ulang jutaan baris, lalu baru menyaring status dan periode.

**b. Struktur yang diusulkan**

Menambahkan tabel agregat baru (atau *materialized view*) yang menyimpan hasil agregasi per periode dan kategori, hanya untuk transaksi berstatus **Selesai**:

```sql
CREATE TABLE Ringkasan_Penjualan_Bulanan (
    periode          DATE         NOT NULL,   -- tanggal awal bulan, mis. 2026-01-01
    kategori         VARCHAR(50)  NOT NULL,
    total_penjualan  NUMERIC(18,2) NOT NULL,
    jumlah_transaksi INT          NOT NULL,
    diperbarui_pada  TIMESTAMP    NOT NULL DEFAULT now(),
    PRIMARY KEY (periode, kategori)
);

-- Pengisian / refresh satu bulan
INSERT INTO Ringkasan_Penjualan_Bulanan (periode, kategori, total_penjualan, jumlah_transaksi)
SELECT DATE '2026-01-01', P.kategori,
       SUM(D.jumlah * D.harga_satuan),
       COUNT(DISTINCT T.id_transaksi)
FROM Transaksi T
JOIN Detail_Transaksi D ON D.id_transaksi = T.id_transaksi
JOIN Produk P           ON P.id_produk    = D.id_produk
WHERE T.status = 'Selesai'
  AND T.tanggal >= '2026-01-01' AND T.tanggal < '2026-02-01'
GROUP BY P.kategori
ON CONFLICT (periode, kategori) DO UPDATE
   SET total_penjualan = EXCLUDED.total_penjualan,
       jumlah_transaksi = EXCLUDED.jumlah_transaksi,
       diperbarui_pada = now();
```

Tabel dasar (`Pelanggan`, `Produk`, `Transaksi`, `Detail_Transaksi`) **tetap tidak berubah** dan tetap menjadi sumber kebenaran (*single source of truth*).

**c. Alasan perubahan meningkatkan performa**

- Laporan menjadi `SELECT ... FROM Ringkasan_Penjualan_Bulanan WHERE periode = '2026-01-01'` yang hanya membaca puluhan atau ratusan baris (jumlah kategori), bukan memproses jutaan baris dan 3 operasi join.
- Biaya agregasi dibayar **sekali** saat refresh (misalnya malam hari), bukan setiap kali laporan dibuka.
- Beban pada tabel transaksi (OLTP) berkurang karena laporan tidak lagi bersaing dengan pelanggan yang sedang berbelanja.

**d. Dampak terhadap konsistensi data dan proses pembaruan**

| Aspek | Dampak | Cara Mengendalikan |
|---|---|---|
| Redundansi | Total penjualan kini tersimpan dua kali (di detail dan di ringkasan). | Ringkasan dianggap *derived data*, bukan sumber utama, dan dapat dibangun ulang kapan saja dari tabel dasar. |
| Risiko data basi (*stale*) | Jika transaksi berubah status (Selesai → Dibatalkan/Retur) setelah ringkasan dibuat, angka ringkasan menjadi salah. | Refresh terjadwal untuk bulan berjalan, *refresh* ulang bulan yang terdampak ketika ada perubahan status/retur, atau pembaruan inkremental dengan *trigger*. Bulan yang sudah ditutup (*closing*) dikunci dan hanya dikoreksi lewat prosedur resmi. |
| Beban pembaruan | Setiap perubahan transaksi Selesai membutuhkan pembaruan tambahan jika memakai trigger. | Lebih ringan memakai refresh berkala (*batch*), atau `REFRESH MATERIALIZED VIEW CONCURRENTLY`. |
| Kebenaran hasil | Hasil laporan harus **identik** dengan query pada tabel dasar. | Validasi rekonsiliasi: bandingkan `SUM` ringkasan dengan query langsung pada sampel bulan. Proses refresh dan pembaruan data dijalankan dalam satu transaksi agar atomik. |

---

### Catatan Tambahan: Redundansi `total_bayar`

`Transaksi.total_bayar` dapat dihitung dari `SUM(jumlah * harga_satuan)` pada `Detail_Transaksi`, tetapi juga disimpan. Ini adalah **redundansi turunan** yang menimbulkan risiko anomali pembaruan. Jika detail diubah dan `total_bayar` tidak ikut diperbarui, datanya tidak konsisten. Karena riwayat transaksi sering dibaca, nilai ini sebaiknya **tetap disimpan** (denormalisasi yang disengaja untuk kecepatan baca), asalkan dijaga konsistensinya. Pada materi, ini adalah contoh *Adding Derived Columns*: keuntungannya menghilangkan join dan agregasi berulang, kerugiannya butuh penyimpanan dan pemeliharaan tambahan. Dari sisi integritas, *trigger* adalah solusi terbaik walau mahal dari sisi performa.

- Pembuatan/perubahan `Detail_Transaksi` dan pembaruan `total_bayar` dilakukan **dalam satu transaksi ACID**, atau dengan *trigger*.
- Pemeriksaan berkala (*audit query*):

  ```sql
  SELECT T.id_transaksi
  FROM Transaksi T
  JOIN (SELECT id_transaksi, SUM(jumlah*harga_satuan) AS hitung
        FROM Detail_Transaksi GROUP BY id_transaksi) X USING (id_transaksi)
  WHERE T.total_bayar <> X.hitung;
  ```

- `harga_satuan` pada `Detail_Transaksi` sudah tepat disimpan sebagai harga saat transaksi (*snapshot*), karena harga di `Produk` dapat berubah. Ini bukan redundansi yang salah.

---

## Jawaban Soal 3 — Normalisasi vs Denormalisasi untuk Laporan Penjualan Bulanan

### Pengertian Singkat

- **Normalisasi**: menyusun tabel agar redundansi minimal (1NF, 2NF, 3NF, BCNF) sehingga anomali *insert/update/delete* dapat dihindari.
- **Denormalisasi**: sengaja menambahkan redundansi (menggabungkan tabel, menyimpan nilai turunan atau agregat) untuk mempercepat pembacaan dengan mengurangi *join* dan agregasi saat query.

### Kondisi Skema Saat Ini

Skema `Pelanggan`, `Produk`, `Transaksi`, `Detail_Transaksi` sudah **cukup ternormalisasi** (kira-kira 3NF). Atribut non-kunci bergantung pada kunci tabelnya masing-masing, dan relasi *one-to-many* dipisah dengan benar. Artinya, memperlambat laporan **bukan karena kurang normal**, tetapi karena volume data besar dan query harus melakukan join serta agregasi berulang.

### Perbandingan

| Aspek | Normalisasi (dipertahankan/ditingkatkan) | Denormalisasi (terkontrol) |
|---|---|---|
| **Kecepatan laporan bulanan** | Lebih lambat. Perlu join 3–4 tabel dan agregasi `SUM` atas jutaan baris setiap kali laporan dibuat. | Jauh lebih cepat. Data sudah tergabung atau teragregasi sehingga join/agregasi dikurangi atau dihilangkan. |
| **Kebenaran hasil** | Paling terjamin karena hanya ada satu sumber nilai. | Dapat salah jika salinan redundan tidak sinkron (data basi). Perlu mekanisme sinkronisasi. |
| **Redundansi data** | Minimal. | Lebih tinggi. |
| **Penggunaan ruang penyimpanan** | Hemat. | Lebih besar (tabel ringkasan atau kolom tambahan). |
| **Biaya INSERT/UPDATE/DELETE** | Ringan, karena satu fakta disimpan di satu tempat. | Lebih berat, karena beberapa salinan harus diperbarui atau di-*refresh*. |
| **Risiko anomali pembaruan** | Sangat rendah. | Ada, jika tidak dikelola. |
| **Kompleksitas pengelolaan** | Sederhana. | Lebih kompleks (jadwal refresh, trigger, validasi). |
| **Cocok untuk** | Beban transaksional (OLTP): pemesanan, pembayaran, stok. | Beban analitik/pelaporan (OLAP): laporan bulanan, dashboard. |
| **Contoh pada kasus** | Tetap memakai 4 tabel dengan FK dan indeks yang tepat, serta query yang dioptimalkan. | Tabel `Ringkasan_Penjualan_Bulanan`, kolom `kategori` disalin ke detail, atau `total_bayar` tersimpan. |

### Teknik Denormalisasi pada Materi dan Kecocokannya

| Teknik (Pertemuan 4) | Penerapan pada kasus | Kecocokan |
|---|---|---|
| *Adding redundant columns* (hapus join yang sering dipakai) | Menyalin `kategori` ke `Detail_Transaksi` agar tidak join ke `Produk` | Bisa, tetapi menambah redundansi pada 20 juta baris dan harus disinkronkan |
| *Adding derived columns* (hasil hitung/agregat disimpan) | `total_bayar` (sudah ada) | Sudah diterapkan, risiko inkonsistensi perlu dijaga |
| *Collapsing tables* (gabung tabel, syarat relasi *one-to-one*) | Menggabung `Transaksi` dan `Detail_Transaksi` | **Tidak cocok**: relasinya *one-to-many*, penggabungan menggandakan data header di tiap baris detail |
| *Duplicating tables* / tabel ringkasan / *materialized view* | `Ringkasan_Penjualan_Bulanan` | **Paling cocok** untuk laporan bulanan |

Menurut materi, denormalisasi tepat dipakai ketika **banyak query membaca dan jarang ada pembaruan**, membutuhkan akses cepat dalam jumlah besar (seperti *data warehouse*/OLAP), dan aplikasi dapat menangani pengelolaan konsistensi. Laporan bulanan memenuhi kriteria ini, sedangkan transaksi pemesanan dan pembayaran (OLTP) tetap lebih cocok pada skema ternormalisasi.

### Analisis dan Rekomendasi

Syarat soal: laporan harus **lebih cepat** namun **hasilnya tidak berubah**. Maka:

1. **Pertahankan normalisasi pada tabel utama** sebagai sumber kebenaran data transaksi. Alasannya, kebenaran data tetap yang utama, dan tabel utama menanggung beban penulisan tinggi.
2. **Optimasi lebih dulu tanpa mengubah desain**: indeks tepat, partisi, dan penulisan ulang query (hapus join `Pelanggan` yang tidak perlu, saring status/tanggal lebih awal). Cara ini tidak menambah redundansi dan hasilnya terjamin identik.
3. **Tambahkan denormalisasi terkontrol hanya untuk jalur laporan**, yaitu tabel ringkasan atau *materialized view* yang bisa dibangun ulang dari tabel dasar, sehingga risiko inkonsistensi dapat dikendalikan.
4. **Jaga kebenaran hasil** dengan: refresh dalam transaksi, filter `status = 'Selesai'` yang sama dengan query asli, penanganan perubahan status/retur, dan rekonsiliasi berkala antara ringkasan dan query langsung.

**Kesimpulan:** Normalisasi unggul dalam **kebenaran dan efisiensi pembaruan**, sedangkan denormalisasi unggul dalam **kecepatan baca laporan**. Solusi terbaik adalah **hibrida**: basis data inti tetap ternormalisasi, ditambah lapisan denormalisasi terkontrol (ringkasan) khusus untuk pelaporan. Dengan begitu laporan lebih cepat tanpa mengorbankan keakuratan hasil.

---

## Jawaban Soal 4 — Mengapa Indeks pada Seluruh Kolom Bukan Strategi yang Selalu Tepat

### Kondisi Awal

Query:

```sql
SELECT id_transaksi, tanggal, total_bayar
FROM Transaksi
WHERE id_pelanggan = 12580 AND status = 'Selesai'
ORDER BY tanggal DESC;
```

Hanya ada indeks pada `id_transaksi` (PK), padahal filter memakai `id_pelanggan` dan `status`. Akibatnya DBMS melakukan *full table scan* pada 5.000.000 baris lalu mengurutkannya (`filesort`). Wajar jika terpikir untuk "memberi indeks pada semua kolom", tetapi itu bukan strategi yang selalu tepat karena alasan berikut.

### Alasan Utama

1. **Memperlambat operasi tulis (INSERT, UPDATE, DELETE).**
   Setiap indeks adalah struktur data tersendiri (misalnya B-Tree) yang harus diperbarui ketika data berubah. Tabel `Transaksi` bertambah terus (pesanan baru) dan statusnya sering berubah (Menunggu → Dibayar → Dikirim → Selesai). Dengan 5 kolom ber-indeks, satu `INSERT` berarti 5 penulisan indeks, dan `UPDATE status` memperbarui indeks `status` (serta bisa memicu *page split*). Sistem e-commerce yang beban tulisnya tinggi justru bisa melambat.

2. **Boros ruang penyimpanan dan memori.**
   Setiap indeks menyimpan salinan nilai kolom plus penunjuk baris. Pada 5 juta baris, indeks yang berlebihan dapat menyamai atau melebihi ukuran tabel, serta memenuhi *buffer cache* sehingga data yang benar-benar penting terdorong keluar dari memori.

3. **Tidak semua indeks berguna (selektivitas rendah).**
   Kolom `status` hanya memiliki sedikit nilai berbeda (Menunggu, Dibayar, Dikirim, Selesai, Batal), dan (dengan asumsi) sebagian besar baris bernilai `Selesai`. Indeks tunggal pada kolom berselektivitas rendah jarang dipakai optimizer, karena membaca indeks lalu meloncat ke jutaan baris tabel lebih mahal daripada *full scan*. Begitu pula `total_bayar` yang hampir tidak pernah dipakai sebagai filter pada query ini.

4. **Indeks tunggal tidak melayani query ini secara optimal.**
   Query memakai kombinasi `id_pelanggan` (filter) + `status` (filter) + `tanggal` (pengurutan). Optimizer umumnya hanya memakai satu indeks tunggal yang paling selektif. Menggabungkan beberapa indeks (*bitmap/index merge*) dimungkinkan pada sebagian DBMS, tetapi lebih mahal dan tidak menghilangkan langkah sortir untuk `ORDER BY tanggal DESC`. Yang tepat adalah **satu indeks komposit** yang dirancang sesuai pola query.

5. **Beban pada optimizer dan pemeliharaan.**
   Semakin banyak indeks, semakin banyak pilihan rencana yang harus dievaluasi optimizer, dan semakin besar kemungkinan ia memilih rencana kurang optimal. Indeks juga harus dipantau, di-*rebuild* saat terfragmentasi, dan ikut menambah waktu *backup*/pemulihan.

6. **Indeks yang tidak terpakai hanya menjadi beban.**
   Indeks tanpa query yang memanfaatkannya memberi biaya penuh (tulis, ruang, pemeliharaan) tanpa manfaat.

### Strategi yang Tepat

Sesuai materi *Tuning Index* (Pertemuan 3) dan *Use Indexes Wisely* (Pertemuan 7), tuning indeks meliputi: **membuat indeks yang tepat, menghapus indeks yang berlebihan, memilih jenis indeks yang sesuai (mis. *clustered* vs *non-clustered*, B-Tree vs Hash), dan melakukan analisis beban kerja (*workload analysis*)**. Indeks dibuat untuk kolom yang sering muncul di `WHERE`, `JOIN`, `ORDER BY`, dan `GROUP BY`, bukan pada semua kolom.

Rancang indeks berdasarkan **pola query nyata (*workload*)**.

Untuk query di atas, indeks komposit yang sesuai:

```sql
-- PostgreSQL / SQL Server (kolom non-kunci disertakan agar covering)
CREATE INDEX idx_transaksi_pelanggan_status_tanggal
ON Transaksi (id_pelanggan, status, tanggal DESC)
INCLUDE (total_bayar);

-- MySQL (tanpa INCLUDE; id_transaksi PK otomatis ikut di indeks sekunder InnoDB)
CREATE INDEX idx_transaksi_pelanggan_status_tanggal
ON Transaksi (id_pelanggan, status, tanggal, total_bayar);
```

**Mengapa urutan kolom demikian?**

- `id_pelanggan` lebih dulu karena kondisi kesamaan (`=`) dengan selektivitas tinggi, sehingga langsung mempersempit ke sebagian kecil baris milik satu pelanggan.
- `status` berikutnya karena juga kondisi kesamaan.
- `tanggal` terakhir agar baris di dalam kombinasi `(id_pelanggan, status)` **sudah terurut menurut tanggal**, sehingga `ORDER BY tanggal DESC` dibaca langsung dari indeks tanpa sortir tambahan.
- `total_bayar` (dan `id_transaksi` yang sudah ada) membuat indeks bersifat **covering**: seluruh kolom pada `SELECT` terpenuhi dari indeks, tanpa membaca tabel utama (*index-only scan*).

Hasil: dari *full scan* 5 juta baris + sortir menjadi pencarian indeks `O(log n)` yang hanya membaca beberapa puluh baris milik pelanggan tersebut. Satu indeks yang tepat ini jauh lebih bermanfaat daripada lima indeks sembarangan.

**Prinsip umum:** indeks adalah *trade-off* antara **kecepatan baca** dan **biaya tulis + ruang**. Buat indeks hanya untuk kolom yang sering dipakai di `WHERE`, `JOIN`, `ORDER BY`, dan `GROUP BY`, utamakan indeks komposit sesuai pola query, serta evaluasi secara berkala dengan `EXPLAIN` dan statistik penggunaan indeks.

> Catatan: indeks untuk pencarian email (`CREATE UNIQUE INDEX idx_pelanggan_email ON Pelanggan(email)`) memang diperlukan, karena pencarian berdasarkan email sering dan nilainya sangat selektif. Ini contoh indeks yang tepat sasaran.

---

## Jawaban Soal 5 — Optimasi Query Total Penjualan per Kategori (Januari 2026)

### Persiapan: Notasi yang Digunakan

| Simbol | Makna |
|---|---|
| σ | Seleksi (memilih baris sesuai kondisi) |
| π | Proyeksi (memilih kolom) |
| ⋈ | Join (*theta join*; bila kondisinya kesamaan atribut kunci, ditulis dengan nama atribut saja) |
| 𝒢 | Agregasi dengan pengelompokan: `kolom_kelompok 𝒢 fungsi_agregat(...)` |

Alias: C = Pelanggan, T = Transaksi, D = Detail_Transaksi, P = Produk.

Aturan ekuivalensi yang dipakai mengikuti numbering materi Pertemuan 7: **Rule-1** cascade seleksi, **Rule-2** komutatif seleksi, **Rule-3** cascade proyeksi, **Rule-4** seleksi–proyeksi dapat ditukar, **Rule-5** komutatif theta join, **Rule-6** seleksi didistribusikan ke dalam join, **Rule-7** proyeksi didistribusikan ke dalam join, **Rule-9** asosiatif join.

---

### 5a. Pengaruh Urutan Pelaksanaan Operasi terhadap Biaya Pemrosesan (6 poin)

Beberapa urutan operasi dapat **ekuivalen secara logis (hasil sama)** tetapi **biayanya sangat berbeda**. Biaya terutama ditentukan oleh **ukuran hasil antara** (*intermediate result*). Prinsip heuristik pada materi: **perkecil hasil antara sebelum operasi yang mahal seperti JOIN**, yaitu seleksi lebih awal, proyeksi lebih awal, dan join belakangan.

**Poin 1 — Seleksi didahulukan (Rule-6).**
Pada query awal, kondisi `status = 'Selesai'` dan rentang tanggal Januari 2026 secara logis dikenakan setelah keempat tabel di-join, sehingga jutaan baris digabung lalu dibuang. Bila seleksi dilakukan lebih awal pada `Transaksi`, baris yang ikut join menyusut drastis.

*Ilustrasi (asumsi, hanya untuk perbandingan):* 5 juta transaksi tersebar ±5 tahun (±60 bulan) → ±83.000 transaksi/bulan; ±70% berstatus Selesai → ±58.000 transaksi. Rata-rata detail per transaksi = 20.000.000 / 5.000.000 = 4 baris.

| Strategi | Transaksi yang di-join ke Detail_Transaksi | Perkiraan baris hasil join |
|---|---|---|
| Join dahulu, seleksi belakangan | ±5.000.000 (semua) | ±20.000.000 baris, baru kemudian difilter |
| Seleksi dahulu, join belakangan | ±58.000 | ±232.000 baris |

Selisihnya sekitar 2 orde besaran (±86 kali lebih sedikit baris).

**Poin 2 — Proyeksi didahulukan (Rule-3, 4, 7).**
Tiap tabel hanya dibutuhkan beberapa kolomnya (mis. `Detail_Transaksi` cukup `id_transaksi, id_produk, jumlah, harga_satuan`). Membuang kolom yang tidak dipakai (`email`, `nama_produk`, `stok`, `total_bayar`, dst.) membuat baris antara lebih sempit, sehingga lebih banyak baris muat per blok/halaman dan I/O berkurang. Kolom untuk join dan agregasi tetap harus dipertahankan.

**Poin 3 — Urutan dan pengelompokan join (Rule-5, Rule-9).**
Join bersifat komutatif dan asosiatif, sehingga urutan `((C ⋈ T) ⋈ D) ⋈ P` boleh diubah. Gabungkan lebih dulu relasi yang menghasilkan baris paling sedikit (T hasil seleksi), baru ke relasi besar. Urutan buruk, misalnya `Pelanggan` (500.000) dengan seluruh `Transaksi` (5 juta) di awal, menghasilkan hasil antara besar tanpa manfaat. Hindari pula hasil kali Kartesius, yang menghasilkan m × n pasangan baris.

**Poin 4 — Pemilihan algoritma dan pemanfaatan indeks.**
Setiap operasi punya beberapa algoritma: seleksi (*linear scan* vs *index scan*), join (*nested loop* vs *sort-merge* vs *hash join*), agregasi (sorting vs hash). Urutan operasi menentukan apakah indeks dapat dipakai. Seleksi awal pada `Transaksi` dapat memakai indeks `(status, tanggal)`, dan hasil kecilnya dapat di-join ke `Detail_Transaksi` lewat indeks `id_transaksi` (*index nested loop*). Bila seleksi terlambat (setelah join), peluang indeks hilang dan DBMS terpaksa memindai seluruh tabel.

**Poin 5 — Komponen biaya (cost-based optimization).**
Hasil antara yang besar menaikkan beberapa komponen biaya sekaligus: *disk I/O* (membaca/menulis blok), penyimpanan sementara, komputasi (sorting, hashing, pembandingan), dan penggunaan memori. Jika hasil antara melebihi memori, DBMS menulisnya ke disk sehingga jauh lebih lambat. Optimizer memilih rencana dengan estimasi biaya terendah berdasarkan statistik (jumlah tuple, nilai unik atribut). Urutan operasi yang baik juga memungkinkan *pipelining* (hasil langsung diteruskan ke operasi berikutnya) dibanding *materialization* (hasil antara disimpan dulu).

**Poin 6 — Menghilangkan operasi yang tidak perlu (optimasi lanjutan).**
Tabel `Pelanggan` di-join tetapi tidak ada kolomnya yang dipakai di `SELECT`, `WHERE`, maupun `GROUP BY`. Jika `Transaksi.id_pelanggan` adalah *foreign key* NOT NULL dengan integritas referensial terjaga, join ini tidak menambah atau mengurangi baris sehingga hanya menjadi biaya (membaca 500.000 baris + satu operasi join). Ini bukan aturan aljabar murni, melainkan eliminasi join berdasarkan batasan integritas, sehingga syaratnya harus dipastikan agar hasil tetap benar.

**Kesimpulan 5a:** semua urutan menghasilkan hasil yang sama, tetapi biayanya dapat berbeda berkali-kali lipat. Heuristiknya: **seleksi dan proyeksi sedini mungkin, join belakangan dengan urutan dari relasi terkecil, hindari Kartesius, dan pilih algoritma/indeks yang sesuai.**

---

### 5b. Ekspresi Aljabar Relasional Sebelum Optimasi (6 poin)

Pemetaan SQL: `FROM/JOIN` → join, `WHERE` → seleksi, `GROUP BY + SUM` → agregasi, `SELECT` → proyeksi.

**Langkah penyusunan:**

1. `C ⋈ T` pada `C.id_pelanggan = T.id_pelanggan`.
2. Hasilnya di-join dengan `D` pada `T.id_transaksi = D.id_transaksi`.
3. Hasilnya di-join dengan `P` pada `D.id_produk = P.id_produk`.
4. Seleksi: `T.status = 'Selesai' ∧ T.tanggal ≥ '2026-01-01' ∧ T.tanggal < '2026-02-01'`.
5. Agregasi: kelompokkan menurut `P.kategori`, hitung `SUM(D.jumlah × D.harga_satuan)` sebagai `total_penjualan`.
6. Proyeksi: `kategori, total_penjualan`.

**Ekspresi aljabar relasional (sebelum optimasi):**

```
π P.kategori, total_penjualan (
    P.kategori 𝒢 SUM(D.jumlah × D.harga_satuan) → total_penjualan (
        σ T.status = 'Selesai'
           ∧ T.tanggal ≥ '2026-01-01'
           ∧ T.tanggal < '2026-02-01' (
            ( ( C ⋈ C.id_pelanggan = T.id_pelanggan  T )
                  ⋈ T.id_transaksi = D.id_transaksi  D )
                  ⋈ D.id_produk = P.id_produk        P
        )
    )
)
```

Bentuk ini ekuivalen dengan menuliskan seleksi di atas hasil kali Kartesius, yaitu `σ (kondisi join ∧ kondisi filter) (C × T × D × P)`. Menurut materi, seleksi di atas Kartesius diubah menjadi theta join (`σθ(E1 × E2) ≡ E1 ⋈θ E2`), sehingga bentuk di atas sudah memakai join.

**Pohon query (sebelum optimasi)**: seleksi berada di atas semua join, sehingga seluruh data digabung dahulu.

```
π kategori, total_penjualan
└─ 𝒢 kategori ; SUM(jumlah × harga_satuan)
   └─ σ status='Selesai' ∧ tanggal ≥ '2026-01-01' ∧ tanggal < '2026-02-01'
      └─ ⋈ D.id_produk = P.id_produk
         ├─ ⋈ T.id_transaksi = D.id_transaksi
         │  ├─ ⋈ C.id_pelanggan = T.id_pelanggan
         │  │  ├─ Pelanggan C (500.000)
         │  │  └─ Transaksi T (5.000.000)
         │  └─ Detail_Transaksi D (20.000.000)
         └─ Produk P (200.000)
```

---

### 5c. Optimasi Ekspresi Aljabar Relasional

Optimasi memakai **aturan ekuivalensi** sehingga hasil tetap identik dengan query awal. Hasilnya disajikan dalam dua versi: **Versi A** (hanya aturan aljabar pada materi, tetap memuat `Pelanggan`) dan **Versi B** (optimasi lanjutan).

#### Langkah 1 — Pecah seleksi konjungtif (Rule-1, sigma-cascade)

```
σ status='Selesai' ( σ tanggal ≥ '2026-01-01' ( σ tanggal < '2026-02-01' ( … ) ) )
```

Tujuan: tiap kondisi dapat dipindahkan secara terpisah ke relasi asalnya.

#### Langkah 2 — Atur urutan seleksi (Rule-2, komutatif)

Seleksi bersifat komutatif, sehingga dalam praktik kondisi yang paling sedikit meloloskan baris diterapkan lebih dulu. Rentang tanggal Januari (±1 dari 60 bulan, ±1,7% baris) jauh lebih selektif daripada `status = 'Selesai'` (±70% baris), sehingga seleksi tanggal didahulukan.

#### Langkah 3 — Dorong seleksi ke relasi dasar (Rule-6)

Ketiga kondisi hanya memakai atribut `Transaksi`, sehingga `σc(C ⋈ T ⋈ D ⋈ P) ≡ C ⋈ σc(T) ⋈ D ⋈ P`:

```
T' = σ status='Selesai' ∧ tanggal ≥ '2026-01-01' ∧ tanggal < '2026-02-01' ( Transaksi )
```

Dari ±5.000.000 baris menjadi ±58.000 baris.

#### Langkah 4 — Dorong proyeksi (Rule-3, 4, 7)

Atribut untuk join dan agregasi tetap dipertahankan:

```
T'' = π id_transaksi, id_pelanggan ( T' )
C'  = π id_pelanggan ( Pelanggan )
D'  = π id_transaksi, id_produk, jumlah, harga_satuan ( Detail_Transaksi )
P'  = π id_produk, kategori ( Produk )
```

#### Langkah 5 — Susun ulang join (Rule-5 dan Rule-9)

Mulai dari relasi terkecil (T'' ≈ 58.000 baris), lalu ke relasi berikutnya lewat atribut kunci yang berindeks:

```
( ( T'' ⋈ id_pelanggan  C' ) ⋈ id_transaksi  D' ) ⋈ id_produk  P'
```

#### Versi A — Ekspresi Akhir (Sesuai Materi)

```
π kategori, total_penjualan (
    kategori 𝒢 SUM(jumlah × harga_satuan) → total_penjualan (
        (
            (
                π id_transaksi, id_pelanggan (
                    σ status = 'Selesai'
                       ∧ tanggal ≥ '2026-01-01'
                       ∧ tanggal < '2026-02-01' ( Transaksi ) )
                ⋈ id_pelanggan  π id_pelanggan ( Pelanggan )
            )
            ⋈ id_transaksi  π id_transaksi, id_produk, jumlah, harga_satuan ( Detail_Transaksi )
        )
        ⋈ id_produk  π id_produk, kategori ( Produk )
    )
)
```

**Pohon query Versi A**: seleksi dan proyeksi berada di dasar pohon, join belakangan.

```
π kategori, total_penjualan
└─ 𝒢 kategori ; SUM(jumlah × harga_satuan)
   └─ ⋈ id_produk
      ├─ ⋈ id_transaksi
      │  ├─ ⋈ id_pelanggan
      │  │  ├─ π id_transaksi, id_pelanggan
      │  │  │  └─ σ tanggal ≥ '2026-01-01' ∧ tanggal < '2026-02-01' ∧ status='Selesai'
      │  │  │     └─ Transaksi (5.000.000)
      │  │  └─ π id_pelanggan
      │  │     └─ Pelanggan (500.000)
      │  └─ π id_transaksi, id_produk, jumlah, harga_satuan
      │     └─ Detail_Transaksi (20.000.000)
      └─ π id_produk, kategori
         └─ Produk (200.000)
```

#### Versi B — Optimasi Lanjutan (di Luar Aturan Dasar Materi)

Dua langkah tambahan yang tetap menjaga hasil identik, dengan syarat yang disebutkan:

**B1 — Hapus join redundan dengan `Pelanggan`.** Syarat: `Transaksi.id_pelanggan` adalah FK NOT NULL dengan integritas referensial terjaga, sehingga `C ⋈ T` tidak menambah atau mengurangi baris. Jika ada transaksi yatim tanpa pelanggan, query awal akan membuangnya sehingga hasil bisa berbeda.

**B2 — Agregasi parsial per produk.** Karena `id_produk → kategori` (satu produk satu kategori) dan `SUM` dapat didekomposisi, jumlahkan dulu per `id_produk`, baru gabungkan dengan `Produk`:

```
π kategori, total_penjualan (
    kategori 𝒢 SUM(subtotal) → total_penjualan (
        (
            id_produk 𝒢 SUM(jumlah × harga_satuan) → subtotal (
                π id_transaksi ( σ status = 'Selesai'
                                    ∧ tanggal ≥ '2026-01-01'
                                    ∧ tanggal < '2026-02-01' ( Transaksi ) )
                ⋈ id_transaksi
                π id_transaksi, id_produk, jumlah, harga_satuan ( Detail_Transaksi )
            )
        )
        ⋈ id_produk  π id_produk, kategori ( Produk )
    )
)
```

#### Ringkasan Aturan yang Dipakai

| Langkah | Aturan / Heuristik | Manfaat |
|---|---|---|
| 1 | Rule-1: cascade seleksi | Tiap kondisi dapat didorong turun. |
| 2 | Rule-2: komutatif seleksi | Kondisi paling selektif dijalankan lebih dulu. |
| 3 | Rule-6: seleksi didistribusikan ke join | Transaksi turun dari ±5 juta menjadi ±58 ribu baris sebelum join. |
| 4 | Rule-3, 4, 7: proyeksi didorong ke bawah | Baris antara lebih sempit, I/O berkurang. |
| 5 | Rule-5 dan Rule-9: komutatif dan asosiatif join | Join dimulai dari relasi terkecil. |
| B1 | Eliminasi join redundan (berdasar FK) | Hemat pembacaan 500.000 baris dan satu join. |
| B2 | Agregasi parsial (berdasar `id_produk → kategori`) | Baris yang di-join ke Produk berkurang. |

#### Query SQL Hasil Optimasi

**Versi A:**

```sql
SELECT P.kategori,
       SUM(D.jumlah * D.harga_satuan) AS total_penjualan
FROM (SELECT id_transaksi, id_pelanggan
      FROM Transaksi
      WHERE tanggal >= '2026-01-01'
        AND tanggal <  '2026-02-01'
        AND status = 'Selesai') AS T
JOIN Pelanggan C ON C.id_pelanggan = T.id_pelanggan
JOIN Detail_Transaksi D ON D.id_transaksi = T.id_transaksi
JOIN Produk P ON P.id_produk = D.id_produk
GROUP BY P.kategori;
```

**Versi B:**

```sql
SELECT P.kategori,
       SUM(A.subtotal) AS total_penjualan
FROM (
    SELECT D.id_produk,
           SUM(D.jumlah * D.harga_satuan) AS subtotal
    FROM Transaksi T
    JOIN Detail_Transaksi D ON D.id_transaksi = T.id_transaksi
    WHERE T.status = 'Selesai'
      AND T.tanggal >= '2026-01-01'
      AND T.tanggal <  '2026-02-01'
    GROUP BY D.id_produk
) AS A
JOIN Produk P ON P.id_produk = A.id_produk
GROUP BY P.kategori;
```

> Pada DBMS modern, optimizer sering melakukan *selection pushdown* otomatis. Menulis ulang secara eksplisit tetap berguna untuk memperjelas rencana, dan `EXPLAIN` dapat dipakai untuk memverifikasi rencana eksekusinya.

**Indeks pendukung rencana eksekusi:**

```sql
-- PostgreSQL / SQL Server (INCLUDE); di MySQL letakkan kolom tambahan di akhir daftar kolom indeks
CREATE INDEX idx_transaksi_status_tanggal ON Transaksi (status, tanggal) INCLUDE (id_transaksi);
CREATE INDEX idx_detail_transaksi ON Detail_Transaksi (id_transaksi) INCLUDE (id_produk, jumlah, harga_satuan);
-- Join ke Produk dan Pelanggan sudah dilayani PK id_produk dan id_pelanggan
```

#### Jaminan Kebenaran Hasil

- Memindahkan seleksi dan proyeksi ke bawah join tidak mengubah hasil karena kondisi hanya memakai atribut `Transaksi` dan atribut join/agregasi tetap dipertahankan (Rule-6, Rule-7).
- Mengubah urutan join aman karena join komutatif dan asosiatif (Rule-5, Rule-9).
- Versi B bergantung pada integritas referensial (B1) dan pada `id_produk → kategori` serta sifat `SUM` yang dapat didekomposisi (B2).
- Rentang tanggal memakai `>=` dan `<` (bukan `BETWEEN` atau fungsi `MONTH()`/`YEAR()`), sehingga tetap dapat memakai indeks (*sargable*) dan akurat untuk tipe `DATETIME`/`TIMESTAMP`.

**Kesimpulan 5c:** dengan seleksi dan proyeksi lebih awal serta urutan join dari relasi terkecil, biaya turun dari penggabungan ±20 juta baris menjadi ratusan ribu baris dengan hasil laporan tetap identik. Optimasi lanjutan (B1 dan B2) menurunkan biaya lebih jauh dengan syarat yang telah disebutkan.

---

*Catatan: angka dalam ilustrasi (±58.000 transaksi/bulan, ±70% berstatus Selesai, rata-rata 4 detail per transaksi, dsb.) adalah asumsi untuk memperlihatkan perbandingan biaya relatif. Nilai aktual bergantung pada distribusi data dan statistik yang dimiliki optimizer.*
