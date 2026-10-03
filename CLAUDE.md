# CLAUDE.md

## Proyek

**indonesian-retrieval-benchmark** — riset yang membandingkan algoritma pencarian vektor untuk retrieval teks berbahasa Indonesia. Semua dokumen dan query di-embed dengan satu model yang sama, lalu dicari dengan beberapa algoritma, sehingga perbedaan hasil hanya berasal dari algoritma pencariannya.

- **Jenis proyek:** riset & training model. Template struktur mengikuti `writer-code` (`.claude/skills/writer-code/references/structure-riset.md`), kecuali rancangan red-chan yang disetujui menetapkan lain.
- **Repo:** https://github.com/AryaSeptiaputra/indonesian-retrieval-benchmark

### Ditetapkan Arya (dikunci)

Bagian ini tidak boleh diganti tanpa persetujuan Arya.

| Hal | Ketetapan |
|---|---|
| Bahasa pemrograman | Python |
| Model embedding | `LazarusNLP/congen-indobert-base` (Hugging Face) |
| Algoritma pencarian | Exact dense search (brute-force, baseline) dan Approximate Nearest Neighbor: HNSW, IVF, LSH |

### Belum diputuskan

Keputusan dibuat lewat red-chan dan dicatat di `docs/keputusan-produk.md`; yang berlaku adalah dokumen itu dan file `docs/rancangan/<nomor>a_...`.

Sudah diputuskan:

| Hal | Ringkasan | Rancangan | Dokumen |
|---|---|---|---|
| Pengguna | Portofolio pribadi Arya tentang kemampuan mengimplementasikan retrieval | 003 | `docs/keputusan-produk.md` |
| Dataset | `miracl/miracl-corpus` id (1.446.315 passage); query dan qrels `miracl/miracl` id dev (960 query, 9.668 penilaian) | 002 | `docs/dataset.md` |
| Library | sentence-transformers 6.1.0 (hanya memuat), encode loop PyTorch, datasets 5.0.1, faiss-cpu 1.15.1 | 002 | `docs/tech-stack.md` |
| Encode | Dokumen `title + " " + text`, fp32, batch mulai 1024, pemeriksaan 100 sampel vs `model.encode` (1e-5) | 003 | `docs/tech-stack.md` |
| Parameter dan sapuan | HNSW M 32, efConstruction 200, efSearch ×5; IVF nlist 4096, nprobe ×7; LSH nbits 768/1536/3072; 16 run val | 003 | `docs/tech-stack.md` |
| Metrik | 12 metrik, k = 5, latensi p50, aturan slot −1 | 002 | `docs/metrik-evaluasi.md` |
| Pengukuran efisiensi | `perf_counter_ns` hanya membungkus `index.search`; p50: 1 query per panggilan, 10 pemanasan, 3 putaran; QPS: satu batch semua query, 5 ulangan, n_query ÷ median; #12 = byte `faiss.serialize_index` | 005 | `docs/metrik-evaluasi.md` |
| Pemanasan QPS | Satu panggilan batch T₀ yang tidak diukur sebelum 5 ulangan QPS, sekali per konfigurasi | 006 | `docs/metrik-evaluasi.md` |
| #8 di dekat nol | d = √max(L2², 0); suku dᴱˣᵢ ≤ 1e-3 dikeluarkan; rata-rata per query atas suku tersisa; dua kolom diagnostik di Run | 005 | `docs/metrik-evaluasi.md` |
| Pemilihan konfigurasi dan tanda berhasil | QPS tertinggi dengan k-NN Recall@5 ≥ 0,95 (kalau tidak ada, recall tertinggi); D4: 12 metrik lengkap di test, exact #7 = 1 dan #8 = 0, exact ulang identik | 003 | `docs/metrik-evaluasi.md` |
| Format berkas | Parquet, CSV append-only, `.npy` float32, JSON | 003 | `docs/dataset.md` |
| Hardware dan tempat menjalankan | Vast.ai Linux, RTX 3090 24 GB, RAM ≥ 32 GB, CPU ≥ 24, disk 50 GB; thread = min(jatah cgroup, 24); jatah dibaca dari cgroup v2 atau v1 | 002, 005 | `docs/lingkungan-eksekusi.md` |
| Pencatatan resource dan env_id | Otomatis oleh kode ke Lingkungan, manual oleh Arya di `README.md`; env_id = hash semua field statis + hostname | 002, 003, 004 | `docs/lingkungan-eksekusi.md` |
| Seed | IVF: seed k-means bawaan FAISS 1234, dibaca dari index; LSH: seed rotasi bawaan FAISS 5 (konstanta kode sumber 1.15.1); keduanya dicatat di Run. Seed project 42 hanya untuk split | 004 | `docs/tech-stack.md` |

