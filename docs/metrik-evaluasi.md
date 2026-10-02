# Metrik Evaluasi

Ringkasan keputusan K7, K4, K11, dan D4 dari rancangan 002a, 003a, 004a, 005a, dan 006a. Kalau isi dokumen ini berbeda dengan file `a` yang disetujui (`docs/rancangan/002a_2026-10-02_mvp-dataset-library-metrik.md`, `docs/rancangan/003a_2026-10-02_mvp-parameter-format-encode.md`, `docs/rancangan/004a_2026-10-02_mvp-seed-env-id.md`, `docs/rancangan/005a_2026-10-02_mvp-pengukuran-cgroup.md`, `docs/rancangan/006a_2026-10-03_mvp-pemanasan-qps.md`) atau `docs/keputusan-produk.md`, dokumen-dokumen itu yang berlaku. Dokumen ini bukan `docs/rencana-evaluasi.md` (milik red-chan).

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

## Pengukuran efisiensi (005a)

| Metrik | Cara ukur |
|---|---|
| #10 Latensi p50 | `time.perf_counter_ns` hanya membungkus `index.search(1 query, k = 5)`; 10 query pemanasan sekali sebelum putaran pertama, tidak dihitung; 3 putaran atas semua query split; p50 = median semua pengukuran (ms) |
| #9 QPS | Semua query split dalam satu panggilan `index.search` dengan n thread, 5 ulangan T₁..T₅; QPS = n_query ÷ median(T₁..T₅). Sebelum T₁ dijalankan satu panggilan batch pemanasan T₀ yang tidak diukur dan hasilnya tidak dipakai, sekali per konfigurasi (006a) |
| #12 Ukuran index | `len(faiss.serialize_index(index))` byte; puncak RAM proses dicatat di Lingkungan |

QPS dan p50 mengukur hal berbeda (QPS ≠ 1000 / p50); K11 memakai QPS batch. Kartu K7 005a, apa adanya:

```
K7 · Latency and throughput measurement — Umum [S15]  ✎ H4
Alat ukur    time.perf_counter_ns, hanya membungkus index.search;
             memuat data dan menghitung metrik tidak termasuk
p50 (#10)    t_{r,q} = waktu index.search(1 query, k = 5)
             10 query pemanasan sekali sebelum putaran pertama, tidak dihitung
             r = 1..3 putaran atas semua query split
             p50 = median { t_{r,q} } (ms)
QPS (#9)     Tⱼ = waktu index.search(semua query split, k = 5) dengan n thread
             j = 1..5;  QPS = n_query / median(T₁, …, T₅)
Catatan      QPS ≠ 1000 / p50: panggilan 1 query praktis satu thread; untuk
             exact, 1 query memakai jalur jarak langsung dan batch ≥ 167 query
             memakai jalur BLAS (nq · d ≥ 128.000). K11 memakai QPS batch
```

```
K7 · Index size (#12) — Umum  ✎ H5
Definisi     #12 = len(faiss.serialize_index(index)) byte
Catatan      Serialisasi membuat salinan sementara sebesar index (exact ≈ 4,44 GB,
             HNSW ≈ 4,8 GB); buffer dilepas segera setelah panjangnya diambil;
             puncak RAM proses dicatat di Lingkungan (field dinamis K8)
```

Kartu K7 #9 006a, apa adanya:

```
K7 #9 · QPS warm-up — Umum  ✎ H17
Pemanasan    T₀ = index.search(semua query split, k = 5) dengan n thread;
             tidak diukur, hasilnya tidak dipakai
Pengukuran   Tⱼ = waktu index.search(semua query split, k = 5) dengan n thread,
             j = 1..5, dijalankan setelah T₀; time.perf_counter_ns, hanya
             index.search
Rumus        QPS = n_query / median(T₁, …, T₅)   (tidak berubah dari 005a)
Frekuensi    Sekali per konfigurasi, tepat sebelum 5 ulangan QPS konfigurasi
             itu (asumsi: sapuan efSearch/nprobe mengganti parameter search
             pada index yang sama)
```

## #8 Relative distance error di dekat nol (005a)

Skor L2² dipotong ke 0 sebelum diakarkan. Suku dengan dᴱˣᵢ ≤ 1e-3 dikeluarkan; RDE per query dirata-rata atas suku yang tersisa; query tanpa suku tersisa tidak ikut rata-rata antarquery. Tanpa suku yang dikeluarkan, hasilnya sama dengan rumus 002. Suku 0/0 milik exact ikut dikeluarkan sehingga #8 exact = 0 (D4). Kartu K7 005a, apa adanya (menggantikan baris #8 di kartu 002a untuk kasus dᴱˣᵢ kecil):

```
K7 · Relative distance error (#8) near-zero rule — Umum [S15]  ✎ H6
Jarak        d = √max(skor L2² FAISS, 0)
Ambang       τ = 1e-3 pada d (setara skor L2² ≤ 1e-6)  — titik periksa 1a
Rumus        I(q)   = { i ∈ 1..5 : dᴱˣᵢ > τ }
             RDE(q) = (1 / |I(q)|) · Σ_{i ∈ I(q)} (dᴬᴺᴺ₍ᵢ₎ − dᴱˣᵢ) / dᴱˣᵢ
             #8     = rata-rata RDE(q) atas query dengan |I(q)| ≥ 1
                                                           — titik periksa 2a
             tanpa suku dikeluarkan, |I(q)| = 5 → sama dengan rumus 002
Diagnostik   Σ_q (5 − |I(q)|) dan jumlah query dengan |I(q)| = 0, dicatat di Run
Exact        Suku 0/0 milik exact ikut dikeluarkan → #8 exact = 0 (D4)
Tetap        Urutan ulang L2 untuk LSH, penalti slot −1 = 2
```

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
| jumlah suku #8 yang dikeluarkan | Kolom diagnostik (005a): Σ_q (5 − \|I(q)\|) |
| jumlah query tanpa suku #8 tersisa | Kolom diagnostik (005a): query dengan \|I(q)\| = 0 |

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

Tidak ada untuk metrik. H4 (cara ukur QPS dan p50), H5 (#12), dan H6 (#8 di dekat nol) diputuskan di 005a; pemanasan sebelum ulangan QPS (H17) diputuskan di 006a.
