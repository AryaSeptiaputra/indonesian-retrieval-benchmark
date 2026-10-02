# 006b · MVP · Pemanasan QPS

Status: disetujui 2026-10-03
Dari: docs/rancangan/006a_2026-10-03_mvp-pemanasan-qps.md
Kondisi kode: rencana 003b sedang dibangun (5 dari 12; notebook `00`–`02` ada). Rencana 005b selesai: `02_embedding` membaca cgroup v1 dan v2 dan berhenti di host hybrid (H18); versi cgroup tidak dicatat (H19). Fungsi `measure_qps` belum ditulis; tempatnya di 003b langkah 6 (`03_exact`, salinan pertama) lalu 7–9 dan 11. `docs/metrik-evaluasi.md` baru merujuk 006a untuk pemanasan QPS; `CLAUDE.md` mencatat "Pemanasan QPS" dengan rujukan langsung ke file 006a.

Rencana ini tidak mengubah 003b maupun 005b. 006a hanya mengubah isi pengukuran QPS (H17) dan mencatat perilaku sementara H18, H19; tidak ada bagian sistem atau notebook baru. Kodenya ditulis di langkah 003b yang sudah merencanakan `measure_qps`; rencana ini menetapkan isinya, pemeriksaan tambahannya, dan pembaruan dokumen.

## Hubungan dengan 003b

| Keputusan 006a | Dibangun di (003b) | Fungsi | Isi menurut 006a |
|---|---|---|---|
| K7 #9 pemanasan (H17) | Langkah 6 `03_exact` (salinan pertama), lalu 7–9 dan 11 | `measure_qps` | T₀ = `index.search(semua query split, k = 5)` dengan n thread, tidak diukur dan hasilnya tidak dipakai; lalu T₁..T₅ diukur dengan `time.perf_counter_ns` yang hanya membungkus `index.search`; QPS = n_query ÷ median(T₁..T₅). T₀ dijalankan sekali per konfigurasi, tepat sebelum T₁..T₅ konfigurasi itu (efSearch/nprobe diganti pada index yang sama sebelum T₀) |
| H18, H19 perilaku sementara | Sudah dibangun di `02_embedding` (005b langkah 1); disalin identik ke `03`–`06` | `fetch_cgroup_cpu_quota`, `fetch_memory_quota`, `collect_environment` | Host hybrid → berhenti dengan error sebelum mengukur apa pun; versi cgroup tidak dicatat |

Urutan p50, QPS, dan #12 di dalam satu konfigurasi tidak diputuskan 006a, kecuali T₀ tepat sebelum T₁..T₅. Notebook memakai urutan yang sama di semua salinan; urutannya disebut di laporan 003b langkah 6.

Pemeriksaan tambahan saat langkah 003b di atas dikerjakan (cara memeriksa mengikuti 003b):

| Langkah 003b | Lokal (pink-chan, `.venv` pustaka kecil) | Vast.ai (Arya) |
|---|---|---|
| 6 | `measure_qps` dengan index palsu (objek berisi `search` buatan yang mencatat panggilan dan waktunya): tepat 6 panggilan batch berisi semua query; panggilan pertama tidak masuk perhitungan; hasil = n_query ÷ median 5 waktu berikutnya; T₀ tidak tercatat di Run | QPS terisi untuk run exact |
| 7–9, 11 | Salinan `measure_qps` identik dengan `03`; T₀ dijalankan ulang untuk setiap nilai efSearch, nprobe, dan nbits | QPS terisi untuk 16 run val dan run test |

## Peta keputusan → kode

| Keputusan | Folder / file | Isi | Peran (writer-code) |
|---|---|---|---|
| K7 #9 pemanasan (H17) | `docs/metrik-evaluasi.md` | Baris #9 di tabel pengukuran memuat T₀; kartu K7 #9 006a apa adanya; bagian "Belum diputuskan" tidak lagi menyebut 006b | Dokumen keputusan |
| H18, H19 perilaku sementara | `docs/lingkungan-eksekusi.md` | Baris H18 dan H19 memuat perilaku sementara menurut 006a (berhenti dengan error; tidak dicatat) | Dokumen keputusan |
| Status keputusan | `CLAUDE.md` bagian "Belum diputuskan" | Baris "Pemanasan QPS" merujuk `docs/metrik-evaluasi.md`; status H18 dan H19 memuat perilaku sementara | Dokumen project |

## Langkah

| # | Yang dibangun | File disentuh | Test | Gate (dari rencana-evaluasi) | Bergantung pada |
|---|---|---|---|---|---|
| 1 | Dokumen turunan sesuai 006a (tanpa kode) | `docs/metrik-evaluasi.md`, `docs/lingkungan-eksekusi.md` | `python -c` (pustaka standar): kartu K7 #9 006a tersalin identik; baris #9 menyebut T₀ yang tidak diukur; tidak ada lagi rujukan "lewat rencana 006b"; H18 dan H19 memuat perilaku sementara; kode belum diputuskan di empat dokumen tetap tepat H9, H10, H12, H13, H16, H18, H19 | — (belum ada `docs/rencana-evaluasi.md`) | — |
| 2 | `CLAUDE.md` "Belum diputuskan" sesuai 006a (tanpa kode) | `CLAUDE.md` | `python -c`: baris "Pemanasan QPS" merujuk `docs/metrik-evaluasi.md`; status H18 dan H19 memuat perilaku sementara; tabel H tetap tepat H9, H10, H12, H13, H16, H18, H19; perubahan hanya di bagian itu | — | 1 |

## Tidak dibangun di rencana ini

- `measure_qps` dan notebook `03`–`06`: ditulis di 003b langkah 6–9 dan 11.
- Perubahan isi file 003b dan 005b.
- Kolom T₀ di Run: 006a menetapkan T₀ tidak dicatat dan model data tidak berubah.
- H18 dan H19: tetap belum pasti; perilaku sementaranya sudah dibangun di 005b langkah 1.

## Risiko teknis

- **T₀ untuk LSH.** LSH tidak punya parameter search; tiap nbits adalah build terpisah, sehingga T₀ dijalankan sekali per build (3 kali), sesuai "sekali per konfigurasi".
- **T₀ di benchmark final.** `06_final_benchmark` menjalankan T₀ untuk setiap konfigurasi terkunci di split test, termasuk run ulang exact; T₀ memakai query test tetapi hasilnya dibuang dan tidak memengaruhi metrik.
- **Waktu tambahan.** T₀ menambah satu panggilan batch per konfigurasi (16 untuk val termasuk exact, 5 untuk test termasuk exact ulang); untuk exact pada 480 query, satu panggilan berjalan di jalur BLAS dan biayanya sebanding dengan satu ulangan QPS.

## Koreksi selama putaran

- Disetujui Arya tanpa koreksi. Perubahan `CLAUDE.md` di langkah 2 disetujui Arya sendiri.
