# Lingkungan Eksekusi

Ringkasan keputusan K6 dan K8 dari rancangan 002a. Kalau isi dokumen ini berbeda dengan `docs/rancangan/002a_2026-10-02_mvp-dataset-library-metrik.md` atau `docs/keputusan-produk.md`, kedua dokumen itu yang berlaku.

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

Lingkungan memakai env_id (hash isi). Angka efisiensi hanya dibandingkan antar-run dengan env_id sama.

## Belum diputuskan

| Kode | Hal |
|---|---|
| H10 | Tempat penyimpanan di luar Vast.ai |
| H11 | Field yang membentuk env_id. Lingkungan memuat nilai yang berubah di setiap notebook (puncak VRAM dan RAM, sisa disk, timestamp sesi), sehingga hash seluruh isi tidak pernah sama antar-run dan aturan "efisiensi hanya dibandingkan antar-run dengan env_id sama" tidak bisa terpenuhi |
| H12 | Perilaku kode pencatatan resource di luar Linux (pembacaan cgroup hanya ada di Linux; laptop Arya Windows) |
