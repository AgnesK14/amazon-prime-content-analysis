# 📊 Amazon Prime Content Analysis

> Analisis tren konten, distribusi genre, dan karakteristik audiens pada katalog Amazon Prime Video.

---

## 🗂️ Project Overview

| Item            | Detail                                                                                                        |
| --------------- | ------------------------------------------------------------------------------------------------------------- |
| **Dataset**     | Amazon Prime Video Catalog                                                                                    |
| **Jumlah Data** | 9.687 rows, 9 columns                                                                                         |
| **Tools**       | Microsoft Excel (Power Query, Pivot Table, Chart)                                                             |
| **Tujuan**      | Mengidentifikasi tren konten, distribusi genre, pola rating audiens, dan karakteristik durasi Movie & TV Show |

---

## 📁 Struktur File

```bash
📦 amazon-prime-content-analysis
 ┣ 📄 amazon_prime_titles.csv          # Dataset mentah
 ┣ 📄 amazon_prime_cleaned.xlsx        # Dataset setelah cleaning (Power Query)
 ┣ 📄 content_genre.xlsx               # Tabel relasional genre
 ┣ 📊 dashboard_preview.png            # Dashboard utama
 ┣ 📊 data_model.png                   # Relasi antar tabel
 ┣ 📊 power_query_workflow.png         # Workflow transformasi data
 ┗ 📄 README.md
```

---

## 🔍 Dataset Info

```python
RangeIndex: 9.687 entries
Columns: 9

#   Column         Non-Null    Dtype
0   show_id        9687        object
1   type           9687        object
2   title          9687        object
3   country        1060        object   ← 89% missing value
4   release_year   9687        int64
5   rating         9687        object   ← label tidak konsisten
6   duration       9687        object   ← mixed value
7   listed_in      9687        object   ← multi-value column
8   description    9687        object
```

---

## 🛠️ Data Cleaning Process (Excel – Power Query)

### 1. Promoted Headers

Baris pertama dataset dijadikan nama kolom menggunakan fitur:

**Home → Use First Row as Headers**

---

### 2. Removed Irrelevant Columns

Kolom `country` dan `description` dihapus menggunakan:

**Home → Remove Columns**

Tujuan:

* `country` memiliki ~89% missing value
* `description` tidak digunakan dalam analisis dashboard

---

### 3. Filtered Invalid Content Type

Hanya mempertahankan kategori:

* `Movie`
* `TV Show`

Langkah:

**Dropdown Filter pada kolom type**

Tujuan:

Menghapus record kosong atau kategori yang tidak relevan.

---

### 4. Standardized Rating Labels

Beberapa label rating tidak konsisten sehingga dilakukan standarisasi menggunakan:

**Transform → Replace Values**

Perubahan label:

| Sebelum   | Sesudah |
| --------- | ------- |
| blank     | NR      |
| 16        | 16+     |
| ALL_AGES  | ALL     |
| AGES_16+_ | 16+     |
| 16++      | 16+     |
| AGES_18_  | 18+     |
| NOT_RATE  | NR      |
| UNRATED   | NR      |

Tujuan:

Memastikan konsistensi kategori rating untuk analisis audiens.

---

### 5. Added AGE_CATEGORY

Kolom baru dibuat untuk mengelompokkan rating berdasarkan kategori usia audiens.

Langkah:

**Add Column → Conditional Column**

Kategori:

| Kategori | Rating                            |
| -------- | --------------------------------- |
| Kids     | TV-Y, TV-Y7, G, ALL, 7+, TV-G     |
| Teen     | TV-PG, PG, PG-13, TV-14, 13+, 16+ |
| Adult    | R, NC-17, 18+, TV-MA              |
| Unknown  | NR, TV-NR                         |

Tujuan:

Mempermudah segmentasi konten berdasarkan target audiens.

---

### 6. Split Duration Column

Kolom `duration` dipisahkan menjadi:

* `duration_value`
* `duration_unit`

Langkah:

**Transform → Split Column → By Delimiter**

Contoh:

| Sebelum   | duration_value | duration_unit |
| --------- | -------------- | ------------- |
| 113 min   | 113            | min           |
| 2 Seasons | 2              | Seasons       |

---

### 7. Standardized Duration Unit

Nilai unit durasi distandarisasi menggunakan:

**Transform → Replace Values**

Perubahan:

| Sebelum | Sesudah |
| ------- | ------- |
| Seasons | Season  |
| min     | Minute  |

Tujuan:

Menghindari inkonsistensi label unit durasi.

---

### 8. Added MOVIE_DURATION_MINS

Kolom baru dibuat untuk menyimpan durasi khusus Movie.

Langkah:

**Add Column → Conditional Column**

Logic:

```python
if duration_unit = "Minute"
then duration_value
else null
```

---

### 9. Added TV_SEASON_COUNT

Kolom baru dibuat untuk menyimpan jumlah season khusus TV Show.

Langkah:

**Add Column → Conditional Column**

Logic:

```python
if duration_unit = "Season"
then duration_value
else null
```

---

### 10. Changed Data Type

Kolom numerik diubah ke tipe integer menggunakan:

**Transform → Data Type**

Kolom:

* `duration_value`
* `Movie_Duration_Mins`
* `TV_Season_Count`

---

### 11. Trimmed Text – listed_in

Spasi tersembunyi pada kolom `listed_in` dihapus menggunakan:

**Transform → Format → Trim**

Tujuan:

Menghindari duplikasi genre akibat whitespace.

---

### 12. Filtered Invalid Duration

Record dengan:

```python
duration_value = 0
```

dihapus menggunakan:

**Number Filters**

Tujuan:

Menghapus durasi tidak valid.

---

### 13. Filtered Duration Outliers

Durasi tidak realistis dihapus dari analisis.

Durasi yang dihapus:

```python
1–10 minutes
479, 480, 481, 485,
540, 541, 550, 601 minutes
```

Tujuan:

Mengurangi noise dan outlier ekstrem pada analisis durasi film.

---

### 14. Added Decade

Kolom baru dibuat berdasarkan `release_year` untuk analisis tren per dekade.

Langkah:

```python
Duplicate release_year
→ Integer Divide by 10
→ Multiply by 10
→ Add suffix "s"
→ Rename Column
```

Contoh:

```python
2018 → 2010s
1994 → 1990s
```

Tujuan:

Mempermudah visualisasi tren pertumbuhan konten jangka panjang.

---

## 🗃️ Data Modeling — Genre Normalization

Karena satu konten dapat memiliki lebih dari satu genre, dibuat tabel relasional terpisah untuk normalisasi genre.

### Proses Transformasi `content_genre`

Langkah:

```python
Select show_id & listed_in
→ Split by delimiter ","
→ Expand to rows
→ Trim whitespace
→ Replace "and Culture" → "Culture"
```

Contoh sebelum normalisasi:

| show_id | listed_in              |
| ------- | ---------------------- |
| s1      | Drama, Comedy, Romance |

Setelah normalisasi:

| show_id | genre   |
| ------- | ------- |
| s1      | Drama   |
| s1      | Comedy  |
| s1      | Romance |

Relasi tabel:

```python
amazon_prime_titles[show_id]
        1 ──────── *
content_genre[show_id]
```

Tujuan:

Memungkinkan analisis genre yang lebih akurat tanpa multi-value cell.

---

## 📊 Dashboard Preview

### Content Performance Overview

Menampilkan:

KPI utama total konten
Total Movie dan TV Show
Average movie duration
Tren jumlah konten berdasarkan dekade rilis
Top 10 genre
Distribusi tipe konten (Movie vs TV Show)
Distribusi kategori rating
Distribusi kategori usia audiens
Distribusi durasi film
Interactive slicer:

---

## 💡 Key Insights

* **9.547 total konten** berhasil dianalisis
* **Movie** mendominasi katalog dengan proporsi **81%**
* **Drama** menjadi genre terbesar dengan **3.672 konten**
* Lonjakan pertumbuhan konten tertinggi terjadi pada era **2010s**
* **Teen** merupakan kategori audiens terbesar dengan **4.669 konten**
* Rata-rata durasi film adalah **91 menit**
* Terdapat **127 konten** dengan rating tidak teridentifikasi (`Unknown`)

---

## 🚀 How to Use

### 1. Download Dataset

Download file:

```bash
amazon_prime_cleaned.xlsx
```

---

### 2. Open in Excel

Gunakan:

**Microsoft Excel 2016+**

---

### 3. Refresh Power Query

Buka:

**Data → Refresh All**

Untuk memuat ulang seluruh transformasi data.

---

### 4. Explore Dashboard

Buka dashboard utama melalui file:

```bash
dashboard_preview.png
```

atau workbook dashboard Excel.

---

## Author

**Agnes Kamelia Narjun**

📧 [agneskameliaa@gmail.com]
🔗 [LinkedIn Profile]
