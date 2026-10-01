# ⚠️ Troubleshooting chroot-ng

Gejala → sebab → solusi. Untuk panduan pemakaian umum lihat
[README.md](README.md).

---

## 0. Tabel cepat

| Gejala | Penyebab & Solusi |
|---|---|
| `Segmentation fault` saat masuk shell | Readline crash. **Solved** — default pakai `--noediting`. Paksa: `cng --shell dash` |
| `dipanggil dari dalam proot` | chroot-ng tidak bisa nested. `exit` dulu ke shell Termux |
| `rootfs 'debian' tidak ada` | Container hilang. `proot-distro list` untuk cek |
| `bad interpreter` | bash belum ada di Termux. `pkg install bash` |
| `cannot load -b (.../rootfs/-b)` | Urutan option salah. Semua option harus **sebelum** `<rootfs>` |
| `chroot-ng: syscall 99 ... -> emulated` | **Bukan error.** Emulasi by-design. Di-filter wrapper |
| `setlocale: LC_CTYPE: cannot change locale` | Guest tidak punya locale itu. Fixed: wrapper pakai `C.UTF-8` |
| `command not found: python3` | Hanya ada `python3.13`. `ln -s /usr/bin/python3.13 /usr/bin/python3` |
| `--list` menampilkan distro yang rusak | Sudah diperbaiki: wrapper cek isi rootfs, bukan cuma `isdir` |
| Tidak ada output di mode perintah | Sudah diperbaiki: ganti process substitution ke temp file |
| `PATH` memuat `/data/data/com.termux/...` | Profile proot bocor ke guest. Sudah ada guard, lihat §6 |
| `container=proot-distro` di dalam chroot-ng | Gejala yang sama. Guard di §6 meng-unset-nya |
| `Permission denied` saat `uv sync` / `hermes` | Symlink `.l2s` proot menunjuk path host. **Solved** — 22.123 symlink di-rewrite, lihat §8 |
| `ImportError: cannot import name 'YAML'` | Akibat dari baris di atas: `uv sync` gagal jadi `site-packages` kosong |
| `ls .l2s` selalu 0 padahal file ada | Normal — isinya dotfile `.l2s.*`. Pakai `ls -a` |
| `Segmentation fault` saat jalanin `opencode` | **Belum jelas.** Tidak ter-reproduksi dengan harness valid. Curigai memori: 209 MB free, binary 199 MB |
| `python3` di proot berperilaku aneh | `python3` di `proot-distro login` itu **bionic Termux**, bukan glibc. Lihat §9 |

---

## 1. `Segmentation fault` saat masuk shell

### Gejala

```
$ cng
chroot-ng: syscall 99 not permitted here -> emulated
root@localhost:~# Segmentation fault
```

### Diagnosis

Penyebabnya **readline** (library yang menangani input TTY), bukan bash,
bukan job control, bukan locale, bukan ukuran terminal.

Bukti:

| Kondisi | Hasil |
|---|---|
| `bash -l -i` baca dari TTY | **crash** |
| `bash -l -i --noediting` | aman |
| `bash -l -i` baca dari pipe / `< /dev/null` | aman |
| `dash -i` (tanpa readline) | aman selalu |
| `set +m` (job control dimatikan) | masih crash |
| `TERM` = `dumb`, `C.UTF-8`, `en_US.UTF-8`, kosong | semua crash (bukan penyebab) |
| `TIOCGWINSZ` 0x0 / 1x1 / 24x80 / 38135x10869 / 5000x5000 | semua aman (bukan penyebab) |

Solusi sudah diterapkan: wrapper memakai `--noediting` secara default untuk
shell interaktif.

**Trade-off:** tombol panah dan pencarian riwayat tidak berfungsi. Perintah,
pipe, redirect, `exit` — semuanya normal.

### Kalau tetap perlu readline penuh

```bash
cng --editing
```

Kalau `--noediting` masih crash (tidak seharusnya — 5/5 aman), pakai dash:

```bash
cng --shell dash
```

---

## 2. Error-option wrapper

### 2.1 `cannot load -b (.../rootfs/-b)`

**Sebab:** urutan salah. Formatnya:

```
chroot-ng [OPSI...] <rootfs> <program> [argumen...]
```

Semua option harus **sebelum** path rootfs. Kalau tidak, `-b` dianggap
sebagai bagian dari path rootfs.

### 2.2 `chroot-ng-distro: dipanggil dari dalam proot`

**Sebab:** chroot-ng tidak bisa di dalam proot — semua path syscall
mengembalikan `errno=38` ENOSYS.

**Solusi:** `exit` dari sesi proot sampai kembali ke shell Termux, lalu jalankan
lagi. Ini perilaku yang benar, bukan bug.

