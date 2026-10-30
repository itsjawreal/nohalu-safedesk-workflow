# 08 — Strategi Pengujian

## Lapisan test

| Lapisan | Jumlah | Butuh jaringan? | Cakupan |
|---|---|---|---|
| Test case end-to-end | 25 | tidak | Perilaku jawaban lintas skenario |
| Unit safety engine | 13 | tidak | Izin, konflik, refusal, PII, injection, grounding |
| Unit portal | 17 | tidak | Approval berjenjang, notifikasi, RBAC portal |
| Unit ingestion | 8 | tidak | PDF/DOCX/CSV/MD/TXT |
| Smoke API | 2 alur | tidak | Endpoint nyata via TestClient |
| Live LLM | opsional | ya | Perilaku model sungguhan |

Semua lapisan kecuali live-LLM berjalan offline dan deterministik — LLM dimatikan
lewat environment saat test, sehingga hasilnya tidak bergantung pada model atau
koneksi.

## Prinsip pengujian

**Uji perilaku keamanan lebih dulu.** Test yang gagal harus berarti ada
kebocoran atau jawaban salah, bukan sekadar perubahan format.

**Setiap bug yang ditemukan menjadi test.** Contoh nyata yang kini dijaga:

| Bug yang pernah terjadi | Test penjaganya |
|---|---|
| LLM menjawab "tidak tahu" padahal draft punya sumber | `llm_fallback_does_not_override_grounded` |
| LLM menyebut email yang tidak ada di dokumen sumber | `ungrounded_entity_rejected` |
| Threshold Rp1jt diabaikan kode | `approver_roles_threshold` |
| Role finance tidak bisa approve (deadlock) | `reimbursement_two_stage_flow` |
| Onboarding ikut jadi sumber jawaban reimbursement | `company_profile_and_workflow` |
| Karyawan tidak dapat arah kontak gaji | `gaji_contact_routing_employee` |

**Test harus bisa gagal.** Setiap checker statis diverifikasi dengan menyuntik
kesalahan sengaja, memastikan ia benar-benar mendeteksi, lalu dipulihkan.

## Skenario end-to-end

25 test case mencakup:

- Konflik kebijakan (nominal lama vs pengumuman baru)
- Penolakan netral untuk data di luar akses
- PII masking per role
- Prompt injection (pertanyaan & isi dokumen)
- RBAC struktur gaji
- Profil perusahaan & mitra publik
- Workflow reimbursement, cuti, onboarding
- Konsistensi audit log
- Perilaku lintas bahasa (Indonesia/Inggris)

## Pemeriksaan statis frontend

| Pemeriksaan | Menangkap |
|---|---|
| Syntax seluruh modul | file rusak / import salah |
| Resolusi import relatif | modul hilang setelah refactor |
| CSP: tanpa inline handler/script/style | pelanggaran Content-Security-Policy |
| Referensi aset | `src`/`href` menunjuk file yang tidak ada |
| Validasi konfigurasi deploy | rewrite/CSP hilang atau salah |
| Serve smoke | entrypoint & modul kunci benar-benar tersaji |

## Yang tidak diuji otomatis

- **Tampilan visual** — tidak ada snapshot/screenshot test.
- **Beban & konkurensi** — belum ada load test.
- **Model LLM secara kontinu** — hanya diuji manual; perilaku model bisa berubah
  antar versi, sehingga validator (bukan test) yang menjadi jaring pengaman.
- **Ketahanan database** — belum ada uji korupsi/restore.

## Cara menjalankan

```bash
# backend (offline)
python tests/run_testcases.py     # 25 skenario -> tests/RESULTS.md
python tests/test_engine.py       # unit safety
python tests/test_portal.py       # unit portal & approval
python tests/test_ingest.py       # unit ingestion
python tests/smoke_api.py         # smoke endpoint

# frontend (tanpa dependency)
npm test                          # static check + serve smoke
```

## Hasil terakhir

```
25/25 PASS   test case end-to-end
13 passed    unit safety engine
17 passed    unit portal
 8 passed    unit ingestion
   PASS      smoke API + alur finance
   PASS      static check + serve smoke
```
