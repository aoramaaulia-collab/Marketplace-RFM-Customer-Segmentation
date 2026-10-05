# Catatan Keputusan: Segmentasi RFM Marketplace

**Analis:** Aulia Aorama
**Tanggal pengerjaan:** 2026-09-14
**Tanggal acuan analisis:** 2026-03-31
**Skenario keputusan:** awal Q2 2026. Data ditarik per 2026-03-31, program berjalan April sampai Juni 2026, hasil dibaca 2026-06-30. Dokumen ini disusun 2026-09-14 sebagai studi kasus; setiap "sekarang", "kuartal ini", dan "kuartal depan" merujuk ke skenario itu.
**Sumber:** `transactions.csv`, dimuat ke tabel `transactions` di DuckDB, 50.980 baris mentah
**Alat:** SQL di DuckDB untuk seluruh pembersihan, agregasi, dan perhitungan; Python hanya untuk memuat data dan merender gambar.

*Catatan data: angka-angka di dokumen ini dihitung dari dataset yang dirancang meniru pola transaksi marketplace. Ini bukan data sebuah perusahaan sungguhan.*

Dokumen ini adalah sumber tunggal angka untuk keempat berkas keluaran. Memo, lampiran bukti, dan portofolio menyalin dari sini, tidak menghitung ulang sendiri.

Analisis yang menghasilkan angka-angka ini dikerjakan di notebook, yang menutup keenam langkah kerangka halaman 02 dan memuat profil data, uji kepekaan, uji bantahan, serta delapan gambar: https://colab.research.google.com/drive/1yj7O5AYiG0rpYbmpyH6Z7fkHKFSVk2J7

---

## 1. Definisi transaksi

**Yang dipakai.** Satu transaksi adalah satu baris unik pada kelima kolom, dengan `TotalValue` tidak NULL dan lebih besar dari nol, bertanggal sampai dengan 2026-03-31.

Dari 50.980 baris mentah, 2.850 baris (5,6%) dibuang dalam empat tahap terpisah, menyisakan 48.130 transaksi dari 5.000 pelanggan, rentang 2025-01-01 sampai 2026-03-31.

| Tahap | Membuang | Dibuang | Sisa |
|---|---|---|---|
| (mentah) | | | 50.980 |
| `deduped` | duplikat persis lima kolom | 2.400 | 48.580 |
| `null_filtered` | `TotalValue` NULL | 350 | 48.230 |
| `positive_filtered` | `TotalValue` ≤ 0 | 50 | 48.180 |
| `cleaned_data` | tanggal > acuan | 50 | 48.130 |

Keempat tahap dilaporkan terpisah, bukan digabung menjadi satu filter. Satu `WHERE` panjang menghasilkan angka akhir yang sama tetapi tidak bisa menjelaskan siapa membuang apa.

**Alasan tiap pembuangan.**

Duplikat persis dibuang karena baris identik termasuk `TransactionID`, dan nomor transaksi yang sama bukan dua kejadian melainkan satu kejadian yang tercatat dua kali. Membuangnya menegakkan definisi bahwa satu `TransactionID` adalah satu transaksi.

Pembayaran gagal (`TotalValue` NULL) dibuang karena RFM mengukur perilaku belanja, dan pembayaran yang gagal bukan belanja. Keputusan ini harus eksplisit: kalau dibiarkan, `sum()` melewati NULL sementara `count()` tetap menghitungnya, sehingga Frequency memakai definisi longgar dan Monetary memakai definisi ketat di kueri yang sama, tanpa satu peringatan pun.

Nilai nol dan negatif dibuang dengan alasan yang sama. Terendah Rp -690.111,44 (refund), dan 20 dari 50 baris bernilai tepat nol (sampel gratis atau voucher yang menutup seluruh harga). Keduanya kejadian nyata, keduanya bukan perilaku belanja yang sedang diukur.

Transaksi bertanggal setelah acuan dibuang karena menurut definisi analisis per 2026-03-31, transaksi yang tercatat sesudahnya belum terjadi. Kalau dibiarkan, selisih tanggal acuan dikurangi tanggal transaksi terakhir menghasilkan Recency negatif, dan Recency negatif lolos syarat `≤ 30` dengan mulus.

**Yang ditolak.**

Memakai semua baris apa adanya ditolak, karena Frequency dan Monetary akan tercemar duplikat dan pembayaran gagal.

Menandai baris kotor alih-alih membuangnya ditolak untuk modul ini, karena datanya salinan dan pipeline bisa diulang dari sumber kapan pun. Catatan untuk penerapan berikutnya: di sistem produksi, bila pipeline ini menjadi satu-satunya jalur data hilir, menandai hampir selalu lebih aman daripada membuang.

Memasang pagar `TransactionID IS NOT NULL` dan `CustomerID IS NOT NULL` ditolak, dan alasannya adalah temuan, bukan kelalaian. Profil kolom menunjukkan keempat kolom selain `TotalValue` nol kosong seluruhnya. Memasang pagar itu boleh, tetapi hari ini ia membuang nol baris, dan menulis "data dibersihkan dari ID kosong" padahal tidak ada satu pun ID kosong berarti mengarang pekerjaan.

**Yang tidak diketahui dari data.** Penyebab pencatatan ganda tidak diketahui. Penyebab 50 transaksi bertanggal di luar periode juga tidak diketahui. Pemeriksaan tambahan menunjukkan ke-50 baris itu milik 50 pelanggan berbeda, tersebar merata di kelima kategori (Grocery 13, Electronics 12, Fashion 9, Beauty 9, Home & Living 7), dengan tanggal membentang 2026-04-12 sampai 2026-12-24. Sebaran yang merata itu menyingkirkan dugaan satu batch impor rusak atau satu kategori pre-order, tetapi tidak menunjuk penyebab penggantinya.

