# Keputusan Produk

Produk: indonesian-retrieval-benchmark
Jenis: riset ML (benchmark algoritma pencarian vektor)
Fase: MVP
Data: tingkat 0 — dataset korpus dan query belum dipilih (dirancang di 002). Mode data rahasia: tidak aktif.
Status: siap dikerjakan
Diperbarui: 2026-10-02

## Ringkasan
Benchmark yang membandingkan exact dense search (baseline) dengan tiga algoritma ANN (HNSW, IVF, LSH) untuk retrieval teks berbahasa Indonesia, dijalankan dan dibaca Arya sebagai peneliti. Semua dokumen dan query di-embed sekali dengan `LazarusNLP/congen-indobert-base`, lalu keempat algoritma mencari di atas vektor yang sama. Rancangan 001 menetapkan bagian-bagian sistem, alirannya, dan entitas datanya sebagai dasar struktur project; tech stack, library, tools, kontrak metrik evaluasi, pemilihan korpus, dan implementasi LSH ditetapkan di rancangan 002.

## Titik periksa
1. [Korpus] Korpus mana yang dipakai di MVP?
   a. MIRACL-id hard negatives dari MTEB, sekitar 168 ribu dokumen; korpus penuh dipertimbangkan di Dev (Usulan) — embedding selesai dalam hitungan menit sampai jam di CPU laptop dan muat di RAM, tetapi korpusnya dikumpulkan di sekitar query dan lebih kecil, sehingga selisih kecepatan ANN terhadap exact belum terlihat sebesar di skala jutaan.
   b. Korpus penuh MIRACL-id, sekitar 1,45 juta passage, sejak MVP — skala realistis tempat keunggulan ANN tampak jelas, tetapi embedding memakan berjam-jam di CPU dan vektornya sekitar 4,4 GB.
   Jawaban Arya: "Ini pekerjaan berbeda." ✓ 2026-10-02 — dikeluarkan dari rancangan 001; dirancang di 002 (titik periksa 5).
2. [Panjang tks] Berapa panjang maksimum token saat embedding?
   a. 32 token, bawaan model, untuk dokumen dan query (Usulan) ✓ 2026-10-02 — sesuai cara model dilatih dan paling cepat, tetapi passage terpotong sehingga angka nDCG mutlak rendah; perbandingan antaralgoritma tetap adil.
   b. 128 token untuk dokumen, 32 untuk query — isi passage lebih banyak terbaca, tetapi model tidak dilatih di panjang ini (efeknya belum diketahui) dan embedding sekitar 4 kali lebih lambat.
   c. 512 token untuk dokumen — seluruh passage terbaca, tetapi paling jauh dari kondisi latih dan paling berat.
3. [LSH] Implementasi LSH mana yang dipakai?
   a. FAISS IndexLSH: proyeksi acak ke kode biner, lalu scan jarak Hamming (Usulan) — satu library dengan tiga algoritma lain dan paling sederhana, tetapi scan-nya tetap menyeluruh (bukan tabel hash bucket), jadi percepatannya datang dari kode biner yang ringkas.
   b. LSH bucket multi-tabel (kode biner proyeksi acak + FAISS IndexBinaryMultiHash) — lebih dekat dengan definisi LSH klasik yang sublinear, tetapi parameternya lebih banyak dan perakitannya lebih rumit untuk MVP.
   c. Implementasi sendiri dengan numpy — kendali penuh atas setiap langkah, tetapi kode buatan sendiri berisiko salah dan tidak teroptimasi sehingga perbandingan waktunya tidak setara.
   Jawaban Arya: "Ini pekerjaan berbeda" ✓ 2026-10-02 — dikeluarkan dari rancangan 001; dirancang di 002 (titik periksa 5).
4. [Split query] Bagaimana query evaluasi dipakai?
   a. Dibagi val/test 50:50 dengan seed tetap; MVP hanya memakai val (Usulan) ✓ 2026-10-02 — test tetap bersih untuk benchmark final di Dev, tetapi angka MVP hanya dari separuh query.
   b. Semua query dipakai di MVP tanpa pembagian — angka MVP lebih stabil, tetapi pemilihan parameter di Dev dan laporan akhir memakai query yang sama sehingga hasil akhir bias.
