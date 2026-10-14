# Orchestrator

Tim agent OpenCode satu arah: **orchestrator** (lead) mendelegasikan ke **planner / coder / reviewer / explorer**.

Repo ini siap pakai: salin config, jalankan, delegasikan. Dibuat dari debugging nyata error `Subagent session not found: ses_invalid*`.

## Isi

```
orchestrator/
├── opencode.json                              # config tim agent (OpenCode V2)
├── AGENTS.md                                  # instruksi project untuk agent
├── README.md                                  # file ini
├── .gitignore
└── .opencode/
    └── skills/
        ├── orchestrate-delegation/SKILL.md    # cara memecah tugas & mendelegasikan
        └── subagent-troubleshooting/SKILL.md  # memperbaiki delegasi yang gagal
```

## Cara pakai

```sh
git clone https://github.com/ekslimau/orchestrator
cd orchestrator
opencode --agent orchestrator
```

Non-interaktif:

```sh
opencode run --agent orchestrator "Delegasikan ke explorer: cari semua file .md"
```

`default_agent` sudah `orchestrator`, jadi sesi baru langsung memakainya.

## Peran tim

| Agent | Mode | Model | Boleh | Tidak boleh |
|---|---|---|---|---|
| `orchestrator` | primary | gpt-6-astra | delegasi ke 4 worker | edit, shell, panggil diri sendiri |
| `planner` | subagent | gpt-6-astra | baca + rencana | edit, shell, delegasi |
| `coder` | subagent | deepseek-v4.1-flash | edit, shell | delegasi |
| `reviewer` | subagent | gpt-6-astra | baca + review | edit, shell, delegasi |
| `explorer` | subagent | deepseek-v4.1-flash | baca + cari | edit, shell, delegasi |

Alur satu arah: orchestrator → worker. Worker **tidak bisa** memanggil agent lain.

## Tool `subagent` — argumen yang benar

```
agent        (wajib)   ID worker: planner | coder | reviewer | explorer
description  (wajib)   label singkat untuk UI
prompt       (wajib)   task lengkap, self-contained
model        (opsional) override model
sessionID    (opsional) continue sesi subagent yang sudah ada
background   (opsional) jalankan async, langsung balik
```

- **Mulai kerja baru**: JANGAN kirim `sessionID`.
- **Lanjutkan sesi**: kirim `sessionID` **persis** dari hasil tool sebelumnya.
- **Jangan mengarang ID** (`ses_invalid`, dst.) — ditolak dengan `Subagent session not found`.

## Skills

Dua skill disediakan dan dimuat otomatis oleh OpenCode saat relevan:

- **orchestrate-delegation** — memecah pekerjaan, memilih worker, menulis task self-contained, melanjutkan sesi.
- **subagent-troubleshooting** — mendiagnosis `Subagent session not found` dan kegagalan delegasi lain.

Panggil manual dengan `@orchestrate-delegation` atau `@subagent-troubleshooting`.

## Kustomisasi

1. Ganti pengaturan provider (`baseURL`, `apiKey`, nama model) di `opencode.json` sesuai provider kamu.
2. Tambah worker: salin satu blok agent, ubah `description`, `system`, `model`, dan `permissions`.
3. Izinkan worker baru dari orchestrator: tambah rule `subagent` + `resource: <nama>` `effect: allow`.

## Lisensi

MIT.
