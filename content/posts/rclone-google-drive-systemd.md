---
title: "Auto-mount Google Drive dengan rclone + systemd di Fedora"
date: 2026-11-21
tags: ["rclone", "systemd", "fedora", "google-drive"]
summary: "Membuat Google Drive ter-mount otomatis setiap login dengan systemd user service, lalu mengganti shared client rclone dengan Client ID milik sendiri."
ShowToc: true
---

## Latar belakang

Selama ini aku menjalankan perintah ini secara manual di terminal setiap kali menyalakan laptop:

```bash
rclone mount "Google Drive:" ~/Clouds/Drive --vfs-cache-mode full -v
```

Repot, dan begitu terminalnya tertutup, mount-nya ikut hilang. Solusinya: jadikan **systemd user service**. Tidak perlu root, memakai config rclone milik user sendiri, dan otomatis restart kalau gagal.

> Post ini mengasumsikan rclone sudah terpasang dan remote bernama `Google Drive` sudah dibuat lewat `rclone config`.

## Membuat user service

Cek lokasi binary-nya dulu:

```bash
which rclone fusermount3
# /usr/bin/rclone
# /usr/bin/fusermount3
```

Buat folder dan file service:

```bash
mkdir -p ~/.config/systemd/user
nano ~/.config/systemd/user/rclone-gdrive.service
```

Isi file-nya:

```ini
[Unit]
Description=rclone mount Google Drive
Wants=network-online.target
After=network-online.target

[Service]
Type=notify
ExecStartPre=/usr/bin/mkdir -p "%h/Clouds/Drive"
ExecStart=/usr/bin/rclone mount "Google Drive:" "%h/Clouds/Drive" --vfs-cache-mode full -v
ExecStop=/usr/bin/fusermount3 -uz "%h/Clouds/Drive"
Restart=on-failure
RestartSec=10

[Install]
WantedBy=default.target
```

Beberapa catatan:

- `%h` = home folder user, jadi path-nya `~/Clouds/Drive`.
- `Type=notify` membuat systemd menunggu sampai rclone benar-benar selesai mount.
- `"Google Drive:"` adalah **nama remote** di config rclone, bukan nama folder.
- `network-online.target` di user service praktis tidak berpengaruh (target itu milik system manager). Kalau rclone jalan sebelum jaringan siap, yang menyelamatkan adalah `Restart=on-failure` + `RestartSec=10`: service akan mencoba lagi sampai berhasil.

## Enable & cek

Hentikan dulu mount manual yang masih jalan (Ctrl+C), lalu:

```bash
systemctl --user daemon-reload
systemctl --user enable --now rclone-gdrive.service
```

Cek hasilnya:

```bash
systemctl --user status rclone-gdrive
ls ~/Clouds/Drive
```

Kalau status-nya `active (running)` dan isi Drive muncul, selesai. Restart laptop untuk memastikan mount-nya jalan sendiri setelah login.

> User service jalan saat kamu **login**. Kalau ingin jalan sejak boot sebelum login, tambahkan `loginctl enable-linger $USER`.

## Membuat Client ID sendiri

Secara default rclone memakai **shared client ID** yang dipakai bareng semua pengguna rclone. Dukungannya akan berakhir, jadi lebih aman membuat Client ID sendiri (panduan resminya ada di [dokumentasi rclone](https://rclone.org/drive/#making-your-own-client-id)).

Langkah yang sering bikin macet adalah tombol **PUBLISH APP** yang abu-abu. Google sekarang mewajibkan **homepage** dan **privacy policy** sebelum app bisa dipublish, bahkan untuk app pribadi.

### 1. Siapkan homepage & privacy policy

Cara gratis paling cepat: **GitHub Pages**. Buat repo publik bernama `<username>.github.io` berisi `index.html` dan `privacy.html` sederhana:

```bash
mkdir <username>.github.io && cd <username>.github.io
git init -b main
echo '<h1>rclone personal client</h1>' > index.html
echo '<h1>Privacy Policy</h1><p>This app is used only by its owner to access their own Google Drive with rclone. No data is collected or shared.</p>' > privacy.html
git add . && git commit -m "Add homepage and privacy policy"
gh repo create <username>.github.io --public --source=. --push
```

Setelah 1–2 menit, `https://<username>.github.io` dan `https://<username>.github.io/privacy.html` bisa dibuka.

### 2. Isi Branding di Google Cloud Console

Di **Google Auth Platform → Branding**:

| Field | Isi |
|---|---|
| Application home page | `https://<username>.github.io` |
| Application privacy policy link | `https://<username>.github.io/privacy.html` |
| Authorised domains | `<username>.github.io` |

Perhatikan peringatan merah **"Missing domain"**: tombol **Save** tetap abu-abu sampai domain-nya ditambahkan lewat **+ Add domain** (tanpa `https://`).

### 3. Publish

Buka **Audience → PUBLISH APP → Confirm**. Status berubah menjadi **In production**. Tidak perlu submit verifikasi.

Kenapa harus publish? Kalau app tetap di mode **Testing**, token-nya kedaluwarsa setiap **7 hari** dan kamu harus login ulang setiap minggu.

### 4. Buat client

Di **Clients → Create client**, pilih **Desktop app**. Simpan **Client ID** dan **Client secret**, dan jangan bagikan ke siapa pun.

## Pasang di rclone

```bash
systemctl --user stop rclone-gdrive
rclone config
```

Pilih `e` (edit) lalu remote **Google Drive**, isi `client_id` dan `client_secret` yang baru, Enter untuk sisanya, lalu jawab `y` saat ditanya **refresh token**. Di browser akan muncul *"Google hasn't verified this app"*: klik **Advanced → Go to (nama app)**. Aman, karena itu app milikmu sendiri.

Lewat menu interaktif ini, client secret tidak tersimpan di history shell.

Jalankan lagi service-nya:

```bash
systemctl --user start rclone-gdrive
```

## Troubleshooting

Lihat log:

```bash
journalctl --user -u rclone-gdrive -f
```

- **`invalid_grant`**: token tidak valid lagi. Jalankan `rclone config reconnect "Google Drive:"` lalu `systemctl --user restart rclone-gdrive`.
- **Log `vfs cache: removed cache file`**: normal. Yang dihapus hanya salinan cache lokal yang lama, bukan file di Drive.
- **Sebelum shutdown**, pastikan upload selesai. Dengan `--vfs-cache-mode full`, file disimpan dulu di cache lalu di-upload beberapa detik kemudian. Cari baris `upload succeeded` di log.

Perintah lain yang berguna:

```bash
systemctl --user restart rclone-gdrive       # restart
systemctl --user disable --now rclone-gdrive # matikan auto-start
```