Untuk memastikan tidak ada variabel proot yang bocor:

```bash
echo "PROOT_L2S_DIR=${PROOT_L2S_DIR:-<kosong>} container=${container:-<kosong>}"
```

Keduanya harus kosong.

---

## 3. Noise stderr yang bukan error

```
chroot-ng: syscall 99 not permitted here -> emulated
chroot-ng: syscall 293 not permitted here -> emulated
chroot-ng: syscall 439 not permitted here -> emulated
```

**Bukan error.** chroot-ng memberi tahu syscall mana yang dia emulasikan:

| Nomor | Syscall |
|---|---|
| 99 | `set_robust_list` |
| 293 | `pipe2` |
| 439 | `faccessat2` |

Wrapper memfilter baris `not permitted here` di **mode perintah** saja. Mode
interaktif menampilkan mentah, dan **error asli tetap lewat** — filter cuma
menghapus baris yang persis pola tersebut.

Lihat mentah semua:

```bash
cng --raw 'echo halo'
```

---

## 4. Locale

### Gejala

```
bash: warning: setlocale: LC_CTYPE: cannot change locale (en_US.UTF-8): No such file or directory
```

**Sebab:** rootfs Debian hanya punya locale `C`, `C.utf8`, `POSIX`. Nilai
`LANG` dari Termux (`en_US.UTF-8`) diteruskan tapi tidak ada di guest.

**Sudah diperbaiki:** wrapper tidak lagi mewarisi `LANG`/`LC_ALL` dari host,
selalu memakai `C.UTF-8`.

Cek locale yang tersedia:

```bash
cng 'locale -a'
```

Kalau butuh locale lain:

```bash
cng 'apt install locales && locale-gen en_US.UTF-8'
```

---

## 5. `python3` tidak ditemukan

**Sebab:** Debian 13 hanya menyediakan `python3.13`, tanpa symlink generik.

**Solusi:**

```bash
cng 'ln -s /usr/bin/python3.13 /usr/bin/python3'
```

Tidak mendesak kalau app-mu (mis. Hermes via `uv`) sudah pakai path lengkap.

---

## 6. `PATH` terkontaminasi bionic

### Gejala

```console
$ cng 'echo "$PATH"'
/root/.local/bin:/usr/local/lib/nodejs/current/bin:/root/.opencode/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/data/data/com.termux/files/usr/bin
```

Entry terakhir itu **biner bionic** (Android) di dalam environment **glibc**.
Juga `container=proot-distro` masih ada di dalam guest.

### Penyebab

`/etc/profile.d/termux-profile.sh` — dipasang **proot-distro** saat container
dibuat. Isinya men-append prefix bionic Termux ke `PATH`, mengekspor ~40
variabel `ANDROID_*`, dan men-set `container=proot-distro`.

Untuk proot itu benar dan berguna. Untuk chroot-ng tidak:

| Yang bocor | Akibat |
|---|---|
| prefix bionic di `PATH` | Biner bionic dieksekusi guest glibc |
| `ANDROID_ROOT`, `BOOTCLASSPATH`, dll. (~40) | App salah mendeteksi host — kelas bug yang pernah melukai Hermes |
| `container=proot-distro` | Men memicu guard proot di `chroot-ng-distro` sendiri |

Yang membuatnya subtile: `bash -l` (**interaktif**) menjalankan
`/etc/profile.d`, sedangkan mode perintah juga memulihkan `PATH` dari `-E`.
Jadi gejalanya hanya terlihat di shell interaktif.

### Solusi (sudah terpasang)

`/etc/profile.d/zz-cng-nobionic.sh` di rootfs, aktif **hanya** kalau
`CHROOTNG_GUEST=1` yang di-set wrapper:

```sh
if [ -n "${CHROOTNG_GUEST:-}" ]; then
  case ":${PATH}:" in
    *":/data/data/com.termux/files/usr/bin:"*)
      PATH=$(printf "%s" "$PATH" | tr ":" "\n" \
        | grep -v "^/data/data/com.termux/files/usr/bin$" | paste -sd:)
      export PATH
      unset container PROOT_L2S_DIR
      ;;
  esac
fi
```

Verifikasi:

```console
$ cng 'printenv PATH'
/root/.local/bin:/usr/local/lib/nodejs/current/bin:/root/.opencode/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

$ cng 'echo "${container:-KOSONG}"'
KOSONG
```

Dan proot **tidak berubah**:

```console
$ proot-distro login debian -- bash -lc 'echo "${container:-KOSONG}"'
proot-distro
```

### Kalau `PATH` bocor lagi

`proot-distro` menulis ulang `termux-profile.sh` saat container di-reinstall.
Cek guard-nya masih ada:

