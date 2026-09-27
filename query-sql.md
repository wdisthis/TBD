# Kumpulan Query SQL

1. Menampilkan seluruh data mata kuliah
```sql
SELECT * FROM course;
```

2. Menampilkan mahasiswa dengan total kredit lebih dari 100
```sql
SELECT * FROM student WHERE tot_cred > 100;
```

3. Menampilkan nama dan departemen dosen dengan gaji di atas 50000
```sql
SELECT name, dept_name FROM instructor WHERE salary > 50000;
```

4. Menghitung jumlah mahasiswa per departemen
```sql
SELECT dept_name, COUNT(*) AS jumlah_mhs
FROM student
GROUP BY dept_name;
```

5. Menampilkan nama mahasiswa, departemen, judul mata kuliah, dan nilai
```sql
SELECT s.name, s.dept_name, c.title, t.grade
FROM student s
JOIN takes t ON s.id = t.id
JOIN course c ON t.course_id = c.course_id;
```

6. Menampilkan rata-rata gaji dosen per departemen
```sql
SELECT i.name, d.dept_name, AVG(i.salary) AS rata_gaji
FROM instructor i
JOIN department d ON i.dept_name = d.dept_name
GROUP BY d.dept_name;
```

7. Menampilkan jumlah mahasiswa per mata kuliah, diurutkan dari yang terbanyak
```sql
SELECT c.title, COUNT(t.id) AS jumlah_mhs
FROM course c
JOIN takes t ON c.course_id = t.course_id
GROUP BY c.title
ORDER BY jumlah_mhs DESC;
```

8. Menampilkan mahasiswa yang mengambil lebih dari 5 mata kuliah (top 10)
```sql
SELECT s.name, s.dept_name, COUNT(t.course_id) AS jumlah_matkul
FROM student s
JOIN takes t ON s.id = t.id
GROUP BY s.id
HAVING COUNT(t.course_id) > 5
ORDER BY jumlah_matkul DESC
LIMIT 10;
```

9. Menampilkan jumlah mahasiswa per mata kuliah dan dosen pengajar
```sql
SELECT c.title, i.name, COUNT(t.id) AS jumlah_mhs
FROM teaches te
JOIN instructor i ON te.id = i.id
JOIN course c ON te.course_id = c.course_id
JOIN takes t ON c.course_id = t.course_id
GROUP BY c.title, i.name
ORDER BY jumlah_mhs DESC;
```

10. Menampilkan mahasiswa yang telah memenuhi semua prasyarat mata kuliah yang diambilnya
```sql
SELECT s.name, s.dept_name
FROM student s
WHERE NOT EXISTS (
    SELECT *
    FROM prereq p
    WHERE p.course_id IN (
        SELECT t.course_id FROM takes t WHERE t.id = s.id
    )
    AND p.prereq_id NOT IN (
        SELECT t2.course_id FROM takes t2 WHERE t2.id = s.id
    )
);
```
