# Hermes Agent — Panduan Penggunaan

Hermes Agent terpasang di perangkat ini dengan **glibc** toolchain.
Dokumen ini adalah panduan penggunaan. Untuk penjelasan *mengapa* `install.sh`
mentah gagal di sini, lihat
[troubleshooting.md](troubleshooting.md).

---

## Ringkasan

| Item | Nilai |
|---|---|
| Versi | `v0.21.5+5355.g357f51c` (rilis 2026.9.24) |
| Metode | `git` (source checkout) |
| Python | 3.14.7 — `linux-aarch64`, **glibc** |
| `HERMES_HOME` | `/root/.hermes` |
| Install dir | `/root/.hermes/hermes-agent` |
| Launcher | `/root/.local/bin/hermes` |

Toolchain terkelola (semua glibc): Python 3.14.7 · uv 0.12.3 · Node 26.7.0 ·
npm 12.0.2 · ffmpeg 9.0.1 · ripgrep 15.2.0

---

## Quick Start

```bash
hermes setup        # WAJIB — masukkan API key provider
hermes doctor       # cek kesehatan
hermes chat         # chat interaktif
```

Sekali jalan, non-interaktif:

```bash
hermes -z "jelaskan isi README.md di direktori ini"
```

> `hermes setup` tidak bisa dilewati — tanpa API key, agent tidak bisa
> menjalankan model. `hermes doctor` akan tetap complaining sampai itu diisi.

---

## Struktur direktori

```
/root/.hermes/                     # HERMES_HOME
├── .env                           # API key (chmod 600, JANGAN di-commit)
├── config.yaml                    # setelan perilaku (non-secret)
├── hermes-agent/                  # source checkout
│   └── .hermes/bin/hermes         # launcher asli
├── tools/                         # toolchain terkelola (facts.json = manifest)
├── installs/                      # runtime generation milik PM
├── cache/uv/                      # cache uv
└── logs/                          # install.log, update_receipts/
```

---

## Konfigurasi

### API key (secret)

Semua secret di `~/.hermes/.env` (mode `600`), atau via `hermes setup`:

```bash
vim ~/.hermes/.env
```

```bash
OPENROUTER_API_KEY=sk-or-...
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
DISCORD_BOT_TOKEN=...
```

### Perilaku (non-secret)

Setelan seperti timeout, toolset default, urutan provider, dll ada di
`~/.hermes/config.yaml`. **Jangan** taruh secret di sini.

---

## Perintah yang sering dipakai

| Perintah | Fungsi |
|---|---|
| `hermes chat` | Chat interaktif |
| `hermes -z "<prompt>"` | Sekali jalan, non-interaktif |
| `hermes setup` | Wizard konfigurasi awal |
| `hermes model` | Pilih model + provider default |
| `hermes status` | Ringkasan komponen |
| `hermes tools` | Pasang / matikan tool |
| `hermes doctor` | Cek konfigurasi & dependency |
| `hermes logs --follow` | Tail log agent |
| `hermes skills list` | Skill Hub |
| `hermes cron` | Scheduled job |
| `hermes gateway` | Telegram/Discord/Slack gateway |
| `hermes --tui` | TUI terminal |
| `hermes security` | Audit supply-chain venv/plugin/MCP |
| `hermes worktree` | Bersihkan git worktree menumpuk |
| `hermes update` | Update ke versi terbaru |

---

## Update

```bash
hermes update
```

> **Known issue:** repo di-*clone* dengan `--filter=tree:0` (treeless), jadi
> tiap update harus fetch blob on-demand dari GitHub. Proses ini kadang
> **stall** di koneksi lambat. Kalau macet, itu bukan install rusak — lihat
> [troubleshooting.md](troubleshooting.md) § Update.

Kalau butuh kestabilan dan tidak keberatan tidak update, pin ke commit
tertentu: `hermes update --commit <sha>`.

---

## Melepas

```bash
hermes uninstall --data      # hapus juga config, chat, sessions
```

---

## Lihat juga

- [troubleshooting.md](troubleshooting.md) — post-mortem `install.sh`,
  diagnosis, dan perbaikan environment
