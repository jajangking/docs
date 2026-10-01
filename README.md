# Docs

Dokumentasi environment perangkat ini.

## hermes/

- [hermes/README.md](hermes/README.md) — panduan penggunaan Hermes Agent:
  quick start, konfigurasi API key, perintah sehari-hari, update, uninstall.
- [hermes/troubleshooting.md](hermes/troubleshooting.md) — tabel gejala →
  solusi, plus post-mortem kenapa `install.sh` gagal di Termux/proot dan cara
  memperbaikinya.

## chroot-ng/

- [chroot-ng/README.md](chroot-ng/README.md) — panduan chroot-ng sebagai
  pengganti proot: perbedaan mekanisme, install, pemakaian, batasan, dan
  daftar yang sudah diverifikasi.
- [chroot-ng/troubleshooting.md](chroot-ng/troubleshooting.md) — tabel
  gejala → solusi, termasuk crash readline pada shell interaktif, akses adb
  ke sandbox Termux lewat `run-as`, dan diagnosis manual.

## Catatan environment

Perangkat ini menjalankan **Termux (Android/bionic)** dengan
**Debian 13 arm64 proot-distro (glibc 2.41)** di dalamnya. Karena ada dua
userland, beberapa tool bisa salah mendeteksi yang mana yang dipakai.

Rootfs Debian itu **dipakai bersama**: `proot-distro` dan `chroot-ng`
menjalankan rootfs yang sama persis, jadi `hermes`, `opencode`, dan
`/root/docs` utuh di kedua mode.
