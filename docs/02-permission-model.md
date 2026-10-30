# 02 — Model Izin

## Tiga lapis kontrol

Sebuah dokumen hanya terlihat bila **ketiganya** terpenuhi:

```
1. workspace_ids(dokumen) ∩ allowed_workspaces(user) ≠ ∅
2. role(user) ∈ access_roles(dokumen)            ← dengan pewarisan role
3. status(dokumen) ∈ {indexed, published}
```

Lapis ketiga mencegah SOP lama yang sudah digantikan ikut dikutip.

## Role

| Role | Cakupan | Catatan |
|---|---|---|
| `employee` | Kebijakan umum, cuti, jadwal, direktori | Tidak boleh struktur gaji, proyek restricted, kontak HR |
| `team_lead` | + proyek, persetujuan tahap 1 | Approver pertama semua pengajuan |
| `hr` | + kelola dokumen, audit, struktur gaji | Boleh lihat PII |
| `admin` | Semua | Satu-satunya yang boleh ubah izin user |
| `finance` | + persetujuan reimbursement | **Tidak** boleh struktur gaji |

## Pewarisan role

Role `finance` butuh membaca SOP dan matriks persetujuan tanpa membuka data
kompensasi. Karena `access_roles` dokumen tidak memuat string `finance`, dipakai
peta pewarisan eksplisit:

```
finance  →  { finance, employee, team_lead }
```

Konsekuensi yang disengaja:

| Dokumen | employee | team_lead | finance | hr | admin |
|---|---|---|---|---|---|
| SOP reimbursement | ✅ | ✅ | ✅ | ✅ | ✅ |
| Matriks persetujuan | ✅ | ✅ | ✅ | ✅ | ✅ |
| Struktur gaji | ❌ | ❌ | **❌** | ✅ | ✅ |
| Proyek Kenari (internal) | ❌ | ✅ | ❌ | ❌ | ✅ |
| Kontak HR (PII) | ❌ | ❌ | ❌ | ✅ | ✅ |

Pewarisan **tidak** menaikkan hak ke dokumen yang lebih ketat — ia hanya
memperluas ke role yang secara eksplisit terdaftar di peta.

## Mengapa izin diperiksa sebelum pencarian

Alternatif naif adalah mencari dulu lalu menyaring hasil. Itu bocor lewat tiga
jalur:

1. **Jumlah hasil** — "0 hasil" vs "3 hasil tapi tidak boleh" membedakan ada/tidak.
2. **Skor relevansi** — hasil terlarang tetap memengaruhi peringkat.
3. **Pesan error** — penyaringan pasca-pencarian cenderung memberi pesan berbeda.

Dengan menyaring lebih dulu, himpunan kandidat untuk user tanpa akses benar-benar
kosong, sehingga jalur penolakan identik dengan "data tidak ada".

## Penolakan netral

Satu kalimat untuk semua kegagalan pencarian:

```
Saya tidak memiliki informasi mengenai hal tersebut.
```

Dipakai untuk: tidak ada dokumen cocok, akses ditolak, dan dokumen tidak ada.
Penyerang tidak bisa membedakan ketiganya dari respons.

## Pemeriksaan tambahan untuk output LLM

Selain ACL, output model diperiksa terhadap **bigram eksklusif** dokumen di luar
izin. Bila model menyebut frasa yang hanya muncul di dokumen terlarang, jawaban
dibuang dan fallback deterministik dipakai. Detail di
[04-llm-grounding.md](04-llm-grounding.md).

## Perubahan izin

Hanya `admin` yang dapat mengubah role/workspace user (`PUT /api/users/{id}/permissions`).
Setiap perubahan tercatat di audit log bersama pertanyaan pengguna, sehingga
"kenapa saya bisa/tidak bisa melihat X" dapat ditelusuri.
