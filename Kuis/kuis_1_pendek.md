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
| 3 | **Kontrol konkurensi dan transaksi (ACID)** | Ribuan pelanggan memesan bersamaan. Dua pelanggan bisa membeli stok terakhir yang sama sehingga stok menjadi negatif, atau file rusak karena ditulis bersamaan. | DBMS memakai *locking*/MVCC dan transaksi ACID. Pembuatan pesanan (insert `Transaksi`, insert `Detail_Transaksi`, kurangi `stok`) dijalankan **atomik**: semua berhasil atau semua dibatalkan (*rollback*). |
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

`Transaksi.total_bayar` dapat dihitung dari `SUM(jumlah * harga_satuan)` pada `Detail_Transaksi`, tetapi juga disimpan. Ini adalah **redundansi turunan** yang menimbulkan risiko anomali pembaruan. Jika detail diubah dan `total_bayar` tidak ikut diperbarui, datanya tidak konsisten. Karena riwayat transaksi sering dibaca, nilai ini sebaiknya **tetap disimpan** (denormalisasi yang disengaja untuk kecepatan baca), asalkan dijaga konsistensinya:

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
   Kolom `status` hanya memiliki sedikit nilai berbeda (Menunggu, Dibayar, Dikirim, Selesai, Batal), dan sebagian besar baris bernilai `Selesai`. Indeks tunggal pada kolom berselektivitas rendah jarang dipakai optimizer, karena membaca indeks lalu meloncat ke jutaan baris tabel lebih mahal daripada *full scan*. Begitu pula `total_bayar` yang hampir tidak pernah dipakai sebagai filter pada query ini.

4. **Indeks tunggal tidak melayani query ini secara optimal.**
   Query memakai kombinasi `id_pelanggan` (filter) + `status` (filter) + `tanggal` (pengurutan). Beberapa indeks terpisah paling-paling dipilih salah satunya, dan pengurutan `ORDER BY tanggal DESC` tetap butuh langkah sortir. Yang tepat adalah **satu indeks komposit** yang dirancang sesuai pola query.

5. **Beban pada optimizer dan pemeliharaan.**
   Semakin banyak indeks, semakin banyak pilihan rencana yang harus dievaluasi optimizer, dan semakin besar kemungkinan ia memilih rencana kurang optimal. Indeks juga harus dipantau, di-*rebuild* saat terfragmentasi, dan ikut menambah waktu *backup*/pemulihan.

6. **Indeks yang tidak terpakai hanya menjadi beban.**
   Indeks tanpa query yang memanfaatkannya memberi biaya penuh (tulis, ruang, pemeliharaan) tanpa manfaat.

### Strategi yang Tepat

Rancang indeks berdasarkan **pola query nyata (*workload*)**, bukan memberi indeks pada semua kolom.

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
| ⋈ | Join (*natural/theta join*) |
| γ (atau 𝒢) | Agregasi dengan pengelompokan: `γ kolom_kelompok ; fungsi_agregat` |
| ρ | Penamaan ulang (*rename*) |

Alias: C = Pelanggan, T = Transaksi, D = Detail_Transaksi, P = Produk.

---

### 5a. Pengaruh Urutan Pelaksanaan Operasi terhadap Biaya Pemrosesan (6 poin)

Dalam aljabar relasional, beberapa urutan operasi yang berbeda dapat menghasilkan **hasil akhir yang sama** (ekuivalen), tetapi **biaya pemrosesannya sangat berbeda**. Biaya terutama ditentukan oleh **ukuran hasil antara** (*intermediate result*), yaitu jumlah baris dan lebar kolom yang harus dibaca, dibuat, disimpan, dan diteruskan ke operasi berikutnya (biaya I/O disk, CPU, dan memori). Prinsipnya: **kurangi ukuran data sedini mungkin**.

**Poin 1 — Seleksi harus didahulukan (*selection pushdown*).**
Pada query awal, kondisi `status = 'Selesai'` dan rentang tanggal Januari 2026 secara logis diterapkan setelah keempat tabel di-join. Jika join dikerjakan lebih dulu, DBMS menggabungkan jutaan baris yang kemudian dibuang. Jika seleksi dilakukan lebih awal pada `Transaksi`, jumlah baris yang ikut join menyusut drastis.