---

## 2. Tanggal acuan

**Yang dipakai.** 2026-03-31, ditulis literal sebagai `DATE '2026-03-31'` di setiap kueri dan disebutkan di setiap berkas keluaran.

Dipilih karena ia tanggal terakhir data bersih, sehingga Recency mengukur jarak perilaku pelanggan terhadap ujung data, bukan terhadap jam server, dan hasilnya bisa direproduksi kapan pun kueri dijalankan ulang.

**Yang ditolak.**

`today()` ditolak. Dengan `today()`, definisi "pelanggan aktif" berubah setiap tengah malam: kueri yang sama, data yang sama, dijalankan Senin dan Kamis memberi hasil berbeda tanpa satu baris log pun yang menjelaskan. Simulasi modul menunjukkan harga kesalahan ini: menjalankan kueri 91 hari setelah data berhenti memindahkan 2.800 pelanggan (56,0% populasi) tanpa satu pun mengubah perilakunya, dan mengosongkan segmen Champions karena tidak ada manusia yang bisa memenuhi syarat Recency ≤ 30.

Akhir kuartal fiskal dan tanggal rapat CMO ditolak. Keduanya bisa dipertahankan, tetapi keduanya membuat Recency mengukur jarak terhadap sesuatu yang bukan data.

**Jendela riwayat tetap 15 bulan.** Frequency dan Monetary dihitung hanya dari transaksi dalam 15 bulan sampai tanggal acuan: `TransactionDate >= CAST(DATE '2026-03-31' + 1 - INTERVAL 15 MONTH AS DATE)`. Per 2026-03-31 jendela itu persis seluruh data bersih (2025-01-01 sampai 2026-03-31), sehingga tidak ada angka yang berubah hari ini. Per 2026-06-30 jendelanya bergeser ke 2025-04-01 sampai 2026-06-30.

Alasannya: ambang F dan M absolut, sedangkan riwayat kumulatif memanjang tiga bulan setiap kuartal. Tanpa jendela tetap, segmentasi ulang akan makin mudah meloloskan pelanggan ke Champions dan Loyal hanya karena riwayatnya lebih panjang, dan perbandingan antar-kuartal (termasuk syarat pembatal kedua di memo) mencampur perubahan perilaku dengan perubahan panjang riwayat. Besarnya efek ini terukur di seksi 6: tiga bulan riwayat tambahan menyumbang sekitar 23 Champions.

**Aturan turunan.** Kueri produksi membaca dari pipeline (`FROM cleaned_data`), bukan dari tabel mentah. Keduanya adalah pagar yang mengubah kesalahan pasif menjadi kesalahan aktif: tanpa pagar, merusak segmentasi cukup dengan lupa; dengan pagar, merusaknya butuh tindakan yang kelihatan di kueri dan bisa ditunjuk saat review.

---

## 3. Ambang segmen dan urutan cabang

**Yang dipakai.**

| Segmen | Syarat | Maknanya dalam perilaku |
|---|---|---|
| Champions | R ≤ 30 dan F ≥ 15 dan M > Rp 5.000.000 | Masih hangat, sangat sering, bernilai besar |
| Loyal | R ≤ 90 dan F ≥ 8 | Masih dalam kebiasaan belanja kuartalan |
| At Risk | R > 90 dan F ≥ 5 | Dulu rajin, sudah satu kuartal lebih menghilang |
| Lost | sisanya | Jarang dan lama tak terlihat, atau baru dan belum terbentuk kebiasaannya |

**Pembelaan tiap garis, dari sebaran populasi (5.000 pelanggan, per 2026-03-31).**

| Ukuran | Min | Q1 | Median | Q3 | Maks |
|---|---|---|---|---|---|
| Recency (hari) | 0 | 17 | 61 | 126 | 440 |
| Frequency (transaksi) | 2 | 4 | 7 | 14 | 25 |
| Monetary (Rp) | 44.457 | 650.185 | 1.358.137 | 3.121.396 | 10.878.758 |

R ≤ 30 untuk Champions punya logika bisnis satu siklus belanja bulanan, dan uji kepekaan menunjukkan garis ini praktis tidak menanggung beban apa pun.

R ≤ 90 untuk Loyal punya logika kuartal, sejalan dengan siklus anggaran yang sedang diputuskan.

F ≥ 8 untuk Loyal berdiri tepat di atas median populasi 7, membelah populasi hampir dua sama besar. Ini garis yang paling menanggung beban di seluruh segmentasi, dan karena itu yang paling wajib dibela.

M > Rp 5 juta untuk Champions berdiri jauh di atas Q3 (Rp 3.121.396), memangkas lebih dari separuh jalan ke puncak (Rp 10.878.758). Segmen puncak memang dirancang sempit, dan kesempitan itu disengaja.

**Uji kepekaan: seberapa jauh ambang boleh digeser sebelum rekomendasi berubah.**

