# Docs

Dokumentasi environment perangkat ini.

## hermes/

- [hermes/README.md](hermes/README.md) — panduan penggunaan Hermes Agent:
  quick start, konfigurasi API key, perintah sehari-hari, update, uninstall.
- [hermes/troubleshooting.md](hermes/troubleshooting.md) — tabel gejala →
  solusi, plus post-mortem kenapa `install.sh` gagal di Termux/proot dan cara
  memperbaikinya.

## Catatan environment

Perangkat ini menjalankan **Termux (Android/bionic)** dengan
**Debian 13 arm64 proot-distro (glibc 2.41)** di dalamnya. Karena ada dua
userland, beberapa tool bisa salah mendeteksi yang mana yang dipakai.
