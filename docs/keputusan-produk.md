# Keputusan Produk

Produk: indonesian-retrieval-benchmark
Jenis: riset ML (benchmark algoritma pencarian vektor)
Fase: MVP (satu-satunya fase project)
Data: tingkat 3 — data riset publik lengkap: korpus `miracl/miracl-corpus` id dan query/qrels `miracl/miracl` id dev, dimuat di instance Vast.ai. Bukan data pengganti. Mode data rahasia: tidak aktif.
Status: siap dikerjakan
Diperbarui: 2026-10-03

## Ringkasan
Benchmark yang membandingkan exact search (baseline) dengan tiga algoritma ANN (HNSW, IVF, LSH) untuk retrieval teks berbahasa Indonesia, dikerjakan Arya sebagai portofolio pribadi kemampuan mengimplementasikan retrieval. Semua 1.446.315 passage MIRACL-id (title + text) dan 960 query dev di-embed sekali dalam fp32 dengan `LazarusNLP/congen-indobert-base` di GPU Vast.ai, lalu keempat algoritma FAISS mencari top-5 di CPU atas vektor yang sama dengan sapuan parameter di split val. Per algoritma dipilih konfigurasi tercepat yang mencapai k-NN Recall@5 ≥ 0,95, dikunci, lalu split test dibuka sekali di benchmark final dan dicatat dengan 12 metrik beserta resource mesin.

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
| sentence-transformers 6.1.0 hanya Python 3.10–3.13 | Python wajib 3.10–3.13; versi pasti dijawab saat instance pertama dibuat (H9) |
| Kartu `miracl/miracl-corpus` mencontohkan loading script vs datasets 5.x | K5: data dimuat langsung dari file |
| Penyimpanan Vast.ai tidak permanen vs catatan run yang menumpuk lintas sesi | Vektor, catatan run, dan hasil disalin keluar sebelum instance dihapus; tempat dijawab sebelum instance pertama dihapus (H10) |
| Angka efisiensi hanya sah dalam satu sesi dan satu mesin | Keempat algoritma diukur di satu instance dan satu sesi; notebook final memeriksa env_id |
| env_id memuat hostname dan semua field statis, termasuk revision model dan dataset (H11, H15) | Sapuan val dan benchmark final test wajib di instance yang sama dan dengan revision yang sama; perubahan field statis apa pun menghasilkan env_id baru dan notebook final berhenti |
| Sebagian host Vast.ai masih cgroup v1, sedangkan notebook 02 hanya membaca cgroup v2 dan berhenti di host v1 | K6 dan K8: pembaca jatah CPU dan RAM mendukung v1 dan v2; diterapkan pink-chan di notebook 02 |
| Host hybrid (v1 dan v2 terpasang sekaligus) belum punya urutan deteksi (H18); notebook sementara berhenti dengan error | Tidak menahan langkah. Kalau instance ternyata hybrid, notebook berhenti sebelum mengukur apa pun sehingga tidak ada angka yang salah tercatat; risikonya instance harus diganti atau H18 diputuskan saat itu |
| Pembacaan cgroup hanya ada di Linux vs laptop Arya Windows | Terpisah dari dukungan v1/v2; hanya relevan kalau notebook dijalankan lokal (H12) |
| Tuning parameter bisa membocorkan query test | Sapuan dan pemilihan hanya di val; konfigurasi dikunci (K11) sebelum test dibuka |
| p50 (1 query per panggilan) dan QPS (batch dengan n thread) mengukur hal berbeda: QPS ≠ 1000 / p50. Untuk exact, panggilan 1 query memakai jalur jarak langsung, sedangkan batch ≥ 167 query memakai jalur BLAS [S15] | Diterima sebagai dua sudut pandang (latensi satu permintaan vs throughput); K11 memakai QPS batch. Dicatat terang di laporan hasil |
| #12 dengan `faiss.serialize_index` membuat salinan index di memori (exact ≈ 4,44 GB, HNSW ≈ 4,8 GB) | Puncak RAM naik sebesar ukuran index selama serialisasi; dicatat lewat puncak RAM K8; buffer dilepas segera setelah panjangnya diambil (asumsi) |
| Sapuan "hanya parameter search" vs nbits LSH yang merupakan parameter build | Pilihan 003 titik periksa 1a: pengecualian tertulis, LSH dibangun tiga kali; #11 dan #12 LSH berbeda antar-konfigurasi |
| "IVF dilatih pada seluruh korpus" vs subsampling bawaan FAISS (`max_points_per_centroid` = 256) | Pilihan 003 titik periksa 2a: sampel bawaan 1.048.576 vektor; jumlah sampel dan seed dicatat di Run |
| nlist = 4096 disebut "sekitar 4√N, sesuai panduan FAISS": untuk N = 1.446.315, 4√N ≈ 4.810 (4096 ≈ 3,4·√N), dan untuk N 1M–10M panduan FAISS menyarankan IVF65536 yang butuh ≥ 30·65536 ≈ 1,97 juta vektor latih (lebih dari N) | Keputusan Arya dipertahankan; dicatat apa adanya |
| Run ulang exact (D4) vs "test dibuka sekali" (K2) | Run ulang dilakukan di dalam notebook benchmark final yang sama, tanpa pemilihan apa pun, sehingga tidak membocorkan test (asumsi) |
| Parquet (K9) butuh pyarrow, sedangkan `requirements.txt` hanya tiga paket (002b) | pyarrow ikut terpasang sebagai dependency `datasets`; versinya dicatat K8; penguncian (H16) belum dijadwalkan |
| Seed rotasi IndexLSH "dicatat di Run" (H14), padahal FAISS menulisnya sebagai konstanta `rrot.init(5)` | Nilai yang dicatat adalah konstanta 5 dari kode sumber FAISS 1.15.1, ditandai sebagai nilai dari kode sumber [S14] |
| Notebook 00–02 dan dokumen turunan sudah ditulis dari 003b/004b sebelum 005 dan 006 | Pink-chan menerapkan 005 dan H17 di langkah 6–9 dan 11 rencana 003b serta di notebook 02 (cgroup v1) |