Masih belum diputuskan:

| Kode | Hal | Status |
|---|---|---|
| H9 | Versi Python pasti (3.10–3.13) | Dijawab Arya saat instance Vast.ai pertama dibuat |
| H10 | Tempat penyimpanan di luar Vast.ai | Dijawab Arya sebelum instance pertama dihapus |
| H12 | Perilaku pencatatan resource di luar Linux | Hanya kalau notebook dijalankan lokal |
| H13 | Versi torch dan numpy (dikunci dari `pip freeze` instance Vast.ai) | Dijawab Arya saat instance Vast.ai pertama dibuat |
| H16 | Penguncian versi pyarrow dan pandas | Belum dijadwalkan |
| H18 | Urutan deteksi cgroup di host hybrid v1+v2 | Belum dijadwalkan; sementara notebook berhenti dengan error di host hybrid sebelum mengukur apa pun (006a) |
| H19 | Versi cgroup dicatat di Lingkungan atau tidak | Belum dijadwalkan; sementara tidak dicatat (006a) |
| — | Lisensi model, grafik laporan | Saat menyusun laporan |

## Dokumen rancangan

| File | Isi | Ditulis oleh |
|---|---|---|
| `docs/keputusan-produk.md` | Rancangan sistem per fase dan statusnya | red-chan |
| `docs/rencana-evaluasi.md` | Rancangan evaluasi per modul (mulai fase Dev) | red-chan |
| `docs/daftar-rancangan.md` | Status semua rancangan bernomor | red-chan (kolom a), pink-chan (kolom b, Pembangunan) |
| `docs/rancangan/<nomor>a_...` | Desain sistem yang disetujui | red-chan |
| `docs/rancangan/<nomor>b_...` | Rencana pembangunan | pink-chan |

Kode hanya ditulis dari rancangan yang sudah berstatus `siap dikerjakan`.

## Kontrak berkas

Notebook terhubung hanya lewat berkas berikut, tanpa import antar-notebook. Entitas dan aturannya dari rancangan 001a–003a; format dari K9 (003a); nama berkas dari rencana 003b. Kolom CSV, kunci JSON, dan kolom Parquet adalah kontrak: mengubahnya berarti mengubah semua notebook pembacanya.

| Entitas | Berkas | Format | Ditulis oleh | Dibaca oleh | Aturan |
|---|---|---|---|---|---|
| Dataset mentah (korpus, topics, qrels) | `data/raw/`: 3 file `docs-*.jsonl.gz`, `topics/*.tsv`, `qrels/*.tsv` | jsonl.gz, TSV | Arya, diunduh manual | `00_load_dataset` | Tidak pernah diubah |
| Data termuat sebelum validasi | `data/interim/corpus.parquet`, `topics.parquet`, `qrels.parquet`, `metadata.json` (revision dataset) | Parquet, JSON | `00_load_dataset` | `01_preprocessing` | Salinan isi dataset mentah apa adanya, tanpa filter |
| Dokumen | `data/processed/documents.parquet` | Parquet | `01_preprocessing` | `02_embedding` | doc_id unik; tidak diubah setelah disiapkan |
| Query | `data/processed/queries.parquet` | Parquet | `01_preprocessing` | `02_embedding`, `03_exact`, `04a`–`04c`, `06_final_benchmark` | split ∈ {val, test}, 50:50 seed 42, dikunci hash; test hanya dibaca `06_final_benchmark` setelah hash Kunci konfigurasi cocok |
| Penilaian relevansi | `data/processed/qrels.parquet` | Parquet | `01_preprocessing` | `03_exact`, `04a`–`04c`, `06_final_benchmark` | Relevan kalau relevance ≥ 1; id yang tidak ada dibuang dan jumlahnya dicatat |
| Metadata penyiapan data | `data/processed/metadata.json` (hash split, revision dataset, jumlah yang dibuang) | JSON | `01_preprocessing` | `01_preprocessing` (gate checksum), `02_embedding` | Split tidak ditimpa kalau hash berbeda |
| Set embedding | `data/embeddings/<embedding_id>/embedding_set.json` | JSON | `02_embedding` | `03_exact`, `04a`–`04c`, `06_final_benchmark` | Konfigurasi berbeda → embedding_id baru; vektor lama tidak ditimpa |
| Vektor dokumen | `data/embeddings/<embedding_id>/doc_vectors.npy` + `doc_ids.parquet` | `.npy` float32, Parquet | `02_embedding` | `03_exact`, `04a`–`04c`, `06_final_benchmark` | 1.446.315 baris; urutan baris = urutan `doc_ids.parquet`; norma L2 = 1 |
| Vektor query | `data/embeddings/<embedding_id>/query_vectors.npy` + `query_ids.parquet` | `.npy` float32, Parquet | `02_embedding` | `03_exact`, `04a`–`04c`, `06_final_benchmark` | 960 baris; urutan baris = urutan `query_ids.parquet`; norma L2 = 1 |
| Tetangga exact | `data/embeddings/<embedding_id>/exact_neighbors.parquet` | Parquet | `03_exact` (split val) | `04a`–`04c` | Hanya sah untuk embedding_id yang sama; exact dijalankan sebelum ANN |
| Run | `outputs/tuning/runs_<algorithm>.csv` (val); `outputs/metrics/runs_test.csv` (test) | CSV append-only | `03_exact`, `04a`–`04c` (val); `06_final_benchmark` (test) | `05_val_results`, `06_final_benchmark` | Satu konfigurasi sapuan = satu baris; hanya ditambah, tidak pernah ditimpa |
| Lingkungan | `outputs/tuning/environment_<env_id>.json` | JSON | `02_embedding`, `03_exact`, `04a`–`04c`, `06_final_benchmark` | `05_val_results`, `06_final_benchmark` | env_id = hash field statis + hostname; field dinamis tidak masuk hash |
| Kunci konfigurasi | `outputs/tuning/locked_config.json` | JSON | `05_val_results` | `06_final_benchmark` | Ditulis sekali sebelum test dibuka; notebook final berhenti kalau hash tidak cocok |

