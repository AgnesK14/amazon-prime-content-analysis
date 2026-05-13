# 📊 Amazon Prime Content Analysis
**Trends, Genres, and Audience Insights**

> Mini Course Project · RevoU Data Analytics Program

---

## 📌 Project Overview

Proyek ini menganalisis **9.500+ konten Amazon Prime Video** untuk mengidentifikasi:
- Tren penambahan konten dari dekade ke dekade
- Distribusi genre dan tipe konten (Movie vs TV Show)
- Karakteristik audiens berdasarkan kategori rating
- Pola durasi film dan jumlah season TV Show

**Tools:** Microsoft Excel (Power Query, Pivot Table, Chart)  
**Dataset:** 9.687 rows · 9 columns · Amazon Prime Video Catalog

---

## 📁 File Structure

```
amazon-prime-content-analysis/
│
├── data/
│   ├── raw/
│   │   └── amazon_prime_titles.csv
│   │
│   └── processed/
│       ├── amazon_prime_cleaned.csv
│
├── assets/
│   ├── dashboard_preview.png
|   ├── Data Model.png
│   ├── data_model.png
│   └── power_query_workflow.png
│
└── README.md
```

---

## 🗂️ Dataset Overview

|Kolom	     |  Deskripsi          |
|------------|---------------------|
|show_id	   |  ID unik konten     |
|type	       |  Movie / TV Show    |
|title	     |  Judul konten       |
|country	   |  Negara produksi    |
|release_year|	Tahun rilis        |
|rating	     |  Rating konten      |
|duration	   |  Durasi (min/Season)|
|listed_in	 |  Genre              |
|description |  Sinopsis           |

---

## 🔍 Data Quality Issues

Sebelum transformasi, ditemukan beberapa masalah kualitas data:

| Issue                        | Detail                                                   |
| ---------------------------- | -------------------------------------------------------- |
| Missing values tinggi        | `country` memiliki 89% nilai kosong                      |
| Label rating tidak konsisten | Terdapat beberapa variasi label untuk kategori yang sama |
| Mixed data                   | `duration` mencampur nilai menit dan season              |
| Multi-value column           | `listed_in` berisi lebih dari satu genre                 |
| Whitespace                   | Spasi tersembunyi pada beberapa kolom                    |
| Outlier                      | Durasi ekstrem dan tidak realistis                       |

---

## 🔧 Data Cleaning & Transformation (Power Query)

### 1. Remove Irrelevant Columns
Kolom `country` (89% missing) dan `description` dihapus karena tidak relevan untuk analisis.

Power Query Feature:
`Home - Remove Columns`

### 2. Filter Tipe Konten
Hanya mempertahankan baris dengan type = "Movie" atau "TV Show".
Menghapus nilai kosong atau kategori tidak valid.

Power Query Feature:
`Column Filter`

### 3. Standarisasi Label Rating
Ditemukan 10+ variasi label untuk nilai yang sama. Semua distandarisasi ke format konsisten.

| Before | After |
|----------|-----|
| blank    | NR  |
| 16       | 16+ |
| AGES_16_ | 16+ |
| AGES_18_ | 18+ |
| NOT_RATE | NR  |
| UNRATED  | NR  |

Power Query Feature:
`Transform → Replace Values`

### 4. Kategorisasi Usia Audiens
Membuat kolom baru `age_category`:

| Kategori | Rating yang Termasuk              |
|----------|-----------------------------------|
| Kids     | TV-Y, TV-Y7, G, ALL, 7+           |
| Teen     | TV-PG, PG, PG-13, TV-14, 13+, 16+ |
| Adult    | R, NC-17, 18+, TV-MA              |
| Unknown  | NR, TV-NR                         |

Power Query Feature:
`Add Column → Conditional Column`

### 5. Transformasi Durasi
Kolom `duration` awalnya berisi dua jenis informasi berbeda dalam satu kolom, yaitu:
durasi film dalam menit
jumlah season TV Show

Kolom tersebut terlebih dahulu dipisahkan berdasarkan spasi menggunakan delimiter " " sehingga menghasilkan dua kolom baru:

Contoh:
|Sebelum	|  Sesudah      |
|---------|---------------|
|113 min	| min           |
|1 Season	| Season        | 

