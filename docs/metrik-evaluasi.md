# Metrik Evaluasi

Ringkasan keputusan K7, K4, K11, dan D4 dari rancangan 002a, 003a, dan 004a. Kalau isi dokumen ini berbeda dengan file `a` yang disetujui (`docs/rancangan/002a_2026-10-02_mvp-dataset-library-metrik.md`, `docs/rancangan/003a_2026-10-02_mvp-parameter-format-encode.md`, `docs/rancangan/004a_2026-10-02_mvp-seed-env-id.md`) atau `docs/keputusan-produk.md`, dokumen-dokumen itu yang berlaku. Dokumen ini bukan `docs/rencana-evaluasi.md` (milik red-chan).

## Ringkasan

12 metrik dikunci, k = 5 untuk semua algoritma, latensi memakai p50. Semua metrik per query dirata-rata atas query di split yang dijalankan.

| Kelompok | Metrik | Dibandingkan dengan |
|---|---|---|
| Kualitas | (1) nDCG@5, (2) Recall@5, (3) MRR@5, (4) Precision@5, (5) MAP@5, (6) Hit rate@5 | Qrels (relevan = relevance ≥ 1) |
| Kesetiaan ANN | (7) k-NN Recall@5, (8) Relative distance error | Tetangga exact (`IndexFlatL2`) |
| Efisiensi | (9) QPS, (10) Latensi p50, (11) Waktu build/train index, (12) Ukuran index / memori | — |

## Rumus (kartu K7 002a, apa adanya)

```
K7 · Evaluation metrics — Umum
k            5 untuk semua algoritma; latensi p50
Notasi       Rel(q) = dokumen relevance ≥ 1; relᵢ = 1 kalau hasil ke-i ∈ Rel(q);
             slot −1 → relᵢ = 0
1  nDCG@5        = DCG@5 / IDCG@5;  DCG@5 = Σᵢ₌₁⁵ relᵢ / log₂(i + 1)
                   IDCG@5 = DCG@5 dengan min(|Rel(q)|, 5) relevan di atas
2  Recall@5      = |Top5 ∩ Rel(q)| / |Rel(q)|
3  MRR@5         = 1 / peringkat relevan pertama di top-5; 0 kalau tidak ada
4  Precision@5   = |Top5 ∩ Rel(q)| / 5
5  MAP@5         = rata-rata AP@5;
                   AP@5 = (1 / min(|Rel(q)|, 5)) · Σᵢ₌₁⁵ P@i · relᵢ
6  Hit rate@5    = 1 kalau |Top5 ∩ Rel(q)| ≥ 1, selain itu 0
7  k-NN Recall@5 = |ANN₅(q) ∩ Exact₅(q)| / 5
8  Rel. distance error = (1/5) · Σᵢ₌₁⁵ (dᴬᴺᴺ₍ᵢ₎ − dᴱˣᵢ) / dᴱˣᵢ
                   d = √(skor L2 kuadrat FAISS)
                   LSH: d = ‖q − xᵢ‖₂ dihitung ulang dari vektor asli
                   dᴬᴺᴺ₍ᵢ₎ = jarak ke-i setelah lima jarak ANN diurutkan menaik
                   slot −1: d = 2 (jarak L2 maksimum vektor bernorma 1)
9  QPS           = jumlah query diukur / total waktu cari (s)
10 Latensi p50   = median waktu cari per query (ms)
11 Waktu build/train index (s)
12 Ukuran index / memori
Slot −1      Metrik 1–7: tidak relevan / tidak cocok, pembagi tetap 5;
             #8: penalti jarak 2; jumlah query dengan hasil < 5 dicatat
             sebagai kolom diagnostik; −1 tidak boleh dipakai sebagai
             indeks array
Batas atas   960 query dev: Precision@5 ≤ 0,567, Recall@5 ≤ 0,953 (rata-rata)
Belum pasti  Cara ukur QPS/p50 (satu per satu atau batch, pemanasan);
             definisi #12 (byte serialisasi atau memori proses);
             dᴱˣᵢ = 0 pada #8; tanda berhasil benchmark
```

Keterangan tambahan dari `docs/keputusan-produk.md`:
- P@i = jumlah relevan di peringkat 1..i / i.
- dᴱˣᵢ = jarak ke-i hasil `IndexFlatL2`.
- HNSW dan IVF sudah terurut L2; hasil LSH terurut Hamming sehingga lima jarak L2-nya wajib diurutkan ulang menaik sebelum #8 (pilihan Arya 002 titik periksa 3a).
- Pembagi AP@5 = min(|Rel(q)|, 5) supaya setiap query bisa mencapai 1,0 (pilihan Arya 002 titik periksa 4a).

## Hasil kurang dari 5 (slot −1)

