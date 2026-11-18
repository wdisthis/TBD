**MODUL PRAKTIKUM**

**BACKUP & RECOVERY SYSTEM**

*Lanjutan Studi Kasus: Pengambilan KRS (Kartu Rencana Studi)*

**Mata Kuliah** : Teknologi Basis Data

**Dosen Pengampu** : Yoga Aji Sukma, S.Mat., M.Stat.

**Tools Praktikum** : Python (sqlite3) di Google Colab

---

## A. TUJUAN PRAKTIKUM

> a. Mahasiswa mampu menjelaskan konsep recovery (pemulihan) serta pentingnya menjaga Atomicity dan Durability pada basis data.
>
> b. Mahasiswa mampu membedakan teknik recovery: Log-Based Recovery, Shadow Paging, Checkpointing, serta Backup dan Restore.
>
> c. Mahasiswa mampu mengimplementasikan backup, restore, dan pemulihan dari kegagalan menggunakan Python dan SQLite.

---

## B. DASAR TEORI

**1. Konsep Recovery**

Sistem basis data rentan terhadap berbagai kegagalan, baik akibat pemadaman listrik mendadak, bug perangkat lunak, maupun kerusakan perangkat keras. Database Management System (DBMS) harus memiliki mekanisme untuk dapat beroperasi kembali tanpa kehilangan data atau meninggalkannya dalam kondisi rusak.

Proses untuk mengembalikan basis data ke keadaan yang benar dan konsisten disebut recovery (pemulihan). Tujuan utamanya adalah menjaga dua komponen penting ACID (dibahas pada Pertemuan 9):

> a. **Atomicity** — sebuah transaksi harus diselesaikan sepenuhnya atau tidak dijalankan sama sekali.
>
> b. **Durability** — setelah transaksi dinyatakan commit, perubahannya harus tetap tersimpan di basis data, bahkan jika terjadi kegagalan setelahnya.

**2. Teknik Pemulihan Utama**

Beberapa teknik pemulihan utama dalam sistem basis data:

> a. **Log-Based Recovery** — sistem menjaga berkas log (jurnal) yang mencatat setiap perubahan sebelum diterapkan. Saat gagal, DBMS membaca log untuk melakukan Undo (membatalkan transaksi belum selesai, demi Atomicity) dan Redo (menerapkan ulang transaksi yang sudah commit, demi Durability).
>
> b. **Shadow Paging** — teknik tanpa log; menjaga dua versi halaman basis data (shadow page table yang asli dan current page table hasil modifikasi). Jika transaksi gagal, cukup kembali memakai shadow tanpa undo/redo.
>
> c. **Checkpointing** — optimisasi untuk Log-Based Recovery; secara berkala menyimpan log & halaman termodifikasi ke penyimpanan permanen dan menulis checkpoint record, sehingga saat pemulihan DBMS cukup membaca log setelah checkpoint terakhir.
>
> d. **Backup dan Restore** — perlindungan terakhir terhadap kegagalan berat (disk crash) dengan menyimpan salinan cadangan berkas basis data.

**3. Penyebab dan Jenis Kegagalan**

Backup dan recovery diperlukan karena adanya kerusakan atau kegagalan operasi pada basis data. Beberapa penyebabnya:

> a. Aliran listrik terputus, mengakibatkan hilangnya informasi di memori utama dan register.
>
> b. Kesalahan operator (human error), administrator melakukan operasi keliru, sengaja maupun tidak (mis. DELETE tanpa WHERE).
>
> c. Kesalahan perangkat lunak (software error), mengakibatkan hasil pengolahan tidak benar dan basis data menjadi tidak konsisten.
>
> d. Disk rusak (disk failure), mengakibatkan hilang atau rusaknya data yang ada di dalam disk.

Berdasarkan penyebab tersebut, jenis kegagalan dapat dikelompokkan menjadi: transaction failure (kesalahan logika atau deadlock), system crash (perangkat keras macet sehingga penyimpanan sementara hilang), dan disk failure (kerusakan media penyimpanan permanen).

**4. Jenis Backup**

Terdapat beberapa jenis backup berdasarkan cakupan data yang disalin:

| **Jenis Backup**       | **Penjelasan**                                                                                           |
|------------------------|----------------------------------------------------------------------------------------------------------|
| Full Backup            | Menyalin seluruh isi basis data secara lengkap.                                                          |
| Differential Backup    | Menyalin hanya data yang berubah sejak full backup terakhir dilakukan.                                   |
| Transaction Log Backup | Menyalin isi log transaksi; memungkinkan point-in-time recovery (pemulihan hingga titik waktu tertentu). |

Berdasarkan strategi pelaksanaannya, backup juga dibedakan menjadi backup statis (dilakukan dengan menonaktifkan basis data lebih dulu) dan backup dinamis (dilakukan tanpa menonaktifkan basis data).

**Catatan: SQLite vs MySQL/PostgreSQL**

MySQL dan PostgreSQL melakukan backup lewat tool khusus seperti mysqldump, pg_dump, atau GUI seperti PhpMyAdmin (Export/Import) dan DBeaver (Tools -> Backup/Restore), yang membutuhkan server database berjalan.

*SQLite melakukan backup cukup dengan modul bawaan Python, tanpa server maupun tool tambahan: conn.backup() untuk salinan fisik (dinamis), conn.iterdump() untuk salinan logis (.sql), PRAGMA journal_mode=WAL untuk log-based recovery, dan PRAGMA wal_checkpoint untuk checkpointing. Inilah yang akan dipraktikkan pada modul ini.*

---

## C. LATIHAN PRAKTIKUM

*Catatan Tools: seluruh praktikum dikerjakan di Google Colab memakai modul bawaan Python, sqlite3. Berbeda dengan Pertemuan 9 yang memakai database in-memory, di sini kita memakai file on-disk (krs_prod.db) supaya basis data bisa di-backup, dihapus, lalu dipulihkan.*

**Langkah 1 - Setup Database KRS**

Buat file database krs_prod.db (anggap ini database produksi) berisi dua tabel: kelas dan krs.

```python
import sqlite3
import pandas as pd
import os

prod = sqlite3.connect('krs_prod.db')
prod.execute('''CREATE TABLE kelas (
id_kelas INTEGER PRIMARY KEY, nama_matkul TEXT,
kuota INTEGER, terisi INTEGER)''')
prod.execute('''CREATE TABLE krs (
id_krs INTEGER PRIMARY KEY AUTOINCREMENT,
nim TEXT, id_kelas INTEGER, status TEXT)''')
prod.execute("INSERT INTO kelas VALUES"
"(1,'Teknologi Basis Data',40,25),(2,'Struktur Data',35,30)")
prod.executemany("INSERT INTO krs (nim,id_kelas,status) VALUES (?,?,?)",
[('121140001',1,'diambil'),('121140002',1,'diambil'),('121140003',2,'diambil')])
prod.commit()

def tampilkan(sql, koneksi, params=()):
    return pd.read_sql_query(sql, koneksi, params=params)
```

***Hasil - SELECT * FROM krs:***

| **id_krs** | **nim**   | **id_kelas** | **status** |
|------------|-----------|--------------|------------|
| 1          | 121140001 | 1            | diambil    |
| 2          | 121140002 | 1            | diambil    |
| 3          | 121140003 | 2            | diambil    |

**Langkah 2 - Full Backup (Backup API, Fisik & Dinamis)**

conn.backup() menyalin seluruh basis data ke file lain, bahkan saat database sedang dipakai (backup dinamis).

```python
backup = sqlite3.connect('krs_backup.db')
prod.backup(backup) # salin seluruh isi prod -> backup
backup.commit()
tampilkan("SELECT * FROM krs", backup)
```

***Hasil - isi backup identik dengan aslinya (3 baris krs). File krs_backup.db terbentuk.***

**Langkah 3 - Logical Backup (iterdump ke file .sql)**

conn.iterdump() menghasilkan skrip SQL (CREATE TABLE + INSERT) untuk membangun ulang seluruh database, mirip hasil Export di PhpMyAdmin.

```python
with open('krs_backup.sql', 'w') as f:
    for baris in prod.iterdump():
        f.write(baris + '\n')
print(open('krs_backup.sql').read())
```

***Hasil - isi krs_backup.sql (potongan):***