```bash
ls -l "$PREFIX/var/lib/proot-distro/containers/debian/rootfs/etc/profile.d/"
```

Kalau `zz-cng-nobionic.sh` hilang, buat ulang dari blok di atas. Kalau hanya
mengocek sendiri tanpa mengubah apa pun:

```bash
cng 'printenv PATH' | tr ':' '\n' | grep termux
```

---

## 7. `run-as` — akses Termux dari adb

Temuan yang tidak terduga saat debugging: shell adb (`uid 2000`) **tidak bisa** baca
data app Termux. Tapi kalau aplikasi Termux-mu `debuggable`, `run-as` memberi
akses sebagai user Termux sendiri:

```bash
adb shell 'run-as com.termux id'
# uid=10496(u0_a496) ...
```

Dari situ bisa baca/tulis `$PREFIX`, jalankan tes, dan ambil exit code asli:

```bash
P=/data/data/com.termux/files/usr
adb shell "run-as com.termux $P/bin/bash -c 'cat > $P/tmp/tes.sh && chmod 755 $P/tmp/tes.sh'" < tes.sh
adb shell "run-as com.termux $P/bin/bash -s" < tes.sh
```

Kenapa ini penting: `/data/local/tmp` ber-mode `drwxrwx--x shell shell`, jadi
`uid 2000` saja tidak bisa membacanya. `run-as` melewati batas itu.

> `run-as` hanya berhasil kalau app `debuggable` — kalau `run-as` bilang
> `package not debuggable`, berarti perlu pendekatan lain (install ulang
> Termux dari F-Droid, atau pakai `RunCommandService` yang butuh permission).

---

## 8. `Permission denied` saat `uv sync` / `hermes`

### Gejala

```
✗ Installing Python dependencies failed
    error: Failed to install: packaging-26.0-py3-none-any.whl
      Caused by: failed to open file
                 `/root/.hermes/cache/uv/archive-v0/.../WHEEL`:
                 Permission denied (os error 13)
```

Disusul `ImportError: cannot import name 'YAML' from 'ruamel.yaml'`. **Satu
akar masalah, dua gejala** — `uv sync` gagal sehingga `site-packages` tidak
lengkap, lalu import jadi benar-benar tidak ada.

Penjelasan lengkap ada di [README.md §6](README.md). Ringkasnya:

```
proot --link2symlink menulis symlink ke path HOST absolut:
  $RFS/root/.hermes/.../WHEEL -> $RFS/.l2s/.l2s.WHEEL0002

proot    : menerjemahkan -> "Wheel-Version: 1.0"  (OK)
chroot-ng: baca apa adanya -> /data/data/... tidak ada di guest -> EACCES
```

### Perbaikan

Rewrite 22.123 symlink dari host-absolute jadi guest-absolute:

```bash
P=/data/data/com.termux/files/usr
RFS="$P/var/lib/proot-distro/containers/debian/rootfs"

# 1) backup target asli dulu (WAJIB)
python3 "$P/tmp/d20.py" backup

# 2) rewrite
python3 "$P/tmp/d20.py" apply

# 3) verifikasi: harus keluar 0
python3 "$P/tmp/d20.py" verify
```

Atau tanpa skrip, kalau mau satu-offs:

```bash
find "$RFS" -xdev -type l -lname "$RFS/*" -print0 |
while IFS= read -r -d '' l; do
  t=$(readlink "$l")
  case "$t" in "$RFS"/*) ln -sfn "${t#$RFS}" "$l" ;; esac
done
```

> **Jangan pakai loop bash untuk 22.000 symlink.** `readlink` per symlink di
> loop shell butuh ~15 menit untuk 2.689 baris. Versi Python satu `os.walk`
> selesai dalam **57 detik** untuk backup dan **11 detik** untuk apply.

### Kalau proot malah rusak

Restore dari backup:

```bash
python3 "$P/tmp/d20.py" restore
```

Lalu cek symlink sistem tidak ikut berubah:

```bash
for l in bin/sh lib sbin; do readlink "$RFS/$l"; done
# harus: dash / usr/lib / usr/sbin
```

### Kalau muncul lagi setelah `proot-distro install`

`proot-distro` membuat symlink `.l2s` baru dengan bentuk host-path lagi.
Gejalanya sama: `Permission denied`. Perbaikannya idempoten — jalankan ulang
langkah di atas.

### Jebakan: `ls .l2s` selalu kosong

Normal, bukan tanda `.l2s` terhapus. Semua entry bernama `.l2s.*` — dotfile,
yang disembunyikan `ls` tanpa `-a`:

```bash
ls    "$RFS/.l2s" | wc -l    # 0
ls -a "$RFS/.l2s" | wc -l    # 15.622
```

