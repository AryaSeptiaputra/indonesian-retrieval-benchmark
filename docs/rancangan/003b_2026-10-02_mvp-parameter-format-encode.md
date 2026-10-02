# 003b · MVP · Parameter, format, encode

Status: disetujui 2026-10-02
Dari: docs/rancangan/003a_2026-10-02_mvp-parameter-format-encode.md
Kondisi kode: belum ada kode atau notebook. Hasil 001b dan 002b: folder `data/raw/`, `data/processed/`, `data/embeddings/`, `outputs/tuning/` dengan `.gitkeep`; `.gitignore`; `requirements.txt` (sentence-transformers 6.1.0, faiss-cpu 1.15.1, datasets 5.0.1); `README.md`; empat dokumen keputusan di `docs/` (`tech-stack.md`, `metrik-evaluasi.md`, `dataset.md`, `lingkungan-eksekusi.md`); bagian "Belum diputuskan" dan "Kontrak berkas" di `CLAUDE.md`. Dokumen dan `CLAUDE.md` itu kini usang terhadap 003a (bentrokan 9 di 003a).

Template: riset notebook-only (`structure-riset.md`). Rencana ini punya dua bagian:

- **Bagian A — dokumen (langkah 1–2), tanpa kode.** Memperbarui dokumen turunan dan `CLAUDE.md` sesuai 003a.
- **Bagian B — notebook (langkah 3–12), berisi kode.** Membangun seluruh alur benchmark. 002b menyerahkan pembangunan notebook ke rencana bernomor baru setelah hal yang belum pasti diputuskan; 003a memutuskan sebagian besar dan menjadwalkan sisanya. Bagian sistem yang "tidak berubah" di 003a (penyiapan data, exact, penilai) ikut dibangun di sini karena belum pernah dibangun; isinya mengikuti 002a apa adanya. Pemuatan dataset dan preprocessing dipisah menjadi dua notebook, `00_load_dataset` dan `01_preprocessing` (koreksi Arya).

Notebook dijalankan Arya di instance Vast.ai (K6). Pink-chan memeriksa di `.venv` lokal yang hanya berisi pustaka kecil; pustaka besar (torch, faiss, sentence-transformers, transformers) tidak diunduh (koreksi Arya). Cara memeriksa dijelaskan di bagian "Cara memeriksa".

## Penghambat

Kode H melanjutkan 002b. H14 dan H15 adalah hal "belum dijadwalkan" di 003a; Arya menerima usulan jawabannya, dan koordinator meneruskannya ke red-chan untuk dicatat sebagai rancangan bernomor. Sampai rancangan itu disetujui, langkah yang bergantung padanya menunggu.

| Kode | Hal | Status | Menghambat langkah |
|---|---|---|---|
| H4 | Cara mengukur QPS dan latensi p50 | Dijawab Arya sebelum notebook search pertama (003a) | 6–9, 11 |
| H5 | Definisi #12 Ukuran index / memori | Dijawab Arya sebelum notebook search pertama (003a) | 6–9, 11 |
| H6 | Jarak exact = 0 pada #8 | Dijawab Arya saat menulis penilai (003a) | 6–9, 11 |
| H9 | Versi Python | Dijawab Arya saat instance Vast.ai pertama dibuat (003a) | 12; menjalankan notebook di instance |
| H13 | Versi torch dan numpy | Dijawab Arya saat instance Vast.ai pertama dibuat (003a) | 12 |
| H10 | Penyimpanan di luar Vast.ai | Dijawab Arya sebelum instance pertama dihapus (003a) | Tidak ada langkah; salin keluar dikerjakan setelah diputuskan (lihat Tidak dibangun) |
| H12 | Perilaku pencatatan resource di luar Linux | Hanya kalau dijalankan lokal (003a) | Tidak menghambat; tanpa H12, pembaca cgroup melempar error yang menyebut berkas yang tidak ditemukan |
| H14 | Seed k-means IVF dan seed rotasi acak IndexLSH | Menunggu rancangan 004 (red-chan) | 8, 9, 11 |
| H15 | Field statis yang membentuk env_id | Menunggu rancangan 004 (red-chan) | 5–9, 11 |
| H16 | Penguncian versi pyarrow dan pandas | Belum dijadwalkan (003a) | 12 |

## Peta keputusan → kode

Nama berkas di kolom Folder / file adalah usulan pink-chan; formatnya dari K9.