```sql
BEGIN TRANSACTION;
CREATE TABLE kelas (...);
INSERT INTO "kelas" VALUES(1,'Teknologi Basis Data',40,25);
CREATE TABLE krs (...);
INSERT INTO "krs" VALUES(1,'121140001',1,'diambil');
...
COMMIT;
```

**Langkah 4 - Skenario Human Error lalu Restore**

Staf TU hendak menghapus satu data KRS tetapi lupa menulis klausa WHERE, sehingga seluruh tabel krs terhapus. Untung sudah ada full backup di Langkah 2.

```python
# HUMAN ERROR: lupa WHERE
prod.execute("DELETE FROM krs")
prod.commit()
print(prod.execute('SELECT COUNT(*) FROM krs').fetchone()[0]) # -> 0
```

***Hasil - jumlah baris krs: 0 (semua data hilang).***

Pulihkan data dengan restore dari full backup:

```python
sumber = sqlite3.connect('krs_backup.db')
sumber.backup(prod) # salin backup -> prod, menimpa kondisi rusak
sumber.close()
tampilkan("SELECT * FROM krs", prod)
```

***Hasil - data KRS pulih:***

| **id_krs** | **nim**   | **id_kelas** | **status** |
|------------|-----------|--------------|------------|
| 1          | 121140001 | 1            | diambil    |
| 2          | 121140002 | 1            | diambil    |
| 3          | 121140003 | 2            | diambil    |

*Pelajaran: backup rutin adalah satu-satunya penyelamat dari human error dan disk failure; teknik lain tidak bisa menolong jika data sudah benar-benar terhapus dan di-commit.*

**Langkah 5 - Restore dari Logical Backup (.sql) ke Database Baru**

Backup logis (.sql) dapat membangun database baru dari nol memakai executescript(). Berguna saat memindahkan data ke server/perangkat lain.

```python
baru = sqlite3.connect(':memory:')
with open('krs_backup.sql') as f:
    baru.executescript(f.read())
tampilkan("SELECT * FROM krs", baru)
```

***Hasil - database baru berisi 3 baris krs yang identik dengan aslinya.***

**Langkah 6 - Log-Based Recovery: WAL Mode**

SQLite punya mode Write-Ahead Logging (WAL): setiap perubahan ditulis dulu ke berkas log krs_prod.db-wal sebelum masuk ke database utama, persis prinsip Log-Based Recovery.

```python
mode = prod.execute("PRAGMA journal_mode = WAL").fetchone()[0]
print(mode) # -> wal
prod.execute("UPDATE kelas SET terisi = terisi + 1 WHERE id_kelas = 1")
prod.commit()
print(os.path.exists('krs_prod.db-wal')) # -> True
```

***Hasil - journal_mode: wal; berkas krs_prod.db-wal muncul (inilah 'log' untuk undo/redo).***

**Langkah 7 - Checkpointing**

PRAGMA wal_checkpoint(FULL) memindahkan perubahan dari berkas log (WAL) ke database utama, persis operasi checkpoint pada teori. Hasilnya tiga angka: (busy, jumlah_halaman_log, jumlah_yang_dipindahkan).

```python
hasil = prod.execute("PRAGMA wal_checkpoint(FULL)").fetchone()
print(hasil) # contoh: (0, 1, 1)
prod.close()
```

***Hasil - (0, 1, 1): busy=0, 1 halaman log, 1 berhasil dipindahkan. Angka tengah & kanan tergantung banyaknya perubahan sejak checkpoint terakhir.***

**Langkah 8 - Crash Recovery Otomatis**

Transaksi yang belum di-commit saat sistem 'mati' akan otomatis dibatalkan ketika database dibuka kembali, SQLite menjamin Atomicity tanpa intervensi kita.

```python
c = sqlite3.connect('krs_crash.db')
c.execute("CREATE TABLE kelas (id_kelas INTEGER PRIMARY KEY,"
" nama_matkul TEXT, terisi INTEGER)")
c.execute("INSERT INTO kelas VALUES (1,'Teknologi Basis Data',25)")
c.commit() # terisi = 25 (sudah commit)
c.execute("BEGIN")
c.execute("UPDATE kelas SET terisi = 99 WHERE id_kelas = 1")
c.close() # 'crash' tanpa commit
c2 = sqlite3.connect('krs_crash.db')
print(c2.execute("SELECT terisi FROM kelas WHERE id_kelas=1").fetchone()[0])
```

