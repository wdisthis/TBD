# TBD

# Tugas Mandiri – Modul 2: Schema Tuning

**Mata Kuliah** : SD3203 – Teknologi Basis Data
**Nama** : Fitra
**NIM** : 124450097
**Kelas** : (isi kelas)
**DBMS / Tools** : PostgreSQL + DBeaver

---

## 1. Deskripsi Kasus

Sebuah toko elektronik memiliki sistem penjualan daring dengan dua tabel:

| Tabel | Kolom |
|---|---|
| `customers` | id, name, email, address, phone_number |
| `orders` | no_order, date_orders, product, quantity, price, total_price |

Pihak administrasi **sering mengakses** `name`, `address`, `product`, dan `quantity` saat pengepakan dan pengiriman barang. Kolom `email` dan `phone_number` jarang dibutuhkan pada proses tersebut.

### Masalah pada skema awal

1. **Baris `customers` terlalu lebar** untuk kebutuhan pengepakan. Setiap pembacaan ikut membawa `email` dan `phone_number` yang tidak dipakai.
2. **Tabel `orders` tidak punya kolom penghubung ke `customers`.** Padahal untuk mendapatkan `name` + `address` + `product` + `quantity` harus ada relasi antar tabel. Kolom `customer_id` ditambahkan sebagai bagian dari perbaikan skema.
3. **Data pengiriman tersebar di dua tabel**, sehingga setiap kali admin membutuhkannya harus melakukan `JOIN`.

### Teknik schema tuning yang dipilih (sesuai Modul 2)

| No | Teknik | Diterapkan pada | Tujuan |
|---|---|---|---|
| 1 | **Vertical Splitting** | `customers` → `customer_shipping` + `customer_contact` | Memisahkan kolom yang sering diakses (`name`, `address`) dari yang jarang (`email`, `phone_number`) |
| 2 | **Denormalisasi – Kolom Redundan** | `customers` + `orders` → `shipping_orders` | Menghilangkan `JOIN` untuk kebutuhan pengepakan dan pengiriman |

Horizontal Splitting **tidak dipilih**, karena seluruh pesanan hanya berasal dari satu tanggal (2025-03-06) sehingga tidak ada dasar pembagian baris yang bermakna.

---

## 2. Persiapan Database

### 2.1 Membuat schema

```sql
CREATE SCHEMA IF NOT EXISTS toko_elektronik;
SET search_path TO toko_elektronik;
```

### 2.2 Membuat tabel awal

```sql
CREATE TABLE customers (
    id            SERIAL PRIMARY KEY,
    name          VARCHAR(100) NOT NULL,
    email         VARCHAR(100),
    address       VARCHAR(255),
    phone_number  VARCHAR(15)
);

CREATE TABLE orders (
    no_order      INT PRIMARY KEY,
    date_orders   DATE,
    product       VARCHAR(100),
    quantity      INT,
    price         NUMERIC(12,2),
    total_price   NUMERIC(14,2)
);
```

### 2.3 Menyisipkan data (dari soal)

