# 001b · MVP · Benchmark retrieval

Status: disetujui 2026-10-02
Dari: docs/rancangan/001a_2026-10-02_mvp-benchmark-retrieval.md
Kondisi kode: project baru. Yang sudah ada hanya `CLAUDE.md`, `.gitignore`, `docs/`, dan `.claude/`; belum ada notebook, folder data, atau requirements.

Template: riset notebook-only (`structure-riset.md`), sesuai `CLAUDE.md` project. Sesuai D4 dan perintah Arya, rencana ini hanya membangun struktur: folder dan file kerangka, tanpa kode. Nama berkas di dalam folder, kolom, format, library, dan metrik tidak ditulis karena ditetapkan di 002. Label "MVP" di nama file mengikuti file `a`; project ini memakai satu fase saja (koreksi Arya).

## Peta keputusan → kode

Kolom Notebook berisi notebook yang menampung bagian itu. Jumlah, nama, dan nomor notebook ditentukan bersama Arya saat langkah 2 dikerjakan (koreksi Arya), sehingga di sini hanya disebut bagiannya.

| Keputusan | Folder / file | Isi kerangka (tahap) | Peran (writer-code) |
|---|---|---|---|
| K2 Penyiapan data | Notebook penyiapan data → `data/processed/` (masukan dari `data/raw/`) | Baca korpus/query/qrels, periksa kunci dan relasi, bagi val/test 50:50 seed 42, kunci split dengan hash | Notebook tahap; entitas Dokumen, Query, Penilaian relevansi |
| K1 Pembuat embedding | Notebook embedding → `data/embeddings/` (satu subfolder per embedding_id, dibuat saat notebook dijalankan) | Bentuk konfigurasi Set embedding (model, revision, max_seq_length 32, normalisasi L2) dan embedding_id, embed dokumen, embed query, simpan vektor beserta urutan id | Notebook tahap; entitas Set embedding, Vektor dokumen, Vektor query |
| K3 Exact search | Notebook exact → `data/embeddings/` (Tetangga exact, di samping vektor dengan embedding_id yang sama) dan `outputs/tuning/` (Run) | Baca vektor, cari brute-force di split val, simpan Tetangga exact, nilai, catat run | Dijalankan sebelum HNSW, IVF, LSH; entitas Tetangga exact |
| K3 HNSW, IVF, LSH | Notebook per algoritma → `outputs/tuning/` | Baca vektor dan Tetangga exact, bangun index, cari di split val, nilai, catat run. Implementasi LSH ditetapkan di 002 | Skenario setingkat |
| Penilai (002) | Tahap "Penilaian" di notebook exact, HNSW, IVF, LSH | Fungsi penilai disalin identik dan dicatat di tabel fungsi tersalin `CLAUDE.md` saat kodenya ditulis di 002 | Salinan identik (aturan notebook-only) |
| K4 Catatan run dan lingkungan | `outputs/tuning/`; tahap "Catat lingkungan" dan "Catat run" di notebook exact, HNSW, IVF, LSH | Run hanya ditambah, tidak ditimpa; lingkungan dicatat per sesi. Nama berkas dan kolom di 002 | Entitas Run, Lingkungan |
| K4 Tabel hasil | Notebook tabel hasil (membaca `outputs/tuning/`) | Baca catatan run keempat algoritma, tampilkan berdampingan untuk split val | Hanya membaca |
| D4 Kontrak berkas | `CLAUDE.md` bagian baru "Kontrak berkas": entitas → folder, bagian penulis, bagian pembaca | — | Kontrak antarnotebook (aturan wajib 1 template riset) |
| D4 Folder dijaga, data tidak ter-commit | `.gitignore`; `.gitkeep` di `data/raw/`, `data/processed/`, `data/embeddings/`, `outputs/tuning/` | — | Aturan umum template |

## Langkah