FAISS mengembalikan ID −1 kalau hasil kurang dari 5, terutama dari IVF.

| Metrik | Aturan |
|---|---|
| Kualitas (1–6) dan k-NN Recall@5 (7) | Slot −1 dianggap tidak relevan atau tidak cocok; pembagi tetap 5 (Precision@5, k-NN Recall@5) |
| Relative distance error (8) | Slot −1 diberi penalti jarak 2 (jarak L2 maksimum dua vektor bernorma 1) |
| Diagnostik | Jumlah query dengan hasil kurang dari 5 dicatat sebagai kolom di setiap run, di luar 12 metrik |
| Kode | −1 tidak boleh dipakai sebagai indeks array dan tidak pernah disimpan sebagai doc_id |

Penalti 2 hanya sah karena vektor wajib bernorma 1 (aturan integritas Vektor dokumen dan Vektor query).

## Pembacaan angka

- Batas atas rata-rata pada 960 query dev: Precision@5 = 0,567 dan Recall@5 = 0,953 (15,2% query punya lebih dari 5 dokumen relevan).
- nDCG@5 dan Recall@5 tidak sebanding langsung dengan angka resmi MIRACL (nDCG@10, Recall@100).
- Exact menghasilkan k-NN Recall@5 = 1,0 dan Relative distance error = 0 menurut definisinya.
- Angka efisiensi hanya dibandingkan antar-run dengan Lingkungan yang sama (lihat `docs/lingkungan-eksekusi.md`).

## K4 · Catatan run

Setiap run menambah satu baris yang tidak pernah ditimpa. Satu konfigurasi sapuan = satu run (16 run di val, lihat K10 di `docs/tech-stack.md`). Tabel hasil val dan hasil benchmark final disusun dari catatan run. Format: baris CSV yang hanya ditambah (K9).

| Kolom | Isi |
|---|---|
| run_id | Surrogate |
| timestamp | Waktu run |
| embedding_id | Rujukan Set embedding |
| env_id | Rujukan Lingkungan |
| algorithm | flat, hnsw, ivf, atau lsh |
| params | Parameter build dan search; IVF juga jumlah sampel latih dan seed k-means; LSH juga seed rotasi (004a) |
| split | val atau test |
| k | 5 |
| thread FAISS, thread torch | Jumlah thread yang diset (K6) |
| 12 metrik K7 | Nilai rata-rata atas query |
| jumlah query dengan hasil < 5 | Kolom diagnostik |

## K11 · Memilih konfigurasi untuk test

Per algoritma, dari run val: konfigurasi dengan QPS tertinggi di antara yang k-NN Recall@5 ≥ 0,95; kalau tidak ada yang mencapai, konfigurasi dengan k-NN Recall@5 tertinggi. Hasilnya ditulis ke Kunci konfigurasi (JSON, cap waktu + hash isi) sebelum test dibuka. Kartu K11 003a, apa adanya:

```
K11 · Configuration selection — Umum  ★ H3
Rumus        c*(a) = argmax { QPS(c) : c ∈ Cₐ, R(c) ≥ 0,95 }  kalau tidak kosong
             c*(a) = argmax { R(c)   : c ∈ Cₐ }                kalau kosong
             Cₐ = konfigurasi algoritma a di val; R = k-NN Recall@5 di val
Kunci        Hasil ditulis ke Kunci konfigurasi (JSON, cap waktu + hash isi)
             sebelum test dibuka; exact = satu-satunya konfigurasinya
Bergantung   H4 (cara ukur QPS) dijawab sebelum notebook search pertama
```

## Benchmark final

Pemilihan konfigurasi hanya di split val. Notebook benchmark final — notebook terakhir — berhenti kalau hash Kunci konfigurasi atau env_id tidak cocok dengan run val, lalu membuka split test sekali untuk keempat konfigurasi terkunci dan menjalankan ulang exact (K2).

## D4 · Tanda berhasil benchmark

Kartu D4 003a, apa adanya:

```
D4 · Benchmark success criteria — ✎ H3
1            12 metrik lengkap untuk keempat algoritma di split test
2            Exact: k-NN Recall@5 = 1 dan Relative distance error = 0
             (bergantung H6 untuk kasus jarak exact = 0)
3            Run ulang exact di notebook final yang sama → nDCG@5 identik
```

## Belum diputuskan

| Kode | Hal | Dijawab paling lambat |
|---|---|---|
| H4 | Cara mengukur QPS dan latensi p50: query satu per satu atau batch, pemanasan, alat ukur waktu | Sebelum notebook search pertama |
| H5 | Definisi metrik #12: byte hasil serialisasi index atau memori proses | Sebelum notebook search pertama |
| H6 | Penanganan dᴱˣᵢ = 0 pada #8 Relative distance error (pembagian dengan nol) | Saat menulis penilai |
