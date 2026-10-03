# Keputusan Produk

Produk: indonesian-retrieval-benchmark
Jenis: riset ML (benchmark algoritma pencarian vektor)
Fase: MVP (satu-satunya fase project)
Data: tingkat 3 — data riset publik lengkap: korpus `miracl/miracl-corpus` id dan query/qrels `miracl/miracl` id dev, dimuat di instance Vast.ai. Bukan data pengganti. Mode data rahasia: tidak aktif.
Status: siap dikerjakan
Diperbarui: 2026-10-03

## Ringkasan
Benchmark yang membandingkan exact search (baseline) dengan tiga algoritma ANN (HNSW, IVF, LSH) untuk retrieval teks berbahasa Indonesia, dikerjakan Arya sebagai portofolio pribadi kemampuan mengimplementasikan retrieval. Semua 1.446.315 passage MIRACL-id (title + text) dan 960 query dev di-embed sekali dalam fp32 dengan `LazarusNLP/congen-indobert-base` di GPU Vast.ai, lalu keempat algoritma FAISS mencari top-5 di CPU atas vektor yang sama dengan sapuan parameter di split val. Per algoritma dipilih konfigurasi tercepat yang mencapai k-NN Recall@5 ≥ 0,95, dikunci, lalu split test dibuka sekali di benchmark final — dijaga supaya hanya bisa dibuka lagi lewat izin tertulis Arya — dan dicatat dengan 12 metrik beserta resource mesin.

## Titik periksa

### Rancangan 001
1. [Korpus] Korpus mana yang dipakai di MVP?
   a. MIRACL-id hard negatives dari MTEB, sekitar 168 ribu dokumen; korpus penuh dipertimbangkan di Dev (Usulan) — embedding selesai dalam hitungan menit sampai jam di CPU laptop dan muat di RAM, tetapi korpusnya dikumpulkan di sekitar query dan lebih kecil, sehingga selisih kecepatan ANN terhadap exact belum terlihat sebesar di skala jutaan.
   b. Korpus penuh MIRACL-id, sekitar 1,45 juta passage, sejak MVP — skala realistis tempat keunggulan ANN tampak jelas, tetapi embedding memakan berjam-jam di CPU dan vektornya sekitar 4,4 GB.
   Jawaban Arya: "Ini pekerjaan berbeda." ✓ 2026-10-02 — diputuskan di 002: korpus penuh `miracl/miracl-corpus` id.
2. [Panjang tks] Berapa panjang maksimum token saat embedding?
   a. 32 token, bawaan model, untuk dokumen dan query (Usulan) ✓ 2026-10-02 — sesuai cara model dilatih dan paling cepat, tetapi passage terpotong sehingga angka nDCG mutlak rendah; perbandingan antaralgoritma tetap adil.
   b. 128 token untuk dokumen, 32 untuk query — isi passage lebih banyak terbaca, tetapi model tidak dilatih di panjang ini (efeknya belum diketahui) dan embedding sekitar 4 kali lebih lambat.
   c. 512 token untuk dokumen — seluruh passage terbaca, tetapi paling jauh dari kondisi latih dan paling berat.
3. [LSH] Implementasi LSH mana yang dipakai?
   a. FAISS IndexLSH: proyeksi acak ke kode biner, lalu scan jarak Hamming (Usulan) — satu library dengan tiga algoritma lain dan paling sederhana, tetapi scan-nya tetap menyeluruh (bukan tabel hash bucket), jadi percepatannya datang dari kode biner yang ringkas.
   b. LSH bucket multi-tabel (kode biner proyeksi acak + FAISS IndexBinaryMultiHash) — lebih dekat dengan definisi LSH klasik yang sublinear, tetapi parameternya lebih banyak dan perakitannya lebih rumit untuk MVP.
   c. Implementasi sendiri dengan numpy — kendali penuh atas setiap langkah, tetapi kode buatan sendiri berisiko salah dan tidak teroptimasi sehingga perbandingan waktunya tidak setara.
   Jawaban Arya: "Ini pekerjaan berbeda" ✓ 2026-10-02 — diputuskan di 002: FAISS IndexLSH.
4. [Split query] Bagaimana query evaluasi dipakai?
   a. Dibagi val/test 50:50 dengan seed tetap; MVP hanya memakai val (Usulan) ✓ 2026-10-02 — test tetap bersih untuk benchmark final, tetapi angka val hanya dari separuh query.
   b. Semua query dipakai di MVP tanpa pembagian — angka MVP lebih stabil, tetapi pemilihan parameter dan laporan akhir memakai query yang sama sehingga hasil akhir bias.
5. [Ditunda ke] Pemilihan korpus dan implementasi LSH dirancang di mana?
   a. Ikut rancangan 002 bersama tech stack dan kontrak metrik (Usulan) ✓ 2026-10-02 — satu putaran memutuskan semua yang saling bergantung, tetapi putaran 002 menjadi lebih besar.
   b. Nomor rancangan sendiri setelah 002 — tiap pekerjaan kecil dan terpisah, tetapi keputusan library di 002 bisa perlu diubah lagi.

### Rancangan 002
1. [Fase] Bagaimana koreksi "Cukup 1 fase saja" diterapkan?
   a. Satu fase berlabel MVP; benchmark final di split test ikut di fase ini sebagai notebook terakhir, dibuka sekali setelah konfigurasi dikunci dari val (Usulan) ✓ 2026-10-02 — K2 001a tetap utuh dan test tetap bersih, tetapi fase ini memuat notebook final dengan pemeriksaan hash dan lingkungan yang di template riset baru muncul di Dev.
   b. Satu fase berlabel MVP; parameter ditetapkan di awal tanpa tuning, split val/test dihapus, dan 960 query dipakai sekali — angka lebih stabil dan alurnya lebih pendek, tetapi K2 001a yang sudah disetujui berubah dan tidak ada ruang memilih parameter dari hasil.
2. [Teks dok] Teks dokumen apa yang di-embed?
   a. title + " " + text (Usulan) — judul artikel memberi konteks topik pada passage yang tidak menyebut subjeknya, tetapi judul memakai sebagian dari 32 token sehingga isi passage yang terbaca berkurang.
   b. text saja — seluruh 32 token untuk isi passage, tetapi passage tanpa nama subjek kehilangan konteks.
   Jawaban Arya: "Simpan keputusan ini untuk diputuskan dipekerjaan selanjutnya" ✓ 2026-10-02 — diputuskan di 003 (H1): title + " " + text.
3. [RDE LSH] Jarak L2 lima hasil LSH dibandingkan per peringkat dalam urutan apa?
   a. Diurutkan ulang menaik berdasarkan jarak L2 sebelum dibandingkan (Usulan) ✓ 2026-10-02 — galat tiap peringkat selalu ≥ 0 dan setara dengan HNSW dan IVF yang hasilnya sudah terurut L2, tetapi #8 tidak lagi mencerminkan urutan Hamming LSH (urutan itu tetap dinilai metrik kualitas).
   b. Urutan Hamming asli dari LSH — mencerminkan urutan yang dikembalikan LSH, tetapi galat per peringkat bisa negatif sehingga rata-ratanya bisa saling menutupi dan tidak setara dengan tiga algoritma lain.
