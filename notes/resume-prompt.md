# Resume prompts

Two prompts, because the work has changed shape. Track 1's core is settled, so the next step is
either **more design discussion** (prompt A) or **writing the contract** (prompt B).

Pick one, paste it into a fresh session.

---

## A — Continue the design discussion

For the remaining backlog items. Replace `[ITEM]` with the number you want.

```
Lanjutkan pembahasan desain teknis Elora.

Baca dulu, urut:
1. notes/discussion-backlog.md — daftar item diskusi + status per item
2. notes/ADR-0003-offline-first-sync-and-receipt-scan.md — keputusan yang sudah final
3. notes/sync-flow.md — diagram alur sync, hasil §2.11–§2.14 dalam bentuk gambar
4. shared/context/PRD-offline-first-and-receipt-scan.md — scope siklus ini
5. .claude/skills/backend/*.md — aturan arsitektur backend yang mengikat
6. shared/context/PRD.md — baseline status proyek

Konteks: kita sedang merancang perubahan besar — offline-first sync + receipt
scanning pakai LLM. Dibahas satu item backlog per sesi supaya tidak menumpuk.
Track 1 item 1.1–1.5 sudah selesai (ADR §2.11–§2.14), 1.4 tertutup sendiri.
Tidak ada lagi yang kelihatan di kabel yang masih terbuka, jadi kontrak API
dan Track 4 sudah tidak terblokir.

Sekarang bahas item: [ITEM]

Cara kerja yang saya mau:
- Bahas dulu sampai tuntas. Jangan langsung tulis file.
- Saya butuh penilaian jujur, bukan validasi. Kalau rumusan saya lemah,
  bilang lemahnya di mana dan kenapa.
- Kalau ada trade-off, kasih rekomendasi + harga yang saya bayar kalau
  ikut rekomendasi itu. Jangan cuma daftar opsi lalu lempar balik ke saya.
- Kalau ada konsekuensi yang belum saya sadari, angkat — walaupun saya
  tidak menanyakannya.
- Kalau saya usul mekanisme yang menurut kamu tidak menyelesaikan masalahnya,
  bilang langsung dan tunjukkan di kasus mana dia gagal. Jangan diiyakan.
- Setelah saya setuju, baru tulis ke ADR, update backlog dan PRD kalau kena,
  lalu commit + push ke branch kerja sesi ini.

Catatan:
- Repo playgrounds/elora-be-go dan playgrounds/elora_spendos TIDAK ada di
  clone ini. Kalau butuh lihat kode asli, bilang — bisa di-attach lewat
  GitHub, tapi remote-nya tertinggal ~1-2 minggu dari dokumen status.
- Semua file ditulis dalam bahasa Inggris. Chat boleh bahasa Indonesia.
```

**Item yang masih terbuka dan tidak terblokir apa pun:**

| Item | Kenapa layak dibahas |
|---|---|
| **2.4** 🔴 | Tiga kasus login. Cabang tengahnya — user sama, device masih pegang tulisan belum ter-sync — adalah jalur kehilangan data terbesar di seluruh desain |
| **3.2** 🔴 | JSON Schema ekstraksi struk. Menentukan bentuk `output_config.format`, field mana opsional, bagaimana confidence dinyatakan |
| **3.3** 🔴 | Aturan angka Indonesia + validasi server. `Rp 1.500,00`, PPN 11%, service charge |
| **1.3 / 1.6 / 1.7 / 1.8** 🟡 | Sisa Track 1. Semuanya internal server, tidak mengunci kontrak — bisa jalan paralel dengan pekerjaan lain |
| **2.2 / 2.3** 🟡 | Rotasi refresh token + grace window, dan session eviction pada device yang lama offline |

---

## B — Write the API contract (item 5.1)

This is a build task, not a discussion. **It needs both product repos, which are not in this
clone** — attach them first or the session cannot do the work.