5. [Ditunda ke] Pemilihan korpus dan implementasi LSH dirancang di mana?
   a. Ikut rancangan 002 bersama tech stack dan kontrak metrik (Usulan) ✓ 2026-10-02 — satu putaran memutuskan semua yang saling bergantung (implementasi LSH bergantung pada library, parameter IVF bergantung pada ukuran korpus), tetapi putaran 002 menjadi lebih besar.
   b. Nomor rancangan sendiri setelah 002 — tiap pekerjaan kecil dan terpisah, tetapi keputusan library di 002 bisa perlu diubah lagi saat LSH atau korpus diputuskan.

## Bentrokan
| Bentrokan | Cara rancangan menghindarinya |
|---|---|
| Panjang maksimum 32 token vs passage yang biasanya lebih panjang | Passage terpotong; diterima karena pertanyaan riset membandingkan algoritma pada vektor yang sama, bukan kualitas model. Panjang token dicatat di set embedding |
| Lisensi model tidak tercantum di kartu model vs aturan lisensi | Model dikunci Arya, tidak diganti. Dicatat di Belum pasti; perlu dicek sebelum hasil dipublikasikan |
| Struktur dibangun sebelum dataset, library, dan kontrak metrik diputuskan | Struktur hanya menyediakan tempat untuk bagian dan entitas di dokumen ini; tidak ada berkas yang mengandaikan format dataset, nama library, atau kolom metrik tertentu |
| Bagian ANN membutuhkan tetangga exact sebagai pembanding | Exact search ditetapkan sebagai bagian yang dijalankan lebih dulu; keluarannya menjadi masukan Penilai |
| Tuning parameter di Dev bisa membocorkan query test | Query dibagi val/test sejak awal dan dikunci; MVP dan tuning hanya memakai val |

## Asumsi
- Pengguna utama Arya sendiri sebagai peneliti; pembaca hasil adalah penguji skripsi atau reviewer.
- Struktur project mengikuti template riset notebook-only (`structure-riset.md`), sesuai CLAUDE.md project.
- Dataset yang nanti dipilih menyediakan korpus, query, dan penilaian relevansi (qrels), dan diunduh Arya; agent tidak mengunduh.
- Rancangan 002 tidak mengubah bagian sistem dan entitas di dokumen ini, hanya mengisi library, kolom metrik, dataset, implementasi LSH, dan parameter.

## Belum pasti
- Spesifikasi mesin: model CPU, jumlah core, RAM, ada atau tidaknya GPU.
- Lisensi `LazarusNLP/congen-indobert-base`: tidak tercantum di kartu model dan metadata Hugging Face (repo kodenya Apache-2.0).
- Tujuan keluaran: skripsi, paper, atau laporan internal.

## Ditunda
| Topik | Ditunda sampai |
|---|---|
| Tech stack, library, dan tools (index, embedding, pembaca data, versi Python) | Rancangan 002, menunggu instruksi Arya |
| Kontrak metrik evaluasi (metrik, rumus, cara mengukur waktu, kolom berkas run, tanda berhasil benchmark) | Rancangan 002, menunggu instruksi Arya |
| Pemilihan dataset korpus dan query | Rancangan 002 |
| Implementasi LSH | Rancangan 002 |
| Parameter awal tiap algoritma dan nilai k | Rancangan 002 |
| Hardware dan aturan pengukuran efisiensi | Rancangan 002 |
| Sapuan parameter, benchmark final di split test, grafik laporan | Dev |

---
<!-- Bagian teknis — dibaca pink-chan -->

