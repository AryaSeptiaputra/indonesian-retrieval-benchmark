# Keputusan Produk

Produk: indonesian-retrieval-benchmark
Jenis: riset ML (benchmark algoritma pencarian vektor)
Fase: MVP (satu-satunya fase project)
Data: tingkat 3 — data riset publik lengkap: korpus `miracl/miracl-corpus` id dan query/qrels `miracl/miracl` id dev, dimuat di instance Vast.ai. Bukan data pengganti. Mode data rahasia: tidak aktif.
Status: siap dikerjakan
Diperbarui: 2026-10-02

## Ringkasan
Benchmark yang membandingkan exact search (baseline) dengan tiga algoritma ANN (HNSW, IVF, LSH) untuk retrieval teks berbahasa Indonesia, dijalankan dan dibaca Arya sebagai peneliti. Semua 1.446.315 passage MIRACL-id dan 960 query dev di-embed sekali dengan `LazarusNLP/congen-indobert-base` di GPU Vast.ai, lalu keempat algoritma FAISS mencari top-5 di CPU atas vektor yang sama. Setiap run dicatat dengan 12 metrik (kualitas, kesetiaan terhadap exact, efisiensi) beserta resource mesin; konfigurasi dikunci dari split val sebelum split test dibuka sekali di benchmark final.

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
   Jawaban Arya: "Simpan keputusan ini untuk diputuskan dipekerjaan selanjutnya" ✓ 2026-10-02
3. [RDE LSH] Jarak L2 lima hasil LSH dibandingkan per peringkat dalam urutan apa?
   a. Diurutkan ulang menaik berdasarkan jarak L2 sebelum dibandingkan (Usulan) ✓ 2026-10-02 — galat tiap peringkat selalu ≥ 0 dan setara dengan HNSW dan IVF yang hasilnya sudah terurut L2, tetapi #8 tidak lagi mencerminkan urutan Hamming LSH (urutan itu tetap dinilai metrik kualitas).
   b. Urutan Hamming asli dari LSH — mencerminkan urutan yang dikembalikan LSH, tetapi galat per peringkat bisa negatif sehingga rata-ratanya bisa saling menutupi dan tidak setara dengan tiga algoritma lain.
4. [MAP@5] Pembagi Z pada AP@5 per query?
   a. min(|Rel(q)|, 5) (Usulan) ✓ 2026-10-02 — setiap query bisa mencapai 1,0 sehingga MAP@5 sebanding antarquery, tetapi relevan di luar top-5 tidak menurunkan nilai (sudah diukur Recall@5).
   b. |Rel(q)| — sama dengan AP penuh yang dipotong di 5, tetapi 15,2% query dengan lebih dari 5 relevan tidak bisa mencapai 1,0.
   c. Jumlah relevan yang ditemukan di top-5 — hanya menilai urutan hasil yang benar, tetapi query yang hanya menemukan satu relevan di peringkat 1 sudah mendapat 1,0.

