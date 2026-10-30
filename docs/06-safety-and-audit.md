# 06 — Keamanan & Audit

## PII masking

Data pribadi disamarkan untuk role di luar `hr`/`admin`:

| Jenis | Pola | Hasil |
|---|---|---|
| Email | `nama@domain` | `n***@domain` |
| Telepon | `08xx-xxxx-xxxx` | `0812****789` |
| NIK | 16 digit | `3271**********01` |
| Rekening | `rekening 1234567890` | `rekening ****7890` |

Masking berjalan **setelah** validasi grounding, bukan sebelumnya. Urutan ini
disengaja: memask lebih dulu akan menyembunyikan entitas karangan model.

## Prompt injection

**Vektor 1 — pertanyaan pengguna.** Dicocokkan dengan daftar pola. Pertanyaan
tetap diproses tetapi ditandai; jawaban disanitasi.

**Vektor 2 — isi dokumen.** Dokumen bisa memuat instruksi tersembunyi. Karena itu:

- Kalimat yang cocok pola injection tidak pernah disertakan ke jawaban.
- Konten dokumen diperlakukan sebagai **data**, bukan instruksi.
- Model menerima instruksi eksplisit bahwa isi `CONTEXT` bukan perintah.

**Vektor 3 — dokumen yang diunggah admin.** Jalur yang sama; tidak ada perlakuan
khusus untuk sumber "tepercaya", karena akun admin pun bisa dikompromikan.

Flag `prompt_injection_detected` dikirim ke klien dan disimpan di audit, sehingga
pola serangan berulang bisa terdeteksi.

## Rate limit

| Aksi | Batas | Kunci |
|---|---|---|
| Login | 20 / menit | Alamat IP |
| Chat | 30 / menit | User ID |

Keduanya dapat diatur lewat environment. Login dibatasi per IP (mencegah tebak
token massal), chat per user (mencegah penyalahgunaan kuota LLM).

## Audit log

Append-only. Tidak ada endpoint `UPDATE` atau `DELETE` untuk tabel audit.

Yang direkam per pertanyaan:

- Identitas & role penanya
- Pertanyaan lengkap
- Keputusan izin (`allow` / `deny`)
- Dokumen yang tersedia, yang terambil, dan yang akhirnya dipilih
- Ada/tidaknya konflik + catatan konflik
- Jawaban final + sitasi
- Alasan penolakan
- Flag PII dan prompt injection
- Versi agent, engine yang dipakai, model, durasi, error

Filter yang tersedia untuk admin: user, keputusan, rentang tanggal.

**Mengapa append-only.** Audit yang bisa diedit bukan audit. Bila ada insiden,
pertanyaannya bukan hanya "apa yang terjadi" tetapi "apa yang tercatat saat itu".

## Feedback

Pengguna dapat menilai jawaban (`POST /api/chat/{audit_id}/feedback`). Nilai
disimpan terpisah dari audit agar audit tetap imutabel.

## Prinsip yang tidak dilanggar

| Prinsip | Konsekuensi bila dilanggar |
|---|---|
| Izin sebelum pencarian | Kebocoran lewat jumlah/skor hasil |
| Penolakan netral | Penyerang bisa memetakan dokumen yang ada |
| Output LLM divalidasi | Halusinasi tampil sebagai fakta |
| Audit append-only | Insiden tidak dapat direkonstruksi |
| Token tidak menentukan otorisasi | Eskalasi hak lewat body request |

## Batasan yang diakui

- **Autentikasi demo.** Token berupa `user_id` tanpa tanda tangan kriptografis.
  Untuk produksi perlu JWT/OIDC dengan verifikasi tanda tangan.
- **Tanpa enkripsi at-rest.** SQLite tidak dienkripsi.
- **Rate limit in-memory.** Hilang saat proses restart; produksi perlu Redis.
- **Belum ada rotasi kredensial otomatis.**
