# Nohalu SafeDesk — Workflow Documentation

Dokumentasi publik **cara kerja** Nohalu SafeDesk: alur keputusan, model izin,
pipeline jawaban, dan mekanisme keamanan. Repo ini **tidak memuat source code**
maupun data perusahaan — hanya desain alur dan kontrak perilaku.

> Live demo: https://app.nohalu.xyz

## Apa itu Nohalu SafeDesk

Asisten chat internal perusahaan. Karyawan bertanya tentang kebijakan, SOP,
status proyek, atau kontak; sistem menjawab **hanya** dari dokumen yang boleh
diakses penanya, dengan sitasi, deteksi konflik, dan audit trail.

Prinsip yang mengikat seluruh desain:

1. **Izin diperiksa sebelum pencarian** — dokumen di luar hak akses tidak pernah
   masuk kandidat, sehingga penolakan tidak bisa membocorkan keberadaan data.
2. **Penolakan tunggal & netral** — "tidak ada data" dan "tidak boleh akses"
   memakai kalimat identik.
3. **Jawaban selalu bersumber** — setiap jawaban membawa sitasi dokumen.
4. **LLM tidak pernah menjadi otoritas** — model hanya merangkai ulang jawaban
   yang sudah diverifikasi; semua gate keamanan berjalan deterministik.
5. **Semua keputusan diaudit** — append-only, tanpa update/delete dari API.

## Dokumen

| Dokumen | Isi |
|---|---|
| [docs/01-architecture.md](docs/01-architecture.md) | Komponen, batas kepercayaan, alur data |
| [docs/02-permission-model.md](docs/02-permission-model.md) | RBAC, workspace ACL, pewarisan role, matriks akses |
| [docs/03-answer-pipeline.md](docs/03-answer-pipeline.md) | Pipeline 8 tahap dari pertanyaan ke jawaban |
| [docs/04-llm-grounding.md](docs/04-llm-grounding.md) | Peran LLM, validator, dan aturan fallback |
| [docs/05-approval-workflow.md](docs/05-approval-workflow.md) | Approval berjenjang cuti & reimbursement |
| [docs/06-safety-and-audit.md](docs/06-safety-and-audit.md) | PII, prompt injection, audit, rate limit |
| [docs/07-api-contract.md](docs/07-api-contract.md) | Endpoint, skema respons, kode error |
| [docs/08-testing-strategy.md](docs/08-testing-strategy.md) | Lapisan test dan cakupannya |

## Ruang lingkup repo ini

**Termasuk:** diagram alur, aturan keputusan, kontrak API, kebijakan keamanan,
strategi pengujian, dan hasil uji.

**Tidak termasuk:** source code, skema database internal, dataset perusahaan,
konfigurasi deployment, kredensial, dan prompt sistem lengkap.

## Lisensi

Dokumentasi ini dirilis untuk keperluan evaluasi dan portofolio. Nama, data
perusahaan, dan seluruh persona di dalam contoh adalah fiktif.