## Asumsi
- Pengguna utama Arya sendiri; keluaran adalah portofolio pribadi, dibaca siapa pun yang melihat portofolio itu.
- Struktur project mengikuti template riset notebook-only (`structure-riset.md`).
- Arya mengunduh dataset; agent tidak mengunduh.
- Relevan berarti relevance ≥ 1 di qrels MIRACL.
- Search mengambil top-5 (k = 5) untuk semua algoritma dan semua konfigurasi sapuan.
- RAM ≥ 32 GB cukup kalau index dibangun dan dilepas satu per satu, termasuk salinan sementara saat serialisasi #12: satu salinan vektor korpus ≈ 4,44 GB; HNSW M = 32 sekitar 1.446.315 × (768·4 + 32·2·4) B ≈ 4,8 GB.
- Exact menghasilkan k-NN Recall@5 = 1,0 dan Relative distance error = 0 menurut definisinya.
- Toleransi pemeriksaan encode 1e-5 dibaca sebagai selisih mutlak maksimum per elemen antara vektor ternormalisasi dari loop sendiri dan dari `model.encode`.
- Run ulang exact pada tanda berhasil dilakukan di split test, di dalam notebook benchmark final yang sama.
- IndexLSH dibuat dengan `rotate_data` = true dan `train_thresholds` = false (bawaan konstruktor Python FAISS).
- Pengukuran p50: 10 query pemanasan dijalankan sekali sebelum putaran pertama, memakai query dari split yang sama; seluruh query split tetap diukur di ketiga putaran.
- Jatah tanpa batas di cgroup v1 (`cpu.cfs_quota_us` = −1, `memory.limit_in_bytes` bernilai sangat besar) diperlakukan sama seperti `max` di cgroup v2: jatah CPU = jumlah CPU `sched_getaffinity`, seperti yang sudah diterapkan notebook 02 untuk v2.
- Pada panggilan 1 query, HNSW, IVF, dan Flat praktis berjalan di satu thread karena FAISS memparalelkan per query (pengetahuan umum), sehingga p50 mencerminkan latensi satu core.
- Galat float32 jalur BLAS untuk vektor bernorma 1 berada di orde 1e-7–1e-6 pada skor L2² (perkiraan dari presisi float32, bukan angka terukur).
- Pemanasan QPS (H17) dijalankan sekali per konfigurasi, tepat sebelum 5 ulangan QPS konfigurasi itu.

## Belum pasti
Diputuskan Arya untuk dijawab saat pengerjaan:

