# 03 — Pipeline Jawaban

Delapan tahap, semuanya deterministik kecuali tahap 6.

## Tahap 1 — Deteksi prompt injection

Pertanyaan dicocokkan dengan daftar pola (Indonesia + Inggris), misalnya
"abaikan aturan", "kamu sekarang admin", "ignore previous", "reveal all".

Bila cocok: pertanyaan **tetap diproses** tetapi ditandai. Jawaban yang dihasilkan
disanitasi dan flag `prompt_injection_detected=true` dikirim ke klien serta audit.
Pendekatan ini dipilih agar serangan tidak otomatis "menang" dengan memicu error.

Pemeriksaan yang sama dijalankan pada **isi dokumen** yang akhirnya dipakai —
kalimat berisi pola injection tidak pernah muncul di jawaban.

## Tahap 2 — Resolusi identitas

Token `Bearer <user_id>` dipetakan ke user (role + `allowed_workspaces`).
Token tidak valid → `401 invalid_token`. Tidak ada otorisasi yang berasal dari
body request.

## Tahap 3 — Filter izin

Himpunan dokumen kandidat dibentuk dari irisan workspace, role (dengan pewarisan),
dan status aktif. Lihat [02-permission-model.md](02-permission-model.md).

## Tahap 4 — Klasifikasi topik

Pertanyaan dinormalisasi: lowercase, dipecah non-alfanumerik, stopword dibuang,
sinonim dipetakan (mis. `salary → gaji`, `leave → cuti`, `travel → transport`).

Topik ditentukan dengan urutan prioritas (yang lebih spesifik menang):

```
pii → project_kenari → cuti → workflow → approval → transport
    → makassar → hr_contact → budget_marketing → gaji
    → company → project → generic
```

Urutan ini bukan sembarang: `reimbursement` didahulukan atas `transport` karena
pertanyaan "how do I request reimbursement for travel" memetakan `travel` ke
`transport` lewat sinonim, padahal maksudnya alur reimbursement.

## Tahap 5 — Penyusunan jawaban

Dua mode, dipilih berdasarkan topik:

**Jalur topik** — untuk topik yang punya aturan eksplisit (transport, cuti,
approval, gaji, kontak, proyek, perusahaan, workflow). Dokumen yang relevan
dipilih dengan skor `|token pertanyaan ∩ token dokumen|`, lalu jawaban disusun
dari konten dokumen.

**Jalur retrieval kalimat** — untuk topik `generic`. Setiap kalimat di setiap
dokumen yang diizinkan diberi skor tumpang tindih token; tiga kalimat teratas
dirangkai. Kalimat berisi pola injection dilewati.

**Deteksi konflik** — bila beberapa dokumen menyebut nominal berbeda untuk topik
yang sama, semua versi dikumpulkan, pemenang dipilih berdasarkan:

1. `source_type = hr_announcement` didahulukan
2. lalu `effective_date` terbaru

Dokumen yang kalah **tetap ditampilkan** sebagai "sudah tidak berlaku", sehingga
pengguna melihat bahwa ada perubahan kebijakan, bukan sekadar angka baru.

## Tahap 6 — Penyusunan ulang oleh LLM (opsional)

Hanya dijalankan bila LLM aktif **dan** tidak ada penolakan.

Aturan penentuan jalur:

| Kondisi | Aksi |
|---|---|
| Draft deterministik punya sumber | LLM **hanya merangkai ulang** draft itu |
| Draft tanpa sumber (tidak ada kecocokan) | LLM boleh memilih dokumen & menjawab |
| Penolakan (`refusal_reason` terisi) | LLM **tidak pernah** dipanggil |

Alasan: bila retrieval deterministik sudah menemukan dokumen yang tepat,
membiarkan model memilih ulang dari nol justru menurunkan kualitas — terbukti
memilih dokumen bertopik mirip tetapi tidak memuat jawabannya.

Semua output model melewati validator. Lihat [04-llm-grounding.md](04-llm-grounding.md).

## Tahap 7 — Sanitasi output

- ID dokumen internal (`doc_*`) dihapus dari teks.
- Baris "Sumber:" dinormalisasi menjadi judul dokumen + tanggal efektif.
- Artefak markdown yatim (`****`, bintang menggantung) dibersihkan.
- PII dimask untuk role di luar `hr`/`admin`.

## Tahap 8 — Audit

Satu baris per pertanyaan, append-only:

```
timestamp, user_id, user_role, question,
permission_decision, allowed_sources, retrieved_sources, selected_sources,
conflict_detected, answer, citations, refusal_reason,
pii_masked, prompt_injection_detected, agent_version,
engine_used, llm_model, llm_ms, llm_error
```

Kolom `engine_used` (`deterministic` | `llm`) memungkinkan audit membedakan
jawaban yang disusun model dari yang murni aturan.

## Ringkasan sifat

| Sifat | Dijamin oleh |
|---|---|
| Tidak membocorkan dokumen terlarang | Filter izin sebelum retrieval + cek bigram |
| Tidak mengarang angka | Validator nominal terhadap sumber terpilih |
| Tidak mengarang kontak | Validator entitas (email/kanal/telepon) |
| Tidak menuruti instruksi tersembunyi | Sanitasi kalimat + isi dokumen sebagai data |
| Dapat direproduksi | Engine deterministik, tanpa jaringan |
| Dapat diaudit | Satu baris audit per pertanyaan |