*Ilustrasi (asumsi, hanya untuk perbandingan):* data 5 juta transaksi tersebar sekitar 5 tahun (±60 bulan) sehingga ±83.000 transaksi per bulan, dan sekitar 70% berstatus Selesai, yaitu ±58.000 transaksi. Rata-rata detail per transaksi = 20.000.000 / 5.000.000 = 4 baris.

| Strategi | Baris yang harus di-join dengan Detail_Transaksi | Perkiraan baris hasil join |
|---|---|---|
| Join dahulu, filter belakangan | ±5.000.000 transaksi (semua) | ±20.000.000 baris (kemudian baru difilter) |
| Filter dahulu, join belakangan | ±58.000 transaksi (sudah terfilter) | ±232.000 baris |

Selisihnya sekitar **2 orde besaran** (±86 kali lebih sedikit baris yang diproses).

**Poin 2 — Proyeksi juga didahulukan (*projection pushdown*).**
Setiap tabel hanya dibutuhkan beberapa kolomnya: `Transaksi` cukup `id_transaksi`; `Detail_Transaksi` cukup `id_transaksi, id_produk, jumlah, harga_satuan`; `Produk` cukup `id_produk, kategori`. Membuang kolom yang tidak diperlukan (`email`, `nama_produk`, `stok`, `total_bayar`, dll.) memperkecil ukuran tiap baris antara sehingga lebih banyak baris muat dalam satu halaman memori/disk dan I/O berkurang.

**Poin 3 — Urutan join (*join ordering*) menentukan ukuran hasil antara.**
Join bersifat komutatif dan asosiatif, sehingga urutan `((C ⋈ T) ⋈ D) ⋈ P` boleh diubah. Urutan yang baik memulai dari relasi hasil seleksi yang **paling kecil** (T yang sudah difilter) lalu menggabungkannya ke relasi besar via kunci yang berindeks. Urutan buruk (misalnya menggabungkan `Pelanggan` 500.000 baris dengan seluruh `Transaksi` 5 juta baris di awal) menghasilkan hasil antara yang besar tanpa manfaat.

**Poin 4 — Menghilangkan join yang tidak perlu (*redundant join elimination*).**
Tabel `Pelanggan` (C) di-join tetapi **tidak ada kolomnya** yang dipakai pada `SELECT`, `WHERE`, maupun `GROUP BY`. Karena `Transaksi.id_pelanggan` adalah *foreign key* yang menjamin setiap transaksi memiliki tepat satu pelanggan (dengan asumsi NOT NULL dan integritas referensial terjaga), join ini tidak menambah, mengurangi, ataupun mengubah baris hasil. Join ini murni biaya tambahan (membaca 500.000 baris dan satu operasi join), sehingga dapat dibuang tanpa mengubah hasil.

**Poin 5 — Agregasi sebaiknya dilakukan sedini mungkin (*aggregation pushdown*).**
Karena `kategori` bergantung secara fungsional pada `id_produk` (id_produk → kategori), jumlah `jumlah × harga_satuan` dapat dijumlahkan **per `id_produk` terlebih dahulu**, baru kemudian digabungkan dengan `Produk` untuk memperoleh kategori. Hal ini memperkecil jumlah baris yang di-join ke `Produk` dari ±232.000 baris detail menjadi paling banyak sejumlah produk berbeda yang terjual pada bulan itu. Hasil akhir identik karena penjumlahan bersifat asosiatif dan penggabungan per kategori dilakukan sesudahnya.

**Poin 6 — Pemanfaatan indeks dan metode join yang sesuai.**
Urutan operasi menentukan apakah indeks dapat dipakai. Seleksi pada `Transaksi` di awal dapat memakai indeks `(status, tanggal)`. Hasil kecilnya lalu menggabungkan ke `Detail_Transaksi` lewat indeks pada `id_transaksi` dengan *nested loop/index join*, bukan *hash join* atas 20 juta baris. Jika seleksi dilakukan terlambat (setelah join), peluang pemakaian indeks hilang dan DBMS terpaksa memindai seluruh tabel.

