# JAWABAN D. SOAL LATIHAN MANDIRI
**Modul Praktikum Spatial-Temporal Database: Studi Kasus Analisis Hotspot Kebakaran (dengan Peta Sungguhan)**
**Tools Praktikum:** Python (GeoPandas + contextily) di Google Colab

---

> **Catatan verifikasi.** Lingkungan tempat jawaban ini disusun tidak bisa memasang GeoPandas/contextily dan tidak bisa mengunduh basemap maupun GeoJSON provinsi. Karena itu:
> - Logika streak (Soal 3) dan jumlah titik per tanggal sudah **dijalankan dan dicek** dengan pandas. Angka di bagian Hasil Soal 3 berasal dari menjalankan kode itu.
> - Hasil **Soal 1, 2, 4, dan 5** (peta, nama provinsi hasil `sjoin`, tampilan basemap, GIF) adalah **prediksi** berdasarkan pengetahuan wilayah. Jalankan kodenya di Colab dan cocokkan dengan output-mu. Jika ada perbedaan, output Colab yang dipakai.
> - Brightness dan confidence pada data tambahan adalah **nilai contoh**, sama seperti data di modul. Koordinat kota itu nyata, tetapi titik-titik ini bukan data kebakaran sungguhan.

---

## Soal 1

Soal:

> Tambahkan 3-4 titik hotspot baru dengan koordinat nyata (boleh cek koordinat kota di Google Maps), lalu tampilkan ulang peta sebaran di atas basemap Esri World Street Map.

**Pemilihan titik.** Dipilih empat kota dengan koordinat yang bisa dicek di Google Maps. Dumai dan Kota Bengkulu tidak berdekatan dengan titik lama. Sampit menambah sebaran di Kalimantan Tengah. Medan memperluas sebaran ke utara.

| No | Kota | Provinsi | longitude | latitude |
|----|------|----------|-----------|----------|
| 1 | Dumai | Riau | 101.45 | 1.67 |
| 2 | Kota Bengkulu | Bengkulu | 102.27 | -3.80 |
| 3 | Sampit | Kalimantan Tengah | 112.95 | -2.53 |
| 4 | Kota Medan | Sumatera Utara | 98.67 | 3.59 |

```python
# 4 titik hotspot baru (koordinat kota nyata, dicek di Google Maps)
rows_baru = [
(101.45, 1.67, '2022-09-12', 318.6, 'nominal'),   # Dumai, Riau
(102.27, -3.80, '2022-09-13', 305.2, 'low'),      # Kota Bengkulu
(112.95, -2.53, '2022-09-09', 331.7, 'high'),     # Sampit, Kalteng
(98.67, 3.59, '2022-09-14', 312.0, 'nominal'),    # Kota Medan, Sumut
]
df_baru = pd.DataFrame(rows_baru, columns=['longitude','latitude','acq_date','brightness','confidence'])
df_baru['acq_date'] = pd.to_datetime(df_baru['acq_date'])

df = pd.concat([df, df_baru], ignore_index=True)   # 16 + 4 = 20 titik
gdf = gpd.GeoDataFrame(df, geometry=gpd.points_from_xy(df.longitude, df.latitude), crs='EPSG:4326')
gdf_3857 = gdf.to_crs(epsg=3857)

print(len(gdf))   # harus 20

fig, ax = plt.subplots(figsize=(10, 7))
gdf_3857.plot(ax=ax, color='red', markersize=60, edgecolor='black')
cx.add_basemap(ax, source=cx.providers.Esri.WorldStreetMap, zoom=5)
ax.set_title('Sebaran Hotspot (20 titik) di Atas Peta Esri World Street Map')
ax.set_axis_off(); plt.show()
```

***Hasil yang diharapkan: 20 titik merah di atas Esri World Street Map. Titik baru di Medan dan Dumai berada di utara sebaran lama, dan titik Bengkulu serta Sampit berada di selatan dan timur. Peta otomatis menyesuaikan batas karena `add_basemap` memakai extent data.***

**Analisis.** Empat titik baru memperluas sebaran ke dua arah. Sebelumnya sebaran berada di antara 5.45°LS dan 0.90°LU. Dengan titik Medan, sebaran kini mencapai 3.59°LU. Ini menunjukkan bahwa perubahan kecil pada data sudah mengubah ruang lingkup analisis, sehingga batas peta sebaiknya dihitung dari data, bukan ditulis tetap. Pada Langkah 9 modul, batas tetap dipakai agar frame sebanding. Batas itu perlu dihitung ulang dengan `total_bounds` setelah data ditambah.