| Ambang digeser | Pelanggan pindah | Pangsa nilai Champions | At Risk | Sasaran win-back | Nilai sasaran |
|---|---|---|---|---|---|
| (resmi) | 0 | 28,7% | 900 | 552 | Rp 1,70 M |
| Recency Champions 30 ke 45 hari | 0 | 28,7% | 900 | 552 | Rp 1,70 M |
| Frequency Loyal 8 ke 6 transaksi | 824 | 28,7% | 900 | 552 | Rp 1,70 M |
| Monetary Champions Rp 5 jt ke Rp 4 jt | 52 | 30,7% | 900 | 552 | Rp 1,70 M |
| Frequency At Risk 5 ke 4 transaksi | 389 | 28,7% | 1.289 | 654 | Rp 1,77 M |
| Frequency At Risk 5 ke 6 transaksi | 0 | 28,7% | 900 | 552 | Rp 1,70 M |
| Garis Recency 90 ke 75 hari | 132 | 28,7% | 1.032 | 486 | Rp 1,24 M |
| Garis Recency 90 ke 105 hari | 354 | 28,7% | 546 | 361 | Rp 1,10 M |

Sasaran win-back didefinisikan relatif terhadap garis: At Risk yang berjeda paling lama 30 hari melewati garis Recency (resmi: 91 sampai 120 hari). Geseran garis Recency menggeser garis Loyal dan At Risk bersamaan.

Ambang-ambang ini sama sekali tidak sama bobotnya, dan hasil ini membalik dugaan yang wajar. Menaikkan Recency Champions 50% (30 ke 45 hari) tidak memindahkan seorang pun, karena median Recency Champions 3 hari, jauh di dalam ambang mana pun yang masuk akal. Menggeser Frequency Loyal dua langkah memindahkan 824 pelanggan, hampir seperenam populasi, karena garisnya berdiri tepat di kerumunan. Menaikkan ambang F At Risk ke 6 juga tidak memindahkan siapa pun, karena tidak ada pelanggan At Risk dengan tepat 5 transaksi; menurunkannya ke 4 memindahkan 389. Garis Recency 90 hari menanggung beban yang nyata: menggesernya ke 105 hari memindahkan 354 orang dan mengecilkan At Risk ke 546.

Pada tingkat rekomendasi, ketujuh geseran tidak mengubah arahnya, dan kali ini itu dihitung, bukan dinalar: Champions tetap memegang 28,7% sampai 30,7% nilai dengan paling banyak 9% pelanggan, dan kelompok yang baru menjauh tetap ada, 361 sampai 654 orang dengan nilai historis Rp 1,10 miliar sampai Rp 1,77 miliar. Keduanya datang dari sebaran, bukan dari garis. Yang berubah hanya ukuran kelompoknya. Karena itu rekomendasi ditulis pada tingkat yang kokoh ("pusatkan pada puncak, uji win-back pada yang menjauh") dengan angka sebagai ilustrasi, bukan sebaliknya.

**Urutan cabang.** `CASE WHEN` berhenti di syarat pertama yang terpenuhi, dan urutan ketat ke longgar (Champions, Loyal, At Risk, Lost) disengaja. Setiap Champions juga memenuhi syarat Loyal, karena R ≤ 30 berarti R ≤ 90 dan F ≥ 15 berarti F ≥ 8. Mereka tertangkap cabang pertama sebelum cabang kedua sempat bertanya. Membalik urutannya akan mengosongkan Champions dan menampung 1.500 orang di Loyal, tanpa satu pesan galat pun. Urutan ini bagian dari definisi segmen, bukan kerapian penulisan.

**Yang ditolak.**

Skoring kuintil 1 sampai 5 per ukuran ditolak untuk iterasi pertama. Metode itu lebih adaptif, tetapi lebih sulit dijelaskan kepada pemegang anggaran pada pembahasan pertama, dan uji kepekaan menunjukkan kedua metode menemukan bentuk yang sama pada data ini. Metode yang lebih mudah dijelaskan menang selama hasilnya tidak berbeda arah. Kandidat untuk iterasi kedua, dan bila suatu saat kedua metode berbeda arah, itu sinyal untuk berhenti dan memahami penyebabnya.

Melonggarkan F Loyal ke 6 ditolak. Ia memindahkan 824 orang dan menggelembungkan Loyal ke 38,5% populasi, mengaburkan justru pembedaan yang dibutuhkan keputusan alokasi.

---

## 4. Pilihan fungsi

**Jumlah unik dihitung eksak.** `count(DISTINCT ...)` dipakai, bukan `approx_count_distinct()`. Versi aproksimasi lebih cepat dan untuk dashboard yang menyegarkan diri tiap menit itu jual beli yang bagus. Untuk angka yang akan masuk memo dan dipakai membagi Rp 150 juta, perkiraan tidak bisa diterima.

**Kuantil dihitung eksak, dengan satu definisi yang dikunci.** DuckDB menyediakan dua fungsi kuantil bawaan, dan keduanya memberi jawaban berbeda pada jumlah data genap: `quantile_cont()` menginterpolasi di antara dua nilai tengah, sedangkan `quantile_disc()` mengambil nilai tengah bawah. Modul menetapkan kuantil eksak dengan definisi elemen ke-floor(n·p)+1 dari data terurut, yang pada median dengan n genap mengambil nilai tengah **atas**. Definisi itu ditulis sebagai satu makro dan dipakai di setiap kueri:

```sql
CREATE OR REPLACE MACRO q_exact(x, p) AS
    list_sort(list(x))[CAST(floor(count(x) * p) AS BIGINT) + 1];
```

Mencampur ketiganya di berkas yang berbeda akan menghasilkan median yang berbeda tipis untuk segmen yang sama, dan selisih tipis itulah yang paling sulit dijelaskan di rapat. Akibat nyatanya tercatat di seksi 9, bagian Metode kuantil.

