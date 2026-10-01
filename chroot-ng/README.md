# chroot-ng (fake-chroot-ng) di Termux

> **TL;DR** — `chroot-ng` adalah pengganti proot yang **tidak pakai ptrace**.
> Di perangkat ini ia sudah terinstall dan **bisa dipakai**. Yang perlu
> diketahui: ia **tidak bisa dijalankan dari dalam proot**, dan shell
> interaktif perlu flag `--noediting` untuk menghindari crash readline yang
> intermiten.
>
> Install: §2 · Pakai: §3 · Masalah: [troubleshooting.md](troubleshooting.md)

---

## 1. Apa itu dan kenapa layak

### Perbandingan mendasar

| | `proot-distro` | `chroot-ng` |
|---|---|---|
| Mekanisme | **ptrace** — menghentikan proses di setiap syscall | **seccomp/SIGSYS** — interupsi in-process |
| Overhead syscall | **tinggi** (1,75x lambat, terukur di perangkat ini) | **rendah** (hampir tidak ada) |
| Startup proses | lambat (node ~159 ms) | jauh lebih cepat |
| Jumlah file rootfs | 212.355 file | sama |
| Bisakah di dalam proot? | ya | **tidak** |

Ptrace menghentikan proses pada *setiap* syscall untuk menerjemahkan path.
Itu mahal. chroot-ng memasang filter seccomp, jadi hanya syscall yang memang
bermasalah yang memicu `SIGSYS` — sisanya jalan native.

### Yang sudah diukur di perangkat ini

```
syscall-rate : 2.206/s (proot+glibc)  vs  3.869/s (native+bionic)   = 1,75x
path-translate: 2.794/s (proot)       vs  3.626/s (native)          = 1,30x
node startup  : 159 ms
rootfs        : 212.355 file
```

> Angka di atas **proot vs native Termux**, bukan proot vs chroot-ng.
> Perbandingan proot vs chroot-ng yang valid belum pernah berhasil diukur —
> lihat §7. Saya tidak akan mengarang angkanya.

---

## 2. Install

### 2.1 Kebutuhan

| Kebutuhan | Status di perangkat ini |
|---|---|
| Termux | ✅ ada |
| bash di Termux | ✅ ada (`pkg install bash`) |
| rootfs Debian via proot-distro | ✅ ada (dipakai bersama proot) |
| Device mendukung seccomp+SIGSYS | ✅ terverifikasi |

### 2.2 Verifikasi capability

```bash
chroot-ng --probe
```

Output yang diharapkan memuat:

```
filter RET_ERRNO WORKS
execmem: RESULT OK
primary in-process tier: LIKELY VIABLE
```

Kalau `LIKELY VIABLE` tidak muncul, chroot-ng tidak akan bekerja — jangan
terus mencoba, kembalikan ke proot.

### 2.3 Sumber binary

Binary dibangun dari source, bukan distro package (belum ada paket resmi
Termux). Yang sudah terpasang di `$PREFIX/bin/chroot-ng` (2.053.208 byte,
static freestanding aarch64).

> **Bug upstream yang ditemukan saat build:** `src/rt/unistd_check.c` gagal
> di-build dengan `-Werror` pada header uapi arm64 Debian. Penyebabnya proyek
> memakai ejaan `__NR3264_fstat`, sedangkan header Debian mendefinisikan
> `__NR_fstat 80` langsung. **Nomornya identik, hanya ejaan berbeda.**
> Compiling file itu manual tanpa `-Werror` membuat build selesai.
> Ini kandidat PR upstream — lihat §7.

### 2.4 Pasang wrapper

Wrapper-nya kompatibel dengan `proot-distro` agar tidak perlu mengingat perintah panjang:

```bash
cp /data/local/tmp/wrap2.sh "$PREFIX/bin/chroot-ng-distro"
chmod 755 "$PREFIX/bin/chroot-ng-distro"
```

### 2.5 Alias (opsional, disarankan)

```bash
echo "alias cng='$PREFIX/bin/chroot-ng-distro'" >> ~/.bashrc
source ~/.bashrc
```

---

## 3. Pemakaian

### 3.1 Perintah dasar

```bash
cng                          # masuk shell interaktif
cng 'git status'             # jalankan satu perintah
cng --list                   # daftar distro
cng --probe                  # cek capability
cng --help                   # bantuan
```

### 3.2 Opsi

| Opsi | Fungsi |
|---|---|
| *(tanpa argumen)* | shell login interaktif, dengan `--noediting` |
| `--list`, `-l` | daftar distro yang terinstall |
| `--probe` | cek capability perangkat |
| `--raw` | jangan filter pesan `not permitted here` |
| `--editing` | aktifkan readline penuh (**risiko crash**) |
| `--shell NAMA` | ganti shell (`bash`, `dash`) |
| `--distro NAMA` | pilih distro (default `debian`) |

### 3.3 Environment

| Variabel | Fungsi |
|---|---|
| `CHROOTNG_DISTRO=NAMA` | distro default |
| `CHROOTNG_BINDS=src:dst,...` | bind mount tambahan, pisahkan dengan koma |
| `CHROOTNG_NOEDIT=0` | matikan `--noediting` (setara `--editing`) |

Contoh bind mount —, kamu perlu akses ke penyimpanan:

```bash
CHROOTNG_BINDS=/sdcard:/mnt/sdcard cng
```

### 3.4 Batasan penting

> **chroot-ng tidak bisa di dalam proot.** Semua path syscall mengembalikan
> `errno=38` ENOSYS. Wrapper menolak pemanggilan dari dalam proot dengan pesan
> jelas — itu perilaku yang benar, bukan bug.
>
> Kalau muncul `dipanggil dari dalam proot`, ketik `exit` dulu sampai kamu
> kembali ke shell Termux, baru jalankan lagi.