## Bentrokan
| Bentrokan | Cara rancangan menghindarinya |
|---|---|
| Panjang maksimum 32 token vs passage yang lebih panjang | Passage terpotong; diterima karena yang dibandingkan algoritma pada vektor yang sama, bukan kualitas model. Panjang token dicatat di Set embedding |
| Lisensi model tidak tercantum di kartu model | Model dikunci Arya, tidak diganti. Dicatat di Belum pasti; dicek sebelum hasil dipublikasikan |
| Satu fase vs 001a K2 ("benchmark final (Dev)") dan template riset fase MVP ("belum perlu benchmark final") | Pilihan 002 titik periksa 1a: notebook benchmark final masuk fase MVP sebagai notebook terakhir; 001a tidak diubah, perubahan dicatat di Riwayat |
| 001a K1 menyebut inner product = cosine vs METRIC_L2 di 002 | Untuk vektor bernorma 1, ‖q − x‖² = 2 − 2·q·x, jadi urutan L2 = urutan cosine; hasil tidak berubah |
| Penalti jarak 2 untuk slot −1 hanya sah untuk vektor bernorma 1 | Aturan integritas Vektor: norma 1 wajib; normalisasi sekali di encode |
| LSH terurut Hamming, bukan L2, saat menghitung #8 | Pilihan 002 titik periksa 3a: lima jarak L2 hasil ANN diurutkan menaik sebelum dibandingkan |
| #8 membagi dengan nol kalau jarak exact = 0 | Belum pasti |
| Teks dokumen belum diputuskan, padahal embedding membutuhkannya | Struktur dan bagian lain bisa disiapkan; embedding baru dijalankan setelah keputusan di pekerjaan berikutnya |
| 001a D4 menjanjikan tanda berhasil benchmark di 002, tetapi diskusi tidak membahasnya | Belum pasti |
| sentence-transformers 6.1.0 hanya Python 3.10–3.13, faiss-cpu dan datasets sampai 3.14 | Python wajib 3.10–3.13; versi pasti Belum pasti |
| Kartu `miracl/miracl-corpus` mencontohkan loading script vs datasets 5.x tanpa dukungan script | K5: data dimuat langsung dari file |
| Penyimpanan Vast.ai tidak permanen vs catatan run yang menumpuk lintas sesi | Vektor, catatan run, dan hasil disalin keluar sebelum instance dihapus dan dibawa ke instance berikutnya; tempat penyimpanan Belum pasti |
| Angka efisiensi hanya sah dalam satu sesi dan satu mesin | Keempat algoritma diukur di satu instance dan satu sesi; benchmark final test sebaiknya di sesi yang sama dengan val; notebook final memeriksa Lingkungan |
| Pembacaan cgroup hanya ada di Linux vs laptop Arya Windows | Perilaku kode di luar Linux Belum pasti |
| Tuning parameter bisa membocorkan query test | Split val/test dikunci; test hanya dibuka di notebook benchmark final setelah hash konfigurasi cocok |
| CLAUDE.md project masih menulis dataset, metrik, library, parameter, hardware, keluaran sebagai "Belum diputuskan" | Red-chan tidak menulis CLAUDE.md; Arya perlu memperbaruinya |

## Asumsi
- Pengguna utama Arya sendiri sebagai peneliti; pembaca hasil adalah penguji skripsi atau reviewer.
- Struktur project mengikuti template riset notebook-only (`structure-riset.md`).
- Arya mengunduh dataset; agent tidak mengunduh.
- Relevan berarti relevance ≥ 1 di qrels MIRACL (sesuai angka rata-rata 3,22 relevan per query).
- Search mengambil top-5 (k = 5) untuk semua algoritma; Tetangga exact berisi peringkat 1–5.
- RAM ≥ 32 GB cukup kalau index dibangun dan dilepas satu per satu: satu salinan vektor korpus = 1.446.315 × 768 × 4 B ≈ 4,44 GB, dan Flat, HNSW, IVF masing-masing menyimpan salinan penuh. Puncak RAM dicatat (K8).
- Exact menghasilkan k-NN Recall@5 = 1,0 dan Relative distance error = 0 menurut definisinya.