| Kode | Hal | Dijawab paling lambat |
|---|---|---|
| H9, H13 | Versi Python; versi torch dan numpy | Saat instance Vast.ai pertama dibuat (`python --version`, `pip freeze`) |
| H10 | Penyimpanan di luar Vast.ai | Sebelum instance pertama dihapus |
| H12 | Perilaku pencatatan resource di luar Linux | Hanya kalau notebook dijalankan lokal |
| — | Grafik laporan, lisensi model | Saat menyusun laporan |

Belum dijadwalkan (tidak menahan langkah):
- H16 Penguncian versi pyarrow (dan pandas) yang dibutuhkan Parquet.
- H18 Urutan deteksi cgroup di host hybrid (v1 dan v2 terpasang sekaligus). Sementara: notebook berhenti dengan error di host seperti itu.
- H19 Versi cgroup dicatat di Lingkungan atau tidak. Sementara: tidak dicatat (kalau kelak dicatat, ia field statis dan ikut env_id menurut H15).

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
| Catatan run dan lingkungan (K4, K8, K9) | Satu baris per run di CSV yang hanya ditambah; Lingkungan (JSON) per sesi dengan env_id dari semua field statis + hostname | Dibaca Tabel hasil, Pemilih konfigurasi, Benchmark final |
| Tabel hasil (K4) | Menampilkan hasil sapuan keempat algoritma untuk split val | Membaca Run |
| Pemilih konfigurasi (K11) | Per algoritma memilih konfigurasi val menurut aturan K11, lalu menulis kunci konfigurasi (JSON, cap waktu + hash isi) | Membaca Run split val; menulis kunci konfigurasi |
| Benchmark final (K2, D4) | Notebook terakhir: memeriksa hash kunci konfigurasi dan env_id, membuka test sekali, menjalankan keempat konfigurasi terkunci, menjalankan ulang exact, memeriksa tanda berhasil | Membaca kunci konfigurasi, Run, Lingkungan; menulis Run split test |

Seluruh pengukuran di satu instance Vast.ai dalam satu sesi (K6); vektor dan hasil disalin keluar instance sebelum instance dihapus.

