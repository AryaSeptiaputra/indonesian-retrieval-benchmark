# 002b · MVP · Dataset, library, metrik

Status: disetujui 2026-10-02
Dari: docs/rancangan/002a_2026-10-02_mvp-dataset-library-metrik.md
Kondisi kode: belum ada kode. Hasil 001b: folder `data/raw/`, `data/processed/`, `data/embeddings/`, `outputs/tuning/` dengan `.gitkeep`; `.gitignore`; bagian "Kontrak berkas" di `CLAUDE.md`; `README.md`. Belum ada notebook dan requirements.

Cakupan (koreksi Arya): hanya `requirements.txt` dan dokumen `.md` yang mencatat keputusan 002a — tech stack, metrik evaluasi, dataset dan data, lingkungan eksekusi — ditambah pembaruan bagian "Belum diputuskan" di `CLAUDE.md`. Tidak ada kode dan tidak ada notebook. Dokumen tidak menambah keputusan; semua yang Belum pasti di 002a ditulis sebagai "belum diputuskan" dengan kode H1–H13. Sumber kebenaran tetap 002a dan `docs/keputusan-produk.md`; setiap dokumen menyatakannya di baris pembuka.

## Belum diputuskan (ditulis apa adanya di dokumen terkait)

| Kode | Hal | Dicatat di |
|---|---|---|
| H1 | Teks dokumen yang di-embed (title + text atau text saja) | tech-stack.md, dataset.md |
| H2 | Parameter tiap algoritma (HNSW M, efConstruction, efSearch; IVF nlist, nprobe, data latih; LSH nbits) dan ada/tidaknya sapuan di val | tech-stack.md |
| H3 | Tanda berhasil benchmark dan aturan memilih konfigurasi yang dikunci dari val | metrik-evaluasi.md |
| H4 | Cara mengukur QPS dan latensi p50 (satu per satu atau batch, pemanasan, alat ukur waktu) | metrik-evaluasi.md |
| H5 | Definisi metrik #12 (byte serialisasi index atau memori proses) | metrik-evaluasi.md |
| H6 | Penanganan dᴱˣᵢ = 0 pada #8 Relative distance error | metrik-evaluasi.md |
| H7 | Presisi encode (fp32/fp16), ukuran batch, pemeriksaan kesamaan dengan `model.encode` | tech-stack.md |
| H8 | Format berkas fisik tabel, vektor, dan metadata | dataset.md |
| H9 | Versi Python pasti (harus 3.10–3.13) | tech-stack.md, `requirements.txt` (komentar) |
| H10 | Tempat penyimpanan di luar Vast.ai | lingkungan-eksekusi.md |
| H11 | Field yang membentuk env_id: Lingkungan memuat nilai yang berubah tiap notebook (puncak VRAM/RAM, sisa disk, timestamp), sehingga hash seluruh isi tidak pernah sama antar-run | lingkungan-eksekusi.md |
| H12 | Perilaku pencatatan resource di luar Linux | lingkungan-eksekusi.md |
| H13 | Versi torch dan numpy, dikunci dari `pip freeze` instance Vast.ai (jawaban Arya) | tech-stack.md, `requirements.txt` (komentar) |

Diputuskan Arya lewat diskusi dengan koordinator, lalu diserahkan ke red-chan di pekerjaan berikutnya.

## Peta keputusan → kode

| Keputusan | Folder / file | Isi | Peran (writer-code) |
|---|---|---|---|
| K1, K3, K5 library | `requirements.txt` | Tepat tiga baris: `sentence-transformers==6.1.0`, `faiss-cpu==1.15.1`, `datasets==5.0.1`; komentar di atasnya: Python 3.10–3.13 (H9), torch dan numpy belum dikunci, menunggu `pip freeze` instance (H13) | Dependency produksi (2.9) |
| K1, K3, K5 | `docs/tech-stack.md` | Model, peran tiap library, loop encode, index FAISS, rumus L2 ↔ cosine, GPU/CPU, pendekatan yang ditolak; belum diputuskan H1, H2, H7, H9, H13 | Dokumen keputusan |
| K7, K4 | `docs/metrik-evaluasi.md` | Notasi, rumus 12 metrik apa adanya dari kartu K7, k = 5, p50, aturan slot −1, aturan #8, batas atas, kolom Run, asumsi exact; belum diputuskan H3–H6 | Dokumen keputusan |
| D3, K2, K5, model data | `docs/dataset.md` | Korpus, query, qrels, berkas sumber, relevan = relevance ≥ 1, split, sembilan entitas; belum diputuskan H1, H8 | Dokumen keputusan |
| K6, K8 | `docs/lingkungan-eksekusi.md` | Spesifikasi Vast.ai, aturan thread, satu instance satu sesi, salin keluar, resource otomatis dan manual, asumsi memori; belum diputuskan H10–H12 | Dokumen keputusan |
| K8 manual | `README.md` | Bagian "Lingkungan Vast.ai" berisi isian kosong untuk Arya; tautan ke empat dokumen; prasyarat merujuk `requirements.txt` dan H13 | Dokumen project |
| Status keputusan | `CLAUDE.md` bagian "Belum diputuskan" | Diganti dua daftar: yang sudah diputuskan di 002a (dengan tautan ke empat dokumen) dan yang masih belum diputuskan (H1–H13, lisensi model, tujuan keluaran, grafik laporan). Bagian "Ditetapkan Arya (dikunci)" dan "Kontrak berkas" tidak disentuh | Dokumen project (disetujui Arya, jawaban 3b) |

