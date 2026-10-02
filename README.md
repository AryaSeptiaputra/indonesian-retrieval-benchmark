# indonesian-retrieval-benchmark

Benchmark algoritma pencarian vektor untuk retrieval teks berbahasa Indonesia. Semua dokumen dan query di-embed sekali dengan `LazarusNLP/congen-indobert-base`, lalu dicari dengan exact dense search (brute-force, baseline) dan tiga algoritma Approximate Nearest Neighbor: HNSW, IVF, dan LSH. Keempatnya membaca vektor yang sama, sehingga perbedaan hasil hanya berasal dari algoritma pencariannya.

## Status

Baru struktur folder, `requirements.txt`, dan dokumen keputusan; belum ada notebook dan kode. Dataset, library, metrik, dan lingkungan eksekusi sudah diputuskan di rancangan 002 (lihat Dokumen rancangan). Teks dokumen yang di-embed, parameter algoritma, dan beberapa hal lain belum diputuskan. Notebook dibuat satu per satu saat pembangunan berjalan.

## Susunan folder

```
indonesian-retrieval-benchmark/
├── data/                  # di-gitignore, susunan dijaga .gitkeep
│   ├── raw/               # dataset mentah (korpus, query, qrels) dari Arya; tidak pernah diubah
│   ├── processed/         # Dokumen, Query beserta split val/test, Penilaian relevansi
│   └── embeddings/        # satu subfolder per embedding_id: Set embedding, Vektor dokumen, Vektor query, Tetangga exact
├── outputs/               # di-gitignore, susunan dijaga .gitkeep
│   └── tuning/            # catatan Run (hanya ditambah) dan Lingkungan per sesi
├── docs/                  # rancangan sistem, rencana pembangunan, dokumen keputusan
├── requirements.txt       # dependency yang versinya sudah diputuskan
└── CLAUDE.md              # aturan project dan kontrak berkas antarnotebook
```

## Bagian sistem dan urutan menjalankan

| Urutan | Bagian | Membaca | Menulis |
|---|---|---|---|
| 1 | Penyiapan data | `data/raw/` | `data/processed/` — split query val/test 50:50 seed 42, dikunci hash |
| 2 | Pembuat embedding | `data/processed/` | `data/embeddings/` — max_seq_length 32, normalisasi L2, revision model dicatat |
| 3 | Exact search | `data/embeddings/` | Tetangga exact di `data/embeddings/`; Run di `outputs/tuning/` |
| 4 | HNSW, IVF, LSH | `data/embeddings/`, Tetangga exact | Run di `outputs/tuning/` |
| 5 | Tabel hasil | `outputs/tuning/` | Tabel keempat algoritma berdampingan untuk split val |

Exact search dijalankan sebelum HNSW, IVF, dan LSH karena tetangganya menjadi pembanding ANN. Penilai menjadi tahap di setiap bagian algoritma: membaca Penilaian relevansi dan Tetangga exact, lalu mencatat hasil sebagai Run beserta Lingkungan. Split test tidak dipakai sampai benchmark final.

Notebook saling terhubung hanya lewat berkas di folder di atas. Rinciannya ada di bagian "Kontrak berkas" di `CLAUDE.md`.

## Prasyarat

- Python 3.10–3.13 (versi pasti belum diputuskan, H9).
- `pip install -r requirements.txt`: sentence-transformers, faiss-cpu, datasets. Versi torch dan numpy dikunci nanti dari `pip freeze` instance Vast.ai (H13).
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