**Kesimpulan 5a:** Walaupun semua urutan menghasilkan hasil yang sama, biayanya bisa berbeda berkali-kali lipat. Optimizer biasanya mengikuti heuristik: **seleksi dan proyeksi sedini mungkin, buang join yang tidak perlu, mulai join dari relasi terkecil, dan pilih metode join/indeks yang sesuai**.

---

### 5b. Ekspresi Aljabar Relasional Sebelum Optimasi (6 poin)

Query SQL dipetakan menjadi: `FROM/JOIN` → operasi join, `WHERE` → seleksi, `GROUP BY + SUM` → agregasi, `SELECT` → proyeksi.

**Langkah penyusunan:**

1. Join: `C ⋈ T` pada `C.id_pelanggan = T.id_pelanggan`.
2. Hasilnya di-join dengan `D` pada `T.id_transaksi = D.id_transaksi`.
3. Hasilnya di-join dengan `P` pada `D.id_produk = P.id_produk`.
4. Seleksi: `T.status = 'Selesai' ∧ T.tanggal ≥ '2026-01-01' ∧ T.tanggal < '2026-02-01'`.
5. Agregasi: kelompokkan menurut `P.kategori`, hitung `SUM(D.jumlah × D.harga_satuan)` sebagai `total_penjualan`.
6. Proyeksi: `kategori, total_penjualan`.

**Ekspresi aljabar relasional (sebelum optimasi):**

```
π P.kategori, total_penjualan (
    P.kategori γ SUM(D.jumlah × D.harga_satuan) → total_penjualan (
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

Notasi LaTeX (opsional):

$$
\pi_{P.kategori,\ total\_penjualan}\Big(\ _{P.kategori}\gamma_{\,SUM(D.jumlah \times D.harga\_satuan)\,\rightarrow\, total\_penjualan}\Big(\sigma_{T.status='Selesai' \,\wedge\, T.tanggal \ge '2026-01-01' \,\wedge\, T.tanggal < '2026-02-01'}\big(((C \bowtie_{C.id\_pelanggan=T.id\_pelanggan} T) \bowtie_{T.id\_transaksi=D.id\_transaksi} D) \bowtie_{D.id\_produk=P.id\_produk} P\big)\Big)\Big)
$$

**Pohon query (sebelum optimasi)** — seleksi berada di atas semua join, sehingga seluruh data digabung dulu:

```
                 π kategori, total_penjualan
                            │
               γ kategori ; SUM(jumlah×harga_satuan)
                            │
        σ status='Selesai' ∧ tanggal ≥ '2026-01-01' ∧ tanggal < '2026-02-01'
                            │
                    ⋈ D.id_produk = P.id_produk
                   ╱                         ╲
   ⋈ T.id_transaksi = D.id_transaksi          P (200.000)
        ╱                      ╲
⋈ C.id_pelanggan = T.id_pelanggan   D (20.000.000)
     ╱              ╲
 C (500.000)     T (5.000.000)