---

## Soal 2

Soal:

> Hitung ulang jumlah hotspot per provinsi dengan gpd.sjoin setelah data ditambah, dan tentukan provinsi dengan titik api terbanyak.

```python
joined = gpd.sjoin(gdf, prov, how='left', predicate='within')

rekap = joined.groupby('provinsi').size().sort_values(ascending=False)
print(rekap)
print('Provinsi terbanyak:', rekap.idxmax(), '-', rekap.max(), 'titik')

# Titik yang tidak masuk polygon provinsi mana pun (perlu dicek manual)
print('Tidak ter-join:', joined['provinsi'].isna().sum())
```

***Hasil yang diharapkan (prediksi, cocokkan dengan output Colab):***

| provinsi | Sebelum (16 titik) | Sesudah (20 titik) |
|----------|--------------------|--------------------|
| RIAU | 5 | **6** (+ Dumai) |
| KALIMANTAN BARAT | 2 | 2 |
| KALIMANTAN TENGAH | 2 | 3 (+ Sampit) |
| SUMATERA SELATAN | 2 | 2 |
| BENGKULU | 0 | 1 (+ Kota Bengkulu) |
| SUMATERA UTARA | 0 | 1 (+ Medan) |
| JAMBI / KALSEL / KALTIM / LAMPUNG / SUMBAR | 1 (masing-masing) | 1 (masing-masing) |

*Provinsi dengan titik api terbanyak adalah **Riau** dengan 6 titik. Nama provinsi di tabel mengikuti properti `Propinsi` pada GeoJSON, jadi huruf besar-kecilnya bisa berbeda.*

**Analisis.**

1. **Jumlah titik tidak sama dengan jumlah lokasi.** Riau unggul karena klaster Riau di (0.52, 101.45) muncul empat hari, sehingga menyumbang 4 dari 6 baris. Jika dihitung lokasi unik, Riau tetap teratas dengan 3 lokasi: (0.52, 101.45), (0.90, 102.10), dan Dumai (1.67, 101.45). Kalimantan Tengah dan Kalimantan Barat masing-masing memiliki 2 lokasi unik. Jadi kesimpulan "Riau paling banyak" tidak bergantung pada cara menghitung, tetapi besarnya selisih banyak dipengaruhi klaster berulang.

2. **Risiko pada titik di pinggir pantai.** Dumai dan Kota Bengkulu berada di pesisir. Garis pantai pada GeoJSON provinsi biasanya sudah disederhanakan, sehingga titik di tepi bisa jatuh di luar polygon dan `predicate='within'` tidak menemukan provinsinya. Titik seperti itu akan menjadi `NaN`, lalu hilang dari `groupby`. Karena itu baris "Tidak ter-join" perlu dicek. Jika ada titik yang hilang, coba `predicate='intersects'` atau `gpd.sjoin_nearest(...)` dengan batas jarak.

3. **Efek ke provinsi lain.** Jika Soal 3 juga dikerjakan sebelum menghitung ulang, Kalimantan Barat bertambah 4 titik dari klaster Singkawang dan menjadi 6, sama dengan Riau. Ini contoh bahwa hasil peringkat sensitif terhadap data yang dimasukkan.

---

## Soal 3

Soal:

> Buat satu klaster baru (koordinat sama persis) yang muncul 4 hari berturut-turut, lalu jalankan ulang analisis streak dan buktikan lokasi tersebut tertandai potensi_kebakaran_besar.

**Pemilihan lokasi.** Klaster baru ditempatkan di Singkawang, Kalimantan Barat, pada koordinat (0.91, 108.99). Lokasi ini belum ada di data, sehingga hasilnya tidak tercampur dengan klaster lain. Empat tanggal dipilih berurutan, yaitu 9 sampai 12 September 2022.

