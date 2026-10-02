# 004b · MVP · Seed dan env_id

Status: disetujui 2026-10-02
Dari: docs/rancangan/004a_2026-10-02_mvp-seed-env-id.md
Kondisi kode: belum ada kode atau notebook. Rencana 003b sedang dibangun (1 dari 12): dokumen turunan (`docs/tech-stack.md`, `docs/metrik-evaluasi.md`, `docs/dataset.md`, `docs/lingkungan-eksekusi.md`) sudah memuat isi 003a dan menulis H14, H15 sebagai "menunggu rancangan 004". Bagian "Belum diputuskan" di `CLAUDE.md` belum diperbarui (003b langkah 2 belum diperintahkan).

Rencana ini tidak mengubah 003b. 004a hanya mengubah isi keputusan (nilai seed dan field env_id), bukan bagian sistem, jadi tidak ada notebook baru. Kodenya ditulis di langkah 003b yang sudah merencanakan fungsi tersebut ("seed menurut H14", "hash menurut H15"); 004a menjadi jawaban H14 dan H15 untuk langkah-langkah itu. Rencana ini membangun pembaruan dokumen dan menetapkan pemeriksaan tambahan yang dipakai saat langkah 003b itu dikerjakan.

## Hubungan dengan 003b

| Keputusan 004a | Dibangun di | Fungsi (nama dari 003b) | Isi menurut 004a |
|---|---|---|---|
| K10 seed IVF (H14) | 003b langkah 8, `notebooks/04b_ivf.ipynb`; salinan di 003b langkah 11, `06_final_benchmark` | `train_ivf_index`, `build_run` | Seed k-means tidak diubah (bawaan FAISS 1234); nilainya dibaca dari parameter clustering index dan dicatat di params Run bersama jumlah sampel latih 1.048.576 |
| K10 seed LSH (H14) | 003b langkah 9, `notebooks/04c_lsh.ipynb`; salinan di 003b langkah 11 | `build_lsh_index`, `build_run` | `IndexLSH` dibuat dengan `rotate_data = true` dan `train_thresholds = false` (bawaan); params Run memuat seed rotasi 5 dengan tanda "nilai dari kode sumber FAISS 1.15.1" |
| K8 env_id (H15) | 003b langkah 5, `notebooks/02_embedding.ipynb` (salinan pertama); salinan di langkah 6–9, 11 | `collect_environment`, `compute_env_id` | Hash semua field statis (model GPU, VRAM total, driver/CUDA, model CPU, flag AVX2/AVX-512, jatah vCPU, total vCPU mesin, jatah RAM, total disk, thread FAISS, thread torch, versi Python dan library, revision model dan dataset) + hostname; field dinamis (puncak RAM dan VRAM, sisa disk, timestamp) dicatat tetapi tidak masuk hash |

Pemeriksaan tambahan yang dijalankan saat langkah 003b di atas dikerjakan (cara memeriksa mengikuti 003b: `.venv` lokal hanya pustaka kecil, sisanya Arya di Vast.ai):

| Langkah 003b | Lokal (pink-chan) | Vast.ai (Arya) |
|---|---|---|
| 5 | `compute_env_id` dengan data palsu: hasil sama untuk field statis sama; berubah kalau salah satu field statis K8 004a atau hostname berubah (termasuk revision); tidak berubah kalau hanya field dinamis berubah | `environment_<env_id>.json` memuat semua field statis dan dinamis |
| 6–9, 11 | Salinan `collect_environment` dan `compute_env_id` identik dengan `02` | env_id notebook `03`–`06` sama dalam satu instance |
| 8 | `build_run` IVF memuat seed dan jumlah sampel latih | Seed yang tercatat = 1234 dan dibaca dari index, bukan ditulis tangan |
| 9 | `build_run` LSH memuat seed 5 dengan tandanya | — |

## Peta keputusan → kode

