# JRN-002 — Konsistensi pembulatan jurnal manual

Tanggal: 25 September 2026. Status: diperbaiki dan lolos tes terarah; belum uji integrasi HTTP/PostgreSQL.

## Perubahan

File yang diperbarui: app/api/v1/endpoints/jurnal.py.
Create jurnal manual tidak lagi menjalankan pemeriksaan total mentah sebelum normalisasi. Validasi baris, minimal dua baris, serta keseimbangan kini diserahkan kepada auto_posting_jurnal → validate_entries, fungsi yang juga digunakan oleh edit dan posting workflow. Setiap nominal dibulatkan dua desimal dengan ROUND_HALF_UP sebelum penjumlahan. Edit/post dan validator bersama tidak perlu diubah.

Perbaikan is_manual=True dari JRN-001 tetap ada pada file ini. Patch ini diterapkan setelah patch JRN-001; file workflow_service.py dari patch sebelumnya tetap diperlukan. Tidak ada migrasi database atau perubahan data historis.

## Catatan wiring frontend

Endpoint: POST /api/v1/jurnal/manual. Method, URL, field request, status sukses 201, dan struktur respons tidak berubah. Header Idempotency-Key tetap wajib sesuai aturan lama.

Contoh nominal request (dalam details, dengan akunPerkiraanId UUID valid):
- Baris pertama: debit "100.004", kredit "0".
- Baris kedua: debit "0", kredit "100.00".

Sebelum: HTTP 400 karena total mentah 100.004 tidak sama dengan 100.00.
Sesudah: bila akun, periode, otorisasi, dan validasi lain memenuhi syarat, jurnal DRAFT dapat dibuat dengan debit/kredit masing-masing 100.00. Respons detail/header memakai nilai hasil normalisasi.

Contoh lain:
- Debit 100.005 versus kredit 100.01 → diterima sebagai 100.01.
- Debit 0.005 + 0.005 versus kredit 0.02 → diterima; pembulatan dilakukan per baris.
- Debit 0.005 + 0.005 versus kredit 0.010 → ditolak: hasil pembulatan 0.02 versus 0.01.
- Baris bernilai 0.004 yang menjadi nol pada kedua sisi → ditolak.

Error tetap HTTP 400 dengan bentuk {"detail": "pesan"}. Karena memakai validator bersama, teks untuk beberapa kasus berubah:
- Kurang dari dua baris: "entries tidak boleh kosong, minimal 2 baris (debit & kredit)".
- Baris nol atau dua sisi positif: "Setiap baris jurnal harus berisi debit atau kredit, bukan keduanya/nol".
- Tidak seimbang: pesan dari validator berisi total setelah pembulatan, dengan format "Jurnal tidak balance: total debit=..., total_kredit=...".

Frontend sebaiknya menampilkan detail error dari server dan nilai respons yang telah dinormalisasi. Bila menghitung pratinjau, gunakan pembulatan desimal HALF_UP per baris, bukan membulatkan total akhir. Jangan mengandalkan pencocokan teks pesan error persis.

Contoh perilaku di atas berasal dari tes fungsi, bukan rekaman HTTP.

## Verifikasi

13 tes lulus: 7 regresi JRN-001 dan 6 tes JRN-002 (termasuk subkasus invalid). Tiga tes pembulatan gagal pada endpoint sebelum perubahan dan berhasil sesudah perubahan.

Tes mengeksekusi body fungsi create/edit/post serta validate_entries asli via AST. Persistence dan dependency disimulasikan; jalur create memakai pengganti auto_posting_jurnal yang menjalankan validator asli. Karena itu hasil ini bukan pengujian decorator transaksi, auto_posting_jurnal secara utuh, HTTP, PostgreSQL, atau approval end-to-end.

Tes berada di tests/test_manual_rounding.py dan menggunakan helper tests/test_manual_control_accounts.py. Dari root backend:

    python -m unittest discover -s tests -p "test_manual_*.py" -v

Sebelum penerapan produksi masih perlu uji integrasi dengan database uji. Belum dilakukan merge/deploy. Patch ZIP hanya memuat file kode yang diperbarui; dokumen dan tes diberikan terpisah.