```python
# Klaster baru: koordinat sama persis, muncul 4 hari berturut-turut (9-12 Sep)
klaster_baru = [
(108.99, 0.91, '2022-09-09', 336.0, 'high'),   # Singkawang, Kalbar
(108.99, 0.91, '2022-09-10', 337.4, 'high'),
(108.99, 0.91, '2022-09-11', 339.1, 'high'),
(108.99, 0.91, '2022-09-12', 341.5, 'high'),
]
df_klaster = pd.DataFrame(klaster_baru, columns=['longitude','latitude','acq_date','brightness','confidence'])
df_klaster['acq_date'] = pd.to_datetime(df_klaster['acq_date'])

df = pd.concat([df, df_klaster], ignore_index=True)
gdf = gpd.GeoDataFrame(df, geometry=gpd.points_from_xy(df.longitude, df.latitude), crs='EPSG:4326')

# Jalankan ulang fungsi streak dari Langkah 8 (tidak diubah)
def streak(tanggal):
    d = sorted(set(tanggal)); terpanjang = berjalan = 1
    for i in range(1, len(d)):
        if (d[i] - d[i-1]).days == 1:
            berjalan += 1; terpanjang = max(terpanjang, berjalan)
        else:
            berjalan = 1
    return terpanjang

hb = (gdf.groupby(['latitude','longitude'])['acq_date']
        .apply(lambda s: streak(list(s)))
        .reset_index(name='hari_berturut'))
hb['potensi_kebakaran_besar'] = hb['hari_berturut'] >= 3
hb.sort_values('hari_berturut', ascending=False).head()

# Pembuktian: lokasi klaster harus bernilai 4 dan True
bukti = hb[(hb['latitude'] == 0.91) & (hb['longitude'] == 108.99)]
print(bukti)
assert bukti['hari_berturut'].iloc[0] == 4
assert bukti['potensi_kebakaran_besar'].iloc[0] == True
print('Terbukti: klaster Singkawang tertandai potensi_kebakaran_besar')
```

***Hasil (sudah dijalankan dengan logika yang sama, pandas):***

| latitude | longitude | hari_berturut | potensi_kebakaran_besar |
|----------|-----------|---------------|-------------------------|
| 0.52 | 101.45 | 4 | True |
| 0.91 | 108.99 | **4** | **True** |
| -2.95 | 104.05 | 2 | False |
| -2.21 | 113.90 | 1 | False |

*Lokasi Singkawang (0.91, 108.99) kini tertandai `potensi_kebakaran_besar = True`, bersama klaster Riau. Keduanya satu-satunya lokasi dengan streak 3 hari atau lebih. Asersi di atas lolos.*

**Analisis.**

1. **Koordinat harus identik.** `groupby(['latitude','longitude'])` hanya menggabungkan baris yang koordinatnya sama persis sampai digit terakhir. Jika salah satu baris ditulis `0.9100001`, streak-nya terpecah. Ini sebabnya Soal meminta koordinat yang sama persis.

2. **Pada data nyata, koordinat hampir tidak pernah identik.** Satelit VIIRS/MODIS di FIRMS mendeteksi titik api per piksel (sekitar 375 m hingga 1 km), dan posisi titik bisa sedikit berubah antar-hari. Dalam praktik, streak harus memakai toleransi jarak, misalnya membulatkan koordinat, memakai buffer, atau memakai clustering spasial seperti DBSCAN dengan jarak sekitar 1 km. Jika tidak, streak nyata akan jauh lebih kecil daripada kenyataan.

3. **Streak hanya menghitung hari yang benar-benar berurutan.** Jika klaster muncul pada 9, 10, 12, dan 13 September, streak-nya 2, bukan 4, karena 11 September tidak ada. Ini sejalan dengan kasus Kalimantan Tengah di modul: titik (-2.21, 113.90) muncul 11 dan 13 September, sehingga streak-nya hanya 1, walaupun lokasinya berulang. Dampaknya dibahas pada bagian interpretasi Soal 5.

---

## Soal 4

Soal:

> Ganti basemap ke sumber lain (mis. cx.providers.CartoDB.Positron atau cx.providers.Esri.WorldImagery) dan bandingkan tampilannya dengan Esri World Street Map.

```python
# Bandingkan tiga basemap pada zoom dan extent yang sama
xmin, ymin, xmax, ymax = gdf_3857.total_bounds; pad = 3e5   # hitung ulang setelah data ditambah

sumber = [
    ('Esri World Street Map', cx.providers.Esri.WorldStreetMap),
    ('CartoDB Positron', cx.providers.CartoDB.Positron),
    ('Esri World Imagery', cx.providers.Esri.WorldImagery),
]

fig, axes = plt.subplots(1, 3, figsize=(18, 6))
for ax, (nama, src) in zip(axes, sumber):
    gdf_3857.plot(ax=ax, color='red', markersize=50, edgecolor='black')
    ax.set_xlim(xmin-pad, xmax+pad); ax.set_ylim(ymin-pad, ymax+pad)
    cx.add_basemap(ax, source=src, zoom=5)   # attribusi dibiarkan tampil
    ax.set_title(nama); ax.set_axis_off()
plt.tight_layout(); plt.show()
```

