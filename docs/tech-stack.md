# Tech Stack

Ringkasan keputusan K1, K3, K5, K9 (dependency), K10, dan K13 dari rancangan 002a, 003a, 004a, dan 007a. Kalau isi dokumen ini berbeda dengan file `a` yang disetujui (`docs/rancangan/002a_2026-10-02_mvp-dataset-library-metrik.md`, `docs/rancangan/003a_2026-10-02_mvp-parameter-format-encode.md`, `docs/rancangan/004a_2026-10-02_mvp-seed-env-id.md`, `docs/rancangan/007a_2026-10-03_mvp-penjaga-test.md`) atau `docs/keputusan-produk.md`, dokumen-dokumen itu yang berlaku.

## Ringkasan

| Bagian | Pilihan | Versi | Lisensi |
|---|---|---|---|
| Bahasa | Python | 3.10–3.13, versi pasti belum diputuskan (H9) | — |
| Model embedding | `LazarusNLP/congen-indobert-base` (dikunci Arya) | revision (commit hash) dicatat di Set embedding | belum pasti (tidak tercantum di kartu model; repo kodenya Apache-2.0) |
| Pemuat model | `sentence-transformers` | 6.1.0 | Apache-2.0 |
| Encode | PyTorch (loop sendiri) | belum dikunci (H13) | — |
| Pemuat dataset | `datasets` | 5.0.1 | Apache-2.0 |
| Index dan search | `faiss-cpu` | 1.15.1 | MIT |
| Array | numpy | belum dikunci (H13) | — |
| Parquet | pyarrow (terpasang lewat `datasets`) | dikunci `==` (K13); nilai versi dari `pip freeze` instance | — |
| Tabel | pandas (terpasang lewat `datasets`) | dikunci `==` (K13); nilai versi dari `pip freeze` instance | — |

Dependency di `requirements.txt` saat ini tiga paket yang versinya ditetapkan 002a. Menurut K13 (007a), torch, numpy, pyarrow, dan pandas juga dikunci `==` dari `pip freeze` instance Vast.ai yang sama, supaya Parquet dan hash isi di notebook 01 dibaca dan ditulis dengan versi yang sama di setiap sesi; nilai versinya ditulis setelah instance pertama ada (H9, H13). Kartu K13 007a, apa adanya:

```
K13 · Dependency pinning (H16) — Umum
requirements.txt  sentence-transformers==6.1.0, faiss-cpu==1.15.1,
                  datasets==5.0.1 (002a) + torch, numpy (H13) + pyarrow,
                  pandas (H16), semuanya == dari pip freeze instance Vast.ai
                  yang sama; Python dari python --version instance itu (H9)
Alasan       Parquet dan hash isi di notebook 01 dibaca dan ditulis dengan
             versi yang sama di setiap sesi
```

 `transformers` ikut ditarik `sentence-transformers` 6.1.0 (PyTorch 2.2+ dan transformers 5.x) dan versinya dicatat di Lingkungan (K8).

## K1 · Model embedding

| Hal | Keputusan |
|---|---|
| Model | `LazarusNLP/congen-indobert-base`, 768 dimensi, mean pooling, tanpa prefix query/dokumen |
| Teks | Dokumen = `title + " " + text`; query apa adanya |
| Peran sentence-transformers | Hanya memuat model beserta revision-nya; `model.encode` hanya dipakai untuk pemeriksaan 100 teks sampel |
| Encode | Loop PyTorch sendiri melewati ketiga modul model: Transformer → Pooling (mean tokens) → Dense 768→768 + Tanh → `F.normalize` (L2) |
| Panjang token | Tokenisasi `max_length = 32` untuk dokumen dan query (bawaan model, pilihan Arya) |
| Presisi dan batch | fp32; batch dari sel konstanta, dicoba mulai 1024, nilai akhir dicatat di Set embedding |
| Normalisasi | L2 sekali saat encode; vektor tersimpan sudah bernorma 1 dan notebook search tidak menormalisasi lagi |
| Tempat | Embedding dibuat sekali di GPU; keempat algoritma membaca vektor yang sama |

Kartu K1 003a, apa adanya:

```
K1 · Embedding encode loop (PyTorch) — Umum [S1, S11]  ✎ H1, H7
Teks         Dokumen = title + " " + text; query apa adanya
Pendekatan   sentence-transformers 6.1.0 hanya memuat model (revision dicatat);
             loop PyTorch: Transformer → Pooling mean → Dense 768→768 + Tanh
             → F.normalize (L2)
Parameter    max_length = 32; presisi fp32; batch = konstanta, dicoba mulai
             1024, nilai akhir dicatat di Set embedding
Pemeriksaan  100 teks sampel: max |v_loop − v_model.encode| ≤ 1e-5 per elemen,
             kedua vektor sudah ternormalisasi (asumsi pembacaan toleransi);
             selisih maksimum dicatat di Set embedding
```

Passage yang lebih panjang dari 32 token terpotong. Ini diterima karena yang dibandingkan adalah algoritma pada vektor yang sama, bukan kualitas model.

## K3 · Index FAISS

Seluruh search berjalan di CPU dengan `faiss-cpu` 1.15.1.

| Algoritma | Index | Jarak |
|---|---|---|
| Exact (baseline, dijalankan lebih dulu) | `IndexFlatL2` | L2 kuadrat |
| HNSW | `IndexHNSWFlat`, METRIC_L2 | L2 kuadrat |
| IVF | `IndexIVFFlat`, quantizer `IndexFlatL2`, METRIC_L2 | L2 kuadrat |
| LSH | `IndexLSH`: proyeksi acak → kode biner → scan jarak Hamming | Hamming |

Untuk vektor bernorma 1: ‖q − x‖² = 2 − 2·q·x, sehingga urutan L2 sama dengan urutan cosine. Exact dijalankan lebih dulu karena hasilnya (Tetangga exact) menjadi pembanding ANN.

## K10 · Parameter dan sapuan

| Algoritma | Build | Search (disapu di val) | Run |
|---|---|---|---|
| Exact | — | — | 1 |
| HNSW | M = 32, efConstruction = 200 | efSearch ∈ {16, 32, 64, 128, 256} | 1 build, 5 run |
| IVF | nlist = 4096; latih dengan sampel acak bawaan FAISS 256 · nlist = 1.048.576 vektor | nprobe ∈ {1, 4, 8, 16, 32, 64, 128} | 1 latih+build, 7 run |
| LSH | nbits ∈ {768, 1536, 3072} (pengecualian tertulis: parameter build yang disapu) | — | 3 build, 3 run |

Total 16 run di split val, k = 5. Kartu K10 003a, apa adanya (seed dilengkapi 004a, di bawah):

```
K10 · Parameter sweep on val — Umum [S7, S13]  ★ H2
Aturan       Parameter build tetap, parameter search disapu; pengecualian
             tertulis untuk LSH (IndexLSH tidak punya parameter search)
Exact        tanpa parameter → 1 run
HNSW         build: M = 32, efConstruction = 200
             search: efSearch ∈ {16, 32, 64, 128, 256} → 1 build, 5 run
IVF          build: nlist = 4096 (≈ 3,4·√N untuk N = 1.446.315)
             latih: sampel acak bawaan FAISS = 256 · nlist = 1.048.576 vektor
             (max_points_per_centroid = 256 tidak diubah); jumlah sampel dan
             seed k-means dicatat di Run
             search: nprobe ∈ {1, 4, 8, 16, 32, 64, 128} → 1 latih+build, 7 run
LSH          build: nbits ∈ {768, 1536, 3072} → 3 build, 3 run
             #11 dan #12 berbeda per konfigurasi
Total        16 run di split val, k = 5
```

### Seed (004a)

| Algoritma | Seed | Sumber nilai | Dicatat di |
|---|---|---|---|
| IVF | k-means = 1234, bawaan FAISS, tidak diubah; dipakai juga untuk sampel latih 1.048.576 vektor | Dibaca dari parameter clustering index | params Run, bersama jumlah sampel latih |
| LSH | Rotasi acak = 5, konstanta `rrot.init(5)` di konstruktor `IndexLSH` | Kode sumber FAISS 1.15.1 (tidak bisa dibaca dari objek index); berlaku selama `rotate_data = true` | params Run, dengan tanda "nilai dari kode sumber FAISS 1.15.1" |