| # | Yang dibangun | File disentuh | Test | Gate (dari rencana-evaluasi) | Bergantung pada |
|---|---|---|---|---|---|
| 1 | Kerangka folder data dan keluaran, aturan gitignore, kontrak berkas | `.gitignore`, `data/raw/.gitkeep`, `data/processed/.gitkeep`, `data/embeddings/.gitkeep`, `outputs/tuning/.gitkeep`, `CLAUDE.md` | `git check-ignore -v`: berkas contoh di `data/` dan `outputs/` ter-ignore, `.gitkeep` tidak; `git status --porcelain` | — (belum ada `docs/rencana-evaluasi.md`) | — |
| 2 | Kerangka notebook (sel markdown saja) untuk setiap bagian di Peta. Jumlah, nama, dan nomornya ditentukan bersama Arya saat langkah ini dikerjakan; kalau lebih dari sekitar 5 notebook, langkah ini dipecah dan pemecahannya dicatat di kolom Pembangunan | `notebooks/*.ipynb`, `CLAUDE.md` (kolom notebook di Kontrak berkas) | `python -c`: tiap notebook JSON nbformat 4 valid, tanpa sel kode, sel pertama `# NN - ...` | — | 1 |
| 3 | README: tujuan project, susunan folder, urutan menjalankan notebook, prasyarat data mentah dari Arya; pemeriksaan D4 | `README.md` | Pemeriksaan D4: setiap bagian di Gambaran sistem dan setiap entitas di Model data tercantum di Kontrak berkas dan di salah satu notebook | — | 2 |

Isi kerangka notebook (langkah 2): sel pertama `# NN - Nama: tujuan`, alur tahap satu kalimat, folder keluaran, prasyarat, rujukan keputusan 001a; lalu satu sel `## n. Judul tahap` per tahap di Peta dengan rujukan K1–K4 dan keterangan "ditetapkan di 002" untuk hal yang ditunda. Tidak ada sel konstanta dan sel kode.

## Tidak dibangun di rencana ini

- Kode apa pun: fungsi, sel konstanta, sel kode notebook. Ditulis setelah 002.
- `requirements.txt` dan `requirements-dev.txt`: library ditetapkan di 002.
- `data/interim/`, `models/`, `outputs/metrics/`, `outputs/artifacts/`, `outputs/notebooks_eksekusi/`, `tuning_grids/`, `PROGRESS.md`: belum ada bagian di 001a yang menulis ke sana.
- Benchmark final di split test, artefak laporan, dan arsip: tidak tercantum di "Yang dirancang atau diubah" 001a; mengikuti rancangan berikutnya.
- Tabel fungsi tersalin di `CLAUDE.md`: dibuat saat fungsi penilai pertama kali ditulis (002).
- `config.py`, `.env.example`: tidak dipakai template notebook-only.

## Risiko teknis

- **Satu fase vs dokumen red-chan.** `docs/keputusan-produk.md` masih menulis `Fase: MVP` dan menunda sapuan parameter, benchmark final di split test, dan grafik laporan "sampai Dev"; K2 menyebut test dibuka di "benchmark final (Dev)". Rencana ini tidak mengubahnya; penyesuaiannya lewat red-chan.
- **Exact harus jalan lebih dulu.** K3 menyebut empat bagian setingkat, tetapi keluaran exact menjadi masukan Penilai ANN; penomoran notebook di langkah 2 harus menempatkan exact sebelum HNSW, IVF, LSH.
- **Tetangga exact disimpan di `data/embeddings/`,** bukan `outputs/`, karena hanya sah untuk embedding_id yang sama dan menjadi masukan notebook ANN. Kalau 002 memilih letak lain, cukup ubah Kontrak berkas.
- **Penilai disalin ke empat notebook.** Setiap perubahan kontrak metrik harus mengubah keempat salinan sekaligus.
- **`.gitignore` belum memuat `*.pem`, `*.key`, `credentials*.json`** yang diwajibkan `berkas-rahasia.md`; ditambahkan di langkah 1.

## Koreksi selama putaran

- "Tidak perlu ada fase MVP, dev dan prod." lalu "Cukup 1 fase saja untuk proyek ini" — rujukan fase Dev dan penomoran ulang notebook untuk Dev dihapus dari rencana.
- "Untuk jumlah notebook akan ditentukan saat progress berjalan" — nama dan nomor notebook tidak dikunci di rencana; ditentukan saat langkah 2 dikerjakan. Usulan 05 untuk tabel hasil val dan `01_eda` dihapus dari rencana.
- Kontrak berkas diletakkan di `CLAUDE.md` (pilihan Arya).