```

---

### 5c. Optimasi Ekspresi Aljabar Relasional

Optimasi dilakukan bertahap menggunakan **aturan ekuivalensi aljabar relasional**, sehingga hasil tetap identik dengan query awal.

#### Langkah 1 — Pecah (cascade) seleksi konjungtif

Aturan: `σ c1 ∧ c2 ∧ c3 (R) ≡ σ c1 (σ c2 (σ c3 (R)))`

```
σ status='Selesai' ( σ tanggal ≥ '2026-01-01' ( σ tanggal < '2026-02-01' (…) ) )
```

Tujuan: setiap kondisi dapat didorong turun secara terpisah menuju tabel asalnya.

#### Langkah 2 — Dorong seleksi sedekat mungkin ke relasi dasar (*selection pushdown*)

Aturan: bila kondisi hanya memakai atribut relasi R, maka `σ c (R ⋈ S) ≡ σ c (R) ⋈ S`.

Ketiga kondisi hanya memakai atribut `Transaksi`, sehingga diterapkan langsung pada T:

```
T' = σ status='Selesai' ∧ tanggal ≥ '2026-01-01' ∧ tanggal < '2026-02-01' (Transaksi)
```

Dari ±5.000.000 baris menjadi ±58.000 baris (sesuai ilustrasi di 5a).

#### Langkah 3 — Hapus join redundan dengan `Pelanggan`

Karena `Pelanggan` tidak dipakai di output, kondisi, maupun pengelompokan, dan `Transaksi.id_pelanggan` adalah FK ke `Pelanggan` (NOT NULL, integritas referensial terjaga), maka:

```
π atribut_T ( C ⋈ T ) ≡ T
```

Join `C ⋈ T` dihilangkan.

#### Langkah 4 — Dorong proyeksi (*projection pushdown*)

Aturan: proyeksi dapat didahulukan selama atribut yang dibutuhkan operasi berikutnya (atribut join, agregasi, pengelompokan) tetap dipertahankan.

```
T'  = π id_transaksi ( σ status='Selesai' ∧ tanggal ≥ '2026-01-01' ∧ tanggal < '2026-02-01' (T) )
D'  = π id_transaksi, id_produk, jumlah, harga_satuan ( D )
P'  = π id_produk, kategori ( P )
```

#### Langkah 5 — Susun ulang urutan join (komutatif dan asosiatif)

Mulai dari relasi hasil seleksi terkecil (T′ ≈ 58.000 baris), lalu join ke D′ lewat `id_transaksi` (dapat memakai indeks), kemudian ke P′ lewat `id_produk`:

```
( T' ⋈ T'.id_transaksi = D'.id_transaksi  D' ) ⋈ D'.id_produk = P'.id_produk  P'
```

#### Langkah 6 — Dorong agregasi parsial (*aggregation pushdown*)

Karena `id_produk → kategori`, jumlahkan dulu per produk sebelum join dengan `Produk`:

```
A = id_produk γ SUM(jumlah × harga_satuan) → subtotal ( T' ⋈ D' )
```

Lalu gabungkan dengan P′ dan jumlahkan lagi per kategori:

```
kategori γ SUM(subtotal) → total_penjualan ( A ⋈ A.id_produk = P'.id_produk  P' )
```

Ini sah karena `SUM` bersifat dapat didekomposisi: jumlah total per kategori = jumlah dari subtotal per produk di kategori itu.

#### Ekspresi Aljabar Relasional Akhir (Setelah Optimasi)

```
π kategori, total_penjualan (
    kategori γ SUM(subtotal) → total_penjualan (
        (
            id_produk γ SUM(jumlah × harga_satuan) → subtotal (
                  π id_transaksi (
                      σ status = 'Selesai'
                         ∧ tanggal ≥ '2026-01-01'
                         ∧ tanggal < '2026-02-01' ( Transaksi )
                  )
                  ⋈ id_transaksi
                  π id_transaksi, id_produk, jumlah, harga_satuan ( Detail_Transaksi )
            )
        )
        ⋈ id_produk
        π id_produk, kategori ( Produk )
    )
)
```

**Versi sederhana** (tanpa agregasi parsial; cukup bila hanya ingin menunjukkan seleksi/proyeksi/join dioptimalkan):

```
kategori γ SUM(jumlah × harga_satuan) → total_penjualan (
    (
        π id_transaksi ( σ status='Selesai' ∧ tanggal ≥ '2026-01-01' ∧ tanggal < '2026-02-01' ( Transaksi ) )
        ⋈ id_transaksi
        π id_transaksi, id_produk, jumlah, harga_satuan ( Detail_Transaksi )
    )
    ⋈ id_produk
    π id_produk, kategori ( Produk )
)
```

#### Pohon Query Setelah Optimasi

```
                     π kategori, total_penjualan
                                │
                     γ kategori ; SUM(subtotal)
                                │
                     ⋈ A.id_produk = P'.id_produk
                    ╱                           ╲
   γ id_produk ; SUM(jumlah×harga_satuan)      π id_produk, kategori
                 │                                       │
     ⋈ T'.id_transaksi = D'.id_transaksi            Produk (200.000)
        ╱                      ╲
 π id_transaksi        π id_transaksi, id_produk,
       │                    jumlah, harga_satuan
 σ status='Selesai'               │
  ∧ tanggal ≥ '2026-01-01'  Detail_Transaksi (20.000.000)
  ∧ tanggal < '2026-02-01'
       │
 Transaksi (5.000.000)
