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
Item 1.1 (bentuk domain sync) sudah selesai, hasilnya ada di ADR §2.11.

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
  kena, lalu commit + push ke branch
  claude/brainstorm-major-technical-changes-tmb3by (PR #1 sudah ada).

Catatan:
- Repo playgrounds/elora-be-go dan playgrounds/elora_spendos TIDAK ada di
  clone ini (symlink ke repo lokal). Kalau butuh lihat kode asli, bilang —
  bisa di-attach lewat GitHub, tapi remote-nya tertinggal ~1-2 minggu dari
  dokumen status.
- Semua file ditulis dalam bahasa Inggris. Chat boleh bahasa Indonesia.
```

---

## Where things stand

**Settled:** 1.1 — sync domain architecture (ADR §2.11)

**Recommended next:** 1.2 — where pull-side changes come from (event + projection vs.
reading entity rows). Note that 1.1 already answered part of it: `ChangesSince` now belongs
to each domain, so what remains is how a domain answers that efficiently, and whether a
changelog earns its keep at all once `server_seq` lives on the row.

**Quick wins available any time** — manager decisions, no design discussion needed:
2.1 refresh-token TTL · 3.4 monthly scan cap · 3.6 model tier

**Independent tracks** — can be picked up without waiting on Track 1:
Track 2 (auth) · Track 3 (receipt scanning)
