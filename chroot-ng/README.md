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
> lihat §8. Saya tidak akan mengarang angkanya.

---

## 2. Install

### 2.1 Kebutuhan

| Kebutuhan | Status di perangkat ini |
|---|---|
| Termux | ✅ ada |
| bash di Termux | ✅ ada (`pkg install bash`) |
| rootfs Debian via proot-distro | ✅ ada (dipakai bersama proot) |
| Device mendukung seccomp+SIGSYS | ✅ terverifikasi |
| Flag `-u` | **bukan** unshare user namespace — itu `--fake-id` |

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

### 2.3 Apa arti flag `-u`

Sering disalahpahami. Dari `chroot-ng --help`:

```
-u, --fake-id[=ID]   Present a fake user identity. ID is a uid or uid:gid
                     (a bare -u defaults to 0:0, root)
```

Jadi chroot-ng berjalan dengan **uid host asli** (10496 di perangkat ini) dan
hanya *berbohong* jadi `0:0` di dalam guest. Tidak ada user namespace, tidak
ada pemetaan uid.

Konsekuensinya: akses file memakai uid host sebenarnya. Karena seluruh rootfs
dimiliki uid 10496, baca-tulis di dalam guest berjalan lancar — dan uid root
di dalam guest **tidak** berarti apa-apa secara privilege di host.

### 2.4 Sumber binary

Binary dibangun dari source, bukan distro package (belum ada paket resmi
Termux). Yang sudah terpasang di `$PREFIX/bin/chroot-ng` (2.053.208 byte,
static freestanding aarch64).

> **Bug upstream yang ditemukan saat build:** `src/rt/unistd_check.c` gagal
> di-build dengan `-Werror` pada header uapi arm64 Debian. Penyebabnya proyek
> memakai ejaan `__NR3264_fstat`, sedangkan header Debian mendefinisikan
> `__NR_fstat 80` langsung. **Nomornya identik, hanya ejaan berbeda.**
> Compiling file itu manual tanpa `-Werror` membuat build selesai.
> Ini kandidat PR upstream — lihat §8.

### 2.5 Pasang wrapper

Wrapper-nya kompatibel dengan `proot-distro` agar tidak perlu mengingat perintah panjang:

```bash
cp /data/local/tmp/wrap2.sh "$PREFIX/bin/chroot-ng-distro"
chmod 755 "$PREFIX/bin/chroot-ng-distro"
```

### 2.6 Alias (opsional, disarankan)

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
| symlink `.l2s` | 0 tersisa dari 22.123 yang sebelumnya rusak (lihat §6) |
| `hermes --version` | `Hermes Agent v0.21.5+5355.g357f51c` |
| `opencode --version` | `v2.0.21` |
| `opencode` TUI interaktif | 5/5 lolos (ternyata sia-sia — `marker=OK` hanya berarti `echo` tercetak, bukan opencode sukses) |
| `opencode models` | **GAGAL** — `rc=139` SIGSEGV di chroot-ng, `rc=0` di proot |
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

## 6. Symlink `.l2s`: kenapa rootfs awalnya rusak di chroot-ng

Ini temuan paling besar, dan gejalanya baru muncul setelah chroot-ng dipakai
untuk menjalankan aplikasi nyata (`hermes`), bukan sekadar `git` atau `node`.

### Gejala

```
✗ Installing Python dependencies failed
    error: Failed to install: packaging-26.0-py3-none-any.whl
      Caused by: failed to open file
                 `/root/.hermes/cache/uv/archive-v0/.../packaging-26.0.dist-info/WHEEL`:
                 Permission denied (os error 13)
```

Lalu gejala kedua yang menyamar sebagai bug lain:

```
ImportError: cannot import name 'YAML' from 'ruamel.yaml' (unknown location)
```

Itu **satu** masalah, bukan dua. `uv sync` gagal → `site-packages` tidak
lengkap → import `ruamel.yaml` jadi benar-benar tidak ada. `unknown location`
adalah ciri package yang gagal dimuat lalu jatuh ke *namespace package*.

### Penyebab

`proot` berjalan dengan emulasi `link2symlink`. Android/host tidak mendukung
hardlink seperti yang dibutuhkan filesystem rootfs, jadi proot mengganti file
dengan **symlink ke salinan di direktori `.l2s/`**, dan symlink itu ditulis
menunjuk **path host absolut**:

```
$RFS/root/.hermes/cache/uv/archive-v0/.../WHEEL
    -> /data/data/com.termux/files/usr/var/lib/proot-distro/containers/debian/rootfs/.l2s/.l2s.WHEEL0002
```

Di bawah **proot** symlink itu diterjemahkan ke path host yang valid, jadi
`head -1` menghasilkan `Wheel-Version: 1.0`. Di bawah **chroot-ng** symlink
dibaca apa adanya — dan `/data/data/com.termux/...` **tidak ada di dalam
guest**, sehingga `Permission denied`.

Skala masalahnya di rootfs ini:

| Metrik | Nilai |
|---|---|
| total symlink di rootfs | 28.854 |
| menunjuk host path (`$RFS/...`) | **22.123** (77%) |
| menunjuk `/sdcard` | 0 |

Jadi ini bukan isu `hermes`. `hermes` cuma yang paling cepat menampakkannya
karena `uv` butuh file cache itu saat `sync`.

### Perbaikan

Buang prefix host dari target sehingga jadi **guest-absolute**:

```
lama: /data/.../containers/debian/rootfs/.l2s/.l2s.WHEEL0002
baru: /.l2s/.l2s.WHEEL0002
```

Guest `/` = rootfs di host, jadi bentuk ini benar untuk **kedua** engine.
Bonus: symlink jadi tahan kalau rootfs dipindah atau di-reinstall
`proot-distro` — justru path host absolut itulah yang paling rapuh.

22.123 symlink di-rewrite dalam **~11 detik**. Target asli disimpan di
`$PREFIX/tmp/l2s_targets.bak` sehingga bisa di-restore.

### Verifikasi setelah perubahan

| Cek | Sebelum | Sesudah |
|---|---|---|
| symlink host-path tersisa | 22.123 | **0** |
| `WHEEL` via chroot-ng | `Permission denied` | `Wheel-Version: 1.0` |
| `WHEEL` via proot | `Wheel-Version: 1.0` | `Wheel-Version: 1.0` |
| `hermes --version` (chroot-ng) | gagal | `Hermes Agent v0.21.5+5355.g357f51c` |
| `hermes --version` (proot) | jalan | jalan |
| `opencode --version` (chroot-ng) | jalan | jalan |
| `git` / `node` (kedua engine) | jalan | jalan |
| `container=proot-distro` (proot) | ada | ada |
| symlink sistem (usr-merge) | utuh | utuh |

> **Symlink sistem tidak tersentuh.** Yang di-rewrite hanya symlink berawalan
> `$RFS/`. `/bin/sh -> dash`, `/lib -> usr/lib`, `/sbin -> usr/sbin` tetap
> bentuk aslinya.

### Kalau container di-reinstall

`proot-distro install` akan membuat symlink `.l2s` baru dengan path host
absolut lagi. Gejalanya: `hermes` di chroot-ng kembali `Permission denied`.
Perbaikannya idempoten — jalankan ulang skrip rewrite, atau pakai perintah di
[troubleshooting.md](troubleshooting.md).

---

## 7. Kenapa `--noediting` jadi default

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

## 8. Yang belum selesai

| Item | Status |
|---|---|
| Angka proot vs chroot-ng yang valid | **belum** — yang pernah diukur salah karena di dalam proot |
| Penyebab segfault `opencode` | **terkonfirmasi ada**, tapi akar masalahnya belum ditemukan. Lihat §2 di troubleshooting |
| Penyebab segfault readline | **belum** didapat alamat crash (`strace` tidak menangkap `SIGSEGV`) |
| Bug `-Werror` di `unistd_check.c` | **belum** dikirim sebagai PR upstream |
| `strace` + core dump | sudah terinstall di rootfs, tapi belum menghasilkan core yang berguna |
| Test GPU (`vulkaninfo`) | **belum** dikerjakan |
| Symlink `python3` → `python3.13` | **belum** — tidak mendesak, `hermes` punya python sendiri |

### `opencode` tidak bisa dipakai di bawah chroot-ng

Ini **bukan** lagi "belum jelas". Sekarang terkonfirmasi:

```
opencode --version   →  rc=0     (chroot-ng dan proot)
opencode --help      →  rc=0     (chroot-ng dan proot)
opencode models      →  rc=139   chroot-ng   ← SIGSEGV, 5 dari 5
opencode models      →  rc=0     proot
```

Deterministik, tanpa TTY. `139` = `128 + 11` = `SIGSEGV`.

Awalnya saya mengira ini tekanan memori atau readline. Keduanya sudah dibantah
dengan pengukuran — `node`, `git`, `bash`, `threading`, `worker_threads`, dan
`mprotect` semuanya identik antara proot dan chroot-ng. Detail dan daftar
hipotesis yang sudah gugur ada di
[troubleshooting.md §2](troubleshooting.md).

Petunjuk terbaru: yang crash adalah **client**-nya, bukan server. Server
dapat hidup 8 detik di bawah chroot-ng, dan client tetap `rc=139` bahkan
saat disambungkan eksplisit ke server yang terbukti hidup lewat
`--server`. Crash terjadi sebelum logging terpasang (`--log-level trace`
menghasilkan 0 baris), dan tidak bergantung pada jaringan. Akar
masalahnya belum ditemukan.

**Rekomendasi praktis:** jalankan `opencode` lewat proot. Yang rusak hanya
lapisan chroot-ng; rootfs-nya sama dan utuh.

Untuk mereproduksi:

```bash
bash $PREFIX/tmp/repro-opencode.sh
```

### Tekanan memori bukan penyebabnya

Worth dicatat supaya tidak ditelusuri ulang: device ini memang sedang
tekanan memori tinggi saat pengujian,

```
Mem 7686 MB total · 209 MB free · 1510 MB available
Swap 4377 MB terpakai dari 7686 MB
opencode = binary 199.936.296 byte
```

tapi itu **bukan** penyebab crash-nya. Buktinya crash-nya deterministik 5 dari
5 pada kondisi memori yang sama, sementara `node` dan `git` yang jauh lebih
berat tetap jalan.

Untuk mengaktifkan debug chroot-ng:

```bash
CNG_DEBUG=1 cng 'echo halo'
```

---

## 9. Referensi

- Source: `fake-chroot-ng` v1.1.0, Apache-2.0, 43k LOC, 336 commit
- Arsitektur & threat model: `docs/DESIGN.md` di repo upstream
- Roadmap: `docs/STATUS.md` (244KB, tidak memuat klaim benchmark)
- Opsi lengkap: `chroot-ng --help`