```
Tulis kontrak API untuk siklus offline-first sync + receipt scanning.

Baca dulu, urut:
1. notes/ADR-0003-offline-first-sync-and-receipt-scan.md — §2.3, §2.4, §2.5,
   §2.11–§2.14 adalah yang mengikat untuk kontrak
2. notes/sync-flow.md — alur push/pull dalam bentuk diagram
3. shared/context/PRD-offline-first-and-receipt-scan.md — §4.2, §4.4, §6
4. .claude/skills/backend/api-design.md — camelCase di JSON, aturan lainnya
5. notes/discussion-backlog.md — item 5.1 dan 5.2

Yang perlu masuk kontrak:
- GET /v1/sync/changes?since=&limit= — changes[], next_cursor, has_more,
  protocol_version
- POST /v1/sync/changes — changes[] dengan kind, id, base_seq, deleted_at,
  payload; hasilnya results[] dengan id, status, reason, server_seq, next_seq.
  status = applied | conflict | rejected, dan conflict ARTINYA TERTULIS.
- POST /v1/receipts/scan — multipart image, hasilnya ekstraksi terstruktur
- Perubahan skema di 4 domain: id jadi UUIDv7 dari klien, deleted_at,
  server_seq, ingested_at
- Endpoint sync lama (POST /v1/sync, GET /v1/sync/status, PUT /v1/sync/{id})
  dihapus, bukan di-deprecate

Sebelum mulai, cek dulu terhadap kode asli — beberapa hal di ADR diturunkan
dari dokumen status, bukan dari skema hidup. Daftarnya ada di bagian
"Verification still owed" di notes/discussion-backlog.md.

PENTING: file api-documentation/*.yaml ada DUA salinan, di elora-be-go dan
di elora_spendos. Keduanya harus bergerak bersamaan. Kalau sudah ada drift
di antara keduanya sekarang, itu bug — catat di notes/, jangan diam-diam
pilih salah satu.

Semua file bahasa Inggris. Chat boleh bahasa Indonesia.
```

**Prasyarat sebelum menjalankan prompt B:**

1. Attach kedua repo ke sesi (`tarlusah/elora-be-go`, `tarlusah/elora_spendos` — sesuaikan
   dengan nama sebenarnya di GitHub).
2. Sadari remote-nya tertinggal ~1–2 minggu dari dokumen status: backend terakhir di-push
   2026-05-09, frontend 2026-05-02, sementara status doc bertanggal 2026-05-15/16.
3. Jawab dulu tiga keputusan manager di PRD §8 kalau ingin kontrak receipt scanning ikut
   lengkap: TTL refresh token, tier model, cap scan bulanan.

---

## Where things stand

**Settled — Track 1 core is closed, nothing wire-visible is left open**

| Item | Outcome |
|---|---|
| 1.1 | Sync domain architecture — thin orchestrator, `SyncPushable` / `SyncPullable`, `pkg/synccontract` (§2.11) |
| 1.2 | Pull reads entity rows; `server_seq` from a per-user in-transaction counter; no changelog (§2.12–§2.13) |
| 1.4 | Collapsed by 1.1 + 1.2 — both candidate read models were cross-domain constructs already excluded |
| 1.5 | `base_seq`; detect / apply / flag; per-record batches; sticky tombstones; budget natural key (§2.14) |

**Recommended next:** **5.1 — the contract** (prompt B). It is what unblocks everyone else:
`@backend` and `@frontend` run in parallel against it, and the frontend mocks from it until the
endpoints land. The remaining Track 1 items are all server-internal and can be settled alongside.

**Quick wins available any time** — manager decisions, no design discussion needed:
2.1 refresh-token TTL · 3.4 monthly scan cap · 3.6 model tier

**Independent tracks:** Track 2 (auth) · Track 3 (receipt scanning)

---

## Things a future session should not re-litigate

Recorded because each was worked through and closed with reasons, and each is the kind of idea
that looks attractive on first encounter:

- **Event + projection as the source of pull changes** — contradicts 1.1 structurally, and the
  entity and projection writes land in different transactions. Kafka does not fix it. (§2.12)
- **Global Postgres sequence for `server_seq`** — a lower sequence committing after a higher one
  lets a paginating client skip a record with no error. (§2.12)
- **Server ingest timestamp instead of an integer** — orders by arrival exactly as the counter
  does, so it changes no outcome, and adds three silent failure modes. (§2.14)
- **Keying sync by device** — detects staleness worse than `base_seq` does, and reintroduces a
  `devices` table on an identity reinstalls do not preserve. (§2.14)
- **Advisory lock per user + entity** — the conflicting writes are days apart, so the lock is
  uncontended exactly when it would be needed. (§2.14)

The general shape: **conflict is semantic, not mechanical.** No sequence, clock, lock, or table
can decide which of two human intentions was right. The levers are only: prevent a second writer,
detect staleness, or set a policy.
