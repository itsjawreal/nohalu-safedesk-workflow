# 04 — LLM Grounding

## Posisi LLM

LLM **bukan** penentu kebenaran. Ia hanya mengubah draft terverifikasi menjadi
kalimat yang lebih natural. Semua keputusan yang berdampak pada keamanan dibuat
sebelum model dipanggil, dan output model diperiksa lagi sesudahnya.

```
filter izin → retrieval → draft terverifikasi → LLM → validator → tampil
                                  ▲                        │
                                  └──── fallback ◄─────────┘ (gagal validasi)
```

## Kapan LLM dipanggil

| Kondisi | LLM |
|---|---|
| Ada penolakan akses | ❌ tidak dipanggil |
| Draft punya sumber | ✅ merangkai ulang draft saja |
| Draft kosong, ada kandidat dokumen | ✅ boleh memilih dokumen + menjawab |
| LLM nonaktif / tanpa API key | ❌ murni deterministik |

Penolakan tidak pernah dikirim ke model — ini mencegah model "membujuk" sistem
untuk menjawab pertanyaan yang seharusnya ditolak.

## Validator output

Setiap output model diperiksa. Kegagalan **tidak** menggagalkan permintaan —
sistem jatuh ke jawaban deterministik.

### 1. Kebocoran entitas restricted

Dihitung **bigram** (pasangan kata) yang eksklusif milik dokumen di luar izin
pengguna. Bila jumlah bigram semacam itu melewati ambang (default 3), jawaban
ditolak.

Ambang dipakai — bukan pencocokan satu kata — karena kata umum seperti "kenari"
atau "internal" bisa muncul di dokumen publik juga. Yang diperiksa adalah
**kombinasi** kata yang hanya ada di dokumen terlarang.

### 2. Nominal karangan

Semua `Rp<nominal>` di output harus ada di dokumen sumber terpilih. Nominal baru
yang tidak ditemukan → ditolak.

### 3. Entitas karangan (email / kanal / telepon)

Email, kanal (`#finance`), dan nomor telepon di output harus benar-benar muncul
di dokumen sumber. Ini menangkap kasus nyata: model menyebut
`finance@…` padahal dokumen yang dipilih hanya berisi jadwal pembayaran gaji,
bukan daftar kontak.

Pemeriksaan dilakukan pada teks **asli**, sebelum PII dimask — kalau tidak,
email karangan akan lolos karena sudah berubah menjadi `f***@…`.

### 4. Klaim gaji tanpa sumber

Bila output membahas gaji tetapi tidak ada dokumen terpilih yang memuat kata
"gaji", jawaban ditolak.

### 5. Band gaji untuk non-HR

Pola `Grade <n>` atau "kisaran gaji"/"salary band" ditolak untuk role di luar
`hr`/`admin`, terlepas dari dokumen sumbernya.

## Fallback berlapis

```
1. LLM menjawab            → lolos validator?  → pakai
2. LLM gagal 3×            → kutipan verbatim dari dokumen terpilih
3. Tidak ada dokumen       → jawaban deterministik
4. Draft deterministik     → penolakan netral (jika memang tidak ada akses)
```

Lapis 2 penting: bila model terus mengarang, sistem tidak menyerahkan jawaban
kosong. Ia mengutip kalimat asli dari dokumen — **grounded by construction**,
tanpa risiko halusinasi.

## Aturan "fallback tidak boleh menang"

Model kadang menjawab "Saya tidak memiliki informasi…" padahal draft
deterministik sudah menemukan sumber yang tepat. Sistem menyimpan draft sebelum
model dipanggil; bila model mengembalikan fallback sementara draft punya
grounding, draft dipulihkan dan model hanya dipakai untuk merangkai ulang.

## Bahasa

Jawaban mengikuti bahasa pertanyaan. Pertanyaan berbahasa Inggris dijawab dalam
Bahasa Inggris; pertanyaan Indonesia dalam Bahasa Indonesia. Dokumen sumber tetap
berbahasa Indonesia — model yang menyesuaikan.

## Konfigurasi

| Variabel | Fungsi |
|---|---|
| `NOHALU_LLM_ENABLED` | `0` memaksa mode deterministik |
| `NOHALU_LLM_API_KEY` | Wajib untuk mengaktifkan |
| `NOHALU_LLM_MODEL` | Nama model |
| `NOHALU_LLM_REASONING` | Tingkat penalaran |
| `NOHALU_LLM_TIMEOUT` | Batas waktu panggilan |
| `NOHALU_LLM_LEAK_THRESHOLD` | Ambang bigram restricted (default 3) |

LLM dikendalikan **server**, bukan klien. Parameter `use_llm` dari klien diabaikan
agar pengguna tidak bisa memaksa jalur yang lebih lemah atau lebih mahal.

## Biaya kegagalan

| Kegagalan | Dampak pengguna |
|---|---|
| LLM timeout / error | Jawaban deterministik (lebih kaku, tetap benar) |
| Validator menolak | Kutipan verbatim dari dokumen |
| Tidak ada dokumen | Penolakan netral |
| LLM mati total | Sistem tetap berfungsi penuh |
