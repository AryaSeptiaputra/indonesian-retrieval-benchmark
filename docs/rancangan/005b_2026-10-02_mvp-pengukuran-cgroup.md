# 005b · MVP · Pengukuran dan cgroup

Status: disetujui 2026-10-02
Dari: docs/rancangan/005a_2026-10-02_mvp-pengukuran-cgroup.md
Kondisi kode: rencana 003b sedang dibangun (5 dari 12). Sudah ada `notebooks/00_load_dataset.ipynb`, `01_preprocessing.ipynb`, `02_embedding.ipynb`. Di `02_embedding`, `fetch_cpu_quota` hanya membaca cgroup v2 `cpu.max`, dan `collect_environment` hanya membaca `memory.max`; keduanya berhenti di host cgroup v1. Notebook `03`–`06` belum ada. Dokumen turunan (`docs/metrik-evaluasi.md`, `docs/lingkungan-eksekusi.md`, `docs/dataset.md`) dan tabel H di `CLAUDE.md` masih menulis H4, H5, H6 sebagai belum diputuskan.

Rencana ini tidak mengubah 003b. Bagian yang baru dirancang di 005a dibangun di dua tempat:
- **cgroup v1/v2 (K6, K8)** — diubah di `02_embedding` lewat langkah 1 rencana ini, karena notebook itu sudah dibangun.
- **Pengukur waktu dan ukuran, #8 dengan ambang (K7, H4–H6), kolom diagnostik Run (K4)** — ditulis di langkah 003b yang sudah merencanakan fungsinya (`measure_search`, `measure_index_size`, `compute_fidelity_metrics`, `build_run` di 003b langkah 6–9 dan 11). 005a menjadi jawaban H4, H5, H6 untuk langkah-langkah itu; rencana ini menetapkan isi fungsinya dan pemeriksaan tambahannya.

## Penghambat baru

005a menyebut tiga hal "belum dijadwalkan". Rencana ini memberi kode lanjutan dari H16.

| Kode | Hal | Menghambat | Cara kode menghindari keputusan sendiri |
|---|---|---|---|
| H17 | Pemanasan sebelum 5 ulangan QPS | Fungsi QPS di 003b langkah 6–9 dan 11 | Tidak ada; urutan "p50 dulu lalu QPS" atau pemanggilan pemanasan sendiri sama-sama keputusan. Langkah 003b 6–9 dan 11 menunggu H17 |
| H18 | Urutan deteksi cgroup di host hybrid v1+v2 | — | Langkah 1 berhenti dengan error kalau berkas v2 dan v1 sama-sama ada, sehingga tidak memilih salah satu |
| H19 | Versi cgroup dicatat di Lingkungan atau tidak | — | Versi cgroup tidak ditambahkan ke Lingkungan (menambah field statis mengubah env_id); dicatat setelah diputuskan |

H16 (penguncian pyarrow dan pandas) tetap belum dijadwalkan. H9, H13, H10, H12 tetap mengikuti batas waktu 003a.

## Hubungan dengan 003b

