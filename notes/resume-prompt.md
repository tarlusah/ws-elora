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
Item 1.1 (bentuk domain sync) selesai di ADR §2.11; item 1.2 (sumber
perubahan sisi pull) selesai di ADR §2.12–§2.13, dan menutup 1.4.

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

**Settled:** 1.1 — sync domain architecture (ADR §2.11) · 1.2 — pull reads entity rows,
`server_seq` from a per-user in-transaction counter (ADR §2.12–§2.13) · 1.4 — collapsed by
1.1 + 1.2, nothing left to decide

**Recommended next:** 1.5 — push semantics. It is the last 🔴 in Track 1 and the last piece of
Track 1 that is **wire-visible and therefore locked at launch**; 1.3 and 1.6 are now small and
mostly mechanical, and 1.7/1.8 are narrowed. Two questions carry it: what `conflict` even means
on a single-device product where the server adjudicates by ingest order (if nothing, the status
should be dropped from the contract before it ships, not after), and whether a batch is atomic
or per-record — ADR §2.11 put transaction granularity in the domain's hands via `ApplyBatch`,
so this decides what the *aggregate* result means when one kind partially fails.

**Quick wins available any time** — manager decisions, no design discussion needed:
2.1 refresh-token TTL · 3.4 monthly scan cap · 3.6 model tier

**Independent tracks** — can be picked up without waiting on Track 1:
Track 2 (auth) · Track 3 (receipt scanning)