## Langkah

| # | Yang dibangun | File disentuh | Test | Gate (dari rencana-evaluasi) | Bergantung pada |
|---|---|---|---|---|---|
| 1 | Dependency dan tech stack | `requirements.txt`, `docs/tech-stack.md` | `python -c`: tepat tiga baris non-komentar `nama==versi` sesuai 002a, tanpa torch/numpy; `tech-stack.md` memuat K1, K3, K5 dan H1, H2, H7, H9, H13 | — (belum ada `docs/rencana-evaluasi.md`) | — |
| 2 | Metrik evaluasi | `docs/metrik-evaluasi.md` | `python -c`: 12 nomor metrik, `k = 5`, aturan slot −1, H3–H6 ada; setiap baris rumus identik dengan kartu K7 di 002a | — | — |
| 3 | Dataset, lingkungan eksekusi, README, CLAUDE.md | `docs/dataset.md`, `docs/lingkungan-eksekusi.md`, `README.md`, `CLAUDE.md` | `python -c`: angka 1.446.315, 960, 9.668, seed 42, spesifikasi K6 sama dengan 002a; sembilan entitas ada; H8, H10–H12 ada; README menautkan keempat dokumen dengan isian manual kosong; "Belum diputuskan" CLAUDE.md memuat H1–H13 dan bagian lain CLAUDE.md tidak berubah (`git diff`) | — | 1, 2 |

Tidak ada langkah yang terhambat.

## Tidak dibangun di rencana ini

- Notebook dan kode dari 002a (penyiapan data, embedding, exact, HNSW, IVF, LSH, tabel hasil val, benchmark final, arsip): lewat rencana nomor baru setelah H1–H13 diputuskan.
- torch dan numpy di `requirements.txt` (H13).
- `requirements-dev.txt` (koreksi Arya: cukup `requirements.txt`).
- `docs/rencana-evaluasi.md`: milik red-chan; dokumen metrik pink-chan diberi nama `metrik-evaluasi.md` supaya tidak tertukar.
- Kalimat di "Kontrak berkas" `CLAUDE.md` ("nama berkas, kolom, dan format ditetapkan di rancangan 002"): tidak diubah karena di luar persetujuan Arya; sebagian kini usang, format berkas masih H8.
- Grafik laporan, sapuan parameter, `tuning_grids/`.

## Risiko teknis

- **Salinan keputusan bisa menyimpang.** Empat dokumen dan `CLAUDE.md` menyalin isi 002a; keputusan yang berubah lewat red-chan harus ikut mengubah dokumen ini. Baris pembuka tiap dokumen menyatakan 002a dan `keputusan-produk.md` yang berlaku kalau berbeda.
- **torch dan numpy belum terkunci** sampai H13; sebelum itu versinya mengikuti image Vast.ai atau ditarik transitif oleh `sentence-transformers`.
- **Versi belum diperiksa terhadap instance.** Kecocokan tiga versi 002a dengan Python dan CUDA instance baru terbukti saat Arya menjalankan `pip install -r requirements.txt` di Vast.ai.
- **transformers transitif.** Versinya mengikuti `sentence-transformers==6.1.0` tanpa dikunci; dicatat K8 saat run.

## Koreksi selama putaran

- "Cukup buat file requirements.txt saja, jangan menulis kode program apapun dulu, cukup penulisan dokumen dalam bentuk file .md mulai dari matriks evaluasi, tech stack yang digunakan dan hasil dari diskusi lainnya tadi." — cakupan dipersempit dari sepuluh langkah notebook menjadi tiga langkah requirements dan dokumen.
- H1–H11 diputuskan Arya lewat diskusi dengan koordinator lalu diserahkan ke red-chan di pekerjaan berikutnya ("Ya, caranya sama seperti tadi").
- Commit dan push dilakukan koordinator setelah implementasi.
- [torch] a: hanya tiga paket dari 002a; torch dan numpy dicatat belum diputuskan (H13), dikunci nanti dari `pip freeze` instance.
- [Dokumen] empat dokumen terpisah: `docs/tech-stack.md`, `docs/metrik-evaluasi.md`, `docs/dataset.md`, `docs/lingkungan-eksekusi.md`.
- [CLAUDE.md] b: pink-chan memperbarui bagian "Belum diputuskan" di langkah 3; Arya memilih sendiri dan menyetujui perubahan CLAUDE.md.
- Persetujuan plan dihitung sebagai perintah Arya untuk ketiga langkah.