Layak ditinjau ulang bila kelak segmentasi ini dipasang di dashboard operasional, di mana kecepatan lebih bernilai daripada digit terakhir.

---

## 5. Hasil

**Segmentasi per 2026-03-31, 5.000 pelanggan, 48.130 transaksi bersih.**

| Segmen | Pelanggan | Pangsa | Median R | Median F | Median M | Total M | Pangsa nilai |
|---|---|---|---|---|---|---|---|
| Champions | 400 | 8,0% | 3 hari | 22 | Rp 8.057.608 | Rp 3,18 M | 28,7% |
| Loyal | 1.100 | 22,0% | 19 hari | 16 | Rp 3.027.313 | Rp 3,35 M | 30,2% |
| At Risk | 900 | 18,0% | 112 hari | 11 | Rp 3.006.102 | Rp 2,74 M | 24,7% |
| Lost | 2.600 | 52,0% | 91 hari | 5 | Rp 669.674 | Rp 1,82 M | 16,4% |

Total nilai transaksi bersih Rp 11.100.722.567.

**Aritmetika silang.** Jumlah pelanggan 400 + 1.100 + 900 + 2.600 = 5.000, cocok dengan jumlah pelanggan unik. Pangsa pelanggan 8,0 + 22,0 + 18,0 + 52,0 = 100%. Pangsa nilai 28,7 + 30,2 + 24,7 + 16,4 = 100%. Tidak ada segmen NULL atau kosong (0 baris).

**Pemeriksaan nalar terhadap hasil, dijalankan sebelum menerimanya.**

Median Frequency Loyal (16) lebih tinggi daripada At Risk (11). Ini masuk akal: keduanya dulu sama rajin, dan At Risk berhenti lebih awal sehingga hitungannya berhenti tumbuh. Bukan tanda kebocoran cabang.

Median Frequency Lost hanya 5 dan median Monetary-nya Rp 669.674. Artinya sebagian besar Lost bukan pelanggan setia yang hilang, melainkan pembeli ringan yang tidak pernah terbentuk kebiasaannya. Nuansa ini mengubah nada rekomendasi tentang mereka.

**Komposisi Lost, diperiksa karena labelnya menyesatkan.** Segmen ini terbelah dua sama besar:

| Jeda sejak transaksi terakhir | Pelanggan | Frequency | Nilai |
|---|---|---|---|
| 0 sampai 30 hari | 606 | 5 sampai 7 | Rp 0,57 M |
| 31 sampai 90 hari | 694 | 5 sampai 7 | Rp 0,65 M |
| lebih dari 90 hari | 1.300 | 2 sampai 4 | Rp 0,61 M |

1.300 anggota Lost masih bertransaksi dalam 90 hari terakhir; mereka masuk Lost karena F di bawah 8, bukan karena pergi. Kesimpulan "pembeli ringan" tetap berdiri, tetapi gambaran "sudah lama pergi" tidak berlaku untuk separuhnya. Label dipertahankan demi konsistensi keempat berkas, setiap penyebutan Lost menyertakan komposisi ini, dan pemecahan menjadi dua segmen bernama jujur menjadi kandidat iterasi berikutnya. Median Recency Lost 91 hari berarti bulatan Lost di gambar median duduk tepat di garis 90 hari, bukan di kanannya.

**Champions dan 400 pelanggan terbesar belanjanya praktis kelompok yang sama.** Dari 400 pelanggan dengan Monetary tertinggi, 399 adalah Champions dan satu At Risk. Karena itu titik 8% pada kurva konsentrasi dan pangsa nilai Champions sama-sama 28,7%, dan sebutan "400 pelanggan teratas" di portofolio sah.

**Kurva konsentrasi nilai.** Pelanggan diurutkan dari yang terbesar belanjanya, lalu pangsa nilai kumulatifnya dihitung.

| Kelompok teratas | Pangsa nilai |
|---|---|
| 8,0% (400 orang) | 28,7% |
| 10% (500 orang) | 32,7% |
| 20% (1.000 orang) | 49,2% |
| 30% (1.500 orang) | 63,3% |
| 50% (2.500 orang) | 85,0% |

Angka 49,2% layak dicatat khusus karena ia mendahului bantahan yang wajar: aturan 80/20 yang biasa dipinjam tidak berlaku di sini, dan menyatakannya lebih dulu lebih kuat daripada membiarkan orang lain menemukannya. Konsentrasi 28,7% pada 8,0% pelanggan setara 3,59 kali lipat porsi seimbangnya.

**Aritmetika anggaran, bukan temuan.** Seluruh angka per kepala di keempat berkas berasal dari tabel ini, supaya tidak ada berkas yang menghitung ulang sendiri.

*Kalau seluruh Rp 150 juta diarahkan ke satu kelompok:*

| Sasaran | Orang | Per kepala |
|---|---|---|
| Semua pelanggan (bagi rata) | 5.000 | Rp 30.000 |
| Champions | 400 | Rp 375.000 |
| At Risk, seluruhnya | 900 | Rp 166.667 |
| At Risk yang baru berhenti (jeda 91 sampai 120 hari) | 552 | Rp 271.739 |
| Separuh At Risk yang baru berhenti (kelompok yang dikirimi) | 276 | Rp 543.478 |
| Champions ditambah seluruh At Risk | 1.300 | Rp 115.385 |

*Kalau anggaran dibagi menurut usulan di memo:*

