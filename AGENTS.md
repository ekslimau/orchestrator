# Delegator — setup team orchestrator OpenCode

Template config tim agent satu arah: **orchestrator** (lead) → **planner / coder / reviewer / explorer** (worker).

Dibuat dari debugging nyata error `Subagent session not found: ses_invalid*`.

## Isi

- `opencode.json` — config OpenCode V2: provider, model, dan 5 agent.
- `AGENTS.md` ini — cara pakai, tangkap masalah, dan aturan prompt.

## Cara pakai

```sh
cd F:\codingan\funny\delegator
opencode --agent orchestrator
```

Di TUI pilih agent `orchestrator`, atau jalankan non-interaktif:

```sh
opencode run --agent orchestrator "Delegasikan ke explorer: cari semua file .md"
```

`default_agent` di config sudah `orchestrator`, jadi sesi baru langsung memakainya.

## Peran

| Agent | Mode | Model | Boleh | Tidak boleh |
|---|---|---|---|---|
| `orchestrator` | primary | gpt-6-astra | delegasi ke 4 worker | edit, shell, panggil diri sendiri |
| `planner` | subagent | gpt-6-astra | baca + rencana | edit, shell, delegasi |
| `coder` | subagent | deepseek-v4.1-flash | edit, shell | delegasi |
| `reviewer` | subagent | gpt-6-astra | baca + review | edit, shell, delegasi |
| `explorer` | subagent | deepseek-v4.1-flash | baca + cari | edit, shell, delegasi |

Alur satu arah: orchestrator → worker. Worker **tidak bisa** memanggil agent lain (`subagent:*` deny).

## Tool `subagent` — argumen yang benar

```
agent        (wajib)  ID worker: planner | coder | reviewer | explorer
description  (wajib)  label singkat untuk UI
prompt       (wajib)  task lengkap, self-contained
model        (opsional) override model
sessionID    (opsional) continue sesi subagent yang sudah ada
background   (opsional) jalankan async, langsung balik
```

Aturan penting:

- **Mulai kerja baru**: JANGAN kirim `sessionID`. Subagent dapat sesi baru.
- **Lanjutkan sesi**: kirim `sessionID` **persis** yang dikembalikan hasil tool sebelumnya.
- **Jangan mengarang ID.** Placeholder seperti `ses_invalid` ditolak:
  `Subagent session not found: ses_invalid`.

## Masalah umum

### `Subagent session not found: ses_invalid*`

**Penyebab:** model mengirim `sessionID` karangan (biasanya karena mau "continue" tapi belum punya ID valid).

**Perbaikan yang sudah tertanam:** system prompt orchestrator di `opencode.json` sudah melarang ini secara eksplisit.

**Kalau masih muncul pada sesi lama:** sesi yang sudah berjalan menyimpan system prompt lama di konteksnya. Edit config **tidak** mengubah sesi aktif. Solusi: **buat sesi orkestrator baru**, jangan lanjutkan sesi lama.

### Warning `configuration normalization diagnostic ... kind=conflict`

Config mencampur bentuk V1 dan V2 di satu entry. Pakai bentuk V2 murni:

| V1 | V2 |
|---|---|
| `provider` | `providers` |
| `npm` | `package` (`@opencode/ai/providers/openai-compatible`) |
| `options` (baseURL/apiKey) | `settings` |
| model `modalities` | `capabilities` (+ `tools: true`) |
| agent `prompt` | `system` |
| agent `permission` | `permissions` |
| agent `maxSteps` | `steps` |

### Warning `configured plugin path must be a directory`

`plugins` menunjuk ke file `.mjs`, bukan direktori. Hapus entri itu atau arahkan ke folder plugin yang benar.

### Delegasi cepat / macet

- Task terlalu besar → pecah jadi beberapa subagent.
- Worker butuh konteks → sertakan path file dan tujuan di `prompt`; child tidak melihat percakapan induk.
- Hanya jalankan paralel untuk task yang benar-benar independen (tidak menulis file yang sama).

## Setup dari nol

1. Salin `opencode.json` ini ke root project atau `~/.config/opencode/`.
2. Ganti `baseURL`, `apiKey`, dan nama model sesuai provider kamu.
3. Jalankan `opencode --agent orchestrator`.
4. Verifikasi tiap agent dengan task kecil ("balas: ok") sebelum kerja besar.

## Verifikasi cepat

```sh
# agent dasar hidup?
opencode run --agent planner "balas: ok"

# delegasi + continue sesi?
opencode run --agent orchestrator "Pakai explorer cari file .gitignore. Lanjutkan sesi explorer yang sama untuk cari README.md, pakai sessionID asli dari hasil tool."
```

Harapan: tidak ada `Subagent session not found`, dan continue memakai ID asli dari return value.

## Sejarah bug (kenapa prompt-nya begini)

Pada OpenCode 2.0.27 tool `subagent` **memang** menerima `sessionID` untuk melanjutkan sesi anak. Bug yang kami temukan bukan skema, tapi model mengisi `sessionID` dengan nilai karangan saat ingin continue tanpa ID valid. Server menolak seluruh call. System prompt orchestrator di sini menutup celah itu dengan menyebut argumen yang benar sekaligus melarang ID karangan.
