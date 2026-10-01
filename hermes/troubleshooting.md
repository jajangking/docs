# ⚠️ Post-mortem: `install.sh` Gagal di Perangkat Ini

> **TL;DR** — `curl ... install.sh | bash` **gagal** di perangkat ini dan akan
> selalu gagal. Penyebabnya: `PATH` memuat prefix Termux, sehingga PM salah
> mendeteksi host sebagai *bionic* (Android) dan mencoba meng-*build* dependency
> dari source dengan Rust — yang mustahil di Android.
>
> **Solusi permanen:** Hermes di sini sudah di-install dengan **glibc**
> toolchain. **Jangan** menjalankan `install.sh` mentah. Untuk perbaikan
> environment, pakai `pm.cli` langsung (lihat §6).

---

## 0. Gejala umum → solusi

| Gejala | Penyebab & Solusi |
|---|---|
| `command not found: hermes` | Shell belum reload → `source ~/.bashrc`. |
| Doctor: *"Run 'hermes setup'"* | API key belum diisi → `hermes setup`. |
| `hermes doctor` exit code 1 | **Normal**, bukan error. Exit 1 = ada issue yang perlu dibereskan. |
| Doctor: ⚠ `browser-cdp` / `computer_use` | Driver eksternal belum di-install → `hermes tools`. Bukan error. |
| Doctor: ⚠ `discord` (missing token) | Token bot belum di-set → `~/.hermes/.env`. |
| `no pinned uv build for this platform` | `uname -m` tidak dikenali. Harus `aarch64`/`arm64`. |
| `uv sync exited 1` | Baca `~/.hermes/logs/install.log` — 33 baris terakhir berisi penyebabnya. |
| `Target triple not supported by rustup` | PM salah deteksi bionic. Lanjut ke § 2. |
| `git fetch` macet saat `hermes update` | Partial clone + jaringan lambat. Lanjut ke § 8. |

Log untuk di-debug:

```bash
tail -50 ~/.hermes/logs/install.log
hermes doctor --fix               # auto-fix yang bisa
hermes security                   # audit supply-chain venv/plugin/MCP
```

---

## 1. Gejala

```
✗ venv: uv sync exited 1
  Python reports SOABI: cpython-314-aarch64-linux-android
  Computed rustc target triple: aarch64-unknown-linux-android
  Target triple not supported by rustup: aarch64-unknown-linux-android
  hint: `firecrawl-anydoc` (v0.2.4) ... depends on `firecrawl-anydoc`
```

Semua dependency berikut gagal di-*build* dari source:
`firecrawl-anydoc`, `cffi`, `cryptography`, `pydantic-core`, `httptools`,
`rpds-py`, `jiter`, `watchfiles`, `markupsafe`, `pillow-heif`.

---

## 2. Akar masalah

Perangkat ini adalah **Termux di atas Android**, dengan **Debian 13 arm64
proot-distro** (`glibc 2.41`) di dalamnya. Jadi secara teori **bisa** jalan di
glibc. Tapi `PATH` masih memuat prefix Termux di bagian akhir:

```
/data/data/com.termux/files/usr/bin     # ← bionic
```

### Rantai penyebab (7 langkah)

| # | Yang terjadi | Kode |
|---|---|---|
| 1 | Installer menjalankan pencarian Python | `install.sh` → `bootstrap_python()` → `uv python find --system 3.14` |
| 2 | Prefix Termux ada di `PATH` → **menemukan Python Termux 3.14.6** (bionic) dan memakainya sebagai bootstrap Python | — |
| 3 | PM berjalan di bawah interpreter itu. `_is_bionic_libc()` membaca `sysconfig` **dari bootstrap Python yang sedang berjalan**, bukan dari userland native | `pm/store.py:83` |
| 4 | `ANDROID_API_LEVEL` terisi → PM menyimpulkan host ini *bionic* | `pm/store.py:99` |
| 5 | Target jadi salah → PM mengunduh toolchain bionic | `current_target()` → `linux-arm64-bionic` |
| 6 | PyPI **tidak punya wheel untuk Android/bionic** → semua paket C/Rust harus di-*build* dari source | — |
| 7 | `firecrawl-anydoc` butuh Rust → `maturin` → `rustup`. Tapi `rustup` **tidak mendukung** `aarch64-unknown-linux-android` → build gagal | — |

**Titik kegagalan sesungguhnya adalah langkah 3.** `_is_bionic_libc()`
bertanya ke *interpreter yang sedang jalan*, bukan ke *userland native*.
ENE-nya benar, tapi pertanyaannya salah — dan disebabkan oleh langkah 2 yang
sudah salah duluan.

---

## 3. Perbaikan yang diterapkan

Buang prefix Termux dari `PATH` saat menjalankan install/PM, supaya bootstrap
Python yang dipakai adalah CPython **glibc** (python-build-standalone).

Bukti perubahan:

| | Sebelum (gagal) | Sesudah (fix) |
|---|---|---|
| platform tag | `android-24-arm64_v8a` | `linux-aarch64` |
| SOABI | `cpython-314-aarch64-linux-android` | `cpython-314-aarch64-linux-gnu` |
| `ANDROID_API_LEVEL` | terisi → bionic | `0` → **glibc** |
| Wheel PyPI | tidak ada → compile | `manylinux_2_17_aarch64` |

