# 07 — Kontrak API

Base: `https://app.nohalu.xyz/api` (di-proksi ke backend).

## Autentikasi

```
Authorization: Bearer <user_id>
```

Tidak ada header → `401 missing_authorization`.
Token tidak dikenal → `401 invalid_token`.

## Chat

### `POST /api/chat`

```json
{ "question": "Berapa tunjangan transport bulanan?", "session_id": "opsional" }
```

Respons:

```json
{
  "user_id": "u_budi",
  "role": "employee",
  "question": "...",
  "audit_id": 87,
  "answer": "Tunjangan transport bulanan adalah Rp750.000 ...",
  "citations": [
    { "id": "doc_hr_transport_2026", "title": "...", "effective_date": "2026-10-12",
      "source_type": "hr_announcement", "excerpt": "..." }
  ],
  "limitations": "Jawaban bersumber dari 2 dokumen yang dapat Anda akses.",
  "conflict_detected": true,
  "conflict_note": "SOP Reimbursement v2 (2025-03-03) menyatakan Rp500.000 — sudah tidak berlaku",
  "permission_decision": "allow",
  "refusal_reason": null,
  "pii_masked": false,
  "prompt_injection_detected": false,
  "selected_sources": ["doc_hr_transport_2026", "doc_sop_reimbursement_v2"],
  "engine_used": "llm",
  "llm_model": "mercury-3",
  "llm_ms": 2362,
  "llm_error": null,
  "suggestions": ["...", "..."],
  "session_id": "uuid"
}
```

Nilai `engine_used`: `deterministic` | `llm`.
`refusal_reason`: `null` | `no_access` | `no_data`.

### `GET /api/chat/{session_id}`
Riwayat sesi. Admin boleh membaca sesi siapa pun.

### `POST /api/chat/{audit_id}/feedback`
```json
{ "value": 1 }
```

`value` harus `1` (membantu) atau `-1` (tidak). Nilai lain → `400 invalid_value`.
`audit_id` tidak ada → `404 audit_not_found`.

## Portal

| Endpoint | Isi | Role |
|---|---|---|
| `GET /api/dashboard` | ringkasan user + statistik | semua |
| `GET /api/schedule` | jadwal kerja | semua |
| `GET /api/payroll` | jadwal gaji (+ struktur gaji bila berhak) | semua |
| `GET /api/directory` | direktori karyawan | semua |
| `GET /api/company` | profil perusahaan (terkelompok) | semua |
| `GET /api/projects` | status proyek | sesuai ACL |
| `GET /api/referensi?q=` | perpustakaan dokumen + pencarian | sesuai ACL |

`GET /api/payroll` mengembalikan `salary_structure: null` untuk role yang tidak
berhak — bukan error, agar UI dapat menampilkan bagian lain tanpa gagal.

## Pengajuan

### `POST /api/requests`
```json
{ "type": "cuti", "start_date": "2026-11-10", "end_date": "2026-11-12" }
{ "type": "reimbursement", "amount": "1500000", "note": "perjalanan dinas" }
```

Respons `201`:
```json
{ "id": 6, "status": "pending",
  "approver_roles": ["team_lead", "finance"],
  "stages": ["team_lead", "finance"], "stage_idx": 0 }
```

### `GET /api/requests`
```json
{ "mine": [...], "approval": [...], "all": [...] }
```

Item menyertakan `history`, `current_role`, `finance_manager_required`,
`approval_threshold`.

### `POST /api/requests/{id}/decide`
```json
{ "action": "approve", "note": "anggaran tersedia" }
```

| Kode | Arti |
|---|---|
| `200` | diproses |
| `400` | `already_decided` / `invalid_action` |
| `403` | `cannot_decide_own` / `not_authorized` |
| `404` | `not_found` |

## Notifikasi

| Endpoint | Fungsi |
|---|---|
| `GET /api/notifications` | daftar + jumlah belum dibaca |
| `POST /api/notifications/{id}/read` | tandai dibaca |

## Admin

| Endpoint | Role | Fungsi |
|---|---|---|
| `GET /api/documents` | hr, admin | daftar dokumen |
| `POST /api/documents` | hr, admin | buat dokumen |
| `POST /api/documents/upload` | hr, admin | unggah PDF/DOCX/CSV/MD/TXT |
| `PUT /api/documents/{id}` | hr, admin | ubah dokumen |
| `GET /api/documents/{id}` | sesuai ACL | preview |
| `GET /api/users` | admin | daftar user |
| `PUT /api/users/{id}/permissions` | admin | ubah role/workspace |
| `GET /api/workspaces` | admin | daftar workspace |
| `GET /api/audit` | hr, admin | audit log + filter |

## Status

| Endpoint | Fungsi |
|---|---|
| `GET /api/health` | status + apakah LLM aktif |
| `GET /api/llm/status` | konfigurasi LLM (tanpa API key) |

## Kode error

| HTTP | `detail` | Kapan |
|---|---|---|
| `400` | `empty_question` | pertanyaan kosong |
| `400` | `invalid_value` | feedback bukan 1/-1 |
| `401` | `missing_authorization` | header tidak ada |
| `401` | `invalid_token` | user tidak dikenal |
| `403` | `admin_required` | butuh admin |
| `403` | `hr_or_admin_required` | butuh hr/admin |
| `403` | `cannot_decide_own` | menyetujui pengajuan sendiri |
| `403` | `not_authorized` | role tidak cocok tahap |
| `404` | `not_found` | resource tidak ada |
| `404` | `audit_not_found` | audit_id feedback tidak ada |
| `429` | rate limit | melewati batas |

Penolakan akses **bukan** error HTTP — selalu `200` dengan
`refusal_reason` terisi. Ini menjaga agar penolakan tidak dapat dibedakan dari
"data tidak ada" lewat status code.