Seed project 42 hanya untuk pembagian query val/test (K2). Kartu K10 seed 004a, apa adanya:

```
K10 · Random seeds (FAISS defaults) — Umum [S13, S14]  ✎ H14
IVF          Seed k-means = 1234 (ClusteringParameters.seed bawaan FAISS),
             tidak diubah; dipakai juga untuk sampel latih 256 · 4096 =
             1.048.576 vektor. Nilai dibaca dari parameter clustering index
             dan dicatat di Run (params) bersama jumlah sampel
LSH          Seed rotasi acak = 5, konstanta di konstruktor IndexLSH
             (rrot.init(5)); bukan parameter dan tidak bisa dibaca dari objek
             index. Dicatat di Run (params) dengan tanda "nilai dari kode
             sumber FAISS 1.15.1"; berlaku selama rotate_data = true
             (bawaan konstruktor Python, asumsi pengetahuan umum)
Seed project 42 tetap hanya untuk pembagian query val/test (K2)
```

## K5 · Pemuatan dataset

`datasets` 5.0.1 tidak mendukung loading script `miracl.py`, sehingga data dimuat langsung dari file:

| Data | Berkas | Builder |
|---|---|---|
| Korpus | `miracl-corpus-v1.0-id`, 3 file `docs-*.jsonl.gz` (atau konversi Parquet Hugging Face) | `json` |
| Query (topics) dan qrels | `miracl-v1.0-id`, `topics/*.tsv` dan `qrels/*.tsv` (TREC), split dev | `csv`, pemisah tab |

Revision dataset dicatat di Set embedding dan Lingkungan. Rincian data dan format berkas (K9) ada di `docs/dataset.md`.

## Pendekatan yang ditolak

| Pendekatan | Ditolak karena |
|---|---|
| WebFAQ (`michaeldinzinger/webfaq-test`) | Arya memilih dataset yang sudah proper: korpus, query, qrels terpisah dan dinilai manusia |
| MIRACL hard negatives versi MTEB | Ditolak Arya; korpus penuh dipakai |
| Memuat MIRACL lewat loading script `miracl.py` | Tidak didukung datasets 5.x |
| `model.encode` sentence-transformers untuk encode | Arya memilih loop PyTorch sendiri; sentence-transformers hanya memuat model |
| `os.cpu_count()` untuk jumlah thread | Di Vast.ai melaporkan total mesin, bukan jatah instance |
| Laptop lokal untuk embedding dan evaluasi | VRAM RTX 3050 4 GB dan RAM kosong sekitar 2 GB tidak cukup |
| max_seq_length 128 atau 512 untuk dokumen | Model tidak dilatih di panjang itu dan embedding lebih lambat; Arya memilih 32 |
| Tiap algoritma meng-embed sendiri | Perbedaan hasil bisa berasal dari embedding, bukan algoritma |
| text saja sebagai teks dokumen | Passage tanpa nama subjek kehilangan konteks; Arya memilih title + text |
| nbits LSH dikunci satu nilai tanpa sapuan | LSH hanya punya satu konfigurasi sehingga K11 tidak punya pilihan (003 titik periksa 1) |
| Menaikkan `max_points_per_centroid` supaya IVF dilatih seluruh 1.446.315 vektor | Latih lebih lama dan melewati rentang panduan FAISS tanpa manfaat terdokumentasi (003 titik periksa 2) |
| nlist = 65536 (panduan FAISS untuk N 1M–10M) | Butuh ≥ 30·65536 ≈ 1,97 juta vektor latih, lebih dari N |
| Seed IVF diganti seed project 42 | Arya memilih seed bawaan FAISS (004a) |

## Belum diputuskan

| Kode | Hal | Dijawab paling lambat |
|---|---|---|
| H9 | Versi Python pasti (harus 3.10–3.13) | Saat instance Vast.ai pertama dibuat |
| H13 | Versi torch dan numpy, dikunci dari `pip freeze` instance Vast.ai | Saat instance Vast.ai pertama dibuat |
