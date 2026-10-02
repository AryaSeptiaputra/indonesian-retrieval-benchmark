# CLAUDE.md

## Proyek

**indonesian-retrieval-benchmark** — riset yang membandingkan algoritma pencarian vektor untuk retrieval teks berbahasa Indonesia. Semua dokumen dan query di-embed dengan satu model yang sama, lalu dicari dengan beberapa algoritma, sehingga perbedaan hasil hanya berasal dari algoritma pencariannya.

- **Jenis proyek:** riset & training model. Template struktur mengikuti `writer-code` (`.claude/skills/writer-code/references/structure-riset.md`), kecuali rancangan red-chan yang disetujui menetapkan lain.
- **Repo:** https://github.com/AryaSeptiaputra/indonesian-retrieval-benchmark

### Ditetapkan Arya (dikunci)

Bagian ini tidak boleh diganti tanpa persetujuan Arya.

| Hal | Ketetapan |
|---|---|
| Bahasa pemrograman | Python |
| Model embedding | `LazarusNLP/congen-indobert-base` (Hugging Face) |
| Algoritma pencarian | Exact dense search (brute-force, baseline) dan Approximate Nearest Neighbor: HNSW, IVF, LSH |

### Belum diputuskan

Keputusan dibuat lewat red-chan dan dicatat di `docs/keputusan-produk.md`; yang berlaku adalah dokumen itu dan file `docs/rancangan/<nomor>a_...`.

Sudah diputuskan di rancangan 002:

| Hal | Ringkasan | Dokumen |
|---|---|---|
| Dataset | `miracl/miracl-corpus` id (1.446.315 passage); query dan qrels `miracl/miracl` id dev (960 query, 9.668 penilaian) | `docs/dataset.md` |
| Library | sentence-transformers 6.1.0 (hanya memuat), encode loop PyTorch, datasets 5.0.1, faiss-cpu 1.15.1 | `docs/tech-stack.md` |
| Metrik | 12 metrik, k = 5, latensi p50, aturan slot −1 | `docs/metrik-evaluasi.md` |
| Hardware dan tempat menjalankan | Vast.ai Linux, RTX 3090 24 GB, RAM ≥ 32 GB, CPU ≥ 24, disk 50 GB; thread = min(jatah cgroup, 24) | `docs/lingkungan-eksekusi.md` |
| Pencatatan resource | Otomatis oleh kode ke Lingkungan, manual oleh Arya di `README.md` | `docs/lingkungan-eksekusi.md` |

Masih belum diputuskan:

| Kode | Hal |
|---|---|
| H1 | Teks dokumen yang di-embed (title + text atau text saja) |
| H2 | Parameter tiap algoritma dan ada/tidaknya sapuan di val |
| H3 | Tanda berhasil benchmark dan aturan memilih konfigurasi yang dikunci dari val |
| H4 | Cara mengukur QPS dan latensi p50 |
| H5 | Definisi metrik #12 (ukuran index / memori) |
| H6 | Penanganan jarak exact = 0 pada #8 Relative distance error |
| H7 | Presisi encode, ukuran batch, pemeriksaan kesamaan dengan `model.encode` |
| H8 | Format berkas fisik tabel, vektor, dan metadata |
| H9 | Versi Python pasti (3.10–3.13) |
| H10 | Tempat penyimpanan di luar Vast.ai |
| H11 | Field yang membentuk env_id |
| H12 | Perilaku pencatatan resource di luar Linux |
| H13 | Versi torch dan numpy (dikunci dari `pip freeze` instance Vast.ai) |
| — | Lisensi model, tujuan keluaran (skripsi, paper, atau laporan internal), grafik laporan |

## Dokumen rancangan

| File | Isi | Ditulis oleh |
|---|---|---|
| `docs/keputusan-produk.md` | Rancangan sistem per fase dan statusnya | red-chan |
| `docs/rencana-evaluasi.md` | Rancangan evaluasi per modul (mulai fase Dev) | red-chan |
| `docs/daftar-rancangan.md` | Status semua rancangan bernomor | red-chan (kolom a), pink-chan (kolom b, Pembangunan) |
| `docs/rancangan/<nomor>a_...` | Desain sistem yang disetujui | red-chan |
| `docs/rancangan/<nomor>b_...` | Rencana pembangunan | pink-chan |

Kode hanya ditulis dari rancangan yang sudah berstatus `siap dikerjakan`.

## Kontrak berkas

Notebook terhubung hanya lewat berkas di folder berikut, tanpa import antar-notebook. Entitas dan aturannya dari rancangan 001a. Nama berkas, kolom, dan format ditetapkan di rancangan 002; notebook penulis dan pembaca dicantumkan saat notebook dibuat.