| Program | Porsi | Orang | Per kepala |
|---|---|---|---|
| Apresiasi dan akses prioritas | Rp 90 juta | 400 | Rp 225.000 |
| Uji win-back terukur | Rp 50 juta | 276 dikirimi, 276 pembanding | Rp 181.159 per orang yang dikirimi |
| Cadangan pengukuran dan kanal | Rp 10 juta | lintas program | tidak berlaku |

Seluruh angka di kedua tabel ini hasil pembagian atas asumsi anggaran, **bukan** angka yang lahir dari data. Bedanya penting karena keduanya runtuh dengan cara berbeda: angka pangsa runtuh kalau kuerinya salah, angka per kepala runtuh kalau anggarannya berubah. Pembagian 90/50/10 sendiri adalah usulan alokasi, bukan temuan analisis, dan tempatnya di memo.

---

## 6. Variasi tanggal acuan 2025-12-31

Dua tempat diganti, bukan satu: batas atas tanggal di `cleaned_data` dan tanggal acuan di perhitungan Recency. Bahan berubah menjadi 39.482 transaksi bersih, tetap dari 5.000 pelanggan unik.

**Dugaan yang ditulis sebelum kueri dijalankan.** Champions mengecil, karena syaratnya menuntut F ≥ 15 dan M > Rp 5 juta sekaligus dan memotong tiga bulan memotong akumulasi keduanya. Lost membesar. Arah At Risk tidak jelas. Loyal tidak diduga secara eksplisit.

**Hasil.**

| Segmen | Per 2026-03-31 | Per 2025-12-31 | Selisih |
|---|---|---|---|
| Champions | 400 (8,0%) | 83 (1,7%) | -317 |
| Loyal | 1.100 (22,0%) | 2.222 (44,4%) | +1.122 |
| At Risk | 900 (18,0%) | 226 (4,5%) | -674 |
| Lost | 2.600 (52,0%) | 2.469 (49,4%) | -131 |

Pangsa nilai per 2025-12-31: Loyal 74,5%, Lost 15,8%, Champions 5,5%, At Risk 4,2%.

**Selisih dugaan dan hasil, dicatat apa adanya.** Dugaan Champions benar, dan jauh lebih ekstrem daripada yang diduga. Dugaan Lost salah; Lost justru sedikit mengecil. Dugaan "arah At Risk tidak jelas" ternyata terlalu berhati-hati: arahnya sangat jelas dan sangat besar, hanya saja ke arah yang tidak dipertimbangkan. Loyal, yang tidak diduga sama sekali, adalah segmen yang paling berubah, dan perubahannya membalik seluruh cerita.

**Mekanismenya ada tiga, dan ketiganya bercampur di tabel di atas.** (1) Panjang riwayat: per 2025-12-31 hanya ada 12 bulan data, per 2026-03-31 ada 15 bulan, dan F serta M hanya bisa naik seiring riwayat memanjang. (2) Penuaan Recency: Recency diukur relatif terhadap tanggal acuan, sehingga orang yang berhenti belanja di Desember belum sempat menua melewati 90 hari per 2025-12-31 (median Recency 35 hari per 2025-12-31 versus 61 hari per 2026-03-31; Q3 79 versus 126 hari). (3) Perilaku Q1 2026 yang memang berubah: sebagian pelanggan berhenti bertransaksi, sebagian yang bertahan belanja lebih sering.

Efek pertama artefak pengukuran, dua lainnya bukan. Ia dipisahkan dengan menyamakan panjang jendela, 12 bulan di kedua tanggal:

| Segmen | Per 2026-03-31, 15 bulan (resmi) | Per 2026-03-31, 12 bulan | Per 2025-12-31, 12 bulan |
|---|---|---|---|
| Champions | 400 | 377 | 83 |
| Loyal | 1.100 | 1.117 | 2.222 |
| At Risk | 900 | 892 | 226 |
| Lost | 2.600 | 2.586 | 2.469 |

Dari kenaikan Champions 83 ke 400, hanya sekitar 23 orang yang berasal dari riwayat lebih panjang; sisanya perilaku Q1 2026. Dugaan awal bahwa Champions "runtuh" per 2025-12-31 karena akumulasinya terpotong hanya benar sebagian kecil, dan keterangan gambar di versi notebook sebelumnya, yang menyebut tidak ada pelanggan yang mengubah perilakunya di antara kedua tanggal, keliru: Q1 2026 adalah kuartal ketika perilaku berubah.

---

## 7. Temuan dari matriks perpindahan

Menggabungkan segmentasi kedua tanggal per pelanggan menghasilkan temuan yang tidak terlihat dari satu tanggal mana pun.

| Dari (2025-12-31) | Ke (2026-03-31) | Pelanggan |
|---|---|---|
| Lost | Lost | 2.468 |
| Loyal | Loyal | 1.060 |
| Loyal | At Risk | 857 |
| Loyal | Champions | 305 |
| At Risk | Lost | 132 |
| Champions | Champions | 83 |
| At Risk | At Risk | 43 |
| At Risk | Loyal | 40 |
| At Risk | Champions | 11 |
| Lost | Champions | 1 |

Aritmetika silang matriks: At Risk 857 + 43 = 900, Champions 305 + 83 + 11 + 1 = 400, Loyal 1.060 + 40 = 1.100, Lost 2.468 + 132 = 2.600. Total 5.000. Sebanyak 1.346 pelanggan (26,9%) berada di segmen berbeda pada kedua tanggal.