```sql
INSERT INTO customers (name, email, address, phone_number) VALUES
('Ahmad Rizky',   'ahmad.rizky@example.com',   'Jl. Melati No.1, Jakarta',      '081234567890'),
('Siti Nuraini',  'siti.nuraini@example.com',  'Jl. Cempaka No.2, Surabaya',    '082345678901'),
('Dedi Prasetyo', 'dedi.prasetyo@example.com', 'Jl. Mawar No.3, Bandung',       '083456789012'),
('Tia Sulastri',  'tia.sulastri@example.com',  'Jl. Anggrek No.4, Yogyakarta',  '084567890123'),
('Bayu Nugroho',  'bayu.nugroho@example.com',  'Jl. Kenanga No.5, Semarang',    '085678901234'),
('Rina Pertiwi',  'rina.pertiwi@example.com',  'Jl. Melur No.6, Medan',         '086789012345'),
('Fajar Kusuma',  'fajar.kusuma@example.com',  'Jl. Seroja No.7, Makassar',     '087890123456'),
('Liana Indah',   'liana.indah@example.com',   'Jl. Angsana No.8, Bali',        '088901234567'),
('Dwi Wibowo',    'dwi.wibowo@example.com',    'Jl. Taman No.9, Palembang',     '089012345678'),
('Rudi Setiawan', 'rudi.setiawan@example.com', 'Jl. Pandan No.10, Malang',      '090123456789');

INSERT INTO orders (no_order, date_orders, product, quantity, price, total_price) VALUES
(1,  '2025-03-06', 'Laptop',              2,  5000000.00, 10000000.00),
(2,  '2025-03-06', 'Smartphone',          3,  3000000.00,  9000000.00),
(3,  '2025-03-06', 'Tablet',              5,  2000000.00, 10000000.00),
(4,  '2025-03-06', 'Headphones',         10,   500000.00,  5000000.00),
(5,  '2025-03-06', 'Monitor',             4,  1500000.00,  6000000.00),
(6,  '2025-03-06', 'Keyboard',            7,   350000.00,  2450000.00),
(7,  '2025-03-06', 'Mouse',              15,   150000.00,  2250000.00),
(8,  '2025-03-06', 'Printer',             3,  1200000.00,  3600000.00),
(9,  '2025-03-06', 'Webcam',              6,   800000.00,  4800000.00),
(10, '2025-03-06', 'External Hard Drive', 4,  1000000.00,  4000000.00);
```

### 2.4 Menghubungkan `orders` dengan `customers`

Tabel `orders` pada soal tidak memiliki kolom pelanggan, sehingga ditambahkan `customer_id` beserta foreign key.

> **Asumsi:** data soal tidak menyebutkan pemesan tiap pesanan. Karena jumlah pelanggan dan pesanan sama (10 dan 10), pesanan nomor *n* dianggap dipesan oleh pelanggan dengan `id` = *n*.

```sql
ALTER TABLE orders ADD COLUMN customer_id INT;

UPDATE orders SET customer_id = no_order;

ALTER TABLE orders
    ALTER COLUMN customer_id SET NOT NULL,
    ADD CONSTRAINT fk_orders_customer
        FOREIGN KEY (customer_id) REFERENCES customers(id);
```

### 2.5 Kondisi sebelum tuning: kueri pengepakan dengan JOIN

```sql
SELECT c.name, c.address, o.product, o.quantity
FROM orders o
JOIN customers c ON o.customer_id = c.id
ORDER BY o.no_order;
```

**Hasil (kondisi awal):**

| name | address | product | quantity |
|---|---|---|---|
| Ahmad Rizky | Jl. Melati No.1, Jakarta | Laptop | 2 |
| Siti Nuraini | Jl. Cempaka No.2, Surabaya | Smartphone | 3 |
| Dedi Prasetyo | Jl. Mawar No.3, Bandung | Tablet | 5 |
| Tia Sulastri | Jl. Anggrek No.4, Yogyakarta | Headphones | 10 |
| Bayu Nugroho | Jl. Kenanga No.5, Semarang | Monitor | 4 |
| Rina Pertiwi | Jl. Melur No.6, Medan | Keyboard | 7 |
| Fajar Kusuma | Jl. Seroja No.7, Makassar | Mouse | 15 |
| Liana Indah | Jl. Angsana No.8, Bali | Printer | 3 |
| Dwi Wibowo | Jl. Taman No.9, Palembang | Webcam | 6 |
| Rudi Setiawan | Jl. Pandan No.10, Malang | External Hard Drive | 4 |

*(Sisipkan screenshot hasil di DBeaver di sini)*

---

## 3. Schema Tuning

## 3.1 Vertical Splitting pada Tabel `customers`

### Penjelasan

*Vertical splitting* membagi satu tabel menjadi beberapa tabel dengan subset kolom yang berbeda. Tabel `customers` dipecah menjadi dua:

- **`customer_shipping`** berisi kolom yang **sering diakses** oleh admin: `id`, `name`, `address`.
- **`customer_contact`** berisi kolom yang **jarang diakses**: `id`, `email`, `phone_number`.