***Hasil yang diharapkan (karakter umum tiap basemap, cocokkan dengan gambarmu):***

| Basemap | Tampilan | Kelebihan untuk titik api | Kekurangan |
|---------|----------|---------------------------|------------|
| Esri World Street Map | Peta jalan berwarna, label kota dan provinsi jelas | Konteks wilayah mudah dibaca, cocok untuk melaporkan lokasi | Label kota bisa tertutup titik merah |
| CartoDB Positron | Abu-abu sangat terang, hampir tanpa warna | Titik merah paling menonjol, cocok untuk peta analisis | Label tipis, konteks medan sulit dibaca |
| Esri World Imagery | Citra satelit | Terlihat hutan, lahan gambut, kebun, dan bekas terbakar (area gelap) | Tanggal citra tidak sama dengan tanggal hotspot, dan resolusi berbeda antarwilayah |

**Analisis.** Basemap hanya memberi konteks visual. Angka dan klasifikasi titik tidak berubah sama sekali. Perbedaan ini penting dalam pelaporan: Positron cocok untuk menonjolkan sebaran titik, Street Map cocok untuk menunjukkan lokasi administratif, dan Imagery cocok untuk memeriksa apakah titik api benar-benar berada di area vegetasi atau perkebunan. Untuk Imagery, kita tidak bisa memastikan bahwa citra diambil pada tanggal hotspot, jadi kesimpulan dari citra hanya bersifat dugaan. Ingat juga bahwa attribusi harus tetap tampil, karena penyedia tile mensyaratkannya. Modul pada Langkah 9 dan 10 mematikan attribusi hanya untuk menghemat ruang.

---

## Soal 5

Soal:

> Buat peta animasi (GIF) untuk rentang tanggal yang sudah diperluas, lalu jelaskan pola pergerakan titik api yang kamu amati.

**Langkah pembuatan.** Rentang diperluas menjadi 9 sampai 16 September. Tambahkan data untuk 15 dan 16 September dengan format yang sama seperti Langkah 2. Setelah itu buat ulang GeoDataFrame dan animasi dengan pola Langkah 10.

```python
# Rentang tanggal yang diperluas (tambahkan data 15-16 Sep ke df dulu, format sama dengan Langkah 2)
tgls = ['2022-09-09','2022-09-10','2022-09-11','2022-09-12',
        '2022-09-13','2022-09-14','2022-09-15','2022-09-16']

gdf = gpd.GeoDataFrame(df, geometry=gpd.points_from_xy(df.longitude, df.latitude), crs='EPSG:4326')
gdf_3857 = gdf.to_crs(epsg=3857)
xmin, ymin, xmax, ymax = gdf_3857.total_bounds; pad = 3e5

from matplotlib.animation import FuncAnimation, PillowWriter
fig, ax = plt.subplots(figsize=(8, 7))
def update(i):
    ax.clear()
    ax.set_xlim(xmin-pad, xmax+pad); ax.set_ylim(ymin-pad, ymax+pad)
    sub = gdf_3857[gdf_3857['acq_date'] == tgls[i]]
    if len(sub): sub.plot(ax=ax, color='red', markersize=80)
    cx.add_basemap(ax, source=cx.providers.Esri.WorldStreetMap, zoom=5, attribution=False)
    ax.set_title('Hotspot ' + tgls[i]); ax.set_axis_off()

anim = FuncAnimation(fig, update, frames=len(tgls))
anim.save('hotspot_diperluas.gif', writer=PillowWriter(fps=1))

# Ringkasan jumlah titik per tanggal (dasar untuk analisis)
print(gdf.groupby(gdf['acq_date'].dt.strftime('%Y-%m-%d')).size())
```

***Hasil: berkas `hotspot_diperluas.gif` berisi delapan frame, dari 9 sampai 16 September. Pada frame 15 dan 16 September, peta menampilkan titik tambahan dari data yang kamu masukkan.***

**Jumlah titik per tanggal (dari data modul, 9-14 September):**

