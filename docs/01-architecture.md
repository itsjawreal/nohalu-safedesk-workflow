# 01 — Arsitektur

## Komponen

```
┌─────────────────────────────────────────────────────────────┐
│ Klien (browser, SPA)                                        │
│  - chat, direktori, jadwal, pengajuan, approval, audit      │
└───────────────────────────┬─────────────────────────────────┘
                            │ HTTPS + Bearer <user_id>
                            ▼
┌─────────────────────────────────────────────────────────────┐
│ API layer (FastAPI)                                         │
│  - autentikasi token, rate limit, RBAC dependency           │
│  - endpoint chat / portal / admin / audit                   │
└───────────────┬─────────────────────────────┬───────────────┘
                │                             │
                ▼                             ▼
┌───────────────────────────────┐  ┌──────────────────────────┐
│ Decision engine (deterministik)│  │ Portal service           │
│  - filter izin                │  │  - dashboard, direktori  │
│  - retrieval                  │  │  - jadwal, payroll       │
│  - deteksi konflik            │  │  - pengajuan, approval   │
│  - refusal / PII / injection  │  │  - notifikasi            │
│  - audit                      │  └──────────────────────────┘
└───────────────┬───────────────┘
                │ grounded draft + allowed context
                ▼
┌─────────────────────────────────────────────────────────────┐
│ LLM layer (opsional)                                        │
│  - merangkai ulang jawaban yang sudah diverifikasi          │
│  - output divalidasi ulang sebelum dipakai                  │
└─────────────────────────────────────────────────────────────┘
```

## Batas kepercayaan

| Zona | Dipercaya? | Catatan |
|---|---|---|
| Klien / browser | **Tidak** | Token bisa dipalsukan; semua otorisasi di server |
| Isi dokumen | **Tidak** | Diperlakukan sebagai data, bukan instruksi |
| Pertanyaan pengguna | **Tidak** | Bisa berisi prompt injection |
| Output LLM | **Tidak** | Harus lolos validator sebelum ditampilkan |
| Decision engine | Ya | Deterministik, dapat diuji ulang |
| Database | Ya | Satu-satunya sumber kebenaran |

Prinsip: **semua yang berasal dari luar dianggap tidak tepercaya**. Otorisasi
tidak pernah bergantung pada nilai yang dikirim klien selain identitas token.

## Alur data satu pertanyaan

```
pertanyaan
   │
   ├─► deteksi prompt injection (pola)
   ├─► resolusi user → role + workspace
   ├─► filter dokumen (workspace ∩ role ∩ status aktif)
   ├─► klasifikasi topik → jalur jawaban
   ├─► retrieval / aturan konflik → draft + sitasi
   ├─► (opsional) LLM merangkai ulang → validator
   ├─► audit log (append-only)
   └─► respons + sitasi + metadata
```

Detail tiap tahap ada di [03-answer-pipeline.md](03-answer-pipeline.md).

## Keputusan desain

**Engine deterministik sebagai inti.** Jawaban dapat direproduksi tanpa jaringan,
tanpa model, dan tanpa biaya. Ini membuat 25 test case berjalan offline dan
membuat perilaku keamanan dapat diaudit baris per baris.

**LLM sebagai lapisan penyusunan, bukan otoritas.** Model tidak pernah memutuskan
izin, tidak pernah melihat dokumen di luar akses, dan tidak pernah dipanggil untuk
kasus penolakan. Lihat [04-llm-grounding.md](04-llm-grounding.md).

**SQLite sebagai penyimpanan.** Cukup untuk skala demo, transaksional, dan mudah
direplikasi. Semua akses DB diserialisasi lewat satu lock untuk menghindari
penulisan bersamaan.

**Frontend terpisah.** UI di-deploy statis; API diproksi same-origin sehingga
tidak ada CORS terbuka dan tidak ada token yang melintasi origin lain.
