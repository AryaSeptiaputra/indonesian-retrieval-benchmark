# 007b · MVP · Penjaga test

Status: disetujui 2026-10-03
Dari: docs/rancangan/007a_2026-10-03_mvp-penjaga-test.md
Kondisi kode: rencana 003b 11 dari 12; notebook `00`–`06` sudah dibangun dan di-commit (3b5e404). `06_final_benchmark` memeriksa hash Kunci konfigurasi dan env_id sebelum membaca query test, tetapi belum punya penjaga run ulang; baris run test memakai 25 kolom yang sama dengan run val (`build_run`, salinan identik di 03, 04a–04c, 06). `05_val_results` sudah berhenti kalau satu konfigurasi val tercatat lebih dari sekali (H20, dibangun di 003b langkah 10). `requirements.txt` masih tiga paket; 003b langkah 12 menunggu keluaran instance. `runs_test.csv` belum pernah ditulis (asumsi 007a).

Rencana ini tidak mengubah file b lain. Pembagiannya:
- **K12, K4 (kolom test), D4 (percobaan terakhir)** — dibangun di `06_final_benchmark` lewat langkah 1.
- **K11 / H20** — perilakunya sudah ada di `05_val_results`; tidak ada kode baru, hanya dicatat di dokumen.
- **K13** — penguncian torch, numpy, pyarrow, pandas dikerjakan di 003b langkah 12, yang sudah merencanakan keempat paket itu; 007a menjawab H16 ("dikunci"), sehingga langkah itu tinggal menunggu `python --version` dan `pip freeze` instance (H9, H13).
- **Dokumen dan `CLAUDE.md`** — langkah 2 dan 3.

## Peta keputusan → kode

| Keputusan | Folder / file | Fungsi utama | Peran (writer-code) |
|---|---|---|---|
| K12 penjaga test | `notebooks/06_final_benchmark.ipynb`, sel konstanta dan tahap baru setelah pemeriksaan Kunci dan env_id, sebelum tahap "Buka split test" | Konstanta `IZIN_BUKA_ULANG = False`, `ALASAN_BUKA_ULANG = ""`; `load_test_runs` (baca `outputs/metrics/runs_test.csv` kalau ada, tanpa membaca data test), `decide_test_attempt` (P = baris dengan embedding_id Kunci; P kosong → 1; P ada dan izin mati → berhenti dengan jumlah percobaan dan cap waktu; izin aktif tanpa alasan → berhenti; izin aktif dengan alasan → cetak peringatan, kembalikan max(test_attempt) + 1) | Pengakses luar, pemeriksa |
| K4 kolom Run test | `06_final_benchmark`, tahap catat run | `to_test_run` menambahkan `config_lock_hash` (content_hash Kunci), `test_attempt`, `reopen_reason` ke baris dari `build_run`; `build_run` dan `append_run` tetap salinan identik, sehingga kolom run val tidak berubah | Pengubah bentuk |
| D4 percobaan terakhir | `06_final_benchmark`, tahap D4 | `validate_success_criteria` tetap menilai baris eksekusi ini (percobaan terakhir); laporan mencetak nomor percobaan, jumlah percobaan untuk embedding_id itu, dan alasan buka ulang | Pemeriksa |
| K11 / H20 | `notebooks/05_val_results.ipynb` (sudah ada) | `validate_run_set` berhenti kalau konfigurasi ganda | — (tidak diubah) |
| K13 | `requirements.txt`, `docs/tech-stack.md` | Dikerjakan di 003b langkah 12 | — |
| Dokumen turunan | `docs/metrik-evaluasi.md`, `docs/dataset.md`, `docs/tech-stack.md`, `README.md` | Kartu K12, K11 (H20), K13 007a apa adanya; kolom Run test; relasi Kunci 1─N Run test; cara memakai izin buka ulang di README | Dokumen keputusan |
| Status keputusan, Kontrak berkas | `CLAUDE.md` bagian "Belum diputuskan" dan "Kontrak berkas" | H16 pindah ke "sudah diputuskan" (nilai versi tetap menunggu instance bersama H9, H13); H20 dicatat sementara; baris Run dan Kunci konfigurasi di Kontrak berkas memuat kolom test dan penjaga | Dokumen project |

