# Technical Evidence & Validation Report — Capstone Project SwiftBite (`SB-P04`)

- **Nama Peserta:** Sintia Tri Wulandari
- **Variant ID:** `SB-P04`
- **Periode Data:** `2026-01-02` s.d. `2026-03-31` (Kuartal I 2026)
- **Grain Analisis:** `1 baris = 1 order (order_id)` (`354` order)
- **File Output Utama:**
  - Dataset Analisis Final: `data/processed/capstone_swiftbite/swiftbite_capstone_analysis_ready.csv`
  - Dashboard HTML Interaktif: `outputs/capstone_swiftbite/swiftbite_capstone_dashboard.html`

---

## 1. Data Intake & Join Validation (Appendix A1)

Penggabungan dilakukan dengan metode **`LEFT JOIN`** dari dataset order-level hasil Praktik Mandiri 2 (`swiftbite_pm2_integrated_order_level.csv`) ke dataset ulasan pelanggan (`swiftbite_reviews.csv`) menggunakan kunci `order_id`.

| No | Pemeriksaan Validasi (Mandatory Check) | Nilai Sebelum Join | Nilai Sesudah Join | Kriteria Lulus | Status |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **1** | **Jumlah Baris (`row_count`)** | `354` baris | `354` baris | Tidak berubah (`354 == 354`), 0 fan-out | **PASS** |
| **2** | **Keunikan `order_id` (`nunique`)** | `354` unik | `354` unik | `0` duplikat, `0` null pada `order_id` | **PASS** |
| **3** | **Order dengan Review (`has_review = True`)** | `203` baris di `reviews.csv` | `203` order (`57,34%`) | Seluruh `203` review terpetakan 1:1 ke order valid | **PASS** |
| **4** | **Order Tanpa Review (`No Review`)** | — | `151` order (`42,66%`) | Dipertahankan utuh sebagai status `No Review` (`312` delivered = `203` reviewed + `109` no review; `42` cancelled = `42` no review) | **PASS** |
| **5** | **Unmatched Reviews (Anti-Join `reviews - orders`)** | `0` baris | `0` baris | Tidak ada review yatim (`orphan review`) di luar daftar `order_id` | **PASS** |
| **6** | **Konservasi Moneter & Safe Data (Non-PII)** | Gross `Rp42.127.000` / Refund `Rp1.850.000` / Net `Rp40.277.000` | Gross `Rp42.127.000` / Refund `Rp1.850.000` / Net `Rp40.277.000` | Nilai pembayaran tidak terduplikasi; `0` kolom PII langsung (`customer_name`, `phone`, `email`, `street_address`) | **PASS** |

---

## 2. Definisi & Referensi Perhitungan KPI (Appendix A2)

| Kode | Nama KPI | Formula Perhitungan | Numerator (Pembilang) | Denominator (Penyebut) | Nilai Aktual (`SB-P04`) | Aturan Edge-Case (`Denominator = 0`) |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **KPI-01** | **Total Orders** | `COUNT(DISTINCT order_id)` | `354` order unik | Seluruh populasi order (`354`) | **`354` order** (`312` delivered, `42` cancelled) | Menampilkan `0` order |
| **KPI-02** | **Net Payment Value** | `SUM(successful_payment_amount) - SUM(refunded_payment_amount)` | `Rp42.127.000 - Rp1.850.000` | `354` order (atau `312` order berbayar bersih) | **`Rp40.277.000`** (AOV keseluruhan: `Rp113.777`; AOV paid: `Rp129.093`) | Menampilkan `Rp0` dan AOV `N/A` |
| **KPI-03** | **Review Coverage** | `COUNT(order dengan review_id NOT NULL) / COUNT(order_id) * 100%` | `203` order dengan review | `354` total order | **`57,34%`** (`203 / 354`) • Status `No Review`: `151` (`42,66%`) | Menampilkan `N/A (0 valid)` |
| **KPI-04** | **Late Delivery Rate** | `COUNT(delivered & is_late = True) / COUNT(delivery_status = 'delivered') * 100%` | `98` order terlambat (`delay_minutes > 0`) | `312` order dengan `delivery_status = 'delivered'` | **`31,41%`** (`98 / 312`) • Rata-rata delay (Late): `+16,34 menit` | Menampilkan `N/A (0 valid)` (tidak membagi dengan `0`) |
| **KPI-05** | **Negative Sentiment Rate** *(Pendukung CX)* | `COUNT(sentiment_label = 'Negative') / COUNT(order dengan review) * 100%` | `33` review bersentimen `Negative` | `203` order yang memiliki review | **`16,26%`** (`33 / 203` review) atau `9,32%` (`33 / 354` total order) | Menampilkan `N/A (0 valid)` |

