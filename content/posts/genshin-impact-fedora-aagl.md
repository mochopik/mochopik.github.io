---
title: "Main Genshin Impact di Fedora 45 dengan An Anime Game Launcher (AAGL)"
date: 2026-11-21
tags: ["fedora", "gaming", "wine", "aagl", "genshin-impact"]
summary: "Install AAGL lewat Flatpak, setting Wine/DXVK yang stabil, memilih GPU di laptop hybrid, dan memperbaiki crash saat teleport/masuk domain dengan NTSync."
ShowToc: true
---

## Sebelum mulai

**An Anime Game Launcher (AAGL)** adalah launcher **tidak resmi** buatan komunitas untuk menjalankan Genshin Impact di Linux lewat Wine. Karena tidak resmi, pemakaiannya tetap jadi tanggung jawab sendiri: dukungan resmi dari developer game tidak ada.

Setup yang dipakai di tutorial ini (dan terbukti jalan):

| Komponen | Versi / pilihan |
|---|---|
| OS | Fedora 45, kernel 7.3 |
| AAGL | 3.19.8 (Flatpak dari Flathub) |
| Wine | `wine-dwproton-wow64-11.0-11` |
| DXVK | 3.1.1 |
| Sync | NTSync |
| GPU | AMD Radeon Vega (iGPU) |

## 1. Install AAGL dari Flathub