## Belum pasti
- Teks dokumen yang di-embed (title + text atau text saja) — diputuskan di pekerjaan berikutnya.
- Parameter tiap algoritma: HNSW M, efConstruction, efSearch; IVF nlist, nprobe, data latih; LSH nbits; ada atau tidaknya sapuan parameter di val.
- Tanda berhasil benchmark (dijanjikan 001a D4).
- Cara mengukur QPS dan latensi p50: query satu per satu atau batch, pemanasan, alat ukur waktu.
- Cara mengukur ukuran index / memori (#12): byte hasil serialisasi index atau memori proses.
- Penanganan jarak exact = 0 pada Relative distance error.
- Encode: presisi (fp32 atau fp16), ukuran batch, dan pemeriksaan kesamaan hasil loop encode sendiri dengan `model.encode`.
- Format berkas fisik tabel, vektor, dan metadata.
- Versi Python pasti (harus 3.10–3.13).
- Tempat penyimpanan di luar Vast.ai.
- Perilaku kode pencatatan resource di luar Linux.
- Grafik laporan.
- Lisensi `LazarusNLP/congen-indobert-base` (tidak tercantum di kartu model; repo kodenya Apache-2.0).
- Tujuan keluaran: skripsi, paper, atau laporan internal.

## Ditunda
| Topik | Ditunda sampai |
|---|---|
| Teks dokumen yang di-embed | Pekerjaan berikutnya (jawaban Arya, 002 titik periksa 2) |
| Hal-hal di Belum pasti | Diputuskan Arya; project tidak punya fase Dev |

---
<!-- Bagian teknis — dibaca pink-chan -->

## Gambaran sistem
| Bagian | Tugasnya | Terhubung ke |
|---|---|---|
| Penyiapan data (K2, K5) | Memuat korpus jsonl.gz dan topics/qrels TSV dev dari file; memeriksa kunci dan relasi; membagi query val/test sekali dan mengunci pembagiannya dengan hash; mencatat revision dataset | Menulis Dokumen, Query, Penilaian relevansi; dibaca Pembuat embedding dan Penilai |
| Pembuat embedding (K1) | Di GPU: tokenisasi max_length 32, Transformer → Pooling mean → Dense 768→768 + Tanh → normalisasi L2, untuk semua dokumen dan query sekali | Membaca Dokumen dan Query; menulis Vektor dokumen, Vektor query, Set embedding |
| Exact search (K3) | `IndexFlatL2` top-5 di CPU: baseline sekaligus Tetangga exact untuk menilai ANN. Dijalankan sebelum bagian ANN | Membaca vektor; menulis Tetangga exact dan hasil ke Penilai |
| HNSW (K3) | `IndexHNSWFlat` (METRIC_L2), bangun dan cari top-5 | Membaca vektor; hasil ke Penilai |
| IVF (K3) | `IndexIVFFlat` dengan quantizer `IndexFlatL2`, latih, bangun, cari top-5 | Membaca vektor; hasil ke Penilai |
| LSH (K3) | `IndexLSH` (jarak Hamming), bangun dan cari top-5; jarak L2 hasil dihitung ulang dari vektor asli untuk #8 | Membaca vektor; hasil ke Penilai |
| Penilai (K7) | Menghitung 12 metrik dan kolom diagnostik; menangani slot −1 | Membaca Penilaian relevansi, Tetangga exact, vektor (untuk LSH); menulis Run |
| Catatan run dan lingkungan (K4, K8) | Satu catatan per run, hanya ditambah; resource dicatat otomatis per sesi | Dibaca Tabel hasil dan Benchmark final |
| Tabel hasil (K4) | Menampilkan hasil keempat algoritma berdampingan untuk split val | Membaca Run |
| Benchmark final (K2) | Notebook terakhir: memeriksa hash konfigurasi yang dikunci dari val dan kesamaan Lingkungan, lalu membuka split test sekali | Membaca konfigurasi terkunci, Run, Lingkungan; menulis Run split test |

Seluruh pengukuran di satu instance Vast.ai dalam satu sesi (K6); vektor dan hasil disalin keluar instance sebelum instance dihapus.

## Keputusan
| # | Keputusan | Rancangan | Alasan | Label |
|---|---|---|---|---|
| D1 | Pengguna | Arya sebagai peneliti yang menjalankan notebook; pembaca hasil penguji atau reviewer yang membaca notebook dari atas ke bawah | Dari jenis project riset dan template notebook-only | — |
| D2 | Cakupan | 001: bagian sistem, aliran, entitas data, pemakaian model, pembagian query. 002: dataset, library, metrik, tempat menjalankan, pencatatan resource, satu fase dengan benchmark final. Tidak dirancang: hal di Belum pasti | Koreksi dan keputusan Arya | — |
| D3 | Sumber data | Korpus `miracl/miracl-corpus` subset id (`miracl-corpus-v1.0-id`, 3 file `docs-*.jsonl.gz`, 1.446.315 passage dari 446.330 artikel; kolom docid, title, text; Apache-2.0). Query dan qrels `miracl/miracl` folder `miracl-v1.0-id` (`topics/*.tsv`, `qrels/*.tsv` format TREC), split dev: 960 query, 9.668 penilaian, rata-rata 3,22 relevan per query (min 1, maks 13). Qrels test-a/test-b tidak dirilis, tidak dipakai. WebFAQ dan versi hard negatives MTEB ditolak | Keputusan Arya: dataset yang sudah proper — korpus, query, qrels terpisah dan dinilai manusia [S3, S10] | — |
| D4 | Tanda berhasil | 001: struktur project menyediakan tempat untuk setiap bagian dan entitas tanpa kode. Tanda berhasil benchmark belum diputuskan (Belum pasti) | Tidak dibahas di diskusi 002 | — |
| K1 | Cara memakai model embedding | `sentence-transformers` 6.1.0 hanya untuk memuat `LazarusNLP/congen-indobert-base` dengan revision (commit hash) dicatat. Encode dengan loop PyTorch sendiri melewati ketiga modul model: Transformer (max_seq_length 32) → Pooling mean tokens → Dense 768→768 + Tanh. Tokenisasi wajib max_length 32 untuk dokumen dan query. Normalisasi L2 sekali saat encode dengan `F.normalize`; vektor tersimpan sudah bernorma 1 dan notebook search tidak menormalisasi lagi. Embedding dibuat sekali di GPU; keempat algoritma membaca vektor yang sama. Teks dokumen ditunda | Keputusan Arya; susunan modul dan 32 token dari model [S1, S11] | Umum |
| K2 | Pembagian query dan benchmark final | 960 query dev dibagi val/test 50:50 acak dengan seed 42; disimpan sebagai atribut split dan dikunci dengan hash isi. Pemilihan konfigurasi hanya di val. Konfigurasi dikunci ke berkas (cap waktu dan hash isi); notebook benchmark final — notebook terakhir di fase MVP — berhenti kalau hash atau Lingkungan tidak cocok, lalu membuka test sekali. Korpus tidak dibagi | 001a K2 + pilihan Arya 002 titik periksa 1a; aturan template riset | Umum |
| K3 | Library index dan algoritma | `faiss-cpu` 1.15.1 untuk keempat algoritma, berjalan di CPU. Exact: `IndexFlatL2`. HNSW: `IndexHNSWFlat` dengan METRIC_L2. IVF: `IndexIVFFlat` dengan quantizer `IndexFlatL2`. LSH: `IndexLSH` (proyeksi acak → kode biner → jarak Hamming). Exact dijalankan lebih dulu sebagai pembanding ANN. Parameter Belum pasti | Keputusan Arya; satu library untuk keempat algoritma [S5, S6, S8] | Umum |
| K4 | Catatan run dan hasil | Setiap run menambah satu catatan yang tidak pernah ditimpa, berisi identitas run, 12 metrik K7, kolom diagnostik jumlah query dengan hasil < 5, jumlah thread FAISS dan torch, serta rujukan Set embedding dan Lingkungan. Tabel hasil val dan hasil benchmark final disusun dari catatan run | 001a K4 + keputusan Arya | Umum |
| K5 | Pemuatan dataset | `datasets` 5.0.1 (Apache-2.0). Kedua repo MIRACL memakai loading script (`miracl.py`) yang tidak didukung datasets 5.x, jadi data dimuat langsung dari file: korpus jsonl.gz dengan builder `json` (atau konversi Parquet Hugging Face); topics dan qrels TSV dengan builder `csv`, pemisah tab. Revision dataset dicatat | Keputusan Arya [S10, S12] | Umum |
| K6 | Tempat menjalankan dan thread | Vast.ai (Linux). Spesifikasi dikunci: GPU RTX 3090 24 GB, RAM ≥ 32 GB, CPU ≥ 24 (jatah efektif vCPU = angka pertama di "x/y CPU" penawaran Vast.ai), disk 50 GB. GPU hanya untuk embedding; search di CPU. Thread = min(jatah CPU dari cgroup, 24), dibulatkan ke bawah, diset eksplisit lewat `faiss.omp_set_num_threads` dan `torch.set_num_threads`, dicatat di setiap run; `os.cpu_count()` tidak dipakai (melaporkan total mesin). Keempat algoritma diukur di satu instance dan satu sesi. Vektor dan hasil disalin keluar sebelum instance dihapus | Keputusan Arya: hardware lokal tidak cukup (VRAM RTX 3050 4 GB, RAM kosong sekitar 2 GB) | Umum |
| K7 | Metrik evaluasi | 12 metrik dikunci, k = 5. Kualitas vs qrels: (1) nDCG@5, (2) Recall@5, (3) MRR@5, (4) Precision@5, (5) MAP@5, (6) Hit rate@5. Kesetiaan ANN vs exact: (7) k-NN Recall@5, (8) Relative distance error. Efisiensi: (9) QPS, (10) Latensi p50, (11) Waktu build/train index, (12) Ukuran index / memori. Rumus dan aturan slot −1 di bawah tabel ini | Keputusan Arya; pilihan Arya 002 titik periksa 3a dan 4a | Umum |
| K8 | Pencatatan resource | Dicatat otomatis oleh kode ke Lingkungan: GPU (model, VRAM, driver, versi CUDA, puncak VRAM terpakai); CPU (model, flag AVX2/AVX-512, jatah vCPU dari cgroup `cpu.max` dan `sched_getaffinity`, total vCPU mesin); RAM (jatah dari cgroup `memory.max`, puncak RAM terpakai); disk (total dan sisa); jumlah thread FAISS dan torch; versi Python dan library; revision model dan dataset; timestamp sesi. Dicatat manual oleh Arya di README.md: data dashboard Vast.ai (ID penawaran/host, harga per jam, reliability, lokasi, status verified) | Keputusan Arya | Umum |

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
8  Relative distance error = (1/5) · Σᵢ₌₁⁵ (dᴬᴺᴺ₍ᵢ₎ − dᴱˣᵢ) / dᴱˣᵢ
                   d      = √(skor L2 kuadrat dari FAISS)  (jarak L2 biasa)
                   LSH    : d = ‖q − xᵢ‖₂ dihitung ulang dari vektor asli untuk ID hasil
                   dᴬᴺᴺ₍ᵢ₎ = jarak ke-i setelah lima jarak ANN diurutkan menaik
                            (HNSW/IVF sudah terurut L2; LSH wajib diurutkan ulang)
                   dᴱˣᵢ   = jarak ke-i hasil IndexFlatL2
                   slot −1: dᴬᴺᴺ = 2 (jarak L2 maksimum dua vektor bernorma 1)
9  QPS           = jumlah query diukur / total waktu cari (detik)
10 Latensi p50   = median waktu cari per query (ms)
11 Waktu build/train index (detik)
12 Ukuran index / memori
Semua metrik per query dirata-rata atas query di split yang dijalankan.
```

Aturan hasil kurang dari 5 (ID −1, terutama dari IVF):
- Metrik kualitas (1–6) dan k-NN Recall@5 (7): slot −1 dianggap tidak relevan atau tidak cocok; pembagi tetap 5 (Precision@5, k-NN Recall@5).
- Relative distance error (8): slot −1 diberi penalti jarak 2.
- Jumlah query dengan hasil kurang dari 5 dicatat sebagai kolom diagnostik di setiap run, di luar 12 metrik.
- −1 tidak boleh dipakai sebagai indeks array.

Fakta pembacaan (960 query dev): batas atas rata-rata Precision@5 = 0,567 dan Recall@5 = 0,953 (15,2% query punya lebih dari 5 dokumen relevan). nDCG@5 dan Recall@5 tidak sebanding langsung dengan angka resmi MIRACL (nDCG@10, Recall@100).

## Desain UI/UX
Tidak berlaku — produk tidak punya antarmuka; hasil dibaca di notebook.

## Model data
Tidak ada basis data; data disimpan sebagai berkas yang menjadi kontrak antarbagian. Format berkas fisik Belum pasti.

| Entitas | Isi | Kunci | Relasi | Aturan integritas |
|---|---|---|---|---|
| Dokumen | doc_id (= `docid` MIRACL, skema "X#Y"), title, text | doc_id (natural) | 1─N Penilaian relevansi; 1─1 Vektor dokumen per set embedding | doc_id unik; tidak diubah setelah disiapkan |
| Query | query_id, text, split | query_id (natural, dari topics MIRACL) | 1─N Penilaian relevansi; 1─1 Vektor query per set embedding; 1─N Tetangga exact | split ∈ {val, test}, 50:50 seed 42, dikunci hash; setiap query punya ≥ 1 dokumen relevan; test hanya dibaca notebook benchmark final setelah hash konfigurasi cocok |
| Penilaian relevansi | query_id, doc_id, relevance (qrels TREC dev) | (query_id, doc_id) | N─1 Query; N─1 Dokumen | Relevan kalau relevance ≥ 1, nilai 0 tidak relevan; query_id dan doc_id wajib ada (yang tidak ada dibuang dan jumlahnya dicatat) |
| Set embedding | embedding_id, model, revision model, revision dataset, max_seq_length 32, normalisasi L2 di PyTorch, dim 768, teks dokumen (menunggu keputusan) | embedding_id (surrogate: hash konfigurasi) | 1─N Vektor dokumen; 1─N Vektor query; 1─N Tetangga exact; 1─N Run | Konfigurasi berbeda menghasilkan embedding_id baru; vektor lama tidak ditimpa |
| Vektor dokumen | float32[768] per dokumen | (embedding_id, row_idx) | N─1 Set embedding; 1─1 Dokumen lewat urutan doc_id yang disimpan bersama | Jumlah baris = 1.446.315; urutan baris = urutan doc_id tersimpan; norma L2 = 1 (wajib, karena penalti 2 pada #8 hanya sah untuk norma 1) |
| Vektor query | float32[768] per query | (embedding_id, row_idx) | N─1 Set embedding; 1─1 Query | Aturan sama dengan Vektor dokumen; jumlah baris = 960 |
| Tetangga exact | embedding_id, query_id, rank (1–5), doc_id, skor L2 kuadrat dari `IndexFlatL2` | (embedding_id, query_id, rank) | N─1 Set embedding; N─1 Query; N─1 Dokumen | Hanya berlaku untuk embedding_id yang sama |
| Run | run_id, timestamp, embedding_id, env_id, algorithm, params, split, k = 5, thread FAISS, thread torch, 12 metrik K7, jumlah query dengan hasil < 5 | run_id (surrogate) | N─1 Set embedding; N─1 Lingkungan | Hanya ditambah; algorithm ∈ {flat, hnsw, ivf, lsh}; split ∈ {val, test}; slot −1 tidak pernah disimpan sebagai doc_id |
| Lingkungan | Isi K8 | env_id (surrogate: hash isi) | 1─N Run | Dicatat setiap sesi; angka efisiensi hanya dibandingkan antar-run dengan env_id sama; notebook final memeriksa kesamaannya dengan run val |

## Sumber
| # | Sumber | Tingkat | Tanggal | Dipakai untuk |
|---|---|---|---|---|
| S1 | Kartu model, `sentence_bert_config.json`, dan metadata API `huggingface.co/LazarusNLP/congen-indobert-base` | 1 | diubah 2025-02-01 | max_seq_length 32, mean pooling, 768 dimensi, tanpa prefix; lisensi tidak tercantum |
| S3 | Kartu dataset `huggingface.co/datasets/miracl/miracl` (Apache-2.0) | 1 | 2022-10-18 | 960 query dev id, format qrels TREC, label test tidak dirilis |
| S5 | FAISS `INSTALL.md`, `github.com/facebookresearch/faiss` | 1 | versi 1.15.1 | Windows hanya CPU (konteks keputusan Vast.ai) |
| S6 | `pypi.org/project/faiss-cpu` | 1 | 2026-09-16 | Versi 1.15.1, MIT, maintainer Meta, Python 3.10–3.14 |
| S8 | FAISS wiki "Faiss indexes" | 1 | — (konsep stabil) | IndexFlatL2/HNSWFlat/IVFFlat/LSH, cara kerja IndexLSH |
| S10 | Kartu dataset `huggingface.co/datasets/miracl/miracl-corpus` (Apache-2.0) | 1 | v1.0 | 1.446.315 passage id dari 446.330 artikel; kolom docid/title/text; jsonl.gz; contoh pemuatan memakai loading script |
| S11 | `pypi.org/project/sentence-transformers` (Apache-2.0) | 1 | 2026-09-18 | Versi 6.1.0, Python 3.10–3.13, PyTorch 2.2+ dan transformers 5.x |
| S12 | `pypi.org/project/datasets` (Apache-2.0) | 1 | 2026-07-28 | Versi 5.0.1, Python 3.10–3.14 |

Sumber riset 001 putaran 1 lain (S2, S4, S7, S9) tidak menjadi dasar keputusan. Fakta "datasets 5.x tidak mendukung loading script" dan statistik qrels dev (9.668 penilaian, rata-rata 3,22, batas atas Precision@5 dan Recall@5) berasal dari diskusi Arya.

## Riwayat
| Tanggal | Perubahan | Alasan |
|---|---|---|
| 2026-10-02 | Usulan pertama rancangan MVP | Perintah Arya |
| 2026-10-02 | 001 putaran 2: korpus dan LSH dikeluarkan dari 001; tech stack/library/tools dan kontrak metrik evaluasi ditunda ke 002; max_seq_length 32 dan split val/test 50:50 dipilih | Jawaban dan koreksi Arya |
| 2026-10-02 | 001 disetujui; korpus dan LSH ikut 002 | Persetujuan Arya |
| 2026-10-02 | 002 disetujui: D3 diisi (MIRACL-id korpus penuh + dev); K1 diisi (sentence-transformers hanya memuat, encode PyTorch tiga modul, L2 sekali); K3 diisi (faiss-cpu, METRIC_L2 menggantikan rumusan inner product di 001a K1 — urutan sama untuk vektor bernorma 1; LSH = IndexLSH); K4 diisi kolom metrik; K5–K8 ditambah | Hasil diskusi Arya |
| 2026-10-02 | Fase tunggal: project hanya punya fase MVP; benchmark final di split test masuk MVP sebagai notebook terakhir (mengubah 001a K2 "benchmark final (Dev)"); baris Ditunda→Dev dihapus | Koreksi Arya "Cukup 1 fase saja", 002 titik periksa 1a |
| 2026-10-02 | Teks dokumen yang di-embed ditunda ke pekerjaan berikutnya | 002 titik periksa 2, jawaban Arya |