`data/` dan `outputs/` di-gitignore; susunan foldernya dijaga dengan `.gitkeep`. Notebook di kolom Ditulis oleh dan Dibaca oleh dibangun lewat rencana 003b langkah 3–11.

## Fungsi tersalin

Fungsi yang dipakai beberapa notebook disalin, tidak di-import, dan setiap salinan harus identik. Saat mengubah satu salinan, ubah semua salinannya dalam pekerjaan yang sama.

| Fungsi | Ada di |
|---|---|
| `load_parquet` | 01, 02, 03, 04a, 04b, 04c |
| `load_json` | 01, 02, 03, 04a, 04b, 04c |
| `parse_cpu_max` | 02, 03, 04a, 04b, 04c |
| `parse_cfs_quota` | 02, 03, 04a, 04b, 04c |
| `fetch_cgroup_cpu_quota` | 02, 03, 04a, 04b, 04c |
| `fetch_cpu_quota` | 02, 03, 04a, 04b, 04c |
| `compute_thread_count` | 02, 03, 04a, 04b, 04c |
| `set_num_threads` | 02, 03, 04a, 04b, 04c |
| `parse_nvidia_smi` | 02, 03, 04a, 04b, 04c |
| `fetch_gpu_info` | 02, 03, 04a, 04b, 04c |
| `parse_cpuinfo` | 02, 03, 04a, 04b, 04c |
| `parse_memory_max` | 02, 03, 04a, 04b, 04c |
| `parse_memory_limit` | 02, 03, 04a, 04b, 04c |
| `fetch_memory_quota` | 02, 03, 04a, 04b, 04c |
| `fetch_library_versions` | 02, 03, 04a, 04b, 04c |
| `collect_environment` | 02, 03, 04a, 04b, 04c |
| `compute_env_id` | 02, 03, 04a, 04b, 04c |
| `save_environment` | 02, 03, 04a, 04b, 04c |
| `fetch_embedding_dir` | 03, 04a, 04b, 04c |
| `load_vectors` | 03, 04a, 04b, 04c |
| `select_split_rows` | 03, 04a, 04b, 04c |
| `build_relevance` | 03, 04a, 04b, 04c |
| `search_index` | 03, 04a, 04b, 04c |
| `to_doc_ids` | 03, 04a, 04b, 04c |
| `to_distances` | 03, 04a, 04b, 04c |
| `to_result_distances` | 03, 04a, 04b, 04c |
| `_score_query` | 03, 04a, 04b, 04c |
| `compute_quality_metrics` | 03, 04a, 04b, 04c |
| `compute_fidelity_metrics` | 03, 04a, 04b, 04c |
| `count_short_results` | 03, 04a, 04b, 04c |
| `measure_latency_p50` | 03, 04a, 04b, 04c |
| `measure_qps` | 03, 04a, 04b, 04c |
| `measure_index_size` | 03, 04a, 04b, 04c |
| `build_run` | 03, 04a, 04b, 04c |
| `append_run` | 03, 04a, 04b, 04c |
| `load_exact_neighbors` | 04a, 04b, 04c |
| `evaluate_index` | 04a, 04b |