**Temuannya.** Dari 900 pelanggan At Risk per 2026-03-31, sebanyak 857 (95,2%) masih tergolong Loyal tiga bulan sebelumnya. Segmen At Risk bukan kolam lama yang mengendap, melainkan kohort yang baru terbentuk dalam satu kuartal.

Temuan ini tidak bergantung pada panjang jendela. Dengan jendela 12 bulan di kedua tanggal, perpindahan Loyal ke At Risk 850 orang. Untuk 857 orang versi resmi, Frequency dan Monetary identik di kedua tanggal, karena mereka tidak bertransaksi sama sekali sesudah 2025-12-30; perpindahan mereka murni perilaku.

Diperiksa dari arah lain dan hasilnya sejalan: dari 900 At Risk, 552 orang terakhir belanja 91 sampai 120 hari lalu, yaitu antara 2025-12-01 dan 2025-12-30, sehingga mereka berhenti sejak Desember 2025 dan tidak muncul sepanjang Q1 2026. Kelompok 552 orang itu membawa Rp 1,70 miliar dari total Rp 2,74 miliar.

**Konteks kuartalan (data bersih).**

| Kuartal | Transaksi | Pelanggan aktif | Nilai |
|---|---|---|---|
| 2025 Q1 | 9.803 | 3.953 | Rp 2,19 M |
| 2025 Q2 | 9.704 | 3.935 | Rp 2,21 M |
| 2025 Q3 | 10.057 | 4.013 | Rp 2,28 M |
| 2025 Q4 | 9.918 | 3.954 | Rp 2,24 M |
| 2026 Q1 | 8.648 | 2.789 | Rp 2,18 M |

Empat kuartal stabil di sekitar 3.950 pelanggan aktif, lalu Q1 2026 turun ke 2.789, turun 29,5% dibanding Q4 2025. Nilainya nyaris tidak bergerak dan transaksinya hanya turun 12,8% dibanding Q4 2025, artinya pelanggan yang tersisa berbelanja lebih banyak masing-masing. Q1 2026 adalah kuartal lengkap di data bersih, jadi ini bukan artefak pemotongan tanggal acuan.

**Batas temuan ini, dinyatakan tegas.** Data menunjukkan bahwa sebuah kohort besar berhenti bertransaksi di Q1 2026. Ia tidak menunjukkan kenapa. Dan hanya ada satu pasang tanggal untuk dibandingkan, sehingga tidak ada pembanding historis yang memberi tahu apakah laju 38,6% (857 dari 2.222 Loyal) tinggi atau normal bagi bisnis ini. Klaim yang bisa dipertahankan berhenti di: terjadi, sebesar ini, pada periode ini.

---

## 8. Desain uji win-back

**Yang dipakai.** 552 pelanggan At Risk berjeda 91 sampai 120 hari dibagi dua sama besar: 276 dikirimi program, 276 sengaja tidak dikirimi apa pun sebagai pembanding. Pembagian ditentukan oleh urutan `md5(CustomerID || '|winback-2026Q2')`; separuh pertama dikirimi. Pembagian ini deterministik, tidak bergantung pada pilihan siapa pun, dan bisa diulang persis. Daftar lengkapnya ditampilkan di notebook, bagian Desain uji win-back.

**Keseimbangan sebelum apa pun dikirim.**

| Kelompok | Pelanggan | Median R | Rerata F | Rerata M | Nilai historis |
|---|---|---|---|---|---|
| Dikirimi | 276 | 103 hari | 11,48 | Rp 3.095.643 | Rp 0,854 M |
| Pembanding | 276 | 101 hari | 11,45 | Rp 3.063.762 | Rp 0,846 M |

**Kenapa pembanding diambil dari 552 yang sama.** Kelompok At Risk berjeda 121 sampai 180 hari bukan pembanding yang sah: jeda yang berbeda berarti peluang kembali yang berbeda, dan selisih apa pun akan tercampur dengan selisih jeda.

**Kenapa pembanding wajib, dalam angka.** Pada tiga tanggal acuan sebelumnya, kelompok berprofil sama (jeda 91 sampai 120 hari, F ≥ 5) yang bertransaksi lagi pada kuartal berikutnya:

| Tanggal acuan | Pelanggan | Kembali kuartal berikutnya |
|---|---|---|
| 2025-06-30 | 29 | 29 |
| 2025-09-30 | 74 | 59 |
| 2025-12-31 | 128 | 96 |

Gabungannya 184 dari 231, atau 79,7%. Patokan ini tidak bersih, karena pada kuartal-kuartal itu promosi masih dikirim rata ke semua orang, dan kohort yang berhenti di Q1 2026 bisa berperilaku lain. Tetapi ia cukup untuk menunjukkan bahwa angka "sekian orang kembali" tanpa pembanding hampir pasti terlihat seperti keberhasilan, berapa pun efek programnya.

**Batas deteksi.** Dengan 276 orang per kelompok, alfa 0,05 dua sisi dan daya 80%, selisih tingkat kembali terkecil yang terbaca adalah 9,6 poin persen pada patokan historis 79,7%, dan 11,9 poin persen pada kasus terburuk 50%. Dibulatkan: sekitar 10 sampai 12 poin persen. Selisih yang lebih kecil dari itu dibaca sebagai "efeknya tidak sebesar itu, atau tidak ada", bukan sebagai bukti program pasti gagal.