| Keputusan | Folder / file | Fungsi utama | Peran (writer-code) |
|---|---|---|---|
| Bentrokan 9 (dokumen usang) | `docs/tech-stack.md`, `docs/metrik-evaluasi.md`, `docs/dataset.md`, `docs/lingkungan-eksekusi.md` | K1 ✎, K9, K10 ke tech-stack; K4 ✎, K11, D4 ke metrik-evaluasi; K9, entitas Kunci konfigurasi ke dataset; K8 env_id ke lingkungan-eksekusi; H1, H2, H3, H7, H8, H11 dipindah dari "Belum diputuskan"; H14, H15 ditulis "menunggu rancangan red-chan" | Dokumen keputusan |
| Bentrokan 9, D1 | `CLAUDE.md` ("Belum diputuskan", "Kontrak berkas"), `README.md` | Tabel H diperbarui (H4–H6, H9, H10, H12–H16 tersisa); Kontrak berkas diisi format, nama berkas, notebook penulis/pembaca, folder `data/interim/`, entitas Kunci konfigurasi; README: tujuan portofolio (D1), status, urutan notebook | Dokumen project |
| K5 Pemuatan dataset | `notebooks/00_load_dataset.ipynb` → `data/interim/corpus.parquet`, `topics.parquet`, `qrels.parquet`, `metadata.json` | `load_corpus` (builder `json`, 3 file `docs-*.jsonl.gz`), `load_topics`, `load_qrels` (builder `csv`, pemisah tab, split dev), `fetch_dataset_revision`, `save_interim` | Pengakses luar |
| K2, K9 Penyiapan data | `notebooks/01_preprocessing.ipynb`, membaca `data/interim/` → `data/processed/documents.parquet`, `queries.parquet`, `qrels.parquet`, `metadata.json` | `load_interim`, `validate_corpus` (doc_id unik), `filter_qrels` (buang id tidak ada, kembalikan jumlah), `validate_queries` (≥ 1 dokumen relevance ≥ 1), `split_queries` (50:50, seed 42), `compute_content_hash` (isi kanonik), `validate_split_unchanged` (gate checksum), `save_processed` | Pengakses luar, pemeriksa, pengubah bentuk |
| K1, K9 Embedding | `notebooks/02_embedding.ipynb` → `data/embeddings/<embedding_id>/doc_vectors.npy`, `query_vectors.npy`, `doc_ids.parquet`, `query_ids.parquet`, `embedding_set.json` | `load_embedding_model` (revision dicatat), `to_document_text` (`title + " " + text`), `build_embedding_config`, `compute_embedding_id`, `encode_texts` (fp32, `max_length = 32`, batch dari konstanta mulai 1024, Transformer → Pooling mean → Dense + Tanh → `F.normalize`), `compare_with_model_encode` (100 teks sampel, max \|selisih\| ≤ 1e-5), `validate_unit_norm`, `save_embeddings` | Pengakses luar, pemeriksa |
| K6 Thread | Salinan di `02`, `03`, `04a`–`04c`, `06` | `fetch_cpu_quota` (cgroup `cpu.max`, `sched_getaffinity`), `set_num_threads` (n = ⌊min(jatah, 24)⌋ ke FAISS dan torch) | Pengakses luar, pembantu |
| K8 Lingkungan dan env_id | Salinan di `02`, `03`, `04a`–`04c`, `06` → `outputs/tuning/environment_<env_id>.json` | `collect_environment` (field statis dan dinamis), `compute_env_id` (hash field statis + hostname menurut K8 dan H15), `save_environment` | Pengakses luar, pembantu |
| K3 Exact, Tetangga exact | `notebooks/03_exact.ipynb` → `data/embeddings/<embedding_id>/exact_neighbors.parquet` | `load_embeddings`, `build_flat_index`, `search_index` (top-5), `save_exact_neighbors` | Pengakses luar |
| K7 Penilai | Salinan identik di `03`, `04a`–`04c`, `06` | `compute_quality_metrics` (#1–#6), `compute_fidelity_metrics` (#7, #8: d = √skor, LSH dihitung ulang dan diurutkan menaik, slot −1 → d = 2), `measure_search` (#9, #10 menurut H4), `measure_index_size` (#12 menurut H5), `count_short_results` | Pengubah bentuk, pengakses luar (pengukuran) |
| K4, K9 Catatan run | Salinan di `03`, `04a`–`04c`, `06` → `outputs/tuning/runs_<algorithm>.csv` (val), `outputs/metrics/runs_test.csv` (test) | `build_run` (params build dan search; IVF juga jumlah sampel latih dan seed k-means), `append_run` (hanya menambah baris, tidak pernah menimpa) | Pengubah bentuk, pengakses luar |
| K10 HNSW | `notebooks/04a_hnsw.ipynb` | `build_hnsw_index` (M = 32, efConstruction = 200); sapuan efSearch ∈ {16, 32, 64, 128, 256} di sel konstanta → 1 build, 5 run | Pengakses luar |
| K10 IVF | `notebooks/04b_ivf.ipynb` | `train_ivf_index` (nlist = 4096, sampel bawaan FAISS 1.048.576 vektor, `max_points_per_centroid` tidak diubah, seed menurut H14), `build_ivf_index`; sapuan nprobe ∈ {1, 4, 8, 16, 32, 64, 128} → 1 latih+build, 7 run | Pengakses luar |
| K10 LSH | `notebooks/04c_lsh.ipynb` | `build_lsh_index` (nbits ∈ {768, 1536, 3072}, seed rotasi menurut H14) → 3 build, 3 run; `compute_l2_distances` | Pengakses luar, pengubah bentuk |
| K4 Tabel hasil, K11 Pemilih konfigurasi | `notebooks/05_val_results.ipynb` → `outputs/tuning/locked_config.json` | `load_runs`, `to_results_table` (16 run val berdampingan, satu env_id), `select_config` (QPS tertinggi dengan k-NN Recall@5 ≥ 0,95, kalau kosong recall tertinggi; exact = satu konfigurasinya), `lock_config` (konfigurasi, run_id val, embedding_id, env_id, cap waktu, hash isi; ditulis sekali) | Pengakses luar, pengubah bentuk |
| K2, D4 Benchmark final | `notebooks/06_final_benchmark.ipynb` → `outputs/metrics/runs_test.csv` | `validate_locked_config` (hash cocok), `validate_environment` (env_id sama dengan run val), test dibuka sekali untuk keempat konfigurasi terkunci, run ulang exact, `validate_success_criteria` (D4: 12 metrik lengkap, exact #7 = 1 dan #8 = 0, nDCG@5 exact ulang identik) | Pemeriksa, proses notebook |
| H9, H13, H16 | `requirements.txt` | torch, numpy (dan pyarrow, pandas kalau H16 memutuskan dikunci) dikunci `==` dari `pip freeze` instance | Dependency (2.9) |

Fungsi yang tersalin (K6, K8, K7, K4, `load_embeddings`, `compute_content_hash`, fungsi build index di `06`) dicatat di tabel fungsi tersalin `CLAUDE.md` pada langkah yang pertama kali menyalinnya.

## Cara memeriksa

Pink-chan membuat `.venv` di root project (sudah di-gitignore) pada langkah notebook pertama yang membutuhkannya, dan hanya menginstal pustaka kecil untuk pemeriksaan: numpy, pandas, pyarrow. Pustaka besar — torch, faiss, sentence-transformers, transformers — tidak diunduh (koreksi Arya). Pustaka kecil lain hanya ditambahkan kalau sebuah pemeriksaan membutuhkannya dan disebut di laporan langkah itu. Isi `.venv` tidak menjadi dependency project; `requirements.txt` tetap mengikuti 002a dan langkah 12.

| Jenis | Siapa | Cara |
|---|---|---|
| Statis | pink-chan, `.venv` lokal | `python -c` dengan `json` dan `ast` dari pustaka standar: notebook JSON nbformat 4 valid, sel kode bisa di-parse, sel konstanta pertama, type hint lengkap, salinan fungsi identik antar-notebook (perbandingan teks) |
| Fungsi tanpa pustaka besar | pink-chan, `.venv` lokal | `python -c` mengekstrak definisi fungsi dari sel notebook (lewat `ast`) dan menjalankannya dengan data palsu kecil, termasuk tulis-baca Parquet, CSV, dan JSON di folder sementara. Untuk fungsi yang cukup memakai pustaka standar, numpy, pandas, dan pyarrow |
| Fungsi yang memakai torch, faiss, sentence-transformers, transformers, datasets, atau cgroup Linux | Arya, Vast.ai | Notebook dijalankan dari atas; sel pemeriksaan di dalam notebook (jumlah baris, norma 1, pemeriksaan encode 1e-5, exact #7 = 1 dan #8 = 0) berhenti dengan error kalau gagal. Perintah dan sel yang perlu dijalankan ditulis di laporan tiap langkah |

## Langkah

| # | Yang dibangun | File disentuh | Test | Gate (dari rencana-evaluasi) | Bergantung pada | Terhambat |
|---|---|---|---|---|---|---|
| 1 | Dokumen turunan sesuai 003a (tanpa kode) | `docs/tech-stack.md`, `docs/metrik-evaluasi.md`, `docs/dataset.md`, `docs/lingkungan-eksekusi.md` | Lokal: kartu K1, K8, K9, K10, K11, D4 003a tersalin identik; H1, H2, H3, H7, H8, H11 tidak lagi di "Belum diputuskan"; H4–H6, H9, H10, H12–H16 ada | — (belum ada `docs/rencana-evaluasi.md`) | — | — |
| 2 | `CLAUDE.md` dan README sesuai 003a (tanpa kode) | `CLAUDE.md`, `README.md` | Lokal: tabel H di `CLAUDE.md` berisi H4–H6, H9, H10, H12–H16; Kontrak berkas memuat sepuluh entitas dengan format K9 dan folder `data/interim/`; `git diff` hanya menyentuh bagian "Belum diputuskan" dan "Kontrak berkas" | — | 1 | — (perubahan `CLAUDE.md` disetujui Arya) |
| 3 | Pemuatan dataset | `notebooks/00_load_dataset.ipynb`, `data/interim/.gitkeep`, `CLAUDE.md` (tabel fungsi tersalin) | Lokal: statis. Vast.ai (Arya): 1.446.315 baris korpus, 960 query dev, 9.668 penilaian, revision dataset tercatat | — | 2 | — |
| 4 | Penyiapan data | `notebooks/01_preprocessing.ipynb`, `CLAUDE.md` | Lokal: statis; fungsi validasi, filter qrels, split 50:50 seed 42 stabil, hash sama untuk isi sama, gate checksum berhenti saat isi berubah (DataFrame palsu kecil). Vast.ai: 480/480 query, berkas `data/processed/` tertulis | — | 3 | — |
| 5 | Embedding | `notebooks/02_embedding.ipynb`, `CLAUDE.md` | Lokal: statis; `to_document_text`, `compute_embedding_id` stabil, `validate_unit_norm` (numpy). Vast.ai: pemeriksaan 100 sampel ≤ 1e-5, norma 1, jumlah baris vektor | — | 4 | H15 |
| 6 | Exact, Penilai, catatan run | `notebooks/03_exact.ipynb`, `CLAUDE.md` | Lokal: statis; kasus hitung tangan #1–#8 termasuk slot −1 dan penalti 2 (numpy), `append_run` hanya menambah baris (pandas CSV di scratchpad). Vast.ai: exact #7 = 1 dan #8 = 0, Tetangga exact tertulis | — | 5 | H4, H5, H6, H15 |
| 7 | HNSW | `notebooks/04a_hnsw.ipynb`, `CLAUDE.md` | Lokal: statis; salinan fungsi identik dengan `03`. Vast.ai: 1 build, 5 run tercatat dengan params build dan search | — | 6 | H4, H5, H6, H15 |
| 8 | IVF | `notebooks/04b_ivf.ipynb`, `CLAUDE.md` | Lokal: statis; salinan identik; penilai menangani slot −1. Vast.ai: 7 run; jumlah sampel latih dan seed tercatat | — | 6 | H4, H5, H6, H14, H15 |
| 9 | LSH | `notebooks/04c_lsh.ipynb`, `CLAUDE.md` | Lokal: statis; salinan identik; `compute_l2_distances` dan pengurutan menaik (numpy). Vast.ai: 3 build, 3 run | — | 6 | H4, H5, H6, H14, H15 |
| 10 | Tabel hasil val dan Pemilih konfigurasi | `notebooks/05_val_results.ipynb`, `CLAUDE.md` | Lokal: statis; `select_config` dengan catatan run palsu memilih QPS tertinggi di atas 0,95 dan jatuh ke recall tertinggi kalau tidak ada; `lock_config` menulis sekali dan hash stabil. Vast.ai: tabel 16 run, kunci tertulis | — | 7, 8, 9 | — |
| 11 | Benchmark final | `notebooks/06_final_benchmark.ipynb`, `CLAUDE.md` | Lokal: statis; `validate_locked_config` dan `validate_environment` berhenti saat hash atau env_id tidak cocok; `validate_success_criteria` gagal saat nDCG@5 exact ulang berbeda. Vast.ai: dijalankan sekali setelah langkah 10, di instance yang sama | — | 10 | H4, H5, H6, H14, H15 |
| 12 | Kunci versi dependency sisa | `requirements.txt`, `docs/tech-stack.md` | Lokal: semua baris `==` dan sama dengan `pip freeze` instance yang dikirim Arya | — | — (kapan saja setelah instance dibuat) | H9, H13, H16 |

Belum ada `docs/rencana-evaluasi.md`, sehingga kolom Gate berisi `—`. Notebook-only tidak punya `tests/`.

## Tidak dibangun di rencana ini

- Notebook arsip / salin keluar instance: cara dan tempatnya bergantung H10; dibangun (atau dikerjakan manual) setelah H10 diputuskan.
- Grafik laporan: dijadwalkan Arya saat menyusun laporan.
- `tuning_grids/`: sapuan K10 hanya 16 konfigurasi, jadi nilainya disimpan di sel konstanta notebook masing-masing.
- `01_eda.ipynb`: tidak ada di 003a maupun 002a.
- `requirements-dev.txt` (koreksi Arya di 002b: cukup `requirements.txt`).
- Pustaka besar di `.venv` lokal: torch, faiss, sentence-transformers, transformers (koreksi Arya).

## Risiko teknis

- **Pemeriksaan lokal terbatas.** Bagian yang memakai torch, faiss, sentence-transformers, transformers, datasets, atau cgroup hanya teruji saat Arya menjalankan notebook di Vast.ai; kesalahan di bagian itu baru terlihat di instance. Versi numpy, pandas, dan pyarrow di `.venv` lokal bisa berbeda dari versi di instance (H13, H16); versi lokal dicatat di laporan langkah.
- **Memori.** Satu salinan vektor korpus ≈ 4,44 GB; HNSW M = 32 ≈ 4,8 GB. Satu notebook per algoritma membuat index dibangun dan dilepas satu per satu. `06_final_benchmark` membangun keempat index berurutan dan harus melepas tiap index sebelum membangun berikutnya.
- **Satu instance untuk val dan test.** env_id memuat hostname (K8); instance yang berganti di antara langkah 6–11 membuat notebook final berhenti. Arya perlu menjalankan `03`–`06` di satu instance.
- **Slot −1 di numpy.** Indeks −1 menunjuk elemen terakhir tanpa error; penilai memakai mask sebelum mengambil doc_id atau vektor.
- **Skor FAISS berbeda jenis.** Flat, HNSW, IVF mengembalikan L2 kuadrat; IndexLSH mengembalikan Hamming. #8 memakai √skor untuk tiga yang pertama dan jarak L2 yang dihitung ulang untuk LSH.
- **IndexLSH dengan nbits > d.** nbits 1536 dan 3072 melebihi d = 768, sehingga rotasi acak menjadi proyeksi ke dimensi lebih tinggi dan index perlu `train`. Ukuran kode 3072 bit = 384 byte per vektor (≈ 0,56 GB).
- **Pemeriksaan encode 1e-5.** Toleransi diasumsikan sebagai selisih mutlak maksimum per elemen vektor ternormalisasi (asumsi 003a). Perbedaan kernel GPU antara loop sendiri dan `model.encode` bisa mendekati batas ini; kalau gagal, hasilnya dilaporkan, tidak dilonggarkan.
- **Banyak salinan fungsi.** Penilai, Lingkungan, thread, catatan run, dan fungsi build index tersalin ke 2–6 notebook; setiap perubahan mengubah semua salinan dalam pekerjaan yang sama.
- **Run ulang exact di test.** Menurut asumsi 003a dijalankan di notebook final yang sama tanpa pemilihan; kalau hasilnya tidak identik (urutan seri skor), D4 gagal dan dilaporkan.

## Koreksi selama putaran

- Cakupan: "file untuk load_dataset dan pre-processing dibuat dengan 2 file berbeda. 00 untuk load_dataset" — pemuatan dataset menjadi `00_load_dataset.ipynb` (menulis `data/interim/`), preprocessing tetap `01_preprocessing.ipynb`; langkah notebook ikut dalam cakupan; total menjadi 12 langkah.
- H14 dan H15: Arya menerima usulan (seed bawaan FAISS dicatat; semua field statis masuk hash env_id); keputusannya dicatat red-chan sebagai rancangan bernomor, dan langkah yang bergantung menunggu rancangan itu.
- `.venv` lokal, putaran 1: "Tidak". Putaran 2: "Bangun virtual env sendiri saja namun jangan mendownload putaka yang besar." — `.venv` project berisi pustaka kecil untuk pemeriksaan (numpy, pandas, pyarrow); torch, faiss, sentence-transformers, transformers tidak diunduh; bagian yang membutuhkannya diperiksa Arya di Vast.ai.
- Perubahan `CLAUDE.md` di langkah 2: disetujui Arya.
- Cakupan: "Benar, dokumen + notebook" — 12 langkah.
- Persetujuan rencana: "Disetujui". H14 dan H15 dirancang red-chan di rancangan 004; langkah yang bergantung menunggu 004a disetujui.