***Hasil - terisi = 25 (bukan 99). Perubahan yang belum di-commit otomatis dibatalkan (UNDO) saat database dibuka kembali.***

---

## D. SOAL LATIHAN MANDIRI

Kerjakan soal-soal berikut pada notebook praktikum.

1. Buat database KRS baru (kelas + krs) berisi beberapa data, lalu buat full backup memakai conn.backup(). Buktikan dengan SELECT bahwa isi backup identik dengan aslinya.

2. Buat logical backup dengan iterdump() ke file .sql, lalu restore ke database baru yang kosong memakai executescript(). Bandingkan jumlah barisnya dengan aslinya.

3. Simulasikan human error: jalankan UPDATE kelas SET terisi = 0 (tanpa WHERE) sehingga semua kuota kelas ter-reset keliru, lalu pulihkan dari backup yang kamu buat di soal 1.

4. Aktifkan WAL mode, lakukan beberapa INSERT, jalankan PRAGMA wal_checkpoint(FULL), lalu jelaskan arti ketiga angka hasil yang muncul.

---

### Jawaban Soal 1 - Full Backup dengan conn.backup()

```python
import sqlite3
import pandas as pd

def tampilkan(sql, koneksi, params=()):
    return pd.read_sql_query(sql, koneksi, params=params)

# Database KRS baru
db = sqlite3.connect('latihan_krs.db')
db.execute('DROP TABLE IF EXISTS kelas')
db.execute('DROP TABLE IF EXISTS krs')
db.execute('''CREATE TABLE kelas (
id_kelas INTEGER PRIMARY KEY, nama_matkul TEXT,
kuota INTEGER, terisi INTEGER)''')
db.execute('''CREATE TABLE krs (
id_krs INTEGER PRIMARY KEY AUTOINCREMENT,
nim TEXT, id_kelas INTEGER, status TEXT)''')
db.executemany("INSERT INTO kelas VALUES (?,?,?,?)", [
    (1, 'Teknologi Basis Data', 40, 4),
    (2, 'Struktur Data', 35, 3),
    (3, 'Basis Data Lanjut', 30, 2)])
db.executemany("INSERT INTO krs (nim,id_kelas,status) VALUES (?,?,?)", [
    ('121140001', 1, 'diambil'), ('121140002', 1, 'diambil'),
    ('121140003', 2, 'diambil'), ('121140004', 1, 'diambil'),
    ('121140005', 3, 'diambil'), ('121140006', 2, 'diambil')])
db.commit()

# Full backup
backup = sqlite3.connect('latihan_backup.db')
db.backup(backup) # salin seluruh isi db -> backup
backup.commit()

# Pembuktian: isi backup identik dengan aslinya
asli = tampilkan("SELECT * FROM krs", db)
salinan = tampilkan("SELECT * FROM krs", backup)
print(asli)
print(salinan)
print('identik:', asli.equals(salinan)) # -> True
```

***Hasil - kedua tabel menampilkan 6 baris krs yang sama, `identik: True`.***

| id_krs | nim       | id_kelas | status  |
|--------|-----------|----------|---------|
| 1      | 121140001 | 1        | diambil |
| 2      | 121140002 | 1        | diambil |
| 3      | 121140003 | 2        | diambil |
| 4      | 121140004 | 1        | diambil |
| 5      | 121140005 | 3        | diambil |
| 6      | 121140006 | 2        | diambil |

***Analisis:***

- `db.backup(backup)` menyalin basis data secara fisik, halaman demi halaman, termasuk skema (CREATE TABLE) dan seluruh isi tabel. Karena itu tidak perlu membuat ulang tabel di file backup.
- Fungsi `.equals()` dari pandas membandingkan nilai, tipe kolom, dan urutan baris sekaligus. Hasil `True` membuktikan isi backup identik dengan aslinya, bukan hanya jumlah barisnya yang sama.
- Backup dilakukan dari `db` yang sedang aktif. Pada SQLite, backup ini bersifat dinamis: aplikasi lain tetap dapat membaca database selama proses backup berlangsung.
- Pembuktian di atas hanya memeriksa tabel `krs`. Untuk memastikan seluruh database tersalin, tabel `kelas` juga bisa dibandingkan dengan cara yang sama: `tampilkan("SELECT * FROM kelas", db).equals(tampilkan("SELECT * FROM kelas", backup))`. Pada percobaan, hasilnya juga `True`.