## Langkah

| # | Yang dibangun | File disentuh | Test | Gate (dari rencana-evaluasi) | Bergantung pada |
|---|---|---|---|---|---|
| 1 | Penjaga test, kolom Run test, laporan percobaan di `06` (kode) | `notebooks/06_final_benchmark.ipynb` | Lokal (`.venv`, faiss/torch palsu, rantai 03 → 04a–c → 05 → 06 dengan data palsu): eksekusi pertama test_attempt 1 dan reopen_reason kosong; eksekusi kedua dengan izin mati berhenti sebelum query test dibaca dan tanpa baris baru; izin aktif tanpa alasan berhenti; izin aktif dengan alasan menghasilkan test_attempt 2 dan alasan tercatat; Kunci lain dengan embedding_id sama tetap berhenti; embedding_id lain tidak terhalang; 28 kolom run test (25 + 3), run val tetap 25 kolom; salinan fungsi tetap identik dengan sumbernya; statis (type hint, docstring, ≤ 50 baris, ≤ 120 karakter) | — (belum ada `docs/rencana-evaluasi.md`) | — |
| 2 | Dokumen turunan sesuai 007a (tanpa kode) | `docs/metrik-evaluasi.md`, `docs/dataset.md`, `docs/tech-stack.md`, `README.md` | `python -c`: kartu K12, K11, K13 007a tersalin identik; kolom `config_lock_hash`, `test_attempt`, `reopen_reason` di tabel Run; H16 tidak lagi di "Belum diputuskan" dan H20 tercatat sementara; README menjelaskan `IZIN_BUKA_ULANG` dan `ALASAN_BUKA_ULANG` | — | 1 |
| 3 | `CLAUDE.md` sesuai 007a (tanpa kode) | `CLAUDE.md` | `python -c`: tabel H berisi H9, H10, H12, H13, H18, H19, H20; baris Run dan Kunci konfigurasi di Kontrak berkas memuat aturan K12 dan kolom test; perubahan hanya di "Belum diputuskan" dan "Kontrak berkas" | — | 2 |

## Tidak dibangun di rencana ini

- Penguncian versi di `requirements.txt` (K13): dikerjakan di 003b langkah 12 setelah keluaran instance ada.
- Perubahan `05_val_results`: perilaku H20 sudah sesuai 007a.
- Perubahan `build_run` dan kolom run val: kolom tambahan hanya untuk split test (007a model data).
- Perubahan file 003b–006b.

## Risiko teknis

- **Baris test lama tanpa kolom baru.** `append_run` berhenti kalau kolom berkas lama berbeda. Asumsi 007a: `runs_test.csv` masih kosong saat 007 dibangun; kalau ternyata sudah ada baris 25 kolom dari uji sebelumnya, notebook berhenti dan dilaporkan (menghapus baris lama melanggar K4).
- **Izin tertinggal aktif.** Penjaga tidak bisa tahu izin lupa dikembalikan; peringatan dicetak dan alasan tercatat di setiap baris (asumsi 007a).
- **Penjaga bergantung pada `runs_test.csv` di instance.** Kalau berkasnya tidak ikut disalin ke instance berikutnya (H10), penjaga tidak melihat percobaan sebelumnya.
- **Urutan pemeriksaan.** Penjaga diletakkan setelah pemeriksaan hash dan env_id dan sebelum tahap "Buka split test", sehingga eksekusi yang berhenti tidak pernah membaca query test.

## Koreksi selama putaran

- Disetujui Arya tanpa koreksi; kode notebook 06 (langkah 1) dan perubahan `CLAUDE.md` (langkah 3) disetujui Arya sendiri. Langkah 1–3 dikerjakan berurutan dalam satu putaran.