Power Query Feature:
`Transform → Split Column → By Delimiter`

Setelah proses split, dilakukan standarisasi unit:
|Sebelum | Sesudah |
|--------|---------|
| min	   | Minute  |
| Seasons|	Season |

Power Query Feature:
`Transform → Replace Values`

### 6. Tambah Kolom `Movie_Duration_Mins` & `TV_Season_Count`
Memisahkan metrik durasi berdasarkan tipe konten.

|type	   |duration_value	| duration_unit	|TV_Season_Count |
|--------|----------------|---------------|----------------|
|Movie	 | 113	          | Minute	      | null           | 
|TV Show |	2	            | Season	      | 2              |

Power Query Feature:
`Add Column → Conditional Column`


### 7. Hapus Anomali Durasi
Record dengan durasi tidak realistis dihapus:
- Terlalu pendek: 1–10 menit
- Ekstrem: 479, 480, 481, 485, 540, 541, 550, 601 menit
- Durasi 0 dihapus

Power Query Feature:
`Number Filters`

### 8. Trim Whitespace
Menghindari duplikasi tersembunyi akibat spasi pada kolom `listed_in`.

Power Query Feature:
`Transform → Format → Trim`

### 9. Hapus Duplikasi
Duplikasi dihapus berdasarkan `show_id` sebagai unique identifier.

### 10. Tambah Kolom `Decade`
Untuk analisis tren jangka panjang berdasarkan dekade rilis.
Formula: Integer divide by 10 → multiply by 10 → tambah suffix "s"
```
Duplicate release_year -> Integer divide -> Multiply -> Add suffix -> Rename
```
```
2018 → 2010s
1994 → 1990s
```

## 🗃️ Data Modeling — Genre Normalization
Karena satu konten bisa memiliki beberapa genre sekaligus, dibuat tabel relasional terpisah.

| show_id | listed_in (original)  |
|---------|-----------------------|
| s1      | Drama, Comedy, Romance|

Setelah normalisasi (tabel `Content_Genre`):
| show_id | genre   |
|---------|---------|
| s1      | Drama   |
| s1      | Comedy  |
| s1      | Romance |

Relasi antar tabel:
`amazon_prime_titles[show_id]  1 ──── *  content_genre[show_id]`

---

## 📊 Final Dataset

### Tabel 1: `amazon_prime_titles`

| Kolom               | Keterangan                    |
|---------------------|-------------------------------|
| show_id             | ID unik konten                |
| type                | Movie / TV Show               |
| title               | Judul konten                  |
| release_year        | Tahun rilis                   |
| rating              | Rating standar                |
| age_category        | Kids / Teen / Adult / Unknown |
| duration_value      | Nilai numerik durasi          |
| duration_unit       | Minute / Season               |
| Movie_Duration_Mins | Durasi film dalam menit       |
| TV_Season_Count     | Jumlah season TV Show         |
| Decade              | Dekade rilis (misal: 2010s)   |

### Tabel 2: `content_genre`
| Kolom   | Keterangan                     |
|---------|--------------------------------|
| show_id | ID unik (FK ke tabel utama)    |
| genre   | Nama genre (1 baris per genre) |

---

## 📈 Key Insights

- **9.547 total konten** — 81% Movie, 19% TV Show
- **Drama** adalah genre terbesar dengan 3.672 konten
- **Lonjakan konten signifikan** terjadi di era 2010s (4.293 konten)
- **Rata-rata durasi film** 91 menit
- **Segmen Teen** mendominasi audiens dengan 4.669 konten
- **127 konten** tidak memiliki rating yang teridentifikasi (Unknown)

---

## Dashboard Preview

<img width="1030" height="592" alt="image" src="https://github.com/user-attachments/assets/1e5f9b81-1839-4c9d-bc9a-1dce78c1f112" />

---

## 🎓 About This Project

Proyek ini merupakan bagian dari **RevoU Mini Course – Data Analytics**.  
Tujuan utama adalah mempraktikkan end-to-end data analysis workflow:  
**Data Cleaning → EDA → Visualization → Storytelling**

---

*Dataset source: Amazon Prime Video catalog (public dataset)*