| Entitas | Folder | Ditulis oleh | Dibaca oleh | Aturan |
|---|---|---|---|---|
| Dataset mentah (korpus, query, qrels) | `data/raw/` | Arya, diunduh manual | Penyiapan data | Tidak pernah diubah |
| Dokumen, Query (beserta split), Penilaian relevansi | `data/processed/` | Penyiapan data (K2) | Pembuat embedding, Penilai | Split val/test 50:50 seed 42, ditetapkan sekali dan dikunci hash; tidak ditimpa tanpa pemeriksaan hash |
| Set embedding, Vektor dokumen, Vektor query | `data/embeddings/<embedding_id>/` | Pembuat embedding (K1) | Exact, HNSW, IVF, LSH | Konfigurasi berbeda → embedding_id baru; vektor lama tidak ditimpa; urutan baris = urutan id yang disimpan bersama |
| Tetangga exact | `data/embeddings/<embedding_id>/` | Exact search (K3) | Penilai di HNSW, IVF, LSH | Hanya sah untuk embedding_id yang sama; exact dijalankan sebelum ANN |
| Run, Lingkungan | `outputs/tuning/` | Exact, HNSW, IVF, LSH | Tabel hasil (K4) | Run hanya ditambah, tidak pernah ditimpa; lingkungan dicatat per sesi |

`data/` dan `outputs/` di-gitignore; susunan foldernya dijaga dengan `.gitkeep`.

## Agent dan skill di repo ini

Agent dan skill disimpan di repo supaya ikut ter-commit:

```
.claude/
├── agents/
│   ├── red-chan.md          # perancang sistem, tidak menulis kode
│   └── pink-chan.md         # pelaksana penulisan kode Python
├── skills/
│   ├── product-design/      # dimuat red-chan
│   ├── writer-code/         # dimuat pink-chan; standar kode Python dan struktur folder
│   └── reader-code/         # dibaca pink-chan dengan Read saat dibutuhkan; tidak dimuat otomatis
└── settings.json            # reader-code dimatikan dari pemanggilan otomatis
```

Path di dalam agent dan skill ditulis relatif terhadap root repo (`.claude/skills/...`).

Catatan perawatan: Claude Code mendahulukan skill personal (`~/.claude/skills/`) di atas skill project dengan nama yang sama, sedangkan untuk agent yang didahulukan versi project. Kalau skill di repo ini diubah, samakan juga salinan personalnya, atau hapus salinan personal supaya versi repo yang dipakai.

### Agent pink-chan

Kalau perintah Arya diawali atau menyebut nama "pink-chan", serahkan pekerjaannya ke agent `pink-chan` — jangan dikerjakan sendiri di percakapan utama. Pink-chan melanjutkan pekerjaan bertahap lewat agent yang sama; saat Arya membalas persetujuan atau jawaban untuk pekerjaan pink-chan, teruskan balasan itu ke pink-chan.

### Agent red-chan

Kalau perintah Arya diawali atau menyebut nama "red-chan", serahkan pekerjaannya ke agent `red-chan` — jangan merancang sendiri di percakapan utama.

Red-chan bekerja per putaran. Saat laporannya berstatus `menunggu jawaban` atau `menunggu persetujuan`:

1. Tampilkan diagram gambaran sistem apa adanya di dalam blok kode (jangan digambar ulang), lalu keputusan, bentrokan, dan asumsi sebagai teks ringkas.
2. Ubah bagian "Pertanyaan / Titik periksa" menjadi `AskUserQuestion`: `[Judul]` menjadi `header`, tiap pilihan menjadi `option` dengan konsekuensinya sebagai `description`, dan pilihan (Usulan) tetap paling atas. Jangan menambah, mengurangi, atau mengubah isi pilihan.
3. Kalau statusnya `menunggu persetujuan`, tanyakan juga apakah rancangan disetujui atau ada koreksi lain.
4. Teruskan jawaban Arya ke red-chan yang sama lewat `SendMessage`, termasuk teks bebas kalau Arya memilih "Other".

Saat statusnya `selesai`, tampilkan perintah untuk pink-chan apa adanya. Jangan menjalankannya sebelum Arya sendiri mengirim perintah itu.

### red-chan dan pink-chan di mode plan

Kalau sesi sedang dalam mode plan, pekerjaan red-chan dan pink-chan tetap diserahkan ke agent-nya, dengan alur berikut:

1. Awali pesan ke agent dengan `MODE PLAN` lalu perintah Arya. Agent tidak menulis file dan mengembalikan laporan lengkap.
2. Titik periksa dari red-chan ditanyakan dulu dengan `AskUserQuestion` (aturan di atas). Teruskan jawabannya ke agent yang sama, juga diawali `MODE PLAN`, sampai tidak ada pertanyaan tersisa.
3. Salin laporan terakhir agent **apa adanya** ke file plan, termasuk diagram di dalam blok kode dan bagian "Akan ditulis/dikerjakan setelah disetujui", lalu panggil `ExitPlanMode`. Jangan meringkas atau menggambar ulang.
4. Setelah Arya menyetujui plan, kirim ke agent yang sama: `PLAN DISETUJUI`, beserta pilihan titik periksa dan koreksi Arya. Agent menulis semua file-nya.
5. Kalau Arya menolak atau mengoreksi plan, teruskan koreksinya ke agent dengan awalan `MODE PLAN`, lalu ulangi dari langkah 3.

## Git

- Branch utama `main`, remote `origin` ke repo GitHub di atas.
- Commit dan push hanya saat Arya memintanya.
- Berkas rahasia (`.env`, kunci, token Hugging Face) tidak pernah di-commit.