```

Seleksi dan proyeksi berada di **dasar pohon** (dikerjakan pertama), tabel `Pelanggan` hilang, dan agregasi dilakukan sebelum join terakhir.

#### Ringkasan Aturan yang Dipakai

| Langkah | Aturan Ekuivalensi / Heuristik | Manfaat |
|---|---|---|
| 1 | Cascade seleksi (`σ c1∧c2 ≡ σ c1(σ c2)`) | Memungkinkan tiap kondisi didorong turun. |
| 2 | Seleksi didorong ke bawah join | Transaksi turun dari ±5 juta menjadi ±58 ribu baris sebelum join. |
| 3 | Eliminasi join redundan berdasarkan FK | Menghemat pemindaian 500.000 baris dan satu operasi join. |
| 4 | Proyeksi didorong ke bawah | Baris antara lebih sempit, I/O berkurang. |
| 5 | Komutatif/asosiatif join + mulai dari relasi terkecil | Hasil antara lebih kecil. |
| 6 | Agregasi parsial didorong ke bawah (memanfaatkan `id_produk → kategori`) | Baris yang di-join ke Produk berkurang. |

#### Query SQL Hasil Optimasi (Setara dengan Ekspresi Akhir)

```sql
SELECT
    P.kategori,
    SUM(A.subtotal) AS total_penjualan
FROM (
    SELECT D.id_produk,
           SUM(D.jumlah * D.harga_satuan) AS subtotal
    FROM Transaksi T
    JOIN Detail_Transaksi D
      ON D.id_transaksi = T.id_transaksi
    WHERE T.status = 'Selesai'
      AND T.tanggal >= '2026-01-01'
      AND T.tanggal <  '2026-02-01'
    GROUP BY D.id_produk
) AS A
JOIN Produk P
  ON P.id_produk = A.id_produk
GROUP BY P.kategori;
```

**Indeks pendukung agar rencana optimal terwujud:**

```sql
-- Seleksi awal pada Transaksi (equality dulu, lalu range)
CREATE INDEX idx_transaksi_status_tanggal ON Transaksi (status, tanggal) INCLUDE (id_transaksi);
-- Join Transaksi → Detail_Transaksi, sekaligus covering untuk kolom yang dipakai
CREATE INDEX idx_detail_transaksi ON Detail_Transaksi (id_transaksi) INCLUDE (id_produk, jumlah, harga_satuan);
-- Join ke Produk sudah dilayani PK id_produk
```

#### Jaminan Kebenaran Hasil

- Pembuangan `Pelanggan` sah karena FK NOT NULL menjamin tiap transaksi punya tepat satu pelanggan, sehingga join itu tidak menambah/mengurangi baris. (Jika ada transaksi yatim tanpa pelanggan, hasil query awal akan membuangnya, sehingga integritas referensial harus ditegakkan agar kesetaraan berlaku.)
- Memindahkan seleksi/proyeksi ke bawah join tidak mengubah hasil karena kondisi hanya memakai atribut `Transaksi`.
- Agregasi parsial benar karena `SUM` dapat didekomposisi dan `id_produk → kategori` (satu produk hanya punya satu kategori).
- Rentang tanggal memakai `>=` dan `<` (bukan `BETWEEN` atau fungsi `MONTH()`), sehingga tetap *sargable* (dapat memakai indeks) dan akurat untuk tipe `DATETIME`/`TIMESTAMP`.

**Kesimpulan 5c:** Dengan seleksi dan proyeksi lebih awal, penghapusan join `Pelanggan`, urutan join dimulai dari relasi terkecil, serta agregasi parsial, biaya pemrosesan turun jauh (dari penggabungan ±20 juta baris menjadi ratusan ribu baris) dengan hasil laporan **tetap identik**.

---

*Catatan: angka dalam ilustrasi (±58.000 transaksi/bulan, rata-rata 4 detail per transaksi, dsb.) adalah asumsi untuk memperlihatkan perbandingan biaya relatif. Nilai aktual bergantung pada distribusi data sebenarnya dan statistik yang dimiliki optimizer.*