### Jawaban Soal 2 - Logical Backup dan Restore ke Database Kosong

```python
import sqlite3
import pandas as pd

def tampilkan(sql, koneksi, params=()):
    return pd.read_sql_query(sql, koneksi, params=params)

# Logical backup: tulis seluruh database sebagai skrip SQL
db = sqlite3.connect('latihan_krs.db')
with open('latihan_backup.sql', 'w') as f:
    for baris in db.iterdump():
        f.write(baris + '\n')

# Database baru yang masih kosong
baru = sqlite3.connect(':memory:')
print('jumlah tabel sebelum restore:',
      baru.execute("SELECT COUNT(*) FROM sqlite_master WHERE type='table'").fetchone()[0]) # -> 0

# Restore dari skrip SQL
with open('latihan_backup.sql') as f:
    baru.executescript(f.read())

# Bandingkan jumlah baris tiap tabel
for tabel in ['kelas', 'krs']:
    n_asli = db.execute(f'SELECT COUNT(*) FROM {tabel}').fetchone()[0]
    n_baru = baru.execute(f'SELECT COUNT(*) FROM {tabel}').fetchone()[0]
    print(f'{tabel}: asli={n_asli}, hasil restore={n_baru}, sama={n_asli == n_baru}')
```

***Hasil:***

```
jumlah tabel sebelum restore: 0
kelas: asli=3, hasil restore=3, sama=True
krs: asli=6, hasil restore=6, sama=True
```

***Analisis:***

- `iterdump()` menghasilkan skrip SQL yang diawali `BEGIN TRANSACTION;` dan diakhiri `COMMIT;`, sehingga proses restore berjalan sebagai satu transaksi utuh. Jika skrip terpotong di tengah jalan, perubahan tidak akan tersimpan sebagian (Atomicity).
- Jumlah tabel di database baru nol sebelum restore, sehingga semua tabel benar-benar dibangun ulang dari file `.sql`, bukan tersisa dari sumber lain.
- `executescript()` cocok untuk skrip berisi banyak pernyataan sekaligus. Berbeda dengan `execute()`, fungsi ini hanya menerima satu pernyataan.
- Backup logis lebih portabel daripada backup fisik karena berupa teks biasa. Kekurangannya, proses restore biasanya lebih lambat untuk data yang besar.

### Jawaban Soal 3 - Human Error UPDATE Tanpa WHERE lalu Restore

```python
import sqlite3
import pandas as pd

def tampilkan(sql, koneksi, params=()):
    return pd.read_sql_query(sql, koneksi, params=params)

prod = sqlite3.connect('latihan_krs.db')
print('SEBELUM ERROR:')
print(tampilkan("SELECT * FROM kelas", prod))

# HUMAN ERROR: lupa WHERE, semua kelas ter-reset
prod.execute("UPDATE kelas SET terisi = 0")
prod.commit()
print('SETELAH ERROR:')
print(tampilkan("SELECT * FROM kelas", prod))

# Restore dari full backup soal 1
sumber = sqlite3.connect('latihan_backup.db')
sumber.backup(prod) # salin backup -> prod, menimpa kondisi rusak
sumber.close()
print('SETELAH RESTORE:')
print(tampilkan("SELECT * FROM kelas", prod))
```

***Hasil:***

| id_kelas | nama_matkul          | kuota | terisi (sebelum) | terisi (setelah error) | terisi (setelah restore) |
|----------|----------------------|-------|------------------|------------------------|--------------------------|
| 1        | Teknologi Basis Data | 40    | 4                | 0                      | 4                        |
| 2        | Struktur Data        | 35    | 3                | 0                      | 3                        |
| 3        | Basis Data Lanjut    | 30    | 2                | 0                      | 2                        |

***Analisis:***