| Keputusan 005a | Dibangun di (003b) | Fungsi | Isi menurut 005a |
|---|---|---|---|
| K7 #10 p50 (H4) | Langkah 6 `03_exact` (salinan pertama), lalu 7–9, 11 | `measure_latency_p50` | `time.perf_counter_ns` hanya membungkus `index.search(1 query, k = 5)`; 10 query pemanasan dari split yang sama sekali sebelum putaran pertama, tidak dihitung; 3 putaran atas semua query split; p50 = median semua pengukuran (ms) |
| K7 #9 QPS (H4, H17) | Sama | `measure_qps` | `index.search(semua query split, k = 5)` dengan n thread, 5 ulangan; QPS = n_query ÷ median(T₁..T₅); pemanasan menurut H17 |
| K7 #12 (H5) | Sama | `measure_index_size` | `len(faiss.serialize_index(index))` byte; buffer dilepas segera setelah panjangnya diambil |
| K7 #8 (H6) | Sama | `compute_fidelity_metrics` | d = √max(skor L2², 0); suku dᴱˣᵢ ≤ 1e-3 dikeluarkan; RDE(q) rata-rata atas suku tersisa; #8 = rata-rata RDE(q) atas query dengan suku tersisa ≥ 1; urutan ulang L2 LSH dan penalti slot −1 = 2 tetap |
| K4 kolom diagnostik #8 | Sama | `build_run` | Jumlah suku #8 yang dikeluarkan; jumlah query tanpa suku #8 tersisa |
| K6, K8 cgroup v1/v2 | Salinan fungsi dari `02` (langkah 1 rencana ini) | `fetch_cgroup_cpu_quota`, `fetch_cpu_quota`, `fetch_memory_quota` | Disalin identik dari `02` |

Pemeriksaan tambahan saat langkah 003b di atas dikerjakan (cara memeriksa mengikuti 003b):

| Langkah 003b | Lokal (pink-chan, `.venv` pustaka kecil) | Vast.ai (Arya) |
|---|---|---|
| 6 | `compute_fidelity_metrics` dengan kasus hitung tangan: suku dᴱˣ = 0 dan 5e-4 dikeluarkan, 2e-3 tidak; query yang kelima sukunya dikeluarkan tidak ikut rata-rata dan terhitung di diagnostik; skor L2² negatif kecil dipotong ke 0; tanpa suku dikeluarkan hasilnya sama dengan rumus 002. `measure_latency_p50` dan `measure_qps` dengan index palsu (objek berisi `search` buatan): jumlah panggilan 10 + 3·n_query dan 5, pemanasan tidak masuk median | Exact: #8 = 0 dan #7 = 1; p50, QPS, #12 terisi; kolom diagnostik tercatat |
| 7–9, 11 | Salinan fungsi pengukur dan penilai identik dengan `03` | Nilai #9, #10, #12 terisi untuk 16 run val dan run test |

## Peta keputusan → kode