## Keputusan
| # | Keputusan | Rancangan | Alasan | Label |
|---|---|---|---|---|
| D1 | Pengguna | Arya sendiri yang menjalankan notebook; keluaran berupa portofolio pribadi tentang kemampuan mengimplementasikan retrieval, dibaca dari atas ke bawah | Jawaban Arya: "hanya portofolio pribadi mengenai kemampuan peng-implementasian retrieval." | — |
| D2 | Cakupan | 001: bagian sistem, aliran, entitas data, pemakaian model, pembagian query. 002: dataset, library, metrik, tempat menjalankan, pencatatan resource, satu fase dengan benchmark final. 003: format berkas, teks dokumen, encode, env_id, parameter dan sapuan, aturan pemilihan, tanda berhasil. 004: seed IVF dan LSH, field env_id. 005: cara ukur QPS dan p50, definisi #12, jarak exact mendekati 0 pada #8, dukungan cgroup v1/v2. 006: pemanasan QPS. Tidak dirancang: hal di Belum pasti | Koreksi dan keputusan Arya | — |
| D3 | Sumber data | Korpus `miracl/miracl-corpus` subset id (`miracl-corpus-v1.0-id`, 3 file `docs-*.jsonl.gz`, 1.446.315 passage dari 446.330 artikel; kolom docid, title, text; Apache-2.0). Query dan qrels `miracl/miracl` folder `miracl-v1.0-id` (`topics/*.tsv`, `qrels/*.tsv` format TREC), split dev: 960 query, 9.668 penilaian, rata-rata 3,22 relevan per query (min 1, maks 13). Qrels test-a/test-b tidak dirilis, tidak dipakai. WebFAQ dan versi hard negatives MTEB ditolak | Keputusan Arya: dataset yang sudah proper [S3, S10] | — |
| D4 | Tanda berhasil | Benchmark berhasil kalau: (1) 12 metrik lengkap untuk keempat algoritma di split test; (2) exact lolos pemeriksaan kewajaran: k-NN Recall@5 = 1 dan Relative distance error = 0; (3) run ulang exact menghasilkan nDCG@5 identik. (Tanda berhasil 001 untuk struktur project sudah tercapai di 001b) | Keputusan Arya (H3) | — |
| K1 | Cara memakai model embedding | `sentence-transformers` 6.1.0 hanya untuk memuat `LazarusNLP/congen-indobert-base` dengan revision (commit hash) dicatat. Encode dengan loop PyTorch sendiri melewati ketiga modul: Transformer (max_seq_length 32) → Pooling mean tokens → Dense 768→768 + Tanh. Tokenisasi max_length 32 untuk dokumen dan query. Teks dokumen = `title + " " + text`; teks query apa adanya. Presisi fp32. Ukuran batch ditetapkan di sel konstanta, dicoba mulai 1024, nilai akhirnya dicatat di Set embedding. Normalisasi L2 sekali dengan `F.normalize`; vektor tersimpan sudah bernorma 1 dan notebook search tidak menormalisasi lagi. Pemeriksaan: 100 teks sampel di-encode dengan loop sendiri dan dengan `model.encode`, selisih ≤ 1e-5. Embedding dibuat sekali di GPU | Keputusan Arya (H1, H7); susunan modul dan 32 token dari model [S1, S11] | Umum |
| K2 | Pembagian query dan benchmark final | 960 query dev dibagi val/test 50:50 acak dengan seed 42; disimpan sebagai atribut split dan dikunci hash isi. Sapuan dan pemilihan hanya di val. Kunci konfigurasi (K11) ditulis sebelum test dibuka; notebook benchmark final berhenti kalau hash kunci atau env_id tidak cocok, lalu membuka test sekali. Korpus tidak dibagi | 001a K2 + 002 titik periksa 1a | Umum |
| K3 | Library index dan algoritma | `faiss-cpu` 1.15.1, CPU. Exact: `IndexFlatL2`. HNSW: `IndexHNSWFlat` METRIC_L2. IVF: `IndexIVFFlat` dengan quantizer `IndexFlatL2`. LSH: `IndexLSH` (proyeksi acak → kode biner → jarak Hamming). Exact dijalankan lebih dulu. Parameter di K10 | Keputusan Arya [S5, S6, S8] | Umum |
| K4 | Catatan run dan hasil | Setiap run menambah satu baris yang tidak pernah ditimpa: identitas run, parameter konfigurasi, 12 metrik K7, kolom diagnostik (jumlah query dengan hasil < 5; jumlah suku #8 yang dikeluarkan karena dᴱˣᵢ ≤ 1e-3; jumlah query yang kelima suku #8-nya dikeluarkan), jumlah thread FAISS dan torch, rujukan Set embedding dan Lingkungan. Satu konfigurasi sapuan = satu run | 001a K4 + keputusan Arya (H6); 005 titik periksa 2a | Umum |
| K5 | Pemuatan dataset | `datasets` 5.0.1, langsung dari file: korpus jsonl.gz dengan builder `json` (atau konversi Parquet Hugging Face); topics dan qrels TSV dengan builder `csv`, pemisah tab. Revision dataset dicatat | Keputusan Arya [S10, S12] | Umum |
| K6 | Tempat menjalankan dan thread | Vast.ai (Linux): RTX 3090 24 GB, RAM ≥ 32 GB, CPU ≥ 24 (jatah efektif vCPU), disk 50 GB. GPU hanya embedding; search di CPU. Thread = min(jatah CPU dari cgroup, 24), dibulatkan ke bawah, diset lewat `faiss.omp_set_num_threads` dan `torch.set_num_threads`, dicatat di setiap run; `os.cpu_count()` tidak dipakai. Jatah CPU dibaca dari cgroup v2 (`cpu.max`) atau cgroup v1 (`cpu.cfs_quota_us` / `cpu.cfs_period_us`), dibatasi `sched_getaffinity`. Satu instance, satu sesi. Vektor dan hasil disalin keluar sebelum instance dihapus | Keputusan Arya; dukungan v1 dari temuan pembangunan 003b langkah 5 (sebagian host Vast.ai masih cgroup v1) | Umum |
| K7 | Metrik evaluasi dan cara ukur | 12 metrik dikunci, k = 5. Rumus, aturan slot −1, aturan jarak exact mendekati 0, dan cara ukur waktu serta ukuran di bawah tabel ini | Keputusan Arya; 002 titik periksa 3a, 4a; H4, H5, H6, H17; 005 titik periksa 1a, 2a | Umum |
| K8 | Pencatatan resource dan env_id | Dicatat otomatis ke Lingkungan: GPU (model, VRAM, driver, CUDA, puncak VRAM); CPU (model, flag AVX2/AVX-512, jatah vCPU dari cgroup v2 `cpu.max` atau v1 `cpu.cfs_quota_us`/`cpu.cfs_period_us`, dan `sched_getaffinity`, total vCPU mesin); RAM (jatah dari cgroup v2 `memory.max` atau v1 `memory.limit_in_bytes`, puncak RAM proses); disk (total, sisa); thread FAISS dan torch; versi Python dan library; revision model dan dataset; timestamp sesi. env_id = hash dari semua field statis — model GPU, VRAM total, driver/CUDA, model CPU, flag AVX2/AVX-512, jatah vCPU, total vCPU mesin, jatah RAM, total disk, jumlah thread FAISS dan torch, versi Python dan library, revision model dan dataset — ditambah hostname instance. Field dinamis (puncak RAM dan VRAM, sisa disk, timestamp) tetap dicatat tetapi tidak membentuk env_id. Dicatat manual oleh Arya di README.md: ID penawaran/host, harga per jam, reliability, lokasi, status verified | Keputusan Arya (H11, H15); dukungan cgroup v1 dari temuan 003b langkah 5 | Umum |
| K9 | Format berkas fisik | Dokumen, Query, Penilaian relevansi, Tetangga exact → Parquet. Catatan run → CSV yang hanya ditambah barisnya. Vektor dokumen dan query → `.npy` float32. Metadata Set embedding, Lingkungan, dan kunci konfigurasi → JSON. Aturan tulis CSV/JSON mengikuti template riset | Keputusan Arya (H8) | Umum |
| K10 | Parameter, sapuan, dan seed | Sapuan di split val. Parameter build tetap dan parameter search disapu, dengan satu pengecualian tertulis untuk LSH. Exact: tanpa parameter, 1 run. HNSW: M = 32, efConstruction = 200 (build); efSearch ∈ {16, 32, 64, 128, 256} → 1 build, 5 run. IVF: nlist = 4096 (build), dilatih dengan sampel acak bawaan FAISS 256 × 4096 = 1.048.576 vektor dari korpus (`max_points_per_centroid` bawaan 256 tidak diubah); seed k-means = bawaan FAISS 1234 (tidak diubah, dibaca dari parameter clustering index dan dicatat di Run bersama jumlah sampel); nprobe ∈ {1, 4, 8, 16, 32, 64, 128} → 1 latih dan build, 7 run. LSH (pengecualian: parameter build disapu karena IndexLSH tidak punya parameter search): nbits ∈ {768, 1536, 3072} → 3 build, 3 run; seed rotasi acak = bawaan FAISS 5 (konstanta di kode sumber, dicatat di Run sebagai nilai dari kode sumber FAISS 1.15.1); #11 dan #12 LSH berbeda per konfigurasi. Total 16 run val | Keputusan Arya (H2, H14); 003 titik periksa 1a, 2a; nlist 4096 ≈ 3,4·√N, lihat Bentrokan [S7, S13, S14] | Umum |
| K11 | Aturan memilih konfigurasi untuk test | Per algoritma, dari run val: konfigurasi dengan QPS tertinggi (QPS batch, K7 #9) di antara yang k-NN Recall@5 ≥ 0,95; kalau tidak ada yang mencapai, konfigurasi dengan k-NN Recall@5 tertinggi. Hasilnya ditulis ke berkas kunci konfigurasi (cap waktu + hash isi) sebelum test dibuka | Keputusan Arya (H3) | Umum |

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
                   (tanpa suku dikeluarkan, |I(q)| = 5 dan rumus sama dengan rumus 002)
                   d      = √max(skor L2 kuadrat dari FAISS, 0)  (skor negatif akibat
                            pembulatan float dipotong ke 0 sebelum diakarkan)
                   LSH    : d = ‖q − xᵢ‖₂ dihitung ulang dari vektor asli untuk ID hasil
                   dᴬᴺᴺ₍ᵢ₎ = jarak ke-i setelah lima jarak ANN diurutkan menaik
                            (HNSW/IVF sudah terurut L2; LSH wajib diurutkan ulang)
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
                   (puncak RAM proses tetap dicatat di Lingkungan sebagai field dinamis)
Pengukuran waktu: time.perf_counter_ns, hanya mencakup index.search; memuat data dan
menghitung metrik tidak termasuk.
Metrik 1–7 per query dirata-rata atas semua query di split yang dijalankan.
```

Aturan hasil kurang dari 5 (ID −1, terutama dari IVF):
- Metrik kualitas (1–6) dan k-NN Recall@5 (7): slot −1 dianggap tidak relevan atau tidak cocok; pembagi tetap 5 (Precision@5, k-NN Recall@5).
- Relative distance error (8): slot −1 diberi penalti jarak 2.
- Jumlah query dengan hasil kurang dari 5 dicatat sebagai kolom diagnostik di setiap run, di luar 12 metrik.
- −1 tidak boleh dipakai sebagai indeks array.

Fakta pembacaan (960 query dev): batas atas rata-rata Precision@5 = 0,567 dan Recall@5 = 0,953 (15,2% query punya lebih dari 5 dokumen relevan). nDCG@5 dan Recall@5 tidak sebanding langsung dengan angka resmi MIRACL (nDCG@10, Recall@100).

**Rumus K11.** Cₐ = konfigurasi algoritma a yang dijalankan di val; R(c) = k-NN Recall@5 di val.

```
c*(a) = argmax { QPS(c) : c ∈ Cₐ, R(c) ≥ 0,95 }    kalau himpunan itu tidak kosong
c*(a) = argmax { R(c)   : c ∈ Cₐ }                  kalau kosong
```

## Desain UI/UX
Tidak berlaku — produk tidak punya antarmuka; hasil dibaca di notebook.

## Model data
Tidak ada basis data; data disimpan sebagai berkas yang menjadi kontrak antarbagian. Format fisik per entitas di kolom Isi (K9).

| Entitas | Isi | Kunci | Relasi | Aturan integritas |
|---|---|---|---|---|
| Dokumen | doc_id (= `docid` MIRACL, skema "X#Y"), title, text — Parquet | doc_id (natural) | 1─N Penilaian relevansi; 1─1 Vektor dokumen per set embedding | doc_id unik; tidak diubah setelah disiapkan |
| Query | query_id, text, split — Parquet | query_id (natural, dari topics MIRACL) | 1─N Penilaian relevansi; 1─1 Vektor query per set embedding; 1─N Tetangga exact | split ∈ {val, test}, 50:50 seed 42, dikunci hash; setiap query punya ≥ 1 dokumen relevan; test hanya dibaca notebook benchmark final setelah hash kunci konfigurasi cocok |
| Penilaian relevansi | query_id, doc_id, relevance (qrels TREC dev) — Parquet | (query_id, doc_id) | N─1 Query; N─1 Dokumen | Relevan kalau relevance ≥ 1; id yang tidak ada dibuang dan jumlahnya dicatat |
| Set embedding | embedding_id, model, revision model, revision dataset, max_seq_length 32, teks dokumen `title + " " + text`, presisi fp32, ukuran batch akhir, normalisasi L2, dim 768, hasil pemeriksaan encode (selisih maksimum pada 100 sampel) — JSON | embedding_id (surrogate: hash konfigurasi) | 1─N Vektor dokumen; 1─N Vektor query; 1─N Tetangga exact; 1─N Run | Konfigurasi berbeda menghasilkan embedding_id baru; vektor lama tidak ditimpa |
| Vektor dokumen | float32[768] per dokumen — `.npy` | (embedding_id, row_idx) | 1─1 Dokumen lewat urutan doc_id yang disimpan bersama | 1.446.315 baris; urutan baris = urutan doc_id tersimpan; norma L2 = 1 |
| Vektor query | float32[768] per query — `.npy` | (embedding_id, row_idx) | 1─1 Query | 960 baris; norma L2 = 1 |
| Tetangga exact | embedding_id, query_id, rank (1–5), doc_id, skor L2 kuadrat dari `IndexFlatL2` — Parquet | (embedding_id, query_id, rank) | N─1 Set embedding; N─1 Query; N─1 Dokumen | Hanya berlaku untuk embedding_id yang sama; skor disimpan apa adanya (pemotongan ke 0 dilakukan saat menghitung #8) |
| Kunci konfigurasi | Per algoritma: konfigurasi terpilih (K11), run_id val asalnya, embedding_id, env_id, cap waktu, hash isi — JSON | hash isi | N─1 Run (val) | Ditulis sekali sebelum test dibuka; notebook final berhenti kalau hash tidak cocok |
| Run | run_id, timestamp, embedding_id, env_id, algorithm, params (build dan search; IVF juga jumlah sampel latih dan seed k-means; LSH juga seed rotasi), split, k = 5, thread FAISS, thread torch, 12 metrik K7, jumlah query dengan hasil < 5, jumlah suku #8 yang dikeluarkan, jumlah query tanpa suku #8 tersisa — baris CSV append-only | run_id (surrogate) | N─1 Set embedding; N─1 Lingkungan | Hanya ditambah; algorithm ∈ {flat, hnsw, ivf, lsh}; split ∈ {val, test}; slot −1 tidak pernah disimpan sebagai doc_id |
| Lingkungan | Isi K8 — JSON | env_id = hash semua field statis K8 + hostname | 1─N Run | Field dinamis dicatat tetapi tidak masuk hash; efisiensi hanya dibandingkan antar-run dengan env_id sama; notebook final memeriksa env_id sama dengan run val |

## Sumber
| # | Sumber | Tingkat | Tanggal | Dipakai untuk |
|---|---|---|---|---|
| S1 | Kartu model, `sentence_bert_config.json`, dan metadata API `huggingface.co/LazarusNLP/congen-indobert-base` | 1 | diubah 2025-02-01 | max_seq_length 32, mean pooling, 768 dimensi, tanpa prefix; lisensi tidak tercantum |
| S3 | Kartu dataset `huggingface.co/datasets/miracl/miracl` (Apache-2.0) | 1 | 2022-10-18 | 960 query dev id, format qrels TREC, label test tidak dirilis |
| S5 | FAISS `INSTALL.md`, `github.com/facebookresearch/faiss` | 1 | versi 1.15.1 | Windows hanya CPU |
| S6 | `pypi.org/project/faiss-cpu` | 1 | 2026-09-16 | Versi 1.15.1, MIT, Python 3.10–3.14 |
| S7 | FAISS wiki "Guidelines to choose an index" | 1 | — (konsep stabil) | Di bawah 1M: K = 4√N–16√N, latih 30·K–256·K; 1M–10M: IVF65536_HNSW32, latih 30·65536–256·65536 |
| S8 | FAISS wiki "Faiss indexes" | 1 | — (konsep stabil) | IndexFlatL2/HNSWFlat/IVFFlat/LSH, cara kerja IndexLSH |
| S10 | Kartu dataset `huggingface.co/datasets/miracl/miracl-corpus` (Apache-2.0) | 1 | v1.0 | 1.446.315 passage id; kolom docid/title/text; jsonl.gz |
| S11 | `pypi.org/project/sentence-transformers` (Apache-2.0) | 1 | 2026-09-18 | Versi 6.1.0, Python 3.10–3.13 |
| S12 | `pypi.org/project/datasets` (Apache-2.0) | 1 | 2026-07-28 | Versi 5.0.1, Python 3.10–3.14 |
| S13 | FAISS `faiss/Clustering.h`, `github.com/facebookresearch/faiss` | 1 | main | `max_points_per_centroid` = 256 (data latih di atasnya disampel), `min_points_per_centroid` = 39, seed bawaan 1234 |
| S14 | FAISS `faiss/IndexLSH.cpp`, `github.com/facebookresearch/faiss` | 1 | main | Konstruktor IndexLSH: rotasi acak `rrot.init(5)` — seed konstanta 5, bukan parameter |
| S15 | FAISS wiki "Implementation notes" | 1 | — (konsep stabil) | IndexFlatL2: jalur langsung kalau nq · d < 128.000, jalur BLAS ‖x‖² + ‖y‖² − 2⟨x, y⟩ selain itu; jalur BLAS kurang stabil secara numerik |

Sumber riset 001 putaran 1 lain (S2, S4, S9) tidak menjadi dasar keputusan. Fakta "datasets 5.x tidak mendukung loading script" dan statistik qrels dev berasal dari diskusi Arya. GitHub issue FAISS tentang jarak L2 negatif (#297, #563) hanya dibaca sebagai konfirmasi masalah yang dikenal (tingkat 3), bukan dasar keputusan.

## Riwayat
| Tanggal | Perubahan | Alasan |
|---|---|---|
| 2026-10-02 | Usulan pertama rancangan MVP | Perintah Arya |
| 2026-10-02 | 001 putaran 2: korpus dan LSH dikeluarkan dari 001; tech stack/library/tools dan kontrak metrik evaluasi ditunda ke 002; max_seq_length 32 dan split val/test 50:50 dipilih | Jawaban dan koreksi Arya |
| 2026-10-02 | 001 disetujui; korpus dan LSH ikut 002 | Persetujuan Arya |
| 2026-10-02 | 002 disetujui: D3 diisi (MIRACL-id korpus penuh + dev); K1 diisi (sentence-transformers hanya memuat, encode PyTorch tiga modul, L2 sekali); K3 diisi (faiss-cpu, METRIC_L2 menggantikan rumusan inner product di 001a K1 — urutan sama untuk vektor bernorma 1; LSH = IndexLSH); K4 diisi kolom metrik; K5–K8 ditambah | Hasil diskusi Arya |
| 2026-10-02 | Fase tunggal: project hanya punya fase MVP; benchmark final di split test masuk MVP sebagai notebook terakhir (mengubah 001a K2 "benchmark final (Dev)"); baris Ditunda→Dev dihapus | Koreksi Arya "Cukup 1 fase saja", 002 titik periksa 1a |
| 2026-10-02 | Teks dokumen yang di-embed ditunda ke pekerjaan berikutnya | 002 titik periksa 2, jawaban Arya |
| 2026-10-02 | 003 usulan: D1 (tujuan portofolio pribadi), D4 (tanda berhasil benchmark), K1 (teks dokumen, fp32, batch, pemeriksaan encode), K8 (pembentuk env_id) diubah; K9 format berkas, K10 parameter dan sapuan, K11 aturan pemilihan ditambah; entitas Kunci konfigurasi ditambah ke Model data; Belum pasti diberi batas waktu | Hasil diskusi Arya (H1, H2, H3, H7, H8, H11) |
| 2026-10-02 | 003 disetujui: sapuan nbits LSH menjadi pengecualian tertulis atas aturan "hanya parameter search" (tiga build); IVF dilatih dengan sampel bawaan FAISS 1.048.576 vektor (bukan seluruh korpus), jumlah sampel dan seed dicatat di Run; status kembali siap dikerjakan | 003 titik periksa 1a dan 2a, persetujuan Arya |
| 2026-10-02 | 004 usulan: K10 seed IVF (bawaan FAISS 1234) dan seed rotasi LSH (bawaan FAISS 5, konstanta kode sumber) dicatat di Run; K8 env_id dibentuk dari semua field statis + hostname (menggantikan daftar field H11); H16 tetap belum dijadwalkan | Keputusan Arya H14, H15 ("Terima usulan pink-chan") |
| 2026-10-02 | 004 disetujui tanpa koreksi; status kembali siap dikerjakan | Persetujuan Arya |
| 2026-10-02 | 005 usulan: K7 diisi cara ukur QPS (batch, 5 ulangan, median) dan p50 (1 query, 10 pemanasan, 3 putaran), definisi #12 (byte `serialize_index`), aturan jarak exact mendekati 0 pada #8 dan pemotongan skor L2² negatif; K4 kolom diagnostik suku #8 yang dikeluarkan; K6 dan K8 cgroup v1 dan v2; bagian Pengukur waktu dan ukuran ditambah ke Gambaran sistem; H4, H5, H6 keluar dari Belum pasti | Keputusan Arya H4, H5, H6 dan temuan 003b langkah 5 ("Lanjutkan") |
| 2026-10-02 | 005 disetujui: ambang #8 menjadi dᴱˣᵢ ≤ 1e-3 (mengganti 1e-6 dari keputusan H6); #8 per query dirata-rata atas suku tersisa (1/|I(q)|), query tanpa suku tersisa tidak ikut rata-rata antarquery dan jumlahnya dicatat di Run; status kembali siap dikerjakan | 005 titik periksa 1a dan 2a, persetujuan Arya |
| 2026-10-02 | 006 usulan: K7 #9 QPS didahului satu panggilan batch pemanasan (semua query split, n thread) yang tidak diukur; rumus QPS tidak berubah. H18 (urutan deteksi cgroup di host hybrid) dan H19 (versi cgroup dicatat atau tidak) dicatat sebagai Belum pasti beserta perilaku sementaranya | Keputusan Arya H17 ("1 batch tanpa diukur"); kode H17–H19 dari pink-chan saat menyusun 005b |
| 2026-10-03 | 006 disetujui tanpa koreksi; status kembali siap dikerjakan | Persetujuan Arya |