- Perintah `UPDATE` tanpa `WHERE` mengenai seluruh baris tabel `kelas`. Kolom `kuota` tidak ikut berubah karena tidak disebutkan dalam `SET`, sedangkan `terisi` menjadi 0 untuk semua kelas.
- Setelah `commit()`, kerusakan sudah permanen di database produksi. Teknik seperti log-based recovery hanya membantu jika transaksi belum di-commit; di sini restore dari backup adalah satu-satunya jalan.
- Restore memakai `backup()` menyalin seluruh isi file backup dan menimpa database. Akibatnya, semua perubahan yang terjadi setelah waktu backup, termasuk data KRS baru, ikut hilang. Inilah alasan backup penuh perlu dilengkapi backup transaction log untuk pemulihan hingga titik waktu tertentu.
- Sebelum melakukan UPDATE atau DELETE massal di database produksi, sebaiknya jalankan dulu `SELECT` dengan kondisi yang sama untuk memastikan baris yang terkena sudah benar.

### Jawaban Soal 4 - WAL Mode, INSERT, dan Checkpoint

```python
import sqlite3
import os

wal = sqlite3.connect('latihan_wal.db')
wal.execute('DROP TABLE IF EXISTS krs')
wal.execute('''CREATE TABLE krs (
id_krs INTEGER PRIMARY KEY AUTOINCREMENT,
nim TEXT, id_kelas INTEGER, status TEXT)''')
wal.commit()

# Aktifkan WAL mode
mode = wal.execute("PRAGMA journal_mode = WAL").fetchone()[0]
print(mode) # -> wal

# Beberapa INSERT, masing-masing di-commit
for i in range(5):
    wal.execute("INSERT INTO krs (nim,id_kelas,status) VALUES (?,?,?)",
                (f'12114001{i}', 1, 'diambil'))
    wal.commit()
print('jumlah baris krs:', wal.execute('SELECT COUNT(*) FROM krs').fetchone()[0]) # -> 5
print(os.path.exists('latihan_wal.db-wal')) # -> True

# Checkpoint: pindahkan perubahan dari WAL ke database utama
hasil = wal.execute("PRAGMA wal_checkpoint(FULL)").fetchone()
print(hasil) # (busy, jumlah_halaman_log, jumlah_yang_dipindahkan)
wal.close()
```

***Hasil:***

```
wal
jumlah baris krs: 5
True
(0, 10, 10)
```

***Arti ketiga angka `(0, 10, 10)`:***

| Posisi | Nama                        | Nilai | Arti                                                                                                 |
|--------|-----------------------------|-------|------------------------------------------------------------------------------------------------------|
| 1      | busy                        | 0     | 0 berarti checkpoint tidak terhalang. Nilai 1 berarti ada koneksi lain yang memblokir checkpoint.     |
| 2      | jumlah halaman di log (WAL) | 10    | Ada 10 frame (halaman) di berkas WAL yang belum dipindahkan ke database utama.                       |
| 3      | jumlah yang dipindahkan     | 10    | Seluruh 10 frame berhasil dipindahkan ke database utama.                                             |

***Analisis:***

- Angka 10 terasa berlebihan untuk 5 baris. Penyebabnya adalah `AUTOINCREMENT`: setiap commit INSERT menulis dua halaman, yaitu halaman tabel `krs` dan halaman tabel internal `sqlite_sequence` yang menyimpan nilai id terakhir. Dengan 5 commit, jumlahnya 2 × 5 = 10.
- Untuk memastikan penyebabnya, tabel yang sama tanpa `AUTOINCREMENT` menghasilkan `(0, 5, 5)`. Jadi jumlah frame sebanding dengan jumlah halaman yang diubah per commit, bukan jumlah baris.
- Jika kolom busy bernilai 1, itu biasanya karena masih ada transaksi pembaca yang aktif. Checkpoint akan memindahkan sebanyak yang mungkin, lalu bisa dijalankan ulang.
- Mode `FULL` menunggu sampai seluruh frame di WAL terpindahkan, sehingga pada akhirnya nilai kedua dan ketiga sama jika tidak ada transaksi lain yang berjalan.

---

*Catatan: sesuai instruksi, seluruh kode di atas mengikuti gaya penulisan modul (import, fungsi `tampilkan()`, komentar `#` singkat, dan `print` dengan keterangan `# -> ...`). Jalankan soal secara berurutan karena soal 3 membutuhkan backup dari soal 1.*