**Anggaran.** Porsi uji tetap Rp 50 juta, sehingga orang yang dikirimi menerima Rp 181.159 per kepala. Pilihan lain yang sah adalah mempertahankan Rp 90.580 per kepala dan mengembalikan Rp 25 juta ke cadangan; yang tidak sah adalah mengirim ke seluruh 552 orang sambil mengklaim ada pembanding.

**Yang ditolak.** Mengirim ke seluruh 552 orang dan membandingkan dengan kelompok berjeda lebih lama, karena pembandingnya tidak sebanding. Memilih kelompok yang dikirimi secara manual, karena pilihan manual hampir selalu condong ke pelanggan yang "terlihat lebih menjanjikan" dan membuat hasilnya tidak bisa dibaca.

---

## 9. Diskrepansi terhadap materi modul

Dua hal ditemukan saat menjalankan sendiri, dan dicatat apa adanya, bukan dipaksa cocok.

**Metode kuantil.** Median Recency dan Frequency segmen Lost keluar 91 dan 5 dengan kuantil eksak `q_exact`, sementara halaman 07 modul menuliskan 90 dan 4. Segmen Lost berjumlah genap (2.600) dan dua nilai tengahnya memang 90/91 dan 4/5. Pemeriksaan pada median Monetary populasi menegaskan penyebabnya: dua nilai tengahnya Rp 1.357.781,85 dan Rp 1.358.137,16; angka modul (Rp 1.357.960) adalah titik tengah interpolasi keduanya, sedangkan kuantil eksak yang modul tetapkan mengembalikan Rp 1.358.137,16, nilai tengah atasnya. Selisih berasal dari metode kuantil, bukan dari data, dan tidak memengaruhi satu pun jumlah segmen. Angka yang dipakai di seluruh berkas ini adalah keluaran `q_exact`, sesuai definisi yang ditetapkan modul. Seandainya `quantile_cont()` bawaan DuckDB yang dipakai, angka modul akan keluar; itulah sebabnya fungsi kuantil dikunci di seksi 4.

Konsekuensi kecil pada narasi: halaman 07 seksi 6 menyebut median Recency Lost "tepat 90 hari, kebetulan sama dengan ambang R Loyal". Pada keluaran `q_exact`, angkanya 91, sehingga kebetulan itu tidak ada.

**Jumlah kemunculan duplikat.** Halaman 05 seksi 3 menulis bahwa setiap `TransactionID` duplikat muncul tepat dua kali. Di data aktual, 2.346 `TransactionID` muncul lebih dari sekali: 2.292 muncul dua kali dan 54 muncul tiga kali. Aritmetikanya tetap konsisten dengan gerbang, karena 2.292 x 1 ditambah 54 x 2 sama dengan 2.400 baris berlebih. Yang meleset hanya narasinya.

---

## 10. Uji bantahan

**"8,0% memegang 28,7% itu artefak ambang."** Setengah benar, dan diakui. Angka 400 dan 28,7% memang bergantung pada ambang. Yang tidak bergantung pada ambang adalah bentuk sebarannya: Monetary membentang Rp 44.457 sampai Rp 10.878.758 dengan median Rp 1.358.137. Konsentrasi nilai di ujung atas adalah sifat data, bukan sifat garis. Data yang akan membatalkan temuan ini adalah sebaran Monetary yang datar dengan kuartil berdekatan, dan data itu tidak ada di sini.

**"Rp 2,74 miliar itu nilai masa lalu, bukan masa depan."** Benar seluruhnya. Redaksi temuan disesuaikan, bukan dibantah balik. Yang ditulis bukan "win-back akan menyelamatkan Rp 2,74 miliar" (klaim proyeksi yang datanya tidak ada), melainkan: 900 pelanggan yang median transaksinya 11 kali terbukti pernah setia, kini median 112 hari tanpa transaksi, dan nilai historis kelompok ini Rp 2,74 miliar. Berapa persen yang bisa dipulihkan tidak diketahui dari data ini, dan justru itu alasan mengujinya dengan campaign kecil terukur sebelum anggaran besar dialokasikan.

**"Duplikat yang kamu buang itu transaksi ganda yang sah."** Bantahan yang pantas dihormati, karena penanyanya membela pelanggan paling setia. Jawabannya satu fakta yang bisa ditunjuk: baris-baris itu identik termasuk `TransactionID`. Dua pembelian sungguhan, bahkan barang sama hari sama nilai sama, tercatat dengan dua nomor transaksi berbeda. Data yang akan membatalkan jawaban ini adalah pasangan duplikat yang sebagian kolomnya berbeda (nomor sama, nilai beda), yang akan mengubah persoalan dari duplikasi menjadi konflik data. Pemeriksaan itu murah, sudah dijalankan, dan hasilnya nol baris.

**Bantahan yang tidak punya jawaban.** Seluruh analisis ini membaca masa lalu, dan tabel transaksi tidak memuat kuartal depan. Kalau pasar bergeser, segmentasi hari ini memotret orang-orang yang sudah berubah. Bantahan ini dinyatakan terbuka sebagai asumsi berikut mitigasinya, bukan disembunyikan dan bukan dijawab dengan keyakinan kosong. Mitigasinya: segmentasi dijalankan ulang tiap kuartal dengan pipeline yang sama, ongkosnya kini hampir nol karena seluruh keputusannya tercatat di dokumen ini, dan pergeseran antar-kuartal justru menjadi sinyal paling awal bahwa pasarnya bergerak. Seksi 7 menunjukkan mitigasi ini bukan formalitas.

---

## 11. Batas klaim