## Gambaran sistem
| Bagian | Tugasnya | Terhubung ke |
|---|---|---|
| Penyiapan data (K2) | Membaca korpus, query, dan qrels mentah; memeriksa kunci dan relasi; membagi query val/test sekali dan mengunci pembagiannya dengan hash | Menulis Dokumen, Query, Penilaian relevansi; dibaca Pembuat embedding dan Penilai |
| Pembuat embedding (K1) | Meng-embed semua dokumen dan query sekali dengan model terkunci, menyimpan vektor beserta metadata set embedding | Membaca Dokumen dan Query; menulis Vektor dokumen, Vektor query, Set embedding |
| Exact search (K3) | Pencarian brute-force: baseline, sekaligus sumber tetangga exact untuk menilai ANN. Dijalankan sebelum bagian ANN | Membaca vektor; menulis Tetangga exact dan hasil ke Penilai |
| HNSW (K3) | Membangun index HNSW dari vektor yang sama dan mencari | Membaca vektor; hasil ke Penilai |
| IVF (K3) | Membangun index IVF dari vektor yang sama dan mencari | Membaca vektor; hasil ke Penilai |
| LSH (K3) | Membangun index LSH dari vektor yang sama dan mencari; implementasinya ditetapkan di 002 | Membaca vektor; hasil ke Penilai |
| Penilai (002) | Menghitung metrik kualitas, kemiripan terhadap exact, dan efisiensi; isi kontraknya di rancangan 002 | Membaca Penilaian relevansi dan Tetangga exact; menulis Run |
| Catatan run dan lingkungan (K4) | Satu catatan per run, hanya ditambah; lingkungan dicatat per sesi | Dibaca Tabel hasil |
| Tabel hasil (K4) | Menampilkan hasil keempat algoritma berdampingan untuk split val | Membaca Run |

## Keputusan
| # | Keputusan | Rancangan | Alasan | Label |
|---|---|---|---|---|
| D1 | Pengguna | Arya sebagai peneliti yang menjalankan notebook; pembaca hasil penguji atau reviewer yang membaca notebook dari atas ke bawah | Dari jenis project riset dan template notebook-only | — |
| D2 | Cakupan | Rancangan 001: bagian sistem, aliran antarbagian, entitas data, cara memakai model, dan pembagian query — cukup untuk membangun struktur project tanpa kode. Sengaja tidak di 001: dataset, tech stack/library/tools, kontrak metrik, parameter, implementasi LSH (semuanya di 002) | Koreksi Arya putaran 2 | — |
| D3 | Sumber data | Belum dipilih; dirancang di 002. Syarat dari rancangan ini: teks bahasa Indonesia dengan korpus, query, dan penilaian relevansi | Jawaban Arya titik periksa 1 dan 5 | — |
| D4 | Tanda berhasil (001) | Struktur project menyediakan tempat untuk setiap bagian di Gambaran sistem dan setiap entitas di Model data, tanpa kode, dan tidak mengandaikan dataset, library, atau kolom metrik tertentu. Tanda berhasil benchmark ditetapkan di 002 | Tujuan 001 dari Arya: dasar struktur project | — |
| K1 | Cara memakai model embedding | Model `LazarusNLP/congen-indobert-base` dengan revision (commit hash) dicatat; max_seq_length = 32 untuk dokumen dan query; vektor dinormalisasi L2 sehingga inner product = cosine; embedding dibuat sekali, dan keempat algoritma membaca vektor yang sama. Library pemanggil ditetapkan di 002 | Bawaan model 32 token, mean pooling, tanpa prefix query/dokumen [S1]; pilihan Arya titik periksa 2 | Umum |
| K2 | Pembagian query | Query evaluasi dibagi val/test 50:50 secara acak dengan seed 42; disimpan sebagai atribut split dan dikunci dengan hash isi. MVP dan tuning hanya memakai val; test hanya dibuka di benchmark final (Dev). Korpus tidak dibagi | Aturan template riset; pilihan Arya titik periksa 4 | Umum |
| K3 | Bagian algoritma | Exact, HNSW, IVF, dan LSH adalah empat bagian setingkat yang masing-masing membaca vektor yang sama dan menyerahkan hasil ke Penilai; exact dijalankan lebih dulu karena keluarannya menjadi pembanding ANN | Perbedaan hasil hanya boleh berasal dari algoritma (CLAUDE.md project) | Umum |
| K4 | Catatan run dan hasil | Setiap run menambah satu catatan yang tidak pernah ditimpa, merujuk set embedding dan lingkungan; tabel hasil disusun dari catatan run. Kolom metrik ditetapkan di 002 | Aturan template riset: angka disimpan, bukan diingat; angka efisiensi hanya sah pada lingkungan yang sama | Umum |

## Desain UI/UX
Tidak berlaku — produk tidak punya antarmuka; hasil dibaca di notebook.