| Tanggal | Jumlah titik | Lokasi aktif |
|---------|--------------|--------------|
| 9 Sep | 2 | Riau (0.52, 101.45), Riau (0.90, 102.10) |
| 10 Sep | 3 | Riau, Sumsel (-2.95, 104.05), Kalbar (-0.05, 109.33) |
| 11 Sep | 4 | Riau, Sumsel, Jambi (-1.61, 103.61), Kalteng (-2.21, 113.90) |
| 12 Sep | 3 | Riau, Lampung (-5.45, 105.26), Kalsel (-3.32, 114.59) |
| 13 Sep | 3 | Sumbar (-0.95, 100.37), Kalteng, Kalbar (-1.50, 110.42) |
| 14 Sep | 1 | Kaltim (-1.27, 116.89) |

**Interpretasi pola dari data modul (9-14 September).** Bagian ini membaca pola dari data yang sudah ada. Setelah data 15-16 September ditambahkan, bagian ini perlu diperbarui dengan hasil GIF-mu.

1. **Klaster Riau memiliki siklus hidup yang jelas.** Titik muncul pada 9 September, bertahan sampai 12 September, lalu hilang pada 13 dan 14 September. Dalam istilah ESTDM pada Dasar Teori, ini adalah **appear** lalu **disappear**, dengan durasi 4 hari. Koordinatnya tidak berubah, sehingga titik ini **tidak bergerak**. Yang terjadi adalah api yang bertahan di satu tempat.

2. **Klaster Kalimantan Tengah bersifat berulang, bukan berkelanjutan.** Titik di (-2.21, 113.90) muncul 11 September, hilang 12 September, lalu muncul lagi 13 September. Kejadian seperti ini tidak dihitung sebagai streak, padahal lokasinya bermasalah berulang. Jika hanya memakai streak, kasus ini akan terlewat.

3. **Sebagian besar titik hanya muncul sekali.** Dari 11 lokasi unik, delapan hanya tercatat sekali. Tiga sisanya adalah Riau (4 hari), Sumsel (2 hari), dan Kalteng (2 hari, tidak berurutan). Pola sporadis seperti ini bisa berarti satu kejadian kecil yang cepat padam, atau noise deteksi satelit. Data ini tidak cukup untuk membedakan keduanya.

4. **Titik tidak dapat menunjukkan pergerakan.** Setiap titik adalah lokasi terpisah pada tanggal tertentu. Dari titik-titik ini, kita hanya bisa melihat kemunculan dan hilangnya titik, tidak bisa melihat arah api. Jika ingin melihat **expansion** atau **contraction** seperti pada Dasar Teori, objek harus berupa polygon, misalnya convex hull atau buffer dari titik-titik per tanggal. Dengan begitu luas area per hari bisa dibandingkan.

5. **Puncak aktivitas pada 11 September.** Jumlah titik naik dari 2 ke 4 lalu turun ke 1 pada 14 September. Dengan 8 hari data, pola ini bisa dibandingkan dengan hari-hari tambahan. Apakah puncak di 11 September tetap menjadi yang tertinggi, atau ada puncak baru di 15-16 September?

**Kerangka interpretasi untuk GIF-mu.** Gunakan pertanyaan berikut untuk menulis jawaban Soal 5:

- Apakah klaster Riau bertahan sampai 15 atau 16 September? Jika tidak, pada hari apa ia hilang?
- Berapa titik baru yang **appear** pada 15-16 September, dan di mana lokasinya?
- Apakah ada lokasi yang **disappear** lalu **muncul kembali** seperti Kalimantan Tengah?
- Apakah jumlah titik per hari naik, turun, atau stabil? Tulis angkanya.
- Apakah sebaran bergeser ke satu arah (misalnya ke timur) atau tersebar di Sumatra dan Kalimantan?
- Apa keterbatasan data ini (koordinat tidak berubah antar-hari, brightness contoh, jumlah titik kecil) yang membuat kesimpulanmu perlu hati-hati?

---

## Ringkasan Hasil

| Soal | Status | Catatan |
|------|--------|---------|
| 1 | Kode siap, hasil peta perlu dijalankan di Colab | 4 titik baru: Dumai, Bengkulu, Sampit, Medan |
| 2 | Prediksi: Riau terbanyak (6 titik) | Cocokkan nama provinsi dan titik NaN dengan output Colab |
| 3 | Sudah dijalankan dan dibuktikan dengan pandas | Singkawang (0.91, 108.99) streak 4, True |
| 4 | Kode siap, perbandingan visual perlu dijalankan | Positron untuk menonjolkan titik, Imagery untuk memeriksa vegetasi |
| 5 | Kode siap, GIF dan interpretasi perlu data 15-16 Sep | Riau appear-disappear 4 hari; Kalteng berulang; titik tidak bergerak |