Kedua tabel memakai `id` yang sama (relasi 1-ke-1) sehingga data asli tetap bisa direkonstruksi. Manfaatnya, baris pada `customer_shipping` menjadi lebih pendek, sehingga lebih banyak baris yang muat dalam satu halaman disk (*page*) dan pembacaan data pengiriman menjadi lebih efisien.

### Kode

```sql
-- Tabel kolom yang sering diakses
CREATE TABLE customer_shipping (
    id       INT PRIMARY KEY,
    name     VARCHAR(100) NOT NULL,
    address  VARCHAR(255)
);

-- Tabel kolom yang jarang diakses
CREATE TABLE customer_contact (
    id            INT PRIMARY KEY,
    email         VARCHAR(100),
    phone_number  VARCHAR(15),
    FOREIGN KEY (id) REFERENCES customer_shipping(id) ON DELETE CASCADE
);

-- Mengisi data dari tabel customers
INSERT INTO customer_shipping (id, name, address)
SELECT id, name, address
FROM customers;

INSERT INTO customer_contact (id, email, phone_number)
SELECT id, email, phone_number
FROM customers;
```

### Verifikasi

```sql
SELECT * FROM customer_shipping ORDER BY id;
SELECT * FROM customer_contact  ORDER BY id;
```

**Hasil `customer_shipping`:**

| id | name | address |
|---|---|---|
| 1 | Ahmad Rizky | Jl. Melati No.1, Jakarta |
| 2 | Siti Nuraini | Jl. Cempaka No.2, Surabaya |
| 3 | Dedi Prasetyo | Jl. Mawar No.3, Bandung |
| 4 | Tia Sulastri | Jl. Anggrek No.4, Yogyakarta |
| 5 | Bayu Nugroho | Jl. Kenanga No.5, Semarang |
| 6 | Rina Pertiwi | Jl. Melur No.6, Medan |
| 7 | Fajar Kusuma | Jl. Seroja No.7, Makassar |
| 8 | Liana Indah | Jl. Angsana No.8, Bali |
| 9 | Dwi Wibowo | Jl. Taman No.9, Palembang |
| 10 | Rudi Setiawan | Jl. Pandan No.10, Malang |

**Hasil `customer_contact`:**

| id | email | phone_number |
|---|---|---|
| 1 | ahmad.rizky@example.com | 081234567890 |
| 2 | siti.nuraini@example.com | 082345678901 |
| 3 | dedi.prasetyo@example.com | 083456789012 |
| 4 | tia.sulastri@example.com | 084567890123 |
| 5 | bayu.nugroho@example.com | 085678901234 |
| 6 | rina.pertiwi@example.com | 086789012345 |
| 7 | fajar.kusuma@example.com | 087890123456 |
| 8 | liana.indah@example.com | 088901234567 |
| 9 | dwi.wibowo@example.com | 089012345678 |
| 10 | rudi.setiawan@example.com | 090123456789 |

*(Sisipkan screenshot hasil di DBeaver di sini)*

Data asli tetap dapat digabung kembali bila diperlukan:

```sql
SELECT s.id, s.name, c.email, s.address, c.phone_number
FROM customer_shipping s
JOIN customer_contact c ON s.id = c.id
ORDER BY s.id;
```

---

## 3.2 Denormalisasi – Kolom Redundan

### Penjelasan

Denormalisasi menambahkan redundansi data demi mempercepat pembacaan. Karena admin hampir selalu membutuhkan `name`, `address`, `product`, dan `quantity` **sekaligus**, keempat kolom itu digabung dalam satu tabel baru bernama **`shipping_orders`**. Dengan begitu, proses pengepakan cukup membaca satu tabel tanpa `JOIN`.

Kolom `name` dan `address` pada tabel ini merupakan **kolom redundan** (salinan dari `customer_shipping`). Kolom lain yang tidak dibutuhkan pengepakan (`price`, `total_price`, `email`, `phone_number`) sengaja tidak dimasukkan agar tabel tetap ramping.

### Kode

```sql
CREATE TABLE shipping_orders AS
SELECT
    o.no_order,
    o.customer_id,
    s.name       AS customer_name,
    s.address    AS customer_address,
    o.product,
    o.quantity,
    o.date_orders
FROM orders o
JOIN customer_shipping s ON o.customer_id = s.id;

ALTER TABLE shipping_orders ADD PRIMARY KEY (no_order);
```