## Model data
Tidak ada basis data; data disimpan sebagai berkas yang menjadi kontrak antarbagian. Rancangan 001 menetapkan tingkat conceptual dan kunci logis. Kolom rinci, format berkas, dan kolom metrik ditetapkan di 002.

| Entitas | Isi | Kunci | Relasi | Aturan integritas |
|---|---|---|---|---|
| Dokumen | Satu dokumen korpus | doc_id (natural, dari dataset) | 1─N Penilaian relevansi; 1─1 Vektor dokumen per set embedding | doc_id unik; tidak diubah setelah disiapkan |
| Query | Satu query evaluasi beserta split | query_id (natural, dari dataset) | 1─N Penilaian relevansi; 1─1 Vektor query per set embedding; 1─N Tetangga exact | split ∈ {val, test}, ditetapkan sekali (K2) dan dikunci hash; setiap query punya ≥ 1 dokumen relevan |
| Penilaian relevansi | Relevansi pasangan query–dokumen | (query_id, doc_id) | N─1 Query; N─1 Dokumen | query_id dan doc_id wajib ada; yang tidak ada dibuang dan jumlahnya dicatat |
| Set embedding | Konfigurasi dan metadata satu kali embedding (model, revision, max_seq_length, normalisasi) | embedding_id (surrogate: hash konfigurasi) | 1─N Vektor dokumen; 1─N Vektor query; 1─N Tetangga exact; 1─N Run | Konfigurasi berbeda menghasilkan embedding_id baru; vektor lama tidak ditimpa |
| Vektor dokumen | Vektor tiap dokumen | (embedding_id, row_idx) | N─1 Set embedding; 1─1 Dokumen lewat urutan doc_id yang disimpan bersama | Jumlah baris = jumlah dokumen; urutan baris = urutan doc_id tersimpan; norma L2 = 1 |
| Vektor query | Vektor tiap query | (embedding_id, row_idx) | N─1 Set embedding; 1─1 Query | Aturan sama dengan Vektor dokumen |
| Tetangga exact | Peringkat dokumen hasil exact search per query | (embedding_id, query_id, rank) | N─1 Set embedding; N─1 Query; N─1 Dokumen | Hanya berlaku untuk embedding_id yang sama |
| Run | Satu kali menjalankan satu algoritma dengan satu konfigurasi di satu split | run_id (surrogate) | N─1 Set embedding; N─1 Lingkungan | Hanya ditambah; algoritma ∈ {flat, hnsw, ivf, lsh}; split MVP = val |
| Lingkungan | Sistem operasi, versi Python dan library, CPU, RAM, GPU | env_id (surrogate: hash isi) | 1─N Run | Dicatat setiap sesi |

## Sumber
| # | Sumber | Tingkat | Tanggal | Dipakai untuk |
|---|---|---|---|---|
| S1 | Kartu model, `sentence_bert_config.json`, dan metadata API `huggingface.co/LazarusNLP/congen-indobert-base` | 1 | diubah 2025-02-01 | max_seq_length 32, mean pooling, 768 dimensi, tanpa prefix; lisensi tidak tercantum |

Sumber riset putaran 1 yang tidak lagi menjadi dasar keputusan 001 (disimpan untuk 002): kartu dataset `miracl/miracl` dan `mteb/MIRACLRetrievalHardNegatives`; FAISS `INSTALL.md`, `pypi.org/project/faiss-cpu`, wiki "Guidelines to choose an index" dan "Faiss indexes"; ANN-Benchmarks (Information Systems 87, 2020); repo `LazarusNLP/indonesian-sentence-embeddings`.

## Riwayat
| Tanggal | Perubahan | Alasan |
|---|---|---|
| 2026-10-02 | Usulan pertama rancangan MVP | Perintah Arya |
| 2026-10-02 | Putaran 2: korpus dan LSH dikeluarkan dari 001; tech stack/library/tools dan kontrak metrik evaluasi ditunda ke 002; max_seq_length 32 dan split val/test 50:50 dipilih | Jawaban dan koreksi Arya |
| 2026-10-02 | Disetujui; korpus dan implementasi LSH ikut rancangan 002; status menjadi siap dikerjakan | Persetujuan Arya |
