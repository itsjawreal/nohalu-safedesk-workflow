# 05 — Approval Berjenjang

## Dua jenis pengajuan

| Jenis | Tahap 1 | Tahap 2 | Syarat input |
|---|---|---|---|
| Cuti | `team_lead` | `hr` | `start_date` + `end_date` wajib |
| Reimbursement | `team_lead` | `finance` | `amount` wajib |

Team Lead selalu tahap pertama karena ia atasan langsung — sesuai SOP:
"reimbursement harus disetujui Team Lead sebelum diproses Finance".

## Ambang nominal

Matriks persetujuan menyatakan:

> Reimbursement sampai Rp1.000.000 disetujui Team Lead.
> Di atas Rp1.000.000 disetujui Finance Manager.

Karena kedua nominal bermuara ke role `finance`, sistem menandai pengajuan besar
secara eksplisit lewat field `finance_manager_required` (dihitung dari
`amount > threshold`). Field ini dipakai UI untuk menampilkan approver yang tepat
tanpa mengubah rantai approval.

Ambang dapat diatur lewat `NOHALU_FINANCE_THRESHOLD` (default `1000000`).

## Siklus status

```
                 ┌──────────┐
   dibuat ──────►│ pending  │  stage_idx = 0
                 └────┬─────┘
                      │ approve oleh approver tahap berjalan
                      ▼
              stage_idx + 1
                      │
        ┌─────────────┴─────────────┐
        │                           │
   masih ada tahap            tahap habis
        │                           │
        ▼                           ▼
   ┌──────────┐              ┌──────────┐
   │ pending  │              │ approved │
   └──────────┘              └──────────┘

   reject (tahap mana pun) ──► ┌──────────┐
                               │ rejected │
                               └──────────┘
```

`reject` menghentikan alur seketika; `stage_idx` tidak maju.

## Aturan penolakan aksi

| Upaya | Hasil |
|---|---|
| Menyetujui pengajuan sendiri | `403 cannot_decide_own` |
| Menyetujui di luar role tahap berjalan | `403 not_authorized` |
| Menyetujui pengajuan yang sudah selesai | `400 already_decided` |
| Aksi selain `approve`/`reject` | `400 invalid_action` |
| Pengajuan tidak ada | `404 not_found` |

Pemisahan tugas ditegakkan: pengaju tidak pernah bisa menjadi approver dirinya
sendiri, bahkan bila role-nya cocok.

## Riwayat

Setiap keputusan menambah satu entri:

```json
{
  "role": "team_lead",
  "action": "approve",
  "by": "u_andi",
  "at": "2026-10-30T01:49:45.698358Z",
  "note": "anggaran tersedia"
}
```

Catatan approver disimpan dan ditampilkan ke pengaju — termasuk saat penolakan,
sehingga alasan tidak hilang.

## Notifikasi

| Kejadian | Penerima |
|---|---|
| Pengajuan dibuat | Semua user dengan role tahap 1 |
| Lolos satu tahap | Semua user dengan role tahap berikutnya |
| Disetujui penuh | Pengaju |
| Ditolak | Pengaju + catatan alasan |

Notifikasi punya status baca (`is_read`) dan dapat ditandai dibaca per item.

## Visibilitas daftar

| Role | `mine` | `approval` | `all` |
|---|---|---|---|
| `employee` | ✅ | — | — |
| `team_lead` | ✅ | ✅ (tahap yang menunggu dia) | — |
| `finance` | ✅ | ✅ (tahap yang menunggu dia) | — |
| `hr` | ✅ | ✅ | ✅ |
| `admin` | ✅ | ✅ | ✅ |

`approval` hanya berisi pengajuan yang **tahap berjalannya** cocok dengan role
pemanggil — bukan semua pengajuan pending.