---

## 3. Aturan Klasifikasi & Validasi Sampel Sentimen (Appendix A3)

### 3.1 Pemisahan Tegas 5 Kategori Resmi Sentimen vs Status `No Review`
- **5 Kategori Resmi Sentimen Review sesuai PPT (Denominator = `203` review masuk):**
  1. **`Positive` (`91` review | `44,83%` dari review | `25,71%` dari total order):** Teks ulasan memuat pujian eksplisit tanpa keluhan (rasa enak/konsisten/fresh, pengantaran cepat/hangat, porsi banyak/pas, kurir ramah/sopan, kemasan rapi/aman, harga *worth it*/seimbang).
  2. **`Neutral` (`50` review | `24,63%` dari review | `14,12%` dari total order):** Teks ulasan datar/standar tanpa pujian maupun keluhan kuat (*"Pengiriman biasa saja dan sesuai perkiraan."*, *"Makanannya cukup oke, tapi tidak terlalu spesial."*, *"Harga dan kualitasnya cukup standar."*).
  3. **`Negative` (`33` review | `16,26%` dari review | `9,32%` dari total order):** Teks ulasan memuat keluhan eksplisit terkait keterlambatan, makanan dingin/kurang enak/kurang fresh, komunikasi kurir buruk/sulit dihubungi, kemasan rusak/tumpah, atau porsi/harga mengecewakan.
  4. **`Unknown` (`16` review | `7,88%` dari review | `4,52%` dari total order):** Teks ulasan memiliki **sentimen campuran/kontradiktif** dalam satu kalimat (memuji satu aspek tetapi mengeluhkan aspek lain, misalnya *"Makanannya enak, tapi datangnya agak lama."*, *"Driver ramah, tetapi makanannya sudah tidak terlalu hangat."*).
  5. **`No Text` (`13` review | `6,40%` dari review | `3,67%` dari total order):** Pelanggan mengirimkan `review_id` dan `rating` bintang, tetapi kolom `review_text` kosong (`null`/whitespace).
- **Status Ketersediaan Review (`review_status = 'No Review'`):**
  - Sebanyak **`151` order (`42,66%` dari `354` order)** tidak memiliki ulasan sama sekali (`109` order `delivered` + `42` order `cancelled`). Status `No Review` disimpan pada kolom `review_status` dan **tidak dicampur** ke dalam pembagi persentase sentimen ulasan.

### 3.2 Tabel Validasi Sampel Manual (`11` Sampel Mewakili 5 Kategori & Anomali Rating vs Teks)

