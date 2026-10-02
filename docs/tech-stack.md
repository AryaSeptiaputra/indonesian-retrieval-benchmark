# Tech Stack

Ringkasan keputusan K1, K3, dan K5 dari rancangan 002a. Kalau isi dokumen ini berbeda dengan `docs/rancangan/002a_2026-10-02_mvp-dataset-library-metrik.md` atau `docs/keputusan-produk.md`, kedua dokumen itu yang berlaku.

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

Dependency di `requirements.txt` hanya tiga paket yang versinya ditetapkan 002a. torch dan numpy dikunci dari `pip freeze` instance Vast.ai (H13); `transformers` ikut ditarik `sentence-transformers` 6.1.0 (PyTorch 2.2+ dan transformers 5.x) dan versinya dicatat di Lingkungan (K8).

## K1 · Model embedding

| Hal | Keputusan |
|---|---|
| Model | `LazarusNLP/congen-indobert-base`, 768 dimensi, mean pooling, tanpa prefix query/dokumen |
| Peran sentence-transformers | Hanya memuat model beserta revision-nya; `model.encode` tidak dipakai |
| Encode | Loop PyTorch sendiri melewati ketiga modul model: Transformer → Pooling (mean tokens) → Dense 768→768 + Tanh |
| Panjang token | Tokenisasi `max_length = 32` untuk dokumen dan query (bawaan model, pilihan Arya) |
| Normalisasi | L2 sekali saat encode dengan `F.normalize`; vektor tersimpan sudah bernorma 1 dan notebook search tidak menormalisasi lagi |
| Tempat | Embedding dibuat sekali di GPU; keempat algoritma membaca vektor yang sama |

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

## K5 · Pemuatan dataset

`datasets` 5.0.1 tidak mendukung loading script `miracl.py`, sehingga data dimuat langsung dari file:

| Data | Berkas | Builder |
|---|---|---|
| Korpus | `miracl-corpus-v1.0-id`, 3 file `docs-*.jsonl.gz` (atau konversi Parquet Hugging Face) | `json` |
| Query (topics) dan qrels | `miracl-v1.0-id`, `topics/*.tsv` dan `qrels/*.tsv` (TREC), split dev | `csv`, pemisah tab |

Revision dataset dicatat di Set embedding dan Lingkungan. Rincian data ada di `docs/dataset.md`.

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

## Belum diputuskan

| Kode | Hal |
|---|---|
| H1 | Teks dokumen yang di-embed: title + text atau text saja |
| H2 | Parameter tiap algoritma (HNSW M, efConstruction, efSearch; IVF nlist, nprobe, data latih; LSH nbits) dan ada/tidaknya sapuan parameter di val |
| H7 | Presisi encode (fp32 atau fp16), ukuran batch, dan pemeriksaan kesamaan hasil loop encode dengan `model.encode` |
| H9 | Versi Python pasti (harus 3.10–3.13) |
| H13 | Versi torch dan numpy, dikunci dari `pip freeze` instance Vast.ai |
