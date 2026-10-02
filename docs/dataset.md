# Dataset dan Data

Ringkasan keputusan D3, K2, K5, dan model data dari rancangan 002a. Kalau isi dokumen ini berbeda dengan `docs/rancangan/002a_2026-10-02_mvp-dataset-library-metrik.md` atau `docs/keputusan-produk.md`, kedua dokumen itu yang berlaku.

## Sumber data (D3)

| Data | Sumber | Isi | Lisensi |
|---|---|---|---|
| Korpus | `miracl/miracl-corpus`, subset id (`miracl-corpus-v1.0-id`) | 1.446.315 passage dari 446.330 artikel; kolom `docid`, `title`, `text` | Apache-2.0 |
| Query | `miracl/miracl`, folder `miracl-v1.0-id`, `topics/*.tsv`, split dev | 960 query | Apache-2.0 |
| Qrels | `miracl/miracl`, folder `miracl-v1.0-id`, `qrels/*.tsv` (format TREC), split dev | 9.668 penilaian; rata-rata 3,22 dokumen relevan per query (min 1, maks 13) | Apache-2.0 |

- Qrels test-a dan test-b tidak dirilis dan tidak dipakai.
- Ditolak: WebFAQ (`michaeldinzinger/webfaq-test`) dan MIRACL hard negatives versi MTEB.
- Data riset publik lengkap (tingkat 3), dimuat di instance Vast.ai; mode data rahasia tidak aktif.
- Dataset diunduh Arya ke `data/raw/`; agent tidak mengunduh.

## Pemuatan (K5)

`datasets` 5.0.1 tidak mendukung loading script `miracl.py`, sehingga data dimuat langsung dari file:

| Data | Berkas | Builder |
|---|---|---|
| Korpus | 3 file `docs-*.jsonl.gz` (atau konversi Parquet Hugging Face) | `json` |
| Topics dan qrels | `topics/*.tsv`, `qrels/*.tsv` | `csv`, pemisah tab |

Revision dataset dicatat di Set embedding dan Lingkungan.

## Relevansi

Dokumen relevan untuk sebuah query kalau relevance ≥ 1 di qrels; nilai 0 tidak relevan.

## Pembagian query (K2)

- 960 query dev dibagi val/test 50:50 secara acak dengan seed 42; korpus tidak dibagi.
- Split disimpan sebagai atribut Query dan dikunci dengan hash isi.
- Pemilihan konfigurasi hanya memakai val.
- Konfigurasi dikunci ke berkas (cap waktu dan hash isi). Notebook benchmark final — notebook terakhir — berhenti kalau hash atau Lingkungan tidak cocok, lalu membuka split test sekali.

## Model data

Tidak ada basis data; data disimpan sebagai berkas yang menjadi kontrak antarbagian. Folder tiap entitas tercatat di bagian "Kontrak berkas" `CLAUDE.md`.

| Entitas | Isi | Kunci | Relasi | Aturan integritas |
|---|---|---|---|---|
| Dokumen | doc_id (= `docid` MIRACL, skema "X#Y"), title, text | doc_id (natural) | 1─N Penilaian relevansi; 1─1 Vektor dokumen per set embedding | doc_id unik; tidak diubah setelah disiapkan |
| Query | query_id, text, split | query_id (natural, dari topics MIRACL) | 1─N Penilaian relevansi; 1─1 Vektor query per set embedding; 1─N Tetangga exact | split ∈ {val, test}, 50:50 seed 42, dikunci hash; setiap query punya ≥ 1 dokumen relevan; test hanya dibaca notebook benchmark final setelah hash konfigurasi cocok |
| Penilaian relevansi | query_id, doc_id, relevance (qrels TREC dev) | (query_id, doc_id) | N─1 Query; N─1 Dokumen | Relevan kalau relevance ≥ 1, nilai 0 tidak relevan; query_id dan doc_id wajib ada (yang tidak ada dibuang dan jumlahnya dicatat) |
| Set embedding | embedding_id, model, revision model, revision dataset, max_seq_length 32, normalisasi L2 di PyTorch, dim 768, teks dokumen (H1) | embedding_id (surrogate: hash konfigurasi) | 1─N Vektor dokumen; 1─N Vektor query; 1─N Tetangga exact; 1─N Run | Konfigurasi berbeda menghasilkan embedding_id baru; vektor lama tidak ditimpa |
| Vektor dokumen | float32[768] per dokumen | (embedding_id, row_idx) | N─1 Set embedding; 1─1 Dokumen lewat urutan doc_id yang disimpan bersama | Jumlah baris = 1.446.315; urutan baris = urutan doc_id tersimpan; norma L2 = 1 (wajib, karena penalti 2 pada #8 hanya sah untuk norma 1) |
| Vektor query | float32[768] per query | (embedding_id, row_idx) | N─1 Set embedding; 1─1 Query | Aturan sama dengan Vektor dokumen; jumlah baris = 960 |
| Tetangga exact | embedding_id, query_id, rank (1–5), doc_id, skor L2 kuadrat dari `IndexFlatL2` | (embedding_id, query_id, rank) | N─1 Set embedding; N─1 Query; N─1 Dokumen | Hanya berlaku untuk embedding_id yang sama |
| Run | run_id, timestamp, embedding_id, env_id, algorithm, params, split, k = 5, thread FAISS, thread torch, 12 metrik K7, jumlah query dengan hasil < 5 | run_id (surrogate) | N─1 Set embedding; N─1 Lingkungan | Hanya ditambah; algorithm ∈ {flat, hnsw, ivf, lsh}; split ∈ {val, test}; slot −1 tidak pernah disimpan sebagai doc_id |
| Lingkungan | Isi K8 (lihat `docs/lingkungan-eksekusi.md`) | env_id (surrogate: hash isi) | 1─N Run | Dicatat setiap sesi; angka efisiensi hanya dibandingkan antar-run dengan env_id sama; notebook final memeriksa kesamaannya dengan run val |

Relasi:

```
[Dokumen] 1──N [Penilaian relevansi] N──1 [Query]
[Set embedding] 1──N [Vektor dokumen]   (1──1 [Dokumen])
[Set embedding] 1──N [Vektor query]     (1──1 [Query])
[Query] 1──N [Tetangga exact] N──1 [Dokumen]
[Set embedding] 1──N [Run] N──1 [Lingkungan]
```

Ukuran: satu salinan vektor korpus = 1.446.315 × 768 × 4 B ≈ 4,44 GB.

## Belum diputuskan

| Kode | Hal |
|---|---|
| H1 | Teks dokumen yang di-embed: title + text atau text saja |
| H8 | Format berkas fisik tabel, vektor, dan metadata |