| No | `review_id` | `order_id` | Zona | `is_late` | Rating | `review_text` | Label Sentimen | Alasan Validasi Manual |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **1** | `RV00016` | `O00234` | West | `False` | `5` | *"Makanannya enak dan rasanya konsisten."* | **`Positive`** | Pujian jelas terhadap rasa dan konsistensi makanan; selaras dengan rating 5. |
| **2** | `RV00002` | `O00148` | South | `False` | `1` | *"Makanannya enak dan rasanya konsisten."* | **`Positive`** | **Anomali Rating vs Teks:** Meskipun rating `1`, isi teks memuji makanan secara eksplisit sehingga diklasifikasikan `Positive` berdasarkan bukti teks. |
| **3** | `RV00005` | `O00273` | South | `False` | `3` | *"Pengiriman biasa saja dan sesuai perkiraan."* | **`Neutral`** | Pernyataan normatif/standar tanpa emosi positif maupun negatif yang dominan. |
| **4** | `RV00014` | `O00034` | West | `False` | `4` | *"Makanannya cukup oke, tapi tidak terlalu spesial."* | **`Neutral`** | Penilaian moderat (*cukup oke, tidak spesial*) sehingga masuk kategori `Neutral`. |
| **5** | `RV00024` | `O00280` | West | `True` | `1` | *"Pesanan datang terlambat dan makanan sudah dingin."* | **`Negative`** | Keluhan langsung akibat keterlambatan pengantaran (`delay +22m`) dan penurunan kualitas suhu makanan. |
| **6** | `RV00008` | `O00144` | North | `False` | `5` | *"Komunikasi dengan driver kurang baik."* | **`Negative`** | **Anomali Rating vs Teks:** Meskipun rating `5`, teks berisi keluhan eksplisit layanan kurir sehingga diklasifikasikan `Negative`. |
| **7** | `RV00027` | `O00208` | North | `False` | `5` | *"Porsinya cukup banyak, tapi harganya terasa agak mahal."* | **`Unknown`** | Sentimen campuran: memuji porsi (`+`) namun mengeluhkan persepsi harga (`-`). |
| **8** | `RV00060` | `O00161` | East | `False` | `4` | *"Driver ramah, tetapi makanannya sudah tidak terlalu hangat."* | **`Unknown`** | Sentimen campuran: memuji keramahan kurir (`+`) namun mengeluhkan suhu makanan (`-`). |
| **9** | `RV00032` | `O00040` | South | `False` | `5` | *"Makanannya enak, tapi datangnya agak lama."* | **`Unknown`** | Sentimen campuran: memuji rasa makanan (`+`) namun mengeluhkan durasi tunggu (`-`). |
| **10** | `RV00080` | `O00058` | North | `True` | `4` | *"Delivery cepat, tetapi packaging-nya kurang rapi."* | **`Unknown`** | Sentimen campuran: memuji kecepatan (`+`) namun mengeluhkan kerapian kemasan (`-`). |
| **11** | `RV00022` | `O00097` | North | `True` | `1` | *(Kosong / `NaN`)* | **`No Text`** | Memiliki `review_id` dan rating `1`, namun tidak ada teks ulasan untuk dianalisis. |

---

## 4. Pengujian Fungsional Dashboard & Traceability Matrix (Appendix A4)

### 4.1 Tabel Uji Skenario Filter Dashboard (`T-01` s.d. `T-04`)

| ID Uji | Skenario Filter pada Dashboard | Total Orders | Net Payment Value | Review Coverage | Late Delivery Rate | Negative Sentiment Rate | Hasil Verifikasi |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`T-01`** | **Default (Semua Zona, Semua Bulan, Semua Status)** | `354` | `Rp40.277.000` | `57,34%` (`203/354`) | `31,41%` (`98/312`) | `16,26%` (`33/203`) | **PASS** (Sesuai 100% dengan baseline Python) |
| **`T-02`** | **Filter Zona = `North` (Semua Bulan)** | `115` | `Rp13.237.000` | `59,13%` (`68/115`) | `33,66%` (`34/101`) | `17,65%` (`12/68`) | **PASS** (KPI & grafik ter-update dinamis) |
| **`T-03`** | **Filter Bulan = `2026-03` (Maret 2026)** | `108` | `Rp11.450.000` | `55,56%` (`60/108`) | `34,04%` (`32/94`) | `21,67%` (`13/60`) | **PASS** (Menampilkan kontraksi Maret & lonjakan Late Rate) |
| **`T-04`** | **Edge-Case: Status = `Cancelled / Tidak Terkirim (42)`** | `42` | `Rp0` (`Refund Rp1.850.000`) | `0,00%` (`0/42`) | **`N/A (0 valid)`** (`0/0`) | **`N/A (0 valid)`** (`0/0`) | **PASS** (Zero-denominator aman, bebas `NaN`/`Infinity`) |

### 4.2 Matriks Ketertelusuran (Traceability Matrix: `BQ -> Finding -> Evidence -> Recommendation`)

