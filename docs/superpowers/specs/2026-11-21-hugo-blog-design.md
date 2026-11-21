# Blog teknis mochopik.github.io (Hugo + PaperMod)

## Tujuan & alasan
Mengubah `mochopik.github.io` dari 2 halaman HTML statis menjadi blog teknis berisi catatan setup (contoh: rclone + systemd). Pembacanya diri sendiri dan orang lain yang mengalami masalah serupa.

- **Goals:** menulis post baru dengan Markdown, `git push` langsung tayang; blok kode rapi + tombol copy; tag & pencarian; preview lokal dengan `hugo server`.
- **Non-goals:** komentar, analytics, domain kustom, multi-bahasa.
- **Anti-goals:** `https://mochopik.github.io/privacy.html` dan halaman utama TIDAK boleh mati/404, karena dipakai Google OAuth app rclone.

## Pendekatan
- **Dipilih: Hugo + tema PaperMod, build via GitHub Actions.** Markdown, preview lokal mudah (Hugo sudah terpasang, v0.169 extended), fitur blog teknis sudah ada di tema.
- Jekyll: preview lokal butuh Ruby/gems, merepotkan di Fedora.
- HTML manual: tiap post & daftar post diurus tangan, tidak skalabel.

## Keputusan desain (usulan saya, boleh dikoreksi)
- **Struktur:** proyek Hugo di root repo `~/Projects/playground/mochopik.github.io`. Tema PaperMod dipasang sebagai git submodule di `themes/PaperMod`.
- **Privacy policy:** pindah ke `static/privacy.html` (isi sama), sehingga URL tetap `/privacy.html`.
- **Deploy:** workflow `.github/workflows/hugo.yml` (Hugo extended), sumber Pages diubah dari "branch main" ke "GitHub Actions". `public/` tidak di-commit.
- **Konfigurasi `hugo.toml`:** `baseURL = "https://mochopik.github.io/"`, bahasa `id`, judul situs "mochopik", mode tema auto (ikut sistem), aktifkan copy-code, reading time, ToC, tag, halaman pencarian (JSON index).
- **Halaman:** Beranda (daftar post), Tags, Search, About singkat, link Privacy di footer.
- **Post pertama:** "Auto-mount Google Drive dengan rclone + systemd di Fedora" (+ membuat Client ID sendiri), disusun dari langkah yang sudah dikerjakan; tanpa data pribadi (email, client ID/secret).

## Diserahkan ke builder
Detail menu, teks About, urutan langkah migrasi, isi workflow persisnya.

## Verifikasi
`hugo server` lokal tampil benar; setelah push, Actions sukses; `/`, `/privacy.html`, post pertama, `/tags/`, `/search/` bisa dibuka.
