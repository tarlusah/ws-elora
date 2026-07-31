# Resume prompt

Paste this into a fresh session to continue the technical design discussion.
Replace `[ITEM]` with the backlog number you want to work on.

---

```
Lanjutkan pembahasan desain teknis Elora.

Baca dulu, urut:
1. notes/discussion-backlog.md — daftar item diskusi + status per item
2. notes/ADR-0003-offline-first-sync-and-receipt-scan.md — keputusan yang sudah final
3. shared/context/PRD-offline-first-and-receipt-scan.md — scope siklus ini
4. .claude/skills/backend/*.md — aturan arsitektur backend yang mengikat
5. shared/context/PRD.md — baseline status proyek

Konteks: kita sedang merancang perubahan besar — offline-first sync + receipt
scanning pakai LLM. Dibahas satu item backlog per sesi supaya tidak menumpuk.
Track 1 item 1.1–1.5 sudah selesai (ADR §2.11–§2.14); 1.4 tertutup
sendiri. Kontrak API sudah tidak terblokir, Track 4 sudah bisa jalan.

Sekarang bahas item: [ITEM]

Cara kerja yang saya mau:
- Bahas dulu sampai tuntas. Jangan langsung tulis file.
- Saya butuh penilaian jujur, bukan validasi. Kalau rumusan saya lemah,
  bilang lemahnya di mana dan kenapa.
- Kalau ada trade-off, kasih rekomendasi + harga yang saya bayar kalau
  ikut rekomendasi itu. Jangan cuma daftar opsi lalu lempar balik ke saya.
- Kalau ada konsekuensi yang belum saya sadari, angkat — walaupun saya
  tidak menanyakannya.
- Setelah saya setuju, baru tulis ke ADR, update backlog dan PRD kalau
  kena, lalu commit + push ke branch kerja sesi ini. (PR #1 sudah
  merged — jangan menumpuk commit di branch itu lagi.)

Catatan:
- Repo playgrounds/elora-be-go dan playgrounds/elora_spendos TIDAK ada di
  clone ini (symlink ke repo lokal). Kalau butuh lihat kode asli, bilang —
  bisa di-attach lewat GitHub, tapi remote-nya tertinggal ~1-2 minggu dari
  dokumen status.
- Semua file ditulis dalam bahasa Inggris. Chat boleh bahasa Indonesia.
```

---

## Where things stand

**Settled — Track 1 core is done, nothing wire-visible is left open:**
1.1 sync domain architecture (§2.11) · 1.2 pull reads entity rows, `server_seq` from a per-user
in-transaction counter (§2.12–§2.13) · 1.4 collapsed by 1.1 + 1.2 · 1.5 push semantics —
`base_seq`, detect/apply/flag, per-record batches, sticky tombstones, budget natural key (§2.14)

**Recommended next:** **5.1 — the OpenAPI contract.** It is now unblocked and it is what
unblocks everyone else: `@backend` and `@frontend` run in parallel against it, and the frontend
mocks from it until endpoints land. Remaining Track 1 items (1.3, 1.6, 1.7, 1.8) are all
server-internal and can be settled while that work proceeds.

If you would rather keep discussing design: **2.4** (three login cases — the middle one is the
primary data-loss path in the whole design) or **3.2** (receipt extraction JSON Schema) are the
two remaining 🔴 items that are not blocked by anything.

**Quick wins available any time** — manager decisions, no design discussion needed:
2.1 refresh-token TTL · 3.4 monthly scan cap · 3.6 model tier

**Independent tracks** — can be picked up without waiting on Track 1:
Track 2 (auth) · Track 3 (receipt scanning)