Menurut [README resmi AAGL](https://github.com/an-anime-team/an-anime-game-launcher), cara yang direkomendasikan adalah **Flatpak dari Flathub**. Paket RPM disebut *"often really outdated"*, jadi lebih baik dihindari.

Fedora biasanya sudah punya remote Flathub, tapi untuk memastikan:

```bash
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
flatpak install flathub moe.launcher.an-anime-game-launcher
```

Lalu buka **An Anime Game Launcher** dari menu aplikasi.

## 2. Setup awal

Saat pertama kali dibuka, AAGL menjalankan wizard setup. Ikuti langkahnya; pilihan default umumnya sudah aman. Hal yang perlu diperhatikan:

- **Folder game**: default-nya di dalam folder Flatpak (`~/.var/app/moe.launcher.an-anime-game-launcher/data/anime-game-launcher/`). Kalau mau di drive lain, beri izin aksesnya dulu dengan **Flatseal** (lihat Troubleshooting).
- **Voice pack**: pilih satu saja (misalnya English) untuk menghemat ruang disk.
- **Wine & DXVK**: biarkan launcher mengunduh komponennya.

Setelah itu klik **Install** dan tunggu download game selesai. Ukurannya puluhan GB, jadi siapkan ruang disk yang cukup.

> Sudah punya file game? Setelah setup awal, ada tombol di sebelah **Install** untuk memilih folder game yang sudah ada.

## 3. Setting yang aku pakai

Buka **Settings** di launcher:

- **Wine version**: `wine-dwproton-wow64-11.0-11`
- **Synchronization**: `NTSync`. **Penting**: baca bagian 5 supaya NTSync benar-benar aktif.
- **DXVK**: `3.1.1`
- **Borderless window**: aktif
- **Disable telemetry**: aktif
- **GameMode**: aktif. Install dulu di host: `sudo dnf install gamemode`.

Kalau mau overlay FPS/MangoHud atau Gamescope versi Flatpak, sesuai [wiki AAGL](https://github.com/an-anime-team/an-anime-game-launcher/wiki/Additional-configuration):

```bash
flatpak install flathub org.freedesktop.Platform.VulkanLayer.MangoHud
flatpak install flathub org.freedesktop.Platform.VulkanLayer.gamescope
```

## 4. Laptop dengan dua GPU: memilih GPU yang dipakai

Laptopku punya iGPU **AMD Vega** dan GPU **NVIDIA GTX 1050 Mobile**, tapi yang NVIDIA bermasalah. Supaya game pasti jalan di Vega, tambahkan dua environment variable di **Settings → Environment** (atau bagian sejenis di launcher):

| Variable | Nilai | Fungsi |
|---|---|---|
| `DXVK_FILTER_DEVICE_NAME` | `Vega` | DXVK hanya memakai GPU yang namanya mengandung "Vega" |
| `MESA_VK_DEVICE_SELECT` | `1002:15dd!` | Mesa memilih GPU dengan vendor:device ID ini (tanda `!` = paksa) |

Vendor:device ID untuk GPU kamu bisa dilihat dengan:

```bash
lspci -nn | grep -iE "vga|3d|display"
# ... VGA compatible controller [0300]: AMD ... Raven Ridge ... [1002:15dd]
```

Ganti `Vega` dan `1002:15dd` sesuai GPU di laptop kamu. Bonus: tambahkan `DXVK_HUD=fps` untuk menampilkan FPS di pojok layar.

## 5. Crash saat teleport atau masuk domain (NTSync)

### Gejalanya

Game tiba-tiba **crash saat teleport ke region lain atau masuk/keluar domain**, kadang hanya beberapa menit setelah main. Masalah ini dilaporkan banyak orang sejak versi 7.1 di [issue #695](https://github.com/an-anime-team/an-anime-game-launcher/issues/695) (*"Genshin randomly crashes during teleport with FileReadCache SetFilePointerEx failure and huge invalid allocation"*). Di log game biasanya muncul:

```text
[FileReadCache] SetFilePointerEx failed with error: 6
Could not allocate memory: System out of memory!
Trying to allocate: 18446744073709289537B ...
```

Angka alokasinya mustahil (puluhan GB sampai 2^64), padahal RAM sebenarnya masih banyak. Jadi ini bukan file game rusak atau RAM kurang.

### Penyebabnya: NTSync "aktif" tapi sebenarnya tidak

Di AAGL, Synchronization sudah diset ke **NTSync**, tapi kalau `/dev/ntsync` tidak ada, launcher **diam-diam fallback ke FSync**. Ini dijelaskan di [komentar issue #695](https://github.com/an-anime-team/an-anime-game-launcher/issues/695#issuecomment-6655018352): setelah NTSync *benar-benar* aktif, pelapornya main lebih dari 3 jam tanpa satu pun crash.

Cek dulu apa yang sebenarnya dipakai:

```bash
grep -E "ntsync: up and running|fsync: up and running" \
  ~/.var/app/moe.launcher.an-anime-game-launcher/data/anime-game-launcher/game.log
```

Kalau yang muncul `fsync: up and running.`, NTSync **belum** aktif.

### Solusinya

Load module kernel `ntsync` sekarang, dan buat supaya otomatis di-load setiap boot:

```bash
sudo modprobe ntsync
echo ntsync | sudo tee /etc/modules-load.d/ntsync.conf
```

Lalu **tutup AAGL sepenuhnya dan buka lagi**. Flatpak menentukan isi `/dev` saat sandbox dibuat, jadi launcher yang sudah terbuka tidak akan melihat `/dev/ntsync` yang baru muncul.

Verifikasi:

```bash
ls -l /dev/ntsync        # harus ada
lsmod | grep ntsync      # module ter-load
```

Main sebentar, lalu cek `game.log` lagi. Hasil yang benar:

```text
[!] [stderr] ntsync: up and running.
```

Di laptopku, setelah ini crash saat teleport dan masuk domain hilang.

### Alternatif kalau masih crash

Dari komentar lain di [issue #695](https://github.com/an-anime-team/an-anime-game-launcher/issues/695), beberapa orang berhasil dengan:

- Ganti Wine ke `spritz-wine-cachyos-wow64-10.0-11` + DXVK `3.0`.
- Menunggu sekitar 1 menit di gerbang login sebelum masuk ke dunia game.

## Troubleshooting lain

Dirangkum dari [wiki Troubleshoot AAGL](https://github.com/an-anime-team/an-anime-game-launcher/wiki/Troubleshoot-Game):

- **Kotak putih saat login**: tunggu sekitar 20 detik sampai captcha muncul.
- **Error 4206 / connection timed out**: biasanya DNS bermasalah. Coba set DNS manual atau pakai VPN.
- **Layar hitam**: DXVK gagal terpasang. Update launcher atau ganti versi DXVK.
- **Game tidak ditemukan di folder/drive lain**: beri izin akses folder itu ke AAGL dengan **Flatseal** (`flatpak install flathub com.github.tchx84.Flatseal`).
- **Error -900x**: ruang disk penuh, atau izin baca/tulis folder game (termasuk subfolder `Persistent`) bermasalah.
- **Game jalan pakai software rendering** (`llvmpipe`): pastikan driver Vulkan terpasang: `sudo dnf install mesa-vulkan-drivers`.

Log lengkap ada di:

```bash
~/.var/app/moe.launcher.an-anime-game-launcher/data/anime-game-launcher/game.log
```