4. [MAP@5] Pembagi Z pada AP@5 per query?
   a. min(|Rel(q)|, 5) (Usulan) ✓ 2026-10-02 — setiap query bisa mencapai 1,0 sehingga MAP@5 sebanding antarquery, tetapi relevan di luar top-5 tidak menurunkan nilai (sudah diukur Recall@5).
   b. |Rel(q)| — sama dengan AP penuh yang dipotong di 5, tetapi 15,2% query dengan lebih dari 5 relevan tidak bisa mencapai 1,0.
   c. Jumlah relevan yang ditemukan di top-5 — hanya menilai urutan hasil yang benar, tetapi query yang hanya menemukan satu relevan di peringkat 1 sudah mendapat 1,0.

### Rancangan 003
1. [nbits LSH] Arya menetapkan "sapuan hanya parameter search; parameter build tetap", tetapi nbits LSH adalah parameter build. Bagaimana sapuan nbits diperlakukan?
   a. Diterima sebagai pengecualian tertulis: LSH dibangun tiga kali, satu per nilai nbits (Usulan) ✓ 2026-10-02 — LSH tetap punya beberapa titik trade-off seperti HNSW dan IVF karena IndexLSH tidak punya parameter search, tetapi waktu build dan ukuran index (#11, #12) LSH berbeda antar-konfigurasi.
   b. nbits dikunci satu nilai (1536 = 2·d), LSH tanpa sapuan — aturan "hanya parameter search" berlaku seragam, tetapi LSH hanya punya satu konfigurasi sehingga aturan pemilihan K11 tidak punya pilihan untuk LSH.
2. [Latih IVF] FAISS otomatis mengambil sampel acak 256 × nlist = 1.048.576 vektor kalau data latih lebih besar, sehingga "dilatih pada seluruh korpus" (1.446.315) tidak terjadi dengan pengaturan bawaan. Mana yang dipakai?
   a. Biarkan sampel bawaan FAISS 1.048.576 vektor, jumlah sampel dan seed dicatat (Usulan) ✓ 2026-10-02 — tetap di batas atas panduan FAISS (256·nlist) dan latih lebih cepat, tetapi tidak benar-benar seluruh korpus seperti keputusan awal.
   b. Naikkan `max_points_per_centroid` supaya seluruh 1.446.315 vektor dipakai — sesuai keputusan Arya apa adanya, tetapi latih lebih lama dan melewati rentang yang disarankan panduan FAISS tanpa manfaat yang terdokumentasi.

### Rancangan 004
Tidak ada titik periksa. H14 dan H15 adalah keputusan Arya ("Terima usulan pink-chan"); disetujui ✓ 2026-10-02.

### Rancangan 005
1. [Ambang d=0] Arya menetapkan suku #8 dengan dᴱˣᵢ ≤ 1e-6 dikeluarkan. Tetangga exact dihitung dalam batch, dan FAISS memakai rumus ‖x‖² + ‖y‖² − 2⟨x, y⟩ lewat BLAS kalau nq · d ≥ 128.000 (≥ 167 query untuk d = 768). Galat float32 di jalur itu menyisakan skor L2² sekitar 1e-7–1e-6 untuk pasangan identik, yaitu d sekitar 3e-4–1e-3, sehingga ambang 1e-6 tidak menangkapnya. Ambang mana yang dipakai?
   a. Ambang d ≤ 1e-3 (setara skor L2² ≤ 1e-6) (Usulan) ✓ 2026-10-02 — menangkap pasangan identik walaupun skor batch FAISS menyisakan galat float32, tetapi pasangan yang sungguh sangat dekat (d ≤ 1e-3) ikut dikeluarkan.
   b. Ambang tetap d ≤ 1e-6 seperti keputusan — sesuai keputusan apa adanya, tetapi pasangan identik yang dihitung lewat jalur BLAS lolos ambang dan satu suku bisa menghasilkan galat relatif yang sangat besar.
   c. Ambang tetap d ≤ 1e-6, tetapi jarak Tetangga exact dihitung lewat jalur langsung FAISS (bukan BLAS) — pasangan identik bernilai tepat 0, tetapi perhitungan Tetangga exact dipisah dari run exact yang diukur waktunya dan berjalan lebih lambat.
2. [Rata-rata #8] Setelah suku dikeluarkan, bagaimana #8 dirata-rata?
   a. Per query: rata-rata atas suku yang tersisa (pembagi = jumlah suku tersisa); query yang kelima sukunya dikeluarkan tidak ikut rata-rata antarquery dan dicatat (Usulan) ✓ 2026-10-02 — sesuai "dikeluarkan dari rata-rata" dan setiap query tetap berbobot sama, tetapi pembagi 1/5 di rumus dibaca sebagai 1/m untuk query yang terkena.
   b. Pembagi tetap 5; suku yang dikeluarkan dihitung 0 — rumus tertulis tidak berubah sama sekali, tetapi galat query yang terkena tampak lebih kecil dari sebenarnya.
   c. Semua suku yang tersisa dari seluruh query dirata-rata langsung (pooled) — setiap suku berbobot sama, tetapi menyimpang dari aturan K7 "metrik per query dirata-rata atas query".

### Rancangan 006
Tidak ada titik periksa. H17 adalah keputusan Arya ("1 batch tanpa diukur"); disetujui ✓ 2026-10-03.

### Rancangan 007
1. [Buka ulang] Penjaga menghentikan notebook 06 kalau `runs_test.csv` sudah berisi run untuk Kunci konfigurasi yang sama. Kalau D4 gagal karena kesalahan kode, atau notebook terputus setelah test dibuka (sebagian baris sudah tertulis), apa jalan untuk membuka test lagi?
   a. Konstanta izin buka ulang di sel konstanta notebook 06 (bawaan mati) yang hanya diaktifkan Arya bersama alasan tertulis; baris lama tetap ada, baris baru membawa nomor percobaan dan alasan, dan laporan hasil memakai percobaan terakhir sambil menyebut jumlah percobaan (Usulan) ✓ 2026-10-03 — kesalahan kode tetap bisa diperbaiki dan jejaknya lengkap, tetapi test bisa dibuka lebih dari sekali lewat keputusan sadar.
   b. Tidak ada jalan buka ulang: test terkunci permanen untuk Kunci itu, dan perbaikan apa pun lewat rancangan baru red-chan — paling ketat, tetapi kesalahan penilai yang baru ketahuan setelah test dibuka membuat hasil test tidak bisa diperbaiki tanpa putaran rancangan.
   c. Arya menghapus baris test lama secara manual lalu menjalankan ulang — paling sederhana, tetapi melanggar aturan catatan run hanya ditambah (K4) dan menghapus jejak bahwa test pernah dibuka.
2. [Kunci baru] Penjaga hanya memeriksa `content_hash` yang sama. Kunci konfigurasi baru (misalnya `locked_config.json` dihapus lalu 05 dijalankan dengan pilihan lain) akan lolos dan membuka test lagi. Mana aturannya?
   a. Penjaga berhenti kalau `runs_test.csv` berisi baris untuk embedding_id yang sama, apa pun Kunci-nya; buka ulang hanya lewat jalan titik periksa 1 (Usulan) ✓ 2026-10-03 — menutup jalan memilih konfigurasi lagi setelah melihat hasil test, tetapi lebih ketat dari usulan pink-chan yang dipilih Arya.
   b. Sesuai usulan pink-chan: hanya `content_hash` yang sama — Kunci baru boleh membuka test lagi, cocok kalau Arya ingin mengulang benchmark dengan konfigurasi baru secara sadar, tetapi konfigurasi yang dipilih setelah melihat test membuat angka test bias.

## Bentrokan
| Bentrokan | Cara rancangan menghindarinya |
|---|---|
| Panjang maksimum 32 token vs passage yang lebih panjang; title + text (H1) memakai sebagian token untuk judul | Passage terpotong; diterima karena yang dibandingkan algoritma pada vektor yang sama, bukan kualitas model. Panjang token dan teks dokumen dicatat di Set embedding |
| Lisensi model tidak tercantum di kartu model | Model dikunci Arya, tidak diganti. Tujuan keluaran portofolio pribadi; lisensi dicek saat menyusun laporan (Belum pasti) |
| Satu fase vs 001a K2 ("benchmark final (Dev)") dan template riset fase MVP | Pilihan 002 titik periksa 1a: notebook benchmark final masuk fase MVP; 001a tidak diubah, perubahan dicatat di Riwayat |
| 001a K1 menyebut inner product = cosine vs METRIC_L2 di 002 | Untuk vektor bernorma 1, ‖q − x‖² = 2 − 2·q·x, jadi urutan L2 = urutan cosine; hasil tidak berubah |
| Penalti jarak 2 untuk slot −1 hanya sah untuk vektor bernorma 1 | Aturan integritas Vektor: norma 1 wajib; normalisasi sekali di encode |
| LSH terurut Hamming, bukan L2, saat menghitung #8 | Pilihan 002 titik periksa 3a: lima jarak L2 hasil ANN diurutkan menaik |
| #8 membagi dengan nol kalau jarak exact = 0, padahal D4 mensyaratkan exact #8 = 0 | H6: suku dengan dᴱˣᵢ ≤ 1e-3 dikeluarkan dan dihitung; untuk exact, suku 0/0 ikut dikeluarkan sehingga #8 = 0 tetap terpenuhi |
| Ambang dᴱˣᵢ ≤ 1e-6 vs galat float32 jalur BLAS FAISS (skor L2² pasangan identik tidak tepat 0) | Pilihan 005 titik periksa 1a: ambang d ≤ 1e-3 (setara L2² ≤ 1e-6) [S15] |
| "Dikeluarkan dari rata-rata" (H6) vs "rumus #8 tidak berubah" (pembagi 1/5) | Pilihan 005 titik periksa 2a: per query dirata-rata atas suku tersisa (1/m); query tanpa suku tersisa tidak ikut rata-rata antarquery dan dicatat |
| K2 "test dibuka sekali" vs notebook 06 yang bisa dijalankan ulang dan menambah 5 baris test lagi (temuan 003b langkah 11) | K12: penjaga di notebook 06 berhenti sebelum query test dibaca kalau test sudah pernah dibuka untuk embedding_id yang sama |
| Penjaga per `content_hash` bisa dilewati dengan Kunci konfigurasi baru | Pilihan 007 titik periksa 2a: penjaga diikat ke embedding_id, apa pun Kunci-nya |
| Run ditulis dulu sebelum D4 diperiksa (notebook 06), jadi D4 yang gagal atau notebook yang terputus tetap meninggalkan baris test | Pilihan 007 titik periksa 1a: buka ulang hanya lewat izin tertulis Arya di sel konstanta; baris lama tetap; percobaan dinomori dan alasannya dicatat |
| Konstanta izin buka ulang bisa tertinggal aktif setelah dipakai, sehingga eksekusi berikutnya membuka test lagi tanpa disadari | Setiap buka ulang menambah nomor percobaan dan menyalin alasan ke setiap baris, sehingga terlihat di laporan; notebook mencetak peringatan saat izin aktif; Arya mengembalikan konstanta ke mati setelah percobaan (asumsi cara pakai) |
| "Test dibuka sekali" (K2) vs izin buka ulang (007-1a) | Pengecualian tertulis: test hanya dibuka lebih dari sekali lewat keputusan sadar Arya; laporan menyebut jumlah percobaan sehingga pembaca tahu |
| Penjaga butuh asal setiap baris test, sedangkan kolom Run belum menyimpannya | K4: baris test menyimpan `config_lock_hash`, `test_attempt`, dan `reopen_reason`; `runs_test.csv` masih kosong sehingga kolom baru tidak merusak baris lama |
| Konfigurasi val yang tercatat lebih dari sekali membuat pilihan K11 ambigu (H20) | Sementara notebook 05 berhenti kalau ada konfigurasi ganda; aturan run mana yang dipakai diputuskan kalau kasusnya terjadi (Belum pasti) |
| 002a/002b menetapkan `requirements.txt` tepat tiga paket, sedangkan H16 menambah pyarrow dan pandas `==` | K13: `requirements.txt` bertambah dua baris di luar daftar 002a; versinya dari `pip freeze` instance yang sama dengan torch dan numpy (H13) |
| Kunci versi pyarrow dan pandas baru tersedia setelah instance pertama dibuat | Sama dengan H9/H13: sampai keluaran instance ada, kedua baris belum bisa ditulis |
| sentence-transformers 6.1.0 hanya Python 3.10–3.13 | Python wajib 3.10–3.13; versi pasti dijawab saat instance pertama dibuat (H9) |
| Kartu `miracl/miracl-corpus` mencontohkan loading script vs datasets 5.x | K5: data dimuat langsung dari file |
| Penyimpanan Vast.ai tidak permanen vs catatan run yang menumpuk lintas sesi | Vektor, catatan run, dan hasil disalin keluar sebelum instance dihapus; tempat dijawab sebelum instance pertama dihapus (H10). Penjaga test bergantung pada `runs_test.csv` ikut disalin dan dibawa kembali |
| Angka efisiensi hanya sah dalam satu sesi dan satu mesin | Keempat algoritma diukur di satu instance dan satu sesi; notebook final memeriksa env_id |
| env_id memuat hostname dan semua field statis, termasuk revision model dan dataset (H11, H15) | Sapuan val dan benchmark final test wajib di instance yang sama dan dengan revision yang sama |
| Sebagian host Vast.ai masih cgroup v1 | K6 dan K8: pembaca jatah CPU dan RAM mendukung v1 dan v2 |
| Host hybrid (v1 dan v2 terpasang sekaligus) belum punya urutan deteksi (H18) | Tidak menahan langkah; notebook berhenti sebelum mengukur apa pun |
| Pembacaan cgroup hanya ada di Linux vs laptop Arya Windows | Hanya relevan kalau notebook dijalankan lokal (H12) |
| Tuning parameter bisa membocorkan query test | Sapuan dan pemilihan hanya di val; konfigurasi dikunci (K11) sebelum test dibuka; test dijaga (K12) |
| p50 dan QPS mengukur hal berbeda: QPS ≠ 1000 / p50 [S15] | Diterima sebagai dua sudut pandang; K11 memakai QPS batch |
| #12 dengan `faiss.serialize_index` membuat salinan index di memori | Puncak RAM naik selama serialisasi; dicatat lewat puncak RAM K8 |
| Sapuan "hanya parameter search" vs nbits LSH yang merupakan parameter build | Pilihan 003 titik periksa 1a: pengecualian tertulis |
| "IVF dilatih pada seluruh korpus" vs subsampling bawaan FAISS | Pilihan 003 titik periksa 2a: sampel bawaan 1.048.576 vektor |
| nlist = 4096 disebut "sekitar 4√N, sesuai panduan FAISS" (sebenarnya ≈ 3,4·√N) | Keputusan Arya dipertahankan; dicatat apa adanya |
| Run ulang exact (D4) vs "test dibuka sekali" (K2) | Exact ulang berjalan di eksekusi notebook 06 yang sama; penjaga memeriksa sebelum query test dibaca |
| Seed rotasi IndexLSH berupa konstanta `rrot.init(5)` | Nilai yang dicatat adalah konstanta 5 dari kode sumber FAISS 1.15.1 [S14] |

## Asumsi
- Pengguna utama Arya sendiri; keluaran adalah portofolio pribadi, dibaca siapa pun yang melihat portofolio itu.
- Struktur project mengikuti template riset notebook-only (`structure-riset.md`).
- Arya mengunduh dataset; agent tidak mengunduh.
- Relevan berarti relevance ≥ 1 di qrels MIRACL.
- Search mengambil top-5 (k = 5) untuk semua algoritma dan semua konfigurasi sapuan.
- RAM ≥ 32 GB cukup kalau index dibangun dan dilepas satu per satu, termasuk salinan sementara saat serialisasi #12.
- Exact menghasilkan k-NN Recall@5 = 1,0 dan Relative distance error = 0 menurut definisinya.
- Toleransi pemeriksaan encode 1e-5 dibaca sebagai selisih mutlak maksimum per elemen antara vektor ternormalisasi dari loop sendiri dan dari `model.encode`.
- Run ulang exact pada tanda berhasil dilakukan di split test, di dalam eksekusi notebook benchmark final yang sama.
- IndexLSH dibuat dengan `rotate_data` = true dan `train_thresholds` = false (bawaan konstruktor Python FAISS).
- Pengukuran p50: 10 query pemanasan dijalankan sekali sebelum putaran pertama, memakai query dari split yang sama.
- Jatah tanpa batas di cgroup v1 diperlakukan sama seperti `max` di cgroup v2.
- Pada panggilan 1 query, HNSW, IVF, dan Flat praktis berjalan di satu thread (pengetahuan umum).
- Galat float32 jalur BLAS untuk vektor bernorma 1 berada di orde 1e-7–1e-6 pada skor L2² (perkiraan, bukan angka terukur).
- Pemanasan QPS (H17) dijalankan sekali per konfigurasi, tepat sebelum 5 ulangan QPS konfigurasi itu.
- `runs_test.csv` belum berisi baris saat 007 dibangun, sehingga kolom baru bisa ditambah tanpa migrasi.
- Arya mengembalikan konstanta izin buka ulang ke mati setelah percobaan yang diizinkan selesai.

## Belum pasti
Diputuskan Arya untuk dijawab saat pengerjaan:

| Kode | Hal | Dijawab paling lambat |
|---|---|---|
| H9, H13, H16 | Versi Python; versi `==` torch, numpy, pyarrow, dan pandas (keputusan mengunci sudah diambil; nilainya dari `pip freeze` instance) | Saat instance Vast.ai pertama dibuat (`python --version`, `pip freeze`) |
| H10 | Penyimpanan di luar Vast.ai | Sebelum instance pertama dihapus |
| H12 | Perilaku pencatatan resource di luar Linux | Hanya kalau notebook dijalankan lokal |
| H20 | Run val mana yang dipakai kalau satu konfigurasi tercatat lebih dari sekali. Sementara: notebook 05 berhenti kalau ada konfigurasi ganda | Kalau kasusnya terjadi |
| — | Grafik laporan, lisensi model | Saat menyusun laporan |

Belum dijadwalkan (tidak menahan langkah):
- H18 Urutan deteksi cgroup di host hybrid (v1 dan v2 terpasang sekaligus). Sementara: notebook berhenti dengan error.
- H19 Versi cgroup dicatat di Lingkungan atau tidak. Sementara: tidak dicatat.

## Ditunda
| Topik | Ditunda sampai |
|---|---|
| Hal-hal di Belum pasti | Batas waktu di tabel Belum pasti; project tidak punya fase Dev |

---
<!-- Bagian teknis — dibaca pink-chan -->

## Gambaran sistem
| Bagian | Tugasnya | Terhubung ke |
|---|---|---|
| Penyiapan data (K2, K5, K9) | Memuat korpus jsonl.gz dan topics/qrels TSV dev dari file; memeriksa kunci dan relasi; membagi query val/test sekali dan mengunci pembagiannya dengan hash; mencatat revision dataset | Menulis Dokumen, Query, Penilaian relevansi (Parquet); dibaca Pembuat embedding dan Penilai |
| Pembuat embedding (K1) | Di GPU, fp32: teks dokumen title + " " + text; tokenisasi max_length 32, Transformer → Pooling mean → Dense 768→768 + Tanh → normalisasi L2; batch dari sel konstanta; pemeriksaan 100 teks sampel terhadap `model.encode`; jatah CPU dan RAM dibaca dari cgroup v1 atau v2 | Membaca Dokumen dan Query; menulis Vektor (.npy) dan Set embedding (JSON) |
| Exact search (K3) | `IndexFlatL2` top-5 di CPU: baseline sekaligus Tetangga exact. Dijalankan sebelum bagian ANN | Membaca vektor; menulis Tetangga exact (Parquet) dan hasil ke Penilai |
| HNSW (K3, K10) | `IndexHNSWFlat` (METRIC_L2), M = 32, efConstruction = 200; satu build, sapuan efSearch di val | Membaca vektor; hasil ke Penilai |
| IVF (K3, K10) | `IndexIVFFlat` dengan quantizer `IndexFlatL2`, nlist = 4096; satu latih (sampel bawaan FAISS 1.048.576 vektor, seed k-means bawaan 1234) dan build, sapuan nprobe di val | Membaca vektor; hasil ke Penilai |
| LSH (K3, K10) | `IndexLSH` (jarak Hamming, rotasi acak seed bawaan 5), tiga build untuk sapuan nbits di val (pengecualian tertulis); jarak L2 hasil dihitung ulang untuk #8 | Membaca vektor; hasil ke Penilai |
| Pengukur waktu dan ukuran (K7) | Untuk setiap konfigurasi: p50 dari panggilan 1 query (10 pemanasan, 3 putaran); QPS dari 1 panggilan batch pemanasan yang tidak diukur lalu 5 panggilan batch yang diukur, semuanya dengan n thread; hanya `index.search` yang diukur dengan `time.perf_counter_ns`; ukuran index = byte `faiss.serialize_index` | Membaca index dan query; hasil ke Penilai |
| Penilai (K7) | Menghitung 12 metrik dan kolom diagnostik; menangani slot −1 dan suku #8 dengan jarak exact ≤ 1e-3 | Membaca Penilaian relevansi, Tetangga exact, vektor (untuk LSH); menulis Run |
| Catatan run dan lingkungan (K4, K8, K9) | Satu baris per run di CSV yang hanya ditambah; Lingkungan (JSON) per sesi dengan env_id dari semua field statis + hostname | Dibaca Tabel hasil, Pemilih konfigurasi, Benchmark final, Penjaga test |
| Tabel hasil (K4) | Menampilkan hasil sapuan keempat algoritma untuk split val | Membaca Run |
| Pemilih konfigurasi (K11) | Per algoritma memilih konfigurasi val menurut aturan K11, lalu menulis kunci konfigurasi (JSON, cap waktu + hash isi); berhenti kalau satu konfigurasi tercatat lebih dari sekali (H20, sementara) | Membaca Run split val; menulis kunci konfigurasi |
| Penjaga test (K12) | Di notebook 06, setelah pemeriksaan hash Kunci dan env_id dan sebelum query test dibaca: berhenti kalau catatan run test sudah berisi baris untuk embedding_id yang sama, kecuali izin buka ulang aktif dengan alasan tertulis; menentukan nomor percobaan | Membaca Kunci konfigurasi, sel konstanta izin, dan `runs_test.csv`; mengizinkan atau menghentikan Benchmark final |
| Benchmark final (K2, D4) | Notebook terakhir: memeriksa hash kunci konfigurasi dan env_id, melewati Penjaga test, membuka test, menjalankan keempat konfigurasi terkunci, menjalankan ulang exact, menulis Run dengan nomor percobaan, lalu memeriksa tanda berhasil | Membaca kunci konfigurasi, Run, Lingkungan; menulis Run split test |
| Laporan hasil test (K12) | Menyajikan percobaan terakhir untuk embedding_id itu sambil menyebut jumlah percobaan dan alasan buka ulang | Membaca Run split test |

Seluruh pengukuran di satu instance Vast.ai dalam satu sesi (K6); vektor dan hasil disalin keluar instance sebelum instance dihapus.

## Keputusan
| # | Keputusan | Rancangan | Alasan | Label |
|---|---|---|---|---|
| D1 | Pengguna | Arya sendiri yang menjalankan notebook; keluaran berupa portofolio pribadi tentang kemampuan mengimplementasikan retrieval, dibaca dari atas ke bawah | Jawaban Arya: "hanya portofolio pribadi mengenai kemampuan peng-implementasian retrieval." | — |
| D2 | Cakupan | 001: bagian sistem, aliran, entitas data, pemakaian model, pembagian query. 002: dataset, library, metrik, tempat menjalankan, pencatatan resource, satu fase dengan benchmark final. 003: format berkas, teks dokumen, encode, env_id, parameter dan sapuan, aturan pemilihan, tanda berhasil. 004: seed IVF dan LSH, field env_id. 005: cara ukur QPS dan p50, definisi #12, jarak exact mendekati 0 pada #8, dukungan cgroup v1/v2. 006: pemanasan QPS. 007: penjaga test dan izin buka ulang, H20, penguncian pyarrow dan pandas (H16). Tidak dirancang: hal di Belum pasti | Koreksi dan keputusan Arya | — |
| D3 | Sumber data | Korpus `miracl/miracl-corpus` subset id (`miracl-corpus-v1.0-id`, 3 file `docs-*.jsonl.gz`, 1.446.315 passage dari 446.330 artikel; kolom docid, title, text; Apache-2.0). Query dan qrels `miracl/miracl` folder `miracl-v1.0-id` (`topics/*.tsv`, `qrels/*.tsv` format TREC), split dev: 960 query, 9.668 penilaian, rata-rata 3,22 relevan per query (min 1, maks 13). Qrels test-a/test-b tidak dirilis, tidak dipakai. WebFAQ dan versi hard negatives MTEB ditolak | Keputusan Arya: dataset yang sudah proper [S3, S10] | — |
| D4 | Tanda berhasil | Benchmark berhasil kalau: (1) 12 metrik lengkap untuk keempat algoritma di split test; (2) exact lolos pemeriksaan kewajaran: k-NN Recall@5 = 1 dan Relative distance error = 0; (3) run ulang exact menghasilkan nDCG@5 identik. D4 dinilai pada percobaan terakhir. Kalau D4 gagal, test hanya bisa dibuka lagi lewat izin buka ulang K12 | Keputusan Arya (H3); 007 titik periksa 1a | — |
| K1 | Cara memakai model embedding | `sentence-transformers` 6.1.0 hanya untuk memuat `LazarusNLP/congen-indobert-base` dengan revision (commit hash) dicatat. Encode dengan loop PyTorch sendiri melewati ketiga modul: Transformer (max_seq_length 32) → Pooling mean tokens → Dense 768→768 + Tanh. Tokenisasi max_length 32 untuk dokumen dan query. Teks dokumen = `title + " " + text`; teks query apa adanya. Presisi fp32. Ukuran batch ditetapkan di sel konstanta, dicoba mulai 1024, nilai akhirnya dicatat di Set embedding. Normalisasi L2 sekali dengan `F.normalize`; vektor tersimpan sudah bernorma 1 dan notebook search tidak menormalisasi lagi. Pemeriksaan: 100 teks sampel di-encode dengan loop sendiri dan dengan `model.encode`, selisih ≤ 1e-5. Embedding dibuat sekali di GPU | Keputusan Arya (H1, H7); susunan modul dan 32 token dari model [S1, S11] | Umum |
| K2 | Pembagian query dan benchmark final | 960 query dev dibagi val/test 50:50 acak dengan seed 42; disimpan sebagai atribut split dan dikunci hash isi. Sapuan dan pemilihan hanya di val. Kunci konfigurasi (K11) ditulis sebelum test dibuka; notebook benchmark final berhenti kalau hash kunci atau env_id tidak cocok, atau kalau Penjaga test (K12) menolak. Test dibuka sekali; pengecualiannya hanya izin buka ulang tertulis Arya (K12). Korpus tidak dibagi | 001a K2 + 002 titik periksa 1a; K12 | Umum |
| K3 | Library index dan algoritma | `faiss-cpu` 1.15.1, CPU. Exact: `IndexFlatL2`. HNSW: `IndexHNSWFlat` METRIC_L2. IVF: `IndexIVFFlat` dengan quantizer `IndexFlatL2`. LSH: `IndexLSH` (proyeksi acak → kode biner → jarak Hamming). Exact dijalankan lebih dulu. Parameter di K10 | Keputusan Arya [S5, S6, S8] | Umum |
| K4 | Catatan run dan hasil | Setiap run menambah satu baris yang tidak pernah ditimpa: identitas run, parameter konfigurasi, 12 metrik K7, kolom diagnostik (jumlah query dengan hasil < 5; jumlah suku #8 yang dikeluarkan karena dᴱˣᵢ ≤ 1e-3; jumlah query yang kelima suku #8-nya dikeluarkan), jumlah thread FAISS dan torch, rujukan Set embedding dan Lingkungan. Baris split test juga menyimpan `config_lock_hash` (= `content_hash` Kunci konfigurasi), `test_attempt` (nomor percobaan, mulai 1), dan `reopen_reason` (alasan izin buka ulang; kosong untuk percobaan 1). Satu konfigurasi sapuan = satu run | 001a K4 + keputusan Arya (H6); 005 titik periksa 2a; K12 | Umum |
| K5 | Pemuatan dataset | `datasets` 5.0.1, langsung dari file: korpus jsonl.gz dengan builder `json` (atau konversi Parquet Hugging Face); topics dan qrels TSV dengan builder `csv`, pemisah tab. Revision dataset dicatat | Keputusan Arya [S10, S12] | Umum |
| K6 | Tempat menjalankan dan thread | Vast.ai (Linux): RTX 3090 24 GB, RAM ≥ 32 GB, CPU ≥ 24 (jatah efektif vCPU), disk 50 GB. GPU hanya embedding; search di CPU. Thread = min(jatah CPU dari cgroup, 24), dibulatkan ke bawah, diset lewat `faiss.omp_set_num_threads` dan `torch.set_num_threads`, dicatat di setiap run; `os.cpu_count()` tidak dipakai. Jatah CPU dibaca dari cgroup v2 (`cpu.max`) atau cgroup v1 (`cpu.cfs_quota_us` / `cpu.cfs_period_us`), dibatasi `sched_getaffinity`. Satu instance, satu sesi. Vektor dan hasil disalin keluar sebelum instance dihapus | Keputusan Arya; dukungan v1 dari temuan 003b langkah 5 | Umum |
| K7 | Metrik evaluasi dan cara ukur | 12 metrik dikunci, k = 5. Rumus, aturan slot −1, aturan jarak exact mendekati 0, dan cara ukur waktu serta ukuran di bawah tabel ini | Keputusan Arya; 002 titik periksa 3a, 4a; H4, H5, H6, H17; 005 titik periksa 1a, 2a | Umum |
| K8 | Pencatatan resource dan env_id | Dicatat otomatis ke Lingkungan: GPU (model, VRAM, driver, CUDA, puncak VRAM); CPU (model, flag AVX2/AVX-512, jatah vCPU dari cgroup v2 `cpu.max` atau v1 `cpu.cfs_quota_us`/`cpu.cfs_period_us`, dan `sched_getaffinity`, total vCPU mesin); RAM (jatah dari cgroup v2 `memory.max` atau v1 `memory.limit_in_bytes`, puncak RAM proses); disk (total, sisa); thread FAISS dan torch; versi Python dan library; revision model dan dataset; timestamp sesi. env_id = hash dari semua field statis ditambah hostname instance; field dinamis (puncak RAM dan VRAM, sisa disk, timestamp) tidak membentuk env_id. Dicatat manual oleh Arya di README.md: ID penawaran/host, harga per jam, reliability, lokasi, status verified | Keputusan Arya (H11, H15) | Umum |
| K9 | Format berkas fisik | Dokumen, Query, Penilaian relevansi, Tetangga exact → Parquet. Catatan run → CSV yang hanya ditambah barisnya. Vektor dokumen dan query → `.npy` float32. Metadata Set embedding, Lingkungan, dan kunci konfigurasi → JSON | Keputusan Arya (H8) | Umum |
| K10 | Parameter, sapuan, dan seed | Sapuan di split val. Parameter build tetap dan parameter search disapu, dengan satu pengecualian tertulis untuk LSH. Exact: 1 run. HNSW: M = 32, efConstruction = 200; efSearch ∈ {16, 32, 64, 128, 256} → 5 run. IVF: nlist = 4096, dilatih dengan sampel bawaan FAISS 1.048.576 vektor, seed k-means bawaan 1234 dicatat; nprobe ∈ {1, 4, 8, 16, 32, 64, 128} → 7 run. LSH: nbits ∈ {768, 1536, 3072} → 3 build, 3 run; seed rotasi bawaan 5 dicatat. Total 16 run val | Keputusan Arya (H2, H14); 003 titik periksa 1a, 2a [S7, S13, S14] | Umum |
| K11 | Aturan memilih konfigurasi untuk test | Per algoritma, dari run val: konfigurasi dengan QPS tertinggi (QPS batch, K7 #9) di antara yang k-NN Recall@5 ≥ 0,95; kalau tidak ada yang mencapai, konfigurasi dengan k-NN Recall@5 tertinggi. Hasilnya ditulis ke berkas kunci konfigurasi (cap waktu + hash isi) sebelum test dibuka. Kalau satu konfigurasi tercatat lebih dari sekali di run val, notebook 05 berhenti (H20, sementara) | Keputusan Arya (H3, H20) | Umum |
| K12 | Penjaga test dan izin buka ulang | Di notebook 06, setelah pemeriksaan hash Kunci dan env_id, sebelum query test dibaca: baca `runs_test.csv` (kalau ada) dan ambil baris dengan embedding_id sama dengan embedding_id Kunci sekarang, apa pun `config_lock_hash`-nya. Tidak ada baris → percobaan 1, lanjut. Ada baris → berhenti dengan pesan yang menyebut jumlah percobaan dan cap waktunya, kecuali sel konstanta berisi izin buka ulang aktif (bawaan mati) dan alasan tertulis yang tidak kosong; dalam hal itu notebook mencetak peringatan, percobaan baru = nomor percobaan terbesar + 1, dan alasan disalin ke setiap baris percobaan itu. Baris lama tidak diubah atau dihapus. Laporan hasil test memakai percobaan terakhir untuk embedding_id itu dan menyebut jumlah percobaan beserta alasannya. Penjaga hanya membaca catatan run, tidak membaca data test | Keputusan Arya (usulan pink-chan, temuan 003b langkah 11); 007 titik periksa 1a dan 2a | Umum |
| K13 | Penguncian dependency | `requirements.txt` berisi `sentence-transformers==6.1.0`, `faiss-cpu==1.15.1`, `datasets==5.0.1` (002a), ditambah `torch`, `numpy` (H13), `pyarrow`, dan `pandas` (H16) yang dikunci `==` dengan versi dari `pip freeze` instance Vast.ai yang sama. Versi Python dari `python --version` instance itu (H9) | Keputusan Arya H16: Parquet dan hash isi di notebook 01 dibaca dan ditulis dengan versi yang sama di setiap sesi | Umum |

**Rumus K7.** Rel(q) = dokumen dengan relevance ≥ 1 di qrels. relᵢ = 1 kalau hasil ke-i ∈ Rel(q), selain itu 0; slot −1 selalu relᵢ = 0.

```
1  nDCG@5        = DCG@5 / IDCG@5
                   DCG@5  = Σᵢ₌₁⁵ relᵢ / log₂(i + 1)
                   IDCG@5 = DCG@5 dengan min(|Rel(q)|, 5) dokumen relevan di peringkat teratas
2  Recall@5      = |Top5 ∩ Rel(q)| / |Rel(q)|
3  MRR@5         = 1 / peringkat dokumen relevan pertama di top-5; 0 kalau tidak ada
4  Precision@5   = |Top5 ∩ Rel(q)| / 5
5  MAP@5         = rata-rata AP@5 atas query
                   AP@5 = (1 / min(|Rel(q)|, 5)) · Σᵢ₌₁⁵ P@i · relᵢ
                   P@i  = jumlah relevan di peringkat 1..i / i
6  Hit rate@5    = 1 kalau |Top5 ∩ Rel(q)| ≥ 1, selain itu 0
7  k-NN Recall@5 = |ANN₅(q) ∩ Exact₅(q)| / 5
8  Relative distance error
                   I(q)     = { i ∈ 1..5 : dᴱˣᵢ > τ },  τ = 1e-3   (setara skor L2² > 1e-6)
                   RDE(q)   = (1 / |I(q)|) · Σ_{i ∈ I(q)} (dᴬᴺᴺ₍ᵢ₎ − dᴱˣᵢ) / dᴱˣᵢ
                   #8       = rata-rata RDE(q) atas query dengan |I(q)| ≥ 1
                   d      = √max(skor L2 kuadrat dari FAISS, 0)
                   LSH    : d = ‖q − xᵢ‖₂ dihitung ulang dari vektor asli untuk ID hasil
                   dᴬᴺᴺ₍ᵢ₎ = jarak ke-i setelah lima jarak ANN diurutkan menaik
                   dᴱˣᵢ   = jarak ke-i hasil IndexFlatL2
                   slot −1: dᴬᴺᴺ = 2 (jarak L2 maksimum dua vektor bernorma 1)
                   Dicatat di Run: Σ_q (5 − |I(q)|) suku dikeluarkan dan jumlah query
                   dengan |I(q)| = 0
9  QPS           = n_query / median(T₁, …, T₅)
                   T₀ = satu panggilan index.search(semua query split, k = 5) dengan
                        n thread sebagai pemanasan; tidak diukur (H17)
                   Tⱼ = waktu satu panggilan index.search(semua query split, k = 5)
                        dengan n thread (K6); j = 1..5, dijalankan setelah T₀
10 Latensi p50   = median { t_{r,q} : r = 1..3, q = semua query split }   (ms)
                   t_{r,q} = waktu index.search(1 query, k = 5);
                   10 query pemanasan sebelum putaran pertama tidak dihitung
11 Waktu build/train index (detik)
12 Ukuran index  = jumlah byte faiss.serialize_index(index)
Pengukuran waktu: time.perf_counter_ns, hanya mencakup index.search.
Metrik 1–7 per query dirata-rata atas semua query di split yang dijalankan.
```

Aturan hasil kurang dari 5 (ID −1, terutama dari IVF):
- Metrik kualitas (1–6) dan k-NN Recall@5 (7): slot −1 dianggap tidak relevan atau tidak cocok; pembagi tetap 5.
- Relative distance error (8): slot −1 diberi penalti jarak 2.
- Jumlah query dengan hasil kurang dari 5 dicatat sebagai kolom diagnostik di setiap run.
- −1 tidak boleh dipakai sebagai indeks array.

Fakta pembacaan (960 query dev): batas atas rata-rata Precision@5 = 0,567 dan Recall@5 = 0,953. nDCG@5 dan Recall@5 tidak sebanding langsung dengan angka resmi MIRACL (nDCG@10, Recall@100).

**Rumus K11.** Cₐ = konfigurasi algoritma a yang dijalankan di val; R(c) = k-NN Recall@5 di val.

```
c*(a) = argmax { QPS(c) : c ∈ Cₐ, R(c) ≥ 0,95 }    kalau himpunan itu tidak kosong
c*(a) = argmax { R(c)   : c ∈ Cₐ }                  kalau kosong
Prasyarat (H20, sementara): setiap konfigurasi c muncul tepat sekali di run val;
kalau tidak, notebook 05 berhenti.
```

**Aturan K12.** L = Kunci konfigurasi yang lolos pemeriksaan hash; E = L.embedding_id; P = baris `runs_test.csv` dengan embedding_id = E.

```
P = ∅                                        → lanjut, test_attempt = 1, reopen_reason kosong
P ≠ ∅ dan IZIN_BUKA_ULANG mati               → berhenti (sebut jumlah percobaan dan cap waktu)
P ≠ ∅ dan IZIN_BUKA_ULANG aktif, alasan kosong → berhenti
P ≠ ∅ dan IZIN_BUKA_ULANG aktif, alasan ada  → peringatan, lanjut,
                                               test_attempt = max(P.test_attempt) + 1,
                                               reopen_reason = alasan
Urutan di notebook 06: hash Kunci cocok → env_id cocok → penjaga K12 → baca query test
Laporan: baris dengan test_attempt terbesar untuk E; sebut jumlah percobaan dan alasannya
D4 dinilai pada percobaan terakhir
```

## Desain UI/UX
Tidak berlaku — produk tidak punya antarmuka; hasil dibaca di notebook.

## Model data
Tidak ada basis data; data disimpan sebagai berkas yang menjadi kontrak antarbagian. Format fisik per entitas di kolom Isi (K9).

| Entitas | Isi | Kunci | Relasi | Aturan integritas |
|---|---|---|---|---|
| Dokumen | doc_id (= `docid` MIRACL, skema "X#Y"), title, text — Parquet | doc_id (natural) | 1─N Penilaian relevansi; 1─1 Vektor dokumen per set embedding | doc_id unik; tidak diubah setelah disiapkan |
| Query | query_id, text, split — Parquet | query_id (natural, dari topics MIRACL) | 1─N Penilaian relevansi; 1─1 Vektor query per set embedding; 1─N Tetangga exact | split ∈ {val, test}, 50:50 seed 42, dikunci hash; setiap query punya ≥ 1 dokumen relevan; test hanya dibaca notebook benchmark final setelah hash kunci konfigurasi cocok dan Penjaga test mengizinkan |
| Penilaian relevansi | query_id, doc_id, relevance (qrels TREC dev) — Parquet | (query_id, doc_id) | N─1 Query; N─1 Dokumen | Relevan kalau relevance ≥ 1; id yang tidak ada dibuang dan jumlahnya dicatat |
| Set embedding | embedding_id, model, revision model, revision dataset, max_seq_length 32, teks dokumen `title + " " + text`, presisi fp32, ukuran batch akhir, normalisasi L2, dim 768, hasil pemeriksaan encode — JSON | embedding_id (surrogate: hash konfigurasi) | 1─N Vektor dokumen; 1─N Vektor query; 1─N Tetangga exact; 1─N Run | Konfigurasi berbeda menghasilkan embedding_id baru; vektor lama tidak ditimpa |
| Vektor dokumen | float32[768] per dokumen — `.npy` | (embedding_id, row_idx) | 1─1 Dokumen lewat urutan doc_id yang disimpan bersama | 1.446.315 baris; norma L2 = 1 |
| Vektor query | float32[768] per query — `.npy` | (embedding_id, row_idx) | 1─1 Query | 960 baris; norma L2 = 1 |
| Tetangga exact | embedding_id, query_id, rank (1–5), doc_id, skor L2 kuadrat dari `IndexFlatL2` — Parquet | (embedding_id, query_id, rank) | N─1 Set embedding; N─1 Query; N─1 Dokumen | Hanya berlaku untuk embedding_id yang sama; skor disimpan apa adanya |
| Kunci konfigurasi | Per algoritma: konfigurasi terpilih (K11), run_id val asalnya, embedding_id, env_id, cap waktu, content_hash — JSON | content_hash | N─1 Run (val); 1─N Run (test) lewat `config_lock_hash` | Ditulis sekali sebelum test dibuka; notebook final berhenti kalau hash tidak cocok |
| Run | run_id, timestamp, embedding_id, env_id, algorithm, params, split, k = 5, thread FAISS, thread torch, 12 metrik K7, kolom diagnostik; untuk split test juga `config_lock_hash`, `test_attempt`, `reopen_reason` — baris CSV append-only | run_id (surrogate) | N─1 Set embedding; N─1 Lingkungan; N─1 Kunci konfigurasi (test) | Hanya ditambah, baris lama tidak pernah diubah atau dihapus; algorithm ∈ {flat, hnsw, ivf, lsh}; split ∈ {val, test}; konfigurasi val unik (H20, sementara); `test_attempt` ≥ 1 dan sama untuk semua baris satu eksekusi notebook 06; `reopen_reason` wajib terisi kalau `test_attempt` > 1 |
| Lingkungan | Isi K8 — JSON | env_id = hash semua field statis K8 + hostname | 1─N Run | Field dinamis tidak masuk hash; notebook final memeriksa env_id sama dengan run val |

## Sumber
| # | Sumber | Tingkat | Tanggal | Dipakai untuk |
|---|---|---|---|---|
| S1 | Kartu model, `sentence_bert_config.json`, dan metadata API `huggingface.co/LazarusNLP/congen-indobert-base` | 1 | diubah 2025-02-01 | max_seq_length 32, mean pooling, 768 dimensi, tanpa prefix; lisensi tidak tercantum |
| S3 | Kartu dataset `huggingface.co/datasets/miracl/miracl` (Apache-2.0) | 1 | 2022-10-18 | 960 query dev id, format qrels TREC, label test tidak dirilis |
| S5 | FAISS `INSTALL.md`, `github.com/facebookresearch/faiss` | 1 | versi 1.15.1 | Windows hanya CPU |
| S6 | `pypi.org/project/faiss-cpu` | 1 | 2026-09-16 | Versi 1.15.1, MIT, Python 3.10–3.14 |
| S7 | FAISS wiki "Guidelines to choose an index" | 1 | — (konsep stabil) | Rentang nlist dan data latih |
| S8 | FAISS wiki "Faiss indexes" | 1 | — (konsep stabil) | IndexFlatL2/HNSWFlat/IVFFlat/LSH, cara kerja IndexLSH |
| S10 | Kartu dataset `huggingface.co/datasets/miracl/miracl-corpus` (Apache-2.0) | 1 | v1.0 | 1.446.315 passage id; kolom docid/title/text; jsonl.gz |
| S11 | `pypi.org/project/sentence-transformers` (Apache-2.0) | 1 | 2026-09-18 | Versi 6.1.0, Python 3.10–3.13 |
| S12 | `pypi.org/project/datasets` (Apache-2.0) | 1 | 2026-07-28 | Versi 5.0.1, Python 3.10–3.14 |
| S13 | FAISS `faiss/Clustering.h` | 1 | main | `max_points_per_centroid` = 256, seed bawaan 1234 |
| S14 | FAISS `faiss/IndexLSH.cpp` | 1 | main | Rotasi acak `rrot.init(5)` |
| S15 | FAISS wiki "Implementation notes" | 1 | — (konsep stabil) | Jalur langsung vs BLAS pada IndexFlatL2 |

Sumber riset 001 putaran 1 lain (S2, S4, S9) tidak menjadi dasar keputusan. Fakta "datasets 5.x tidak mendukung loading script" dan statistik qrels dev berasal dari diskusi Arya.

## Riwayat
| Tanggal | Perubahan | Alasan |
|---|---|---|
| 2026-10-02 | Usulan pertama rancangan MVP | Perintah Arya |
| 2026-10-02 | 001 putaran 2: korpus dan LSH dikeluarkan dari 001; tech stack/library/tools dan kontrak metrik evaluasi ditunda ke 002; max_seq_length 32 dan split val/test 50:50 dipilih | Jawaban dan koreksi Arya |
| 2026-10-02 | 001 disetujui; korpus dan LSH ikut 002 | Persetujuan Arya |
| 2026-10-02 | 002 disetujui: D3 diisi (MIRACL-id korpus penuh + dev); K1 diisi (sentence-transformers hanya memuat, encode PyTorch tiga modul, L2 sekali); K3 diisi (faiss-cpu, METRIC_L2 menggantikan rumusan inner product di 001a K1 — urutan sama untuk vektor bernorma 1; LSH = IndexLSH); K4 diisi kolom metrik; K5–K8 ditambah | Hasil diskusi Arya |
| 2026-10-02 | Fase tunggal: project hanya punya fase MVP; benchmark final di split test masuk MVP sebagai notebook terakhir (mengubah 001a K2 "benchmark final (Dev)"); baris Ditunda→Dev dihapus | Koreksi Arya "Cukup 1 fase saja", 002 titik periksa 1a |
| 2026-10-02 | Teks dokumen yang di-embed ditunda ke pekerjaan berikutnya | 002 titik periksa 2, jawaban Arya |
| 2026-10-02 | 003 usulan: D1, D4, K1, K8 diubah; K9, K10, K11 ditambah; entitas Kunci konfigurasi ditambah; Belum pasti diberi batas waktu | Hasil diskusi Arya (H1, H2, H3, H7, H8, H11) |
| 2026-10-02 | 003 disetujui: sapuan nbits LSH menjadi pengecualian tertulis; IVF dilatih dengan sampel bawaan FAISS 1.048.576 vektor; status kembali siap dikerjakan | 003 titik periksa 1a dan 2a, persetujuan Arya |
| 2026-10-02 | 004 usulan: K10 seed IVF (1234) dan seed rotasi LSH (5) dicatat di Run; K8 env_id dari semua field statis + hostname | Keputusan Arya H14, H15 |
| 2026-10-02 | 004 disetujui tanpa koreksi; status kembali siap dikerjakan | Persetujuan Arya |
| 2026-10-02 | 005 usulan: K7 cara ukur QPS dan p50, definisi #12, aturan jarak exact mendekati 0 pada #8; K4 kolom diagnostik #8; K6 dan K8 cgroup v1 dan v2; bagian Pengukur waktu dan ukuran ditambah | Keputusan Arya H4, H5, H6 dan temuan 003b langkah 5 |
| 2026-10-02 | 005 disetujui: ambang #8 menjadi dᴱˣᵢ ≤ 1e-3; #8 per query dirata-rata atas suku tersisa; status kembali siap dikerjakan | 005 titik periksa 1a dan 2a, persetujuan Arya |
| 2026-10-02 | 006 usulan: K7 #9 QPS didahului satu panggilan batch pemanasan yang tidak diukur; H18 dan H19 dicatat sebagai Belum pasti | Keputusan Arya H17 |
| 2026-10-03 | 006 disetujui tanpa koreksi; status kembali siap dikerjakan | Persetujuan Arya |
| 2026-10-03 | 007 usulan: K12 Penjaga test ditambah; K4 dan Run ditambah kolom `config_lock_hash` untuk baris test; K2 dan Gambaran sistem merujuk penjaga; H20 dicatat sebagai Belum pasti dengan perilaku sementara notebook 05 berhenti kalau ada konfigurasi val ganda; K13 ditambah: pyarrow dan pandas dikunci `==` dari `pip freeze` instance yang sama dengan torch dan numpy (H16), sehingga `requirements.txt` bertambah dua baris di luar daftar tiga paket 002a | Keputusan Arya ("Ya, lewat red-chan"; "Terima sementara"; H16 "Ya, dikunci"); temuan pink-chan di 003b langkah 11 |
| 2026-10-03 | 007 disetujui: penjaga diikat ke embedding_id, apa pun Kunci-nya (bukan hanya `content_hash` seperti usulan pink-chan); test hanya bisa dibuka ulang lewat konstanta izin di notebook 06 (bawaan mati) bersama alasan tertulis; Run test ditambah `test_attempt` dan `reopen_reason`; laporan dan D4 memakai percobaan terakhir sambil menyebut jumlah percobaan; K2 "test dibuka sekali" mendapat pengecualian tertulis ini; status kembali siap dikerjakan | 007 titik periksa 1a dan 2a, persetujuan Arya |