Karena target-nya jadi glibc, `firecrawl-anydoc` langsung memakai wheel
`manylinux_2_17_aarch64` yang sudah tersedia di PyPI — **tidak perlu Rust
sama sekali**.

> Verifikasi: setelah fix, `pm.cli install` selesai dengan
> `✓ ffmpeg  ✓ node  ✓ npm  ✓ python  ✓ ripgrep  ✓ venv`.

---

## 4. Cara mendiagnosis cepat

Kalau suatu saat PM/venv gagal build, cek dulu ini — sebelum blaming apa pun:

```bash
python3 -c "
import sysconfig
print('platform         :', sysconfig.get_platform())
print('ANDROID_API_LEVEL:', repr(sysconfig.get_config_var('ANDROID_API_LEVEL')))
"
```

**Kriteria:** kalau `platform` diawali `android-*` **atau** `ANDROID_API_LEVEL`
bukan `0`, kamu sedang menjalankan interpreter yang salah. Itu root cause-nya.

---

## 5. Perbaikan environment (copy-paste)

Cara paling aman untuk regenerate runtime — interpreter glibc secara eksplisit,
PATH bersih dari Termux:

```bash
cd /root/.hermes/hermes-agent
env -u TERMUX_VERSION -u PREFIX \
  PATH="/root/.hermes/tools/python-3.14.7+20260901-linux-arm64/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin" \
  /root/.hermes/tools/python-3.14.7+20260901-linux-arm64/bin/python3 \
  -m pm.cli install
```

Bisa juga tanpa menentukan interpreter (kalau `python3` di `PATH` sudah glibc):

```bash
env -u TERMUX_VERSION -u PREFIX \
  PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin" \
  <interpreter-glibc> -m pm.cli install
```

Setelah selesai, pastikan target-nya memang glibc:

```bash
python3 -c "import json; [print(f\"{k:8} {v['target']}\") for k,v in sorted(json.load(open('/root/.hermes/tools/facts.json'))['packages'].items())]"
```

Semua baris harus `linux-arm64`, **bukan** `linux-arm64-bionic`.

---

## 6. Catatan: jangan pakai `install.sh` mentah

Kalau kamu butuh reinstall penuh, `install.sh` akan mendeteksi Termux dan
**menolak** — itu memang sengaja dijaga upstream, karena source install di
Termux tidak didukung. Pesannya:

> ✗ Termux is installed from its APT repository, not install.sh:
> `pkg install hermes-agent`

Batas yang sama juga ada di dalam codebase-nya sendiri
(`hermes_cli/steward.py`):

> Source installs are not supported on Termux — `hermes update` would build
> Python packages on the device.

**Tapi** guideline upstream itu mengasumsikan Termux *murni* (bionic, tanpa
proot). Karena kita punya **Debian glibc** di atasnya, source install
sebenarnya *bisa* bekerja — asalkan PM tidak salah mendeteksi. Karena itu
perbaikannya bukan "jangan install Hermes", melainkan "paksa target glibc".

---

## 7. Lokasi kode relevan

| File | Fungsi | Peran |
|---|---|---|
| `pm/store.py:83` | `_is_bionic_libc()` | Penyebab utama — deteksi dari interpreter, bukan userland |
| `pm/store.py:149` | `current_target()` | Mengembalikan `linux-arm64-bionic` atau `linux-arm64` |
| `hermes_cli/steward.py` | — | Pesan "source install tidak didukung di Termux" |
| `pm/lock.json` | — | Manifest toolchain; **punya** baris glibc untuk semua tool |
| `/root/.hermes/tools/facts.json` | — | Manifest toolchain yang *sudah terpasang* di perangkat ini |

---

## 8. Update macet (`git fetch` stall)

**Known issue:** repo ini di-*clone* dengan `--filter=tree:0` (treeless clone).
Setiap update harus fetch blob on-demand dari GitHub. Di koneksi lambat,
proses ini bisa **stall** — saat diuji, 0 byte selama 20 detik lalu hang.

Kalau `hermes update` macet, itu **bukan** install-nya rusak.

Workaround — nonaktifkan partial clone, lalu fetch penuh sekali:

```bash
git -C /root/.hermes/hermes-agent config remote.origin.promisor false
git -C /root/.hermes/hermes-agent config remote.origin.partialclonefilter ""
git -C /root/.hermes/hermes-agent fetch --refetch origin main
```

> ⚠️ Command di atas **belum pernah diuji** di perangkat ini (diturunkan dari
> pengetahuan git, bukan dari percobaan). Kalau tidak berhasil, kabari — jangan
> diasumsikan jalan.

Alternatif yang lebih aman: pin ke commit tertentu dan skip update:

```bash
hermes update --commit <sha>
```

---

## Lihat juga

- [Panduan pengunaan](README.md) — quick start, konfigurasi, update
- [Indeks dokumentasi](../README.md) — daftar semua tool di perangkat ini
