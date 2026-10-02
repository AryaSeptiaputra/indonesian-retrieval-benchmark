# Dataset dan Data

Ringkasan keputusan D3, K2, K5, K9, dan model data dari rancangan 002a, 003a, 004a, dan 005a. Kalau isi dokumen ini berbeda dengan file `a` yang disetujui (`docs/rancangan/002a_2026-10-02_mvp-dataset-library-metrik.md`, `docs/rancangan/003a_2026-10-02_mvp-parameter-format-encode.md`, `docs/rancangan/004a_2026-10-02_mvp-seed-env-id.md`, `docs/rancangan/005a_2026-10-02_mvp-pengukuran-cgroup.md`) atau `docs/keputusan-produk.md`, dokumen-dokumen itu yang berlaku.

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
- Pemilihan konfigurasi hanya memakai val (aturan K11 di `docs/metrik-evaluasi.md`).
- Konfigurasi terpilih ditulis ke Kunci konfigurasi (cap waktu dan hash isi). Notebook benchmark final — notebook terakhir — berhenti kalau hash atau env_id tidak cocok, lalu membuka split test sekali.

## K9 · Format berkas

| Format | Entitas |
|---|---|
| Parquet | Dokumen, Query, Penilaian relevansi, Tetangga exact |
| CSV, hanya ditambah barisnya | Run (catatan run) |
| `.npy` float32 | Vektor dokumen, Vektor query |
| JSON | Set embedding, Lingkungan, Kunci konfigurasi |

Aturan tulis CSV dan JSON mengikuti template riset: UTF-8, CSV tanpa index dengan ujung baris LF, JSON `ensure_ascii=False` indentasi 2; hash dihitung dari isi kanonik. Kartu K9 003a, apa adanya:

```
K9 · Physical file formats — Umum  ★ H8
Parquet      Dokumen, Query, Penilaian relevansi, Tetangga exact
CSV          Catatan run, hanya ditambah barisnya
.npy         Vektor dokumen dan query, float32
JSON         Set embedding, Lingkungan, Kunci konfigurasi
Catatan      Parquet butuh pyarrow (terpasang lewat datasets); versinya dicatat K8
```

## Model data

Tidak ada basis data; data disimpan sebagai berkas yang menjadi kontrak antarbagian. Folder dan nama berkas tiap entitas tercatat di bagian "Kontrak berkas" `CLAUDE.md`.

| Entitas | Isi | Kunci | Relasi | Aturan integritas |
|---|---|---|---|---|
| Dokumen | doc_id (= `docid` MIRACL, skema "X#Y"), title, text — Parquet | doc_id (natural) | 1─N Penilaian relevansi; 1─1 Vektor dokumen per set embedding | doc_id unik; tidak diubah setelah disiapkan |
| Query | query_id, text, split — Parquet | query_id (natural, dari topics MIRACL) | 1─N Penilaian relevansi; 1─1 Vektor query per set embedding; 1─N Tetangga exact | split ∈ {val, test}, 50:50 seed 42, dikunci hash; setiap query punya ≥ 1 dokumen relevan; test hanya dibaca notebook benchmark final setelah hash Kunci konfigurasi cocok |
| Penilaian relevansi | query_id, doc_id, relevance (qrels TREC dev) — Parquet | (query_id, doc_id) | N─1 Query; N─1 Dokumen | Relevan kalau relevance ≥ 1, nilai 0 tidak relevan; query_id dan doc_id wajib ada (yang tidak ada dibuang dan jumlahnya dicatat) |
| Set embedding | embedding_id, model, revision model, revision dataset, max_seq_length 32, teks dokumen `title + " " + text`, presisi fp32, ukuran batch akhir, normalisasi L2 di PyTorch, dim 768, hasil pemeriksaan encode (selisih maksimum pada 100 sampel) — JSON | embedding_id (surrogate: hash konfigurasi) | 1─N Vektor dokumen; 1─N Vektor query; 1─N Tetangga exact; 1─N Run | Konfigurasi berbeda menghasilkan embedding_id baru; vektor lama tidak ditimpa |
| Vektor dokumen | float32[768] per dokumen — `.npy` | (embedding_id, row_idx) | N─1 Set embedding; 1─1 Dokumen lewat urutan doc_id yang disimpan bersama | Jumlah baris = 1.446.315; urutan baris = urutan doc_id tersimpan; norma L2 = 1 (wajib, karena penalti 2 pada #8 hanya sah untuk norma 1) |
| Vektor query | float32[768] per query — `.npy` | (embedding_id, row_idx) | N─1 Set embedding; 1─1 Query | Aturan sama dengan Vektor dokumen; jumlah baris = 960 |
| Tetangga exact | embedding_id, query_id, rank (1–5), doc_id, skor L2 kuadrat dari `IndexFlatL2` — Parquet | (embedding_id, query_id, rank) | N─1 Set embedding; N─1 Query; N─1 Dokumen | Hanya berlaku untuk embedding_id yang sama; skor disimpan apa adanya, pemotongan ke 0 hanya saat menghitung #8 (005a) |
| Kunci konfigurasi | Per algoritma: konfigurasi terpilih (K11), run_id val asalnya, embedding_id, env_id, cap waktu, hash isi — JSON | hash isi | N─1 Run (val) | Ditulis sekali sebelum test dibuka; notebook final berhenti kalau hash tidak cocok |
| Run | run_id, timestamp, embedding_id, env_id, algorithm, params (build dan search; IVF juga jumlah sampel latih dan seed k-means; LSH juga seed rotasi), split, k = 5, thread FAISS, thread torch, 12 metrik K7, jumlah query dengan hasil < 5, jumlah suku #8 yang dikeluarkan, jumlah query tanpa suku #8 tersisa (005a) — baris CSV append-only | run_id (surrogate) | N─1 Set embedding; N─1 Lingkungan | Satu konfigurasi sapuan = satu run; hanya ditambah; algorithm ∈ {flat, hnsw, ivf, lsh}; split ∈ {val, test}; slot −1 tidak pernah disimpan sebagai doc_id |
| Lingkungan | Isi K8, jatah CPU dan RAM dari cgroup v1 atau v2 (lihat `docs/lingkungan-eksekusi.md`) — JSON | env_id (hash semua field statis + hostname) | 1─N Run | Field dinamis dicatat tetapi tidak masuk hash; angka efisiensi hanya dibandingkan antar-run dengan env_id sama; notebook final memeriksa env_id sama dengan run val |

Relasi:

```
[Dokumen] 1──N [Penilaian relevansi] N──1 [Query]
[Set embedding] 1──N [Vektor dokumen]   (1──1 [Dokumen])
[Set embedding] 1──N [Vektor query]     (1──1 [Query])
[Query] 1──N [Tetangga exact] N──1 [Dokumen]
[Set embedding] 1──N [Run] N──1 [Lingkungan]
[Run (val)] 1──N [Kunci konfigurasi]   ← ditulis sekali sebelum test dibuka
```

Ukuran: satu salinan vektor korpus = 1.446.315 × 768 × 4 B ≈ 4,44 GB.

## Belum diputuskan

Tidak ada untuk dataset dan format berkas. Teks dokumen (H1) dan format berkas (H8) diputuskan di 003a.
