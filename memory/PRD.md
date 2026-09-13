# PRD — SIMBENG (Sistem Bengkel Motor) v1.4 fix + Pengembangan Lanjutan

## Problem Statement Asli
Melanjutkan project repo public https://github.com/khaulahcomp/simbeng_v1_4_fix (PHP native + MySQL/MariaDB, target cPanel shared hosting dengan database existing):
1. Kasir/servis: default diskon persen (%), tercetak persen di faktur; input kode item ganda digabung (qty terakumulasi); qty melebihi stok diberi tanda merah/notifikasi dan tidak bisa tersimpan.
2. Cetakan faktur: urutan item berdasarkan kode sparepart, bukan urutan input.
3. Draf otomatis inputan kasir (anti hilang karena kendala perangkat/jaringan) + opsi simpan sementara.
4. Kirim faktur ke WhatsApp: gambar faktur harus otomatis terkirim bersama teks (tanpa link).
5. Jangan merubah fungsi lain yang sudah berjalan.

## Arsitektur
- PHP native (front controller index.php), MySQL/MariaDB, Bootstrap 5, html2canvas, Web Share API.
- Migrasi skema idempoten via includes/schema.php digerbang APP_SCHEMA_VERSION (includes/db.php).
- Hasil kerja ada di /app/simbeng (clone repo) — folder bengkel/ di-upload ke cPanel.

## User Persona
- Kasir bengkel (input transaksi servis/penjualan cepat, scan barcode).
- Owner/admin bengkel (pengaturan, laporan).

## Yang Telah Diimplementasikan (2026-09-13)
1. **Diskon default persen** (`pages/pos.php`): select diskon per item (jasa & part) dan diskon nota default `persen`; kolom `transactions.diskon_jenis` + `diskon_nilai` baru (migrasi otomatis, APP_SCHEMA_VERSION 2026.06.20.2); faktur menampilkan "10%" per item dan "Diskon Nota (10%)"; teks WA menampilkan "disc 10% (Rp 10.000)".
2. **Penggabungan item ganda**: client-side (`addPartRow` merge qty) dan server-side (POST handler menggabungkan part_id sama sebelum validasi stok — atribut baris pertama dipakai).
3. **Validasi stok merah**: `validateStok()` — baris merah (`overstock`), input `is-invalid`, label "Melebihi stok!", alert `#stokWarning`, tombol simpan terkunci; mode edit memperhitungkan qty lama (`POS_OLD_QTY`). Server-side tetap menolak qty > stok.
4. **Urutan faktur per kode sparepart** (`pages/receipt.php`): ORDER BY jasa-dulu, lalu `p.kode`, lalu `ti.id`.
5. **Draf otomatis kasir**: localStorage `pos_draft_v1` tersimpan tiap perubahan (debounce), indikator "Draf tersimpan otomatis", banner Pulihkan/Hapus Draf saat halaman dibuka, auto-restore bila submit gagal (flash danger), auto-hapus setelah submit sukses.
6. **WhatsApp + gambar faktur** (`pages/receipt.php`): tombol "Kirim WhatsApp (rincian + gambar faktur)" memakai Web Share API `navigator.share({files})` — di HP gambar JPG faktur otomatis terlampir bersama teks; fallback desktop: JPG otomatis diunduh + chat wa.me berisi rincian dibuka.

## Verifikasi (lokal: PHP 8 + MariaDB, DB dari bengkel_dump.sql)
- Merge item ganda: part_id=1 qty 2+3 → 1 baris qty 5, stok 10→5. ✓
- Diskon persen nota 10% → diskon 31.500 tersimpan jenis/nilai; faktur menampilkan %. ✓
- Overstock qty 99 stok 3 → ditolak server (302 + flash), UI merah + tombol simpan disabled. ✓
- Urutan faktur: input BMT-010 dulu tetap tercetak AHM-002 lebih dulu. ✓
- Draf: tersimpan otomatis, banner muncul setelah reload, pulihkan berfungsi. ✓
- Tombol WA: render + Web Share (canShare tersedia di browser HTTPS/mobile). ✓

## Catatan Deployment cPanel
- Upload folder `bengkel/` (overwrite). Migrasi kolom baru berjalan otomatis SEKALI pada request pertama (idempoten, aman untuk database existing).
- Fitur lampir gambar WA otomatis butuh HTTPS (cPanel SSL) dan browser HP (Chrome/Safari) — di desktop otomatis fallback unduh+chat.
- Kredensial uji lokal: admin / admin123 (default aplikasi).

## Update 2026-09-13 (sesi 2) — Perbaikan Preview Emergent
- Masalah user: preview "open new tab" menampilkan template React bawaan (port 3000 = node), sehingga aplikasi PHP tidak terlihat dan dikira struktur datanya salah. Remote repo dicek IDENTIK dengan dasar clone; perubahan hanya di 4 file (pos.php, receipt.php, schema.php, db.php).
- Solusi: `sudo supervisorctl stop frontend`, PHP server dipindah ke `0.0.0.0:3000` (docroot `/app/simbeng/bengkel`); MariaDB test di `/app/simbeng/.mariadb_test` (dump + seed uji). Path diperbarui di `bengkel/start_preview.sh` dan `start_mariadb.sh`.
- Preview publik https://simbeng-kasir-fix.preview.emergentagent.com kini menyajikan aplikasi PHP bengkel; seluruh fitur (merge item, tanda merah stok + tombol simpan terkunci, default diskon persen, draf otomatis, urutan faktur per kode, diskon % di faktur, tombol WA+gambar) diverifikasi ulang lewat URL publik — SEMUA LULUS.

## Backlog
- P1: Uji kirim WA di perangkat HP produksi (share sheet dengan lampiran).
- P2: Draf tersimpan di server (per-user) agar lintas perangkat — saat ini localStorage per perangkat/browser.