| Kode BQ | Pertanyaan Bisnis | Kode Temuan (Finding) | Bukti Kuantitatif & Visual (`SB-P04`) | Rekomendasi Prioritas & Target Dampak |
| :--- | :--- | :--- | :--- | :--- |
| **`BQ-01`** | Bagaimana performa pembayaran bersih dan distribusi order antar zona pelanggan serta periode bulan? | **`F-01`** (Konsentrasi 3 Zona Inti) & **`F-02`** (Kontraksi Maret `-22,74%` MoM & Refund di North) | • **North (`Rp13,24 Jt` / `32,86%`), Central (`Rp10,77 Jt` / `26,75%`), South (`Rp8,90 Jt` / `22,08%`)** menyumbang **`81,70%`** Net Payment (`298/354` order).<br>• Net Payment turun dari **`Rp14,82 Jt` (Feb)** ke **`Rp11,45 Jt` (Mar)** (`-22,74%`), diiringi kenaikan refund ke `Rp653 rb` (total refund Q1 `Rp1,85 Jt`, `45,78%` berasal dari zona **North** `Rp847 rb`). | **Prioritas P2 (Growth & Refund Control):** Audit penyebab pembatalan berbayar (`14` order refunded = `Rp1,85 Jt`, terutama di zona North) dan kampanye retensi terfokus di 3 zona inti pada awal Q2 untuk memulihkan volume ke `>=120 order/bulan`. |
| **`BQ-02`** | Bagaimana kinerja ketepatan waktu pengiriman (`Late Delivery Rate`) per zona, segmen pelanggan, dan kategori? | **`F-03`** (Bottleneck SLA di Zona North & West serta Segmen Loyal `40,00%`) | • **Late Delivery Rate keseluruhan `31,41%` (`98/312`)** dengan rata-rata keterlambatan `+16,34 menit`.<br>• Rate tertinggi di **West (`14/38 = 36,84%`)** dan volume terlambat tertinggi di **North (`34/101 = 33,66%`, menyumbang `34,69%` dari seluruh keterlambatan)**.<br>• Segmen **Loyal** mengalami keterlambatan tertinggi (**`30/75 = 40,00%`**) dan kategori **Dessert** mencapai **`26/60 = 43,33%`**. | **Prioritas P1 (Operasional Dispatch & SLA):** Realokasi buffer kurir jam sibuk di zona **North & West**, kalibrasi ulang *promised delivery time* untuk merchant **Dessert & Central/North**, serta *priority dispatch* untuk pesanan pelanggan **Loyal** (target menurunkan Late Rate dari `31,41%` ke `<22%`). |
| **`BQ-03`** | Bagaimana distribusi sentimen ulasan pelanggan dan sejauh mana keterlambatan memicu ulasan negatif? | **`F-04`** (Lonjakan Sentimen Negatif `3,89x` pada Order Terlambat) & **`F-05`** (Gap Coverage `42,66%` & Anomali Rating-Teks) | • Dari `203` review: **`Positive` `44,83%` (`91`)**, **`Neutral` `24,63%` (`50`)**, **`Negative` `16,26%` (`33`)**, **`Unknown` `7,88%` (`16`)**, **`No Text` `6,40%` (`13`)**; terpisah dari **`151` (`42,66%`) `No Review`**.<br>• Order **Late (`is_late=True`)** menghasilkan **`33,33%` (`21/63`) ulasan `Negative`** (rating rata-rata `2,73`), sedangkan order **On-Time** hanya **`8,57%` (`12/140`)** (rating rata-rata `4,03`). **`63,64%` (`21/33`)** ulasan negatif berakar dari keterlambatan/kurir. | **Prioritas P1 & P3 (Customer Experience & Data Quality):** Otomasi kompensasi/notifikasi proaktif saat keterlambatan `>10 menit` untuk meredam ulasan negatif, pengingat ulasan pasca-antar untuk menaikkan Review Coverage dari `57,34%` ke `>70%`, serta validasi UI rating vs teks di aplikasi. |