**Yang boleh diklaim.** Data kini konsisten dengan definisi yang dinyatakan: satu baris satu transaksi, setiap transaksi bernilai positif, tidak ada transaksi setelah tanggal acuan.

**Yang tidak boleh diklaim.** Bahwa datanya kini "benar". Pembersihan tidak memverifikasi bahwa transaksi yang tersisa sungguh terjadi, tidak menemukan transaksi yang gagal tercatat, dan tidak tahu apa pun tentang pelanggan yang berbelanja tunai di tempat lain. Ia menegakkan definisi; ia tidak memperbaiki dunia.

**Yang tidak ada di data ini sama sekali.** Hasil campaign: tidak ada kolom yang menyatakan pelanggan membeli karena voucher, sehingga klaim ROI, uplift, atau konversi tidak bisa dibangun dari data ini. Biaya per kanal: pembagian anggaran yang diusulkan berhenti di tingkat segmen. Alasan pelanggan pergi: pindah ke pesaing, kecewa pengiriman, atau sekadar tidak butuh lagi, ketiganya identik di tabel ini.

**Angka konteks, diterima tanpa verifikasi.** Anggaran Rp 150 juta per kuartal, open rate email 12%, dan redemption voucher 3% berasal dari laporan campaign pemangku kepentingan, bukan dari tabel `transactions`. Ketiganya boleh dikutip sebagai konteks dan tidak menopang satu pun temuan di atas.

**Dampak keputusan pembersihan, diperiksa bukan diasumsikan.** Membuang 350 baris NULL menyentuh 333 pelanggan berbeda. Diperiksa dengan menjalankan ulang segmentasi sambil membiarkan baris NULL tercacah di Frequency: hasilnya nol pelanggan berpindah segmen. Keputusan ini karena itu berdampak rendah pada segmentasi, tetapi tetap diambil eksplisit karena diamnya SQL berarti memakai dua definisi sekaligus tanpa jejak. Membuang 2.850 baris tidak menghilangkan satu pelanggan pun (5.000 unik di mentah, 5.000 di bersih); ini kebetulan yang menyenangkan di dataset ini, bukan jaminan alam, dan diperiksa ulang setiap kali pipeline dijalankan.

---

## 12. Kueri final

Kueri berikut dijalankan apa adanya di DuckDB, atas tabel `transactions` yang dimuat dari `transactions.csv`, dan menghasilkan angka segmentasi yang dikutip ketiga berkas lain. Angka pemeriksaan tambahan (komposisi Lost, uji kepekaan, jendela sama panjang, desain uji win-back) dihitung di notebook dengan SQL yang sama gayanya.

```sql
-- Kuantil eksak: elemen ke-floor(n*p)+1 dari data terurut.
-- Untuk n genap dan p = 0,5 ia mengambil nilai tengah ATAS.
CREATE OR REPLACE MACRO q_exact(x, p) AS
    list_sort(list(x))[CAST(floor(count(x) * p) AS BIGINT) + 1];

WITH deduped AS (
    SELECT DISTINCT
        TransactionID, CustomerID, TransactionDate, TotalValue, ProductCategory
    FROM transactions
),
null_filtered AS (
    SELECT * FROM deduped
    WHERE TotalValue IS NOT NULL
),
positive_filtered AS (
    SELECT * FROM null_filtered
    WHERE TotalValue > 0
),
cleaned_data AS (
    SELECT * FROM positive_filtered
    WHERE TransactionDate <= DATE '2026-03-31'
),
rfm_scores AS (
    SELECT
        CustomerID,
        DATE '2026-03-31' - max(TransactionDate) AS Recency,
        count(*)                                 AS Frequency,
        sum(TotalValue)                          AS Monetary
    FROM cleaned_data
    WHERE TransactionDate >= CAST(DATE '2026-03-31' + 1 - INTERVAL 15 MONTH AS DATE)  -- jendela tetap 15 bulan
    GROUP BY CustomerID
),
rfm_segmented AS (
    SELECT
        *,
        CASE
            WHEN Recency <= 30 AND Frequency >= 15 AND Monetary > 5000000 THEN 'Champions'
            WHEN Recency <= 90 AND Frequency >= 8                         THEN 'Loyal'
            WHEN Recency >  90 AND Frequency >= 5                         THEN 'At Risk'
            ELSE 'Lost'
        END AS Segment
    FROM rfm_scores
)
SELECT
    Segment,
    count(*)                                                     AS Pelanggan,
    round(count(*) * 100.0 / sum(count(*)) OVER (), 1)           AS Pangsa,
    q_exact(Recency, 0.5)                                        AS MedianR,
    q_exact(Frequency, 0.5)                                      AS MedianF,
    round(q_exact(Monetary, 0.5))                                AS MedianM,
    round(sum(Monetary))                                         AS TotalM,
    round(sum(Monetary) * 100.0 / sum(sum(Monetary)) OVER (), 1) AS PangsaNilai
FROM rfm_segmented
GROUP BY Segment
ORDER BY Pelanggan DESC;
```

Untuk variasi 2025-12-31, dua tanggal diganti: batas atas di `cleaned_data` dan tanggal acuan di perhitungan Recency, dan batas bawah jendela ikut bergeser karena diturunkan dari tanggal acuan. Mengganti salah satunya saja menghasilkan angka lain, dan itu sendiri diagnosis yang berguna. Untuk segmentasi ulang per 2026-06-30, ketiga tanggal itu diganti ke 2026-06-30, dan jendelanya otomatis menjadi 2025-04-01 sampai 2026-06-30.