| Keputusan | Folder / file | Isi | Peran (writer-code) |
|---|---|---|---|
| K10 seed (H14) | `docs/tech-stack.md` | Kartu K10 seed 004a apa adanya; baris seed di tabel K10; H14 keluar dari "Belum diputuskan"; penolakan "seed IVF = 42" | Dokumen keputusan |
| K8 env_id (H15) | `docs/lingkungan-eksekusi.md` | Kartu K8 004a apa adanya menggantikan kartu 003a; tabel field statis vs dinamis; H15 keluar dari "Belum diputuskan"; penolakan "env_id hanya field H11" | Dokumen keputusan |
| Model data Run, Lingkungan | `docs/dataset.md`, `docs/metrik-evaluasi.md` | Run: params memuat seed k-means IVF, jumlah sampel latih, dan seed rotasi LSH; Lingkungan: kunci = hash semua field statis + hostname | Dokumen keputusan |
| Status H | `CLAUDE.md` bagian "Belum diputuskan" | H14 dan H15 dipindah ke "sudah diputuskan" dengan rujukan 004a | Dokumen project |

## Langkah

| # | Yang dibangun | File disentuh | Test | Gate (dari rencana-evaluasi) | Bergantung pada |
|---|---|---|---|---|---|
| 1 | Dokumen turunan sesuai 004a (tanpa kode) | `docs/tech-stack.md`, `docs/lingkungan-eksekusi.md`, `docs/dataset.md`, `docs/metrik-evaluasi.md` | `python -c` (pustaka standar): kartu K10 seed dan K8 004a tersalin identik; kartu K8 003a tidak lagi tercantum sebagai yang berlaku; H14 dan H15 tidak ada di "Belum diputuskan"; kode tersisa tepat H4, H5, H6, H9, H10, H12, H13, H16; params Run memuat seed rotasi LSH di dataset dan metrik-evaluasi | — (belum ada `docs/rencana-evaluasi.md`) | — |
| 2 | `CLAUDE.md` "Belum diputuskan" sesuai 004a (tanpa kode) | `CLAUDE.md` | `python -c`: tabel H berisi tepat H4, H5, H6, H9, H10, H12, H13, H16; `git diff` hanya menyentuh bagian "Belum diputuskan" | — | 1 dan 003b langkah 2 (bagian itu ditulis ulang di sana) |

## Tidak dibangun di rencana ini

- Notebook dan kode: dibangun di 003b langkah 5–11 seperti tabel "Hubungan dengan 003b".
- Perubahan isi 003b: 003b sudah disetujui dan tidak diubah. Status penghambat H14 dan H15 di 003b dicatat terjawab lewat laporan dan kolom Pembangunan saat langkah 003b dikerjakan.
- H16 (penguncian pyarrow dan pandas): tetap belum dijadwalkan.

## Risiko teknis

- **Seed LSH tidak bisa dibaca dari objek.** Nilai 5 hanya benar untuk `faiss-cpu==1.15.1` dengan `rotate_data = true`. `requirements.txt` sudah mengunci 1.15.1; konstruktor `IndexLSH` di notebook menulis `rotate_data` dan `train_thresholds` secara eksplisit supaya asumsi 004a terlihat di kode.
- **Seed IVF dibaca dari index.** Atribut parameter clustering pada `IndexIVFFlat` hanya bisa diperiksa di Vast.ai (faiss tidak dipasang lokal); kalau atributnya tidak tersedia di 1.15.1, langkah 003b-8 berhenti dan dilaporkan, nilai tidak ditulis tangan.
- **Daftar library di env_id.** "Versi Python dan library" harus berupa daftar paket tetap yang sama di semua salinan, dibaca lewat `importlib.metadata` tanpa meng-import paketnya; kalau daftar berbeda antar-notebook, env_id `03`–`06` tidak akan sama. Daftarnya ditulis di sel konstanta dan diperiksa identik.
- **Field GPU di notebook search.** Notebook `03`–`06` tidak memakai GPU tetapi env_id memuat model GPU, VRAM total, dan driver/CUDA; field ini harus dibaca dengan cara yang sama di semua notebook.
- **Hostname instance.** env_id memuat hostname; kalau hostname kontainer Vast.ai berubah saat instance di-restart, notebook final berhenti walaupun mesinnya sama.
- **Urutan dengan 003b langkah 2.** Langkah 2 rencana ini menimpa bagian yang ditulis 003b langkah 2; kalau 003b langkah 2 dikerjakan lebih dulu, tabel H di `CLAUDE.md` sempat memuat H14 dan H15 sampai langkah 2 ini selesai.

## Koreksi selama putaran

- Disetujui Arya tanpa koreksi. Perubahan `CLAUDE.md` di langkah 2 disetujui Arya sendiri.