### Verifikasi

```sql
SELECT customer_name, customer_address, product, quantity
FROM shipping_orders
ORDER BY no_order;
```

**Hasil:**

| customer_name | customer_address | product | quantity |
|---|---|---|---|
| Ahmad Rizky | Jl. Melati No.1, Jakarta | Laptop | 2 |
| Siti Nuraini | Jl. Cempaka No.2, Surabaya | Smartphone | 3 |
| Dedi Prasetyo | Jl. Mawar No.3, Bandung | Tablet | 5 |
| Tia Sulastri | Jl. Anggrek No.4, Yogyakarta | Headphones | 10 |
| Bayu Nugroho | Jl. Kenanga No.5, Semarang | Monitor | 4 |
| Rina Pertiwi | Jl. Melur No.6, Medan | Keyboard | 7 |
| Fajar Kusuma | Jl. Seroja No.7, Makassar | Mouse | 15 |
| Liana Indah | Jl. Angsana No.8, Bali | Printer | 3 |
| Dwi Wibowo | Jl. Taman No.9, Palembang | Webcam | 6 |
| Rudi Setiawan | Jl. Pandan No.10, Malang | External Hard Drive | 4 |

*(Sisipkan screenshot hasil di DBeaver di sini)*

---

## 4. Perbandingan Sebelum dan Sesudah Tuning

Keduanya menghasilkan data yang sama. Perbedaannya ada pada cara data diambil.

**Sebelum** (JOIN dua tabel, membaca kolom `email` dan `phone_number` yang tidak terpakai):

```sql
EXPLAIN ANALYZE
SELECT c.name, c.address, o.product, o.quantity
FROM orders o
JOIN customers c ON o.customer_id = c.id;
```

**Sesudah** (satu tabel, tanpa JOIN):

```sql
EXPLAIN ANALYZE
SELECT customer_name, customer_address, product, quantity
FROM shipping_orders;
```

*(Sisipkan screenshot hasil `EXPLAIN ANALYZE` di DBeaver di sini)*

| Aspek | Sebelum tuning | Sesudah tuning |
|---|---|---|
| Tabel yang dibaca | 2 tabel (`orders` + `customers`) | 1 tabel (`shipping_orders`) |
| Operasi JOIN | Ada | Tidak ada |
| Kolom yang terbaca | Baris `customers` lebar (termasuk email dan telepon) | Hanya kolom yang dibutuhkan pengepakan |
| Redundansi data | Rendah | Ada (`name` dan `address` tersalin) |
| Risiko inkonsistensi | Rendah | Perlu sinkronisasi bila data pelanggan berubah |

> **Catatan:** dengan hanya 10 baris, selisih waktu eksekusi hampir tidak terlihat. Manfaat tuning baru terasa pada data berskala besar (ribuan hingga jutaan pesanan), sehingga perbandingan ini menggambarkan rencana kueri, bukan selisih detik yang signifikan.

---

## 5. Kesimpulan

1. **Vertical splitting** memisahkan `customers` menjadi `customer_shipping` (kolom yang sering diakses) dan `customer_contact` (kolom yang jarang diakses), sehingga pembacaan data pengiriman lebih ringan.
2. **Denormalisasi kolom redundan** menggabungkan `name`, `address`, `product`, dan `quantity` ke dalam `shipping_orders`, sehingga proses pengepakan dan pengiriman tidak lagi memerlukan `JOIN`.
3. **Trade-off:** denormalisasi menimbulkan duplikasi data. Jika alamat atau nama pelanggan berubah, `shipping_orders` harus ikut diperbarui, misalnya dengan *trigger* atau proses sinkronisasi berkala. Karena itu teknik ini cocok untuk kebutuhan yang lebih banyak membaca daripada menulis, seperti pengepakan dan pengiriman.
4. Kolom `customer_id` ditambahkan pada `orders` agar relasi antara pesanan dan pelanggan jelas dan dapat digunakan oleh kedua teknik di atas.