## Agent dan skill di repo ini

Agent dan skill disimpan di repo supaya ikut ter-commit:

```
.claude/
├── agents/
│   ├── red-chan.md          # perancang sistem, tidak menulis kode
│   └── pink-chan.md         # pelaksana penulisan kode Python
├── skills/
│   ├── product-design/      # dimuat red-chan
│   ├── writer-code/         # dimuat pink-chan; standar kode Python dan struktur folder
│   └── reader-code/         # dibaca pink-chan dengan Read saat dibutuhkan; tidak dimuat otomatis
└── settings.json            # reader-code dimatikan dari pemanggilan otomatis
```

Path di dalam agent dan skill ditulis relatif terhadap root repo (`.claude/skills/...`).

Catatan perawatan: Claude Code mendahulukan skill personal (`~/.claude/skills/`) di atas skill project dengan nama yang sama, sedangkan untuk agent yang didahulukan versi project. Kalau skill di repo ini diubah, samakan juga salinan personalnya, atau hapus salinan personal supaya versi repo yang dipakai.

### Agent pink-chan

Kalau perintah Arya diawali atau menyebut nama "pink-chan", serahkan pekerjaannya ke agent `pink-chan` — jangan dikerjakan sendiri di percakapan utama. Pink-chan melanjutkan pekerjaan bertahap lewat agent yang sama; saat Arya membalas persetujuan atau jawaban untuk pekerjaan pink-chan, teruskan balasan itu ke pink-chan.

### Agent red-chan

Kalau perintah Arya diawali atau menyebut nama "red-chan", serahkan pekerjaannya ke agent `red-chan` — jangan merancang sendiri di percakapan utama.

Red-chan bekerja per putaran. Saat laporannya berstatus `menunggu jawaban` atau `menunggu persetujuan`:

1. Tampilkan diagram gambaran sistem apa adanya di dalam blok kode (jangan digambar ulang), lalu keputusan, bentrokan, dan asumsi sebagai teks ringkas.
2. Ubah bagian "Pertanyaan / Titik periksa" menjadi `AskUserQuestion`: `[Judul]` menjadi `header`, tiap pilihan menjadi `option` dengan konsekuensinya sebagai `description`, dan pilihan (Usulan) tetap paling atas. Jangan menambah, mengurangi, atau mengubah isi pilihan.
3. Kalau statusnya `menunggu persetujuan`, tanyakan juga apakah rancangan disetujui atau ada koreksi lain.
4. Teruskan jawaban Arya ke red-chan yang sama lewat `SendMessage`, termasuk teks bebas kalau Arya memilih "Other".

Saat statusnya `selesai`, tampilkan perintah untuk pink-chan apa adanya. Jangan menjalankannya sebelum Arya sendiri mengirim perintah itu.

### red-chan dan pink-chan di mode plan

Kalau sesi sedang dalam mode plan, pekerjaan red-chan dan pink-chan tetap diserahkan ke agent-nya, dengan alur berikut:

1. Awali pesan ke agent dengan `MODE PLAN` lalu perintah Arya. Agent tidak menulis file dan mengembalikan laporan lengkap.
2. Titik periksa dari red-chan ditanyakan dulu dengan `AskUserQuestion` (aturan di atas). Teruskan jawabannya ke agent yang sama, juga diawali `MODE PLAN`, sampai tidak ada pertanyaan tersisa.
3. Salin laporan terakhir agent **apa adanya** ke file plan, termasuk diagram di dalam blok kode dan bagian "Akan ditulis/dikerjakan setelah disetujui", lalu panggil `ExitPlanMode`. Jangan meringkas atau menggambar ulang.
4. Setelah Arya menyetujui plan, kirim ke agent yang sama: `PLAN DISETUJUI`, beserta pilihan titik periksa dan koreksi Arya. Agent menulis semua file-nya.
5. Kalau Arya menolak atau mengoreksi plan, teruskan koreksinya ke agent dengan awalan `MODE PLAN`, lalu ulangi dari langkah 3.

## Git

- Branch utama `main`, remote `origin` ke repo GitHub di atas.
- Commit dan push hanya saat Arya memintanya.
- Berkas rahasia (`.env`, kunci, token Hugging Face) tidak pernah di-commit.