---

## 4. Yang sudah diverifikasi

Semua ini diuji di rootfs Debian asli perangkat ini, bukan rootfs uji:

| Uji | Hasil |
|---|---|
| `git --version` | 2.47.3 |
| `node --version` | v26.10.0 (node yang sama dengan opencode) |
| jumlah file `/usr/bin` | 486 (termasuk `strace` yang di-install saat debugging) |
| versi Debian | 13.7 |
| usr-merge symlink | benar (`/lib` → `/usr/lib`, `/bin/sh` → `/usr/bin/dash`) |
| dynamic loader | resolve path guest dengan benar (`libtinfo.so.6` → `/lib/aarch64-linux-gnu/`) |
| environment | bersih: hanya `TERM`/`COLORTERM`/`PWD`/`SHLVL`, tanpa `PATH` warisan host |
| `PATH` di dalam guest | bersih — tidak ada prefix bionic Termux (lihat §5) |
| mode perintah | stabil, noise stderr terfilter |
| shell interaktif | aman dengan `--noediting` |

Rootfs yang dipakai **sama persis** dengan yang dipakai proot. Isi
`/root/docs`, `hermes`, `opencode` — semuanya ada di sana. Ini bukan distro
terpisah.

---

## 5. `PATH` dan profile: kenapa perlu dibereskan

Rootfs Debian punya `/etc/profile.d/termux-profile.sh` yang dipasang
**proot-distro**. Isinya:

- menambahkan `/data/data/com.termux/files/usr/bin` ke `PATH`
- mengekspor ~40 variabel `ANDROID_*` (`ANDROID_ROOT`, `BOOTCLASSPATH`, …)
- men-set `container=proot-distro`

Untuk **proot** itu semua benar dan berguna. Untuk **chroot-ng** tidak:

| Masalah | Akibat |
|---|---|
| Prefix bionic di `PATH` | Biner bionic dieksekusi guest glibc — bisa `Exec format error` atau crash |
| ~40 variabel `ANDROID_*` | App salah mendeteksi host; kelas bug yang pernah bikin Hermes salah deteksi platform |
| `container=proot-distro` | Men memicu guard proot di `chroot-ng-distro` sendiri |

### Solusi yang dipasang

File baru `/etc/profile.d/zz-cng-nobionic.sh` di rootfs, hanya aktif kalau
`CHROOTNG_GUEST=1` (di-set oleh wrapper):

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

Dicek: `bash -l` di bawah chroot-ng menghasilkan `PATH` bersih dan
`container` kosong; di bawah proot tetap `container=proot-distro` dan `PATH`
bionic masih ada. **Perilaku proot tidak berubah.**

> Catatan: `termux-distro` bisa menulis ulang `termux-profile.sh` saat
> container di-reinstall. Kalau `PATH` suddenly bocor lagi, cek file itu.

### Urutan yang perlu diingat

Semua option harus **sebelum** path rootfs:

```
chroot-ng [OPSI...] <rootfs> <program> [argumen...]
```

Salah urutan menghasilkan error yang menyesatkan seperti
`cannot load -b (.../rootfs/-b)`.

---

## 6. Kenapa `--noediting` jadi default

Interaktif dengan readline penuh pernah menghasilkan `Segmentation fault`.
Setelah pelacakan panjang, penyebabnya **readline** (bukan bash, bukan job
control, bukan locale, bukan ukuran terminal):

| Kondisi | Hasil |
|---|---|
| `bash -l -i` baca dari TTY | crash (0/5 lolos) |
| `bash -l -i --noediting` | aman (5/5 lolos) |
| `bash -l -i` baca dari pipe/devnull | aman |
| `dash -i` baca dari TTY | aman (selalu) |
| `set +m` (job control mati) | tetap crash |

Hipotesis yang **sudah** tersingkir: ukuran terminal (`TIOCGWINSZ`) — aman di
0x0, 1x1, 24x80, 50x120, 999x999, 38135x10869, 5000x5000. `LANG` — aman di
`C.UTF-8`, `C.utf8`, `en_US.UTF-8`, kosong. `terminfo` — `xterm-256color`
ada di rootfs.

Trade-off `--noediting`: tombol panah dan pencarian riwayat tidak
berfungsi. Perintah, pipe, redirect, dan exit semuanya normal.

Kalau butuh readline penuh dan siap mengambil risiko:

```bash
cng --editing
```

---

## 7. Yang belum selesai

| Item | Status |
|---|---|
| Angka proot vs chroot-ng yang valid | **belum** — yang pernah diukur salah karena di dalam proot |
| Bug `-Werror` di `unistd_check.c` | **belum** dikirim sebagai PR upstream |
| Penyebab segfault readline | **belum** didapat alamat crash (`strace` tidak menangkap `SIGSEGV`) |
| `strace` + core dump | sudah terinstall di rootfs, tapi belum menghasilkan core yang berguna |
| Test GPU (`vulkaninfo`) | **belum** dikerjakan |

Untuk mengaktifkan debug chroot-ng:

```bash
CNG_DEBUG=1 cng 'echo halo'
```

---

## 8. Referensi

- Source: `fake-chroot-ng` v1.1.0, Apache-2.0, 43k LOC, 336 commit
- Arsitektur & threat model: `docs/DESIGN.md` di repo upstream
- Roadmap: `docs/STATUS.md` (244KB, tidak memuat klaim benchmark)
- Opsi lengkap: `chroot-ng --help`
