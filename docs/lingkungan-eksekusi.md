# Lingkungan Eksekusi

Ringkasan keputusan K6 dan K8 dari rancangan 002a, 003a, dan 004a. Kalau isi dokumen ini berbeda dengan file `a` yang disetujui (`docs/rancangan/002a_2026-10-02_mvp-dataset-library-metrik.md`, `docs/rancangan/003a_2026-10-02_mvp-parameter-format-encode.md`, `docs/rancangan/004a_2026-10-02_mvp-seed-env-id.md`) atau `docs/keputusan-produk.md`, dokumen-dokumen itu yang berlaku.

## Tempat menjalankan (K6)

| Hal | Keputusan |
|---|---|
| Tempat | Vast.ai (Linux) |
| GPU | RTX 3090 24 GB |
| RAM | ≥ 32 GB |
| CPU | ≥ 24 (jatah efektif vCPU = angka pertama di "x/y CPU" penawaran Vast.ai) |
| Disk | 50 GB |
| Peran GPU | Hanya embedding |
| Peran CPU | Search keempat algoritma (`faiss-cpu`) |
| Sesi | Keempat algoritma diukur di satu instance dan satu sesi |

Laptop lokal ditolak: VRAM RTX 3050 4 GB dan RAM kosong sekitar 2 GB tidak cukup.

## Thread

- n = ⌊min(jatah CPU dari cgroup, 24)⌋.
- Diset eksplisit lewat `faiss.omp_set_num_threads(n)` dan `torch.set_num_threads(n)`, lalu dicatat di setiap run.
- `os.cpu_count()` tidak dipakai karena di Vast.ai melaporkan total mesin, bukan jatah instance.

## Memori

Satu salinan vektor korpus = 1.446.315 × 768 × 4 B ≈ 4,44 GB, dan Flat, HNSW, IVF masing-masing menyimpan salinan penuh. RAM ≥ 32 GB diasumsikan cukup kalau index dibangun dan dilepas satu per satu. Puncak RAM dicatat (K8).

## Penyimpanan

Penyimpanan Vast.ai tidak permanen. Vektor, catatan run, dan hasil disalin keluar sebelum instance dihapus dan dibawa ke instance berikutnya. Tempatnya belum diputuskan (H10).

Angka efisiensi hanya sah dalam satu sesi dan satu mesin; benchmark final di split test sebaiknya dijalankan di sesi yang sama dengan val, dan notebook final memeriksa Lingkungan.

## Pencatatan resource (K8)

Dicatat otomatis oleh kode ke entitas Lingkungan:

| Kelompok | Yang dicatat |
|---|---|
| GPU | Model, VRAM, driver, versi CUDA, puncak VRAM terpakai |
| CPU | Model, flag AVX2/AVX-512, jatah vCPU (cgroup `cpu.max` dan `sched_getaffinity`), total vCPU mesin |
| RAM | Jatah (cgroup `memory.max`), puncak RAM terpakai |
| Disk | Total dan sisa |
| Thread | Jumlah thread FAISS dan torch |
| Versi | Python dan library |
| Revision | Model dan dataset |
| Waktu | Timestamp sesi |

Dicatat manual oleh Arya di `README.md` bagian "Lingkungan Vast.ai", dari dashboard Vast.ai: ID penawaran/host, harga per jam, reliability, lokasi, status verified.

## env_id (K8, 004a)

env_id dibentuk dari hash semua field statis ditambah hostname instance; field dinamis dicatat tetapi tidak masuk hash. Angka efisiensi hanya dibandingkan antar-run dengan env_id sama.

| Jenis | Field |
|---|---|
| Statis, masuk hash | Model GPU, VRAM total, driver/CUDA, model CPU, flag AVX2/AVX-512, jatah vCPU, total vCPU mesin, jatah RAM, total disk, thread FAISS, thread torch, versi Python dan library, revision model dan dataset, hostname instance |
| Dinamis, hanya dicatat | Puncak RAM, puncak VRAM, sisa disk, timestamp |

Kartu K8 004a, apa adanya (menggantikan kartu K8 003a):

```
K8 · Environment identity — Umum  ✎ H15
env_id       hash( model GPU, VRAM total, driver/CUDA, model CPU,
                   flag AVX2/AVX-512, jatah vCPU, total vCPU mesin,
                   jatah RAM, total disk, thread FAISS, thread torch,
                   versi Python dan library, revision model dan dataset,
                   hostname instance )
Dicatat saja Puncak RAM, puncak VRAM, sisa disk, timestamp (tidak masuk hash)
Akibat       Perubahan field statis apa pun (termasuk revision) antara sapuan
             val dan benchmark final → env_id baru → notebook final berhenti
```

Akibatnya sapuan val dan benchmark final test wajib dijalankan di instance yang sama dan dengan revision model dan dataset yang sama; perubahan field statis apa pun menghasilkan env_id baru dan notebook final berhenti.

Ditolak: env_id hanya dari field H11 + hostname (Arya memilih semua field statis masuk hash, 004a).

## Belum diputuskan

| Kode | Hal | Dijawab paling lambat |
|---|---|---|
| H10 | Tempat penyimpanan di luar Vast.ai | Sebelum instance pertama dihapus |
| H12 | Perilaku kode pencatatan resource di luar Linux (pembacaan cgroup hanya ada di Linux; laptop Arya Windows) | Hanya kalau notebook dijalankan lokal |