| Keputusan | Folder / file | Fungsi utama | Peran (writer-code) |
|---|---|---|---|
| K6, K8 cgroup v1/v2 | `notebooks/02_embedding.ipynb` | `fetch_cgroup_cpu_quota` (v2 `cpu.max`; v1 `cpu/cpu.cfs_quota_us` dan `cpu/cpu.cfs_period_us`; quota −1 = tanpa batas; berhenti kalau v2 dan v1 sama-sama ada atau tidak ada keduanya), `fetch_cpu_quota` (dibatasi `sched_getaffinity`), `fetch_memory_quota` (v2 `memory.max`; v1 `memory/memory.limit_in_bytes`; nilai ≥ 2⁶² dianggap tanpa batas), `parse_cpu_max`, `parse_memory_max` tetap; `collect_environment` memanggil `fetch_memory_quota` | Pengakses luar, pengubah bentuk |
| Dokumen turunan | `docs/metrik-evaluasi.md`, `docs/lingkungan-eksekusi.md`, `docs/dataset.md` | Kartu K7 (pengukuran, #12, #8) dan K6·K8 cgroup 005a apa adanya; kolom diagnostik Run; skor Tetangga exact disimpan apa adanya; H4, H5, H6 keluar dari "Belum diputuskan"; H17–H19 masuk | Dokumen keputusan |
| Status H | `CLAUDE.md` bagian "Belum diputuskan" | H4, H5, H6 dipindah ke "sudah diputuskan" dengan rujukan 005a; H17, H18, H19 ditambahkan | Dokumen project |

## Langkah

| # | Yang dibangun | File disentuh | Test | Gate (dari rencana-evaluasi) | Bergantung pada |
|---|---|---|---|---|---|
| 1 | Pembaca cgroup v1 dan v2 di `02_embedding` (kode) | `notebooks/02_embedding.ipynb` | Lokal: folder cgroup palsu — v2 (`400000 100000` → 4, `max` → tanpa batas), v1 (quota 400000 / period 100000 → 4, quota −1 → tanpa batas), memori v2 dan v1 (angka dan ≥ 2⁶² → tanpa batas), hybrid → error, tanpa keduanya → error; statis (type hint, docstring, ≤ 50 baris). Vast.ai: tahap 1 dan 7 `02_embedding` berjalan di host v1 maupun v2 | — (belum ada `docs/rencana-evaluasi.md`) | — |
| 2 | Dokumen turunan sesuai 005a (tanpa kode) | `docs/metrik-evaluasi.md`, `docs/lingkungan-eksekusi.md`, `docs/dataset.md` | `python -c` (pustaka standar): tiga kartu K7 dan kartu K6·K8 005a tersalin identik; H4, H5, H6 tidak ada di "Belum diputuskan"; H17, H18, H19 ada; kolom diagnostik #8 ada di tabel Run | — | — |
| 3 | `CLAUDE.md` "Belum diputuskan" sesuai 005a (tanpa kode) | `CLAUDE.md` | `python -c`: tabel H berisi tepat H9, H10, H12, H13, H16, H17, H18, H19; `git diff` hanya menyentuh bagian itu | — | 2 |

## Tidak dibangun di rencana ini

- Notebook `03`–`06` dan fungsi pengukur, penilai, catatan run: dibangun di 003b langkah 6–9 dan 11 seperti tabel "Hubungan dengan 003b".
- Perubahan isi file 003b: 003b sudah disetujui dan tidak diubah. Jawaban H4, H5, H6 dan penghambat baru H17 dicatat di laporan dan kolom Pembangunan saat langkah 003b dikerjakan.
- Versi cgroup di Lingkungan (H19) dan urutan deteksi host hybrid (H18).

## Risiko teknis

- **Menjalankan ulang `02_embedding`.** Kalau 02 sudah pernah dijalankan dan folder embedding_id sudah ada, notebook berhenti di tahap 6 (`FileExistsError`) sebelum mencapai tahap 7. Perubahan langkah 1 tidak mengubah vektor maupun embedding_id; kalau 02 belum pernah dijalankan, tidak ada dampak.
- **env_id tetap sama.** Jatah CPU dan RAM dari v1 memakai satuan yang sama dengan v2, sehingga env_id host v2 tidak berubah karena langkah 1.
- **Path cgroup v1.** Langkah 1 membaca `/sys/fs/cgroup/cpu/` dan `/sys/fs/cgroup/memory/`. Di sebagian distribusi v1 controller CPU dipasang sebagai `cpu,cpuacct/` dengan `cpu/` sebagai symlink; kalau symlink tidak ada, notebook berhenti dengan error yang menyebut path.
- **Ambang "tanpa batas" memori v1.** Kernel menulis angka sangat besar (sekitar 2⁶³) untuk v1 tanpa batas; ambang ≥ 2⁶² mengikuti asumsi 005a "limit sangat besar diperlakukan sama seperti max".
- **H17 menahan langkah 003b 6–9 dan 11.** Seluruh pengukuran lain sudah terjawab; hanya fungsi QPS yang menunggu.
- **Salinan serialize_index.** #12 membuat salinan sementara sebesar index (exact ≈ 4,44 GB, HNSW ≈ 4,8 GB); fungsi melepas buffer segera, dan puncak RAM tercatat di Lingkungan.

## Koreksi selama putaran

- Disetujui Arya tanpa koreksi. Kode notebook 02 di langkah 1 dan perubahan `CLAUDE.md` di langkah 3 disetujui Arya sendiri.
- H17: Arya memilih "1 batch tanpa diukur" sebelum 5 ulangan QPS; dicatat red-chan sebagai rancangan bernomor dan diterapkan setelah file `a`-nya disetujui.
