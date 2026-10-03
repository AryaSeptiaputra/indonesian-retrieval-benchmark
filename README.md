# indonesian-retrieval-benchmark

Benchmark algoritma pencarian vektor untuk retrieval teks berbahasa Indonesia. Semua dokumen dan query di-embed sekali dengan `LazarusNLP/congen-indobert-base`, lalu dicari dengan exact dense search (brute-force, baseline) dan tiga algoritma Approximate Nearest Neighbor: HNSW, IVF, dan LSH. Keempatnya membaca vektor yang sama, sehingga perbedaan hasil hanya berasal dari algoritma pencariannya.

Proyek ini adalah portofolio pribadi tentang kemampuan mengimplementasikan retrieval.

## Status

Struktur folder, `requirements.txt`, dan dokumen keputusan sudah ada; notebook dibangun satu per satu lewat rencana 003b. Dataset, library, metrik, format berkas, parameter sapuan, aturan pemilihan konfigurasi, dan lingkungan eksekusi sudah diputuskan (rancangan 002–004). Yang masih belum diputuskan tercatat di bagian "Belum diputuskan" `CLAUDE.md`.

## Susunan folder

```
indonesian-retrieval-benchmark/
├── notebooks/             # seluruh kode; dijalankan dari folder ini (dibangun lewat rencana 003b)
├── data/                  # di-gitignore, susunan dijaga .gitkeep
│   ├── raw/               # dataset mentah (korpus, topics, qrels) dari Arya; tidak pernah diubah
│   ├── interim/           # data termuat sebelum validasi (Parquet)
│   ├── processed/         # Dokumen, Query beserta split val/test, Penilaian relevansi (Parquet)
│   └── embeddings/        # satu subfolder per embedding_id: Set embedding, Vektor (.npy), Tetangga exact
├── outputs/               # di-gitignore, susunan dijaga .gitkeep
│   ├── tuning/            # catatan Run val (CSV, hanya ditambah), Lingkungan, Kunci konfigurasi
│   └── metrics/           # catatan Run benchmark final di split test
├── docs/                  # rancangan sistem, rencana pembangunan, dokumen keputusan
├── requirements.txt       # dependency yang versinya sudah diputuskan
└── CLAUDE.md              # aturan project dan kontrak berkas antarnotebook
```

Folder `notebooks/`, `data/interim/`, dan `outputs/metrics/` dibuat saat notebook yang memakainya dibangun.

## Urutan menjalankan notebook

| Notebook | Bagian | Membaca | Menulis |
|---|---|---|---|
| `00_load_dataset` | Pemuatan dataset (K5) | `data/raw/` | `data/interim/` — revision dataset dicatat |
| `01_preprocessing` | Penyiapan data (K2) | `data/interim/` | `data/processed/` — split query val/test 50:50 seed 42, dikunci hash |
| `02_embedding` | Pembuat embedding (K1), GPU | `data/processed/` | `data/embeddings/<embedding_id>/` — title + text, 32 token, fp32, norma 1 |
| `03_exact` | Exact search (K3) | `data/embeddings/` | Tetangga exact; Run di `outputs/tuning/` |
| `04a_hnsw`, `04b_ivf`, `04c_lsh` | Sapuan HNSW, IVF, LSH di val (K10) | `data/embeddings/`, Tetangga exact | Run di `outputs/tuning/` (16 run val bersama exact) |
| `05_val_results` | Tabel hasil val dan Pemilih konfigurasi (K11) | `outputs/tuning/` | Kunci konfigurasi |
| `06_final_benchmark` | Benchmark final (K2, D4) | Kunci konfigurasi, Lingkungan | Run test di `outputs/metrics/` |

Exact dijalankan sebelum HNSW, IVF, dan LSH karena tetangganya menjadi pembanding ANN. Penilai menjadi tahap di setiap notebook algoritma. Split test hanya dibuka sekali di `06_final_benchmark`, setelah hash Kunci konfigurasi dan env_id cocok, jadi `03`–`06` dijalankan di instance Vast.ai yang sama.

Notebook saling terhubung hanya lewat berkas. Rinciannya ada di bagian "Kontrak berkas" di `CLAUDE.md`.

## Membuka ulang split test

`06_final_benchmark` berhenti sebelum membaca query test kalau `outputs/metrics/runs_test.csv` sudah berisi percobaan untuk embedding_id yang sama (K12). Untuk membuka ulang, misalnya setelah kesalahan penilai diperbaiki:

1. Di sel konstanta `06_final_benchmark`, isi `IZIN_BUKA_ULANG = True` dan `ALASAN_BUKA_ULANG` dengan alasan tertulis.
2. Jalankan notebook; peringatan dicetak dan setiap baris run test baru membawa `test_attempt` berikutnya dan alasannya.
3. Kembalikan `IZIN_BUKA_ULANG = False` dan `ALASAN_BUKA_ULANG = ""` setelah percobaan selesai.

Baris test lama tidak dihapus atau diubah; laporan memakai percobaan terakhir dan menyebut jumlah percobaan.

## Prasyarat

- Python 3.10–3.13 (versi pasti belum diputuskan, H9).
- `pip install -r requirements.txt`: sentence-transformers, faiss-cpu, datasets. torch, numpy, pyarrow, dan pandas dikunci `==` dari `pip freeze` instance Vast.ai setelah instance pertama ada (K13, H13).
- Instance Vast.ai sesuai `docs/lingkungan-eksekusi.md`.
- Dataset MIRACL id diletakkan Arya di `data/raw/` (lihat `docs/dataset.md`); agent tidak mengunduh data.

## Dokumen rancangan

- `docs/keputusan-produk.md` — kondisi terkini rancangan sistem
- `docs/daftar-rancangan.md` — status semua rancangan bernomor
- `docs/rancangan/` — desain sistem (`a`) dan rencana pembangunan (`b`)
- `docs/tech-stack.md` — model, library, dan index FAISS
- `docs/metrik-evaluasi.md` — 12 metrik, rumus, dan aturan slot −1
- `docs/dataset.md` — sumber data, pembagian query, dan model data
- `docs/lingkungan-eksekusi.md` — Vast.ai, thread, dan pencatatan resource

## Lingkungan Vast.ai

Diisi Arya dari dashboard Vast.ai setiap menyewa instance (K8). Resource lain dicatat otomatis oleh kode.

| Hal | Isi |
|---|---|
| ID penawaran/host | |
| Harga per jam | |
| Reliability | |
| Lokasi | |
| Status verified | |