---

## 9. Diagnosis manual

Kalau perlu menelusuri sendiri, tiga perintah ini paling berguna:

```bash
# capability
cng --probe

# capability detail
CNG_DEBUG=1 cng 'echo halo'

# PATH & user di dalam guest
cng 'echo "$PATH"; id; echo $HOME'
```

Untuk dumping syscall di dalam guest:

```bash
cng 'strace -f -o /tmp/st.txt /bin/bash -l -i'
cng 'tail -30 /tmp/st.txt'
```

Tapi perhatikan: `strace` di bawah chroot-ng **tidak** menangkap `SIGSEGV`
untuk crash readline — dia hanya bisa menunjukkan loop `pselect6`. Itu
sendiri petunjuk bahwa crash-nya race condition, tapi tidak menghasilkan
alamat fault yang berguna.

### Cara menguji dengan benar (hemat waktu)

Pengujian lewat `adb shell` **wajib** lewat `run-as com.termux` (§7), bukan
shell adb biasa. Alasannya: `/data/local/tmp` mode `drwxrwx--x shell shell`,
jadi `uid 2000` tidak bisa membacanya, dan tidak bisa menyentuh `$PREFIX`.

### Jebakan yang memakan jam debugging

| Jebakan | Gejala | Solusi |
|---|---|---|
| `os.environ.clear()` di harness python | `libtinfo.so.6: cannot open shared object file: Error 38` | Jangan hapus semua env. chroot-ng butuh `LD_LIBRARY_PATH` dari host |
| Harness tak membersihkan `PROOT_L2S_DIR` / `container` | `dipanggil dari dalam proot` padahal tidak di proot | `os.environ.pop(k, None)` untuk keduanya sebelum `execv` |
| `proot -r "$RFS"` tanpa bind `/dev`, `/proc` | `can't chdir(...)`, `could not open '/dev/null'`, `tail: not found` | Gunakan `proot-distro login` saja — dia yang pasang bind-nya |
| `python3` di proot dipakai untuk tes | `ModuleNotFoundError` padahal paket ada | `python3` itu **bionic Termux**. Pakai `proot-distro login debian -- python3` atau path glibc eksplisit |

`Error 38` = ENOSYS, dan itu **artefak harness**, bukan bug chroot-ng. Mode
perintah yang terlihat "rusak" karena itu sebenarnya sehat.

### Bentuk harness yang benar

```python
def child_env():
    # buang HANYA penanda proot, sisanya warisi apa adanya
    for k in ("PROOT_L2S_DIR", "container"):
        os.environ.pop(k, None)
    os.environ["TERM"] = "xterm-256color"
    os.environ["PREFIX"] = P
    os.environ["TMPDIR"] = P + "/tmp"
    os.environ["PATH"] = P + "/bin:/system/bin"
    # JANGAN os.environ.clear() -- LD_LIBRARY_PATH wajib ada
```

Matriks yang sudah diuji, supaya tidak perlu mengulang:

| Mode env | Hasil | Arti |
|---|---|---|
| warisi apa adanya | guard proot menyala | **paling realistis** — persis seperti shell Termux |
| `minimal` (tanpa `LD_LIBRARY_PATH`) | `Error 38` ENOSYS | ❌ bukan kontrol yang sah |
| `clear()` + set env baru | `Error 38` ENOSYS | ❌ bukan kontrol yang sah |

Dua baris bawah penting: harness yang menghapus semua env **bukan kontrol**,
dan hasilnya tidak boleh dipakai menyimpulkan apa pun. Semua tes yang
menghasilkan "chroot-ng rusak" pagi itu keliru karena alasan ini.

### Soal `opencode` yang sempat segfault

Dengan harness di atas, `opencode` **tidak** ter-reproduksi: 5 dari 5 kasus
lolos, termasuk TUI di shell interaktif, tanpa signal. Jadi tidak ada bukti
chroot-ng penyebabnya.

Yang tersisa sebagai hipotesis: **tekanan memori**.

```
Mem: 7686 total, 209 free, 1510 available
Swap: 4377 terpakai dari 7686
opencode = binary 199.936.296 byte
```

Kill proses besar dengan sisa 209 MB free memang pantas dicurigai. Tapi
ini **belum diuji** — belum ada yang mengukur apakah allocate gagal. Jangan
sebut ini sebagai penyebab sebelum ada datanya.

---

## 10. Kalau semuanya gagal

Kembali ke proot — tidak ada yang hilang, rootfs-nya sama persis:

```bash
proot-distro login debian
```

`chroot-ng` hanya lapisan eksekusi tambahan di atas rootfs yang sama. Rootfs,
`/root/docs`, `hermes`, `opencode` — semuanya utuh di kedua mode.
