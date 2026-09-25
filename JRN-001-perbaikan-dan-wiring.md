# JRN-001 — Perbaikan pembatasan akun kontrol jurnal manual

Tanggal: 24 September 2026
Status: kode diperbaiki; 7 tes regresi terarah lulus. Uji integrasi HTTP/PostgreSQL masih diperlukan.

## Changelog

- app/api/v1/endpoints/jurnal.py:69 — _update_manual kini meneruskan is_manual=True kepada validate_entries. Penolakan terjadi sebelum detail jurnal diganti.
- app/services/workflow_service.py:200 — cabang jurnal_umum pada post_existing kini meneruskan is_manual=True. Aturan akun diperiksa kembali sebelum status menjadi POSTED.
- Workflow get_document membatasi jurnal yang diproses ke ref_module MANUAL dan bukan reversal. Posting transaksi sistem tetap menggunakan cabang service masing-masing.
- Aturan lama tetap berlaku: akun kontrol boleh dipakai manual jika allow_manual_posting secara eksplisit True. Akun kontrol dengan flag False ditolak.
- Tidak ada migrasi database, perubahan schema, atau perubahan pada source Java. Pembulatan JRN-002 belum diperbaiki.

## Catatan wiring frontend

Method, URL, field request/response sukses tidak berubah.

Endpoint yang perilaku validasinya diperbaiki:
- PUT /api/v1/jurnal/manual/{jurnal_id}
- POST /api/v1/workflow/jurnal_umum/{document_id}/post

Sebelum: akun kontrol aktif/DETAIL dengan allow_manual_posting=False bisa lolos validasi edit dan posting.
Sesudah: request tersebut ditolak HTTP 400 melalui penanganan ValueError yang sudah ada. Bentuk error tetap {"detail": "..."}. Tidak ada kode error baru.

Contoh request edit, sebelum dan sesudah memakai bentuk yang sama (ganti placeholder dengan UUID nyata):

```json
{
  "tanggal": "2026-09-24T10:00:00+07:00",
  "keterangan": "Jurnal manual",
  "details": [
    {"akunPerkiraanId": "<uuid-akun-kontrol>", "debit": "100.00", "kredit": "0.00"},
    {"akunPerkiraanId": "<uuid-akun-biasa>", "debit": "0.00", "kredit": "100.00"}
  ]
}
```

Contoh request posting tetap:

```json
{"expectedVersion": 2}
```

Angka expectedVersion harus berasal dari workflow terbaru, bukan nilai tetap 2. Posting tetap memerlukan approval dan otorisasi yang berlaku.

Contoh respons baru untuk akun kontrol yang dilarang (kode/nama/jenis mengikuti data akun):

```json
{
  "detail": "Akun 114001 (Persediaan) adalah CONTROL ACCOUNT (INVENTORY). Tidak bisa dipakai jurnal manual — hanya sistem yang boleh posting ke akun ini. Gunakan modul transaksi yang sesuai (Penjualan/Pembelian/Kas-Bank/Persediaan)."
}
```

Sebelumnya request dapat menghasilkan respons sukses: detail jurnal pada edit atau WorkflowResponse dengan state POSTED saat post. Sesudah perbaikan, respons sukses untuk akun yang diizinkan tetap sama; hanya jalur terlarang di atas yang ditolak. Contoh error ini berasal dari format validator, bukan rekaman HTTP integration test.

Frontend perlu menampilkan detail error dan mempertahankan input agar pengguna dapat memilih akun yang diizinkan. Bila posting ditolak setelah approval, gunakan alur koreksi draft yang sudah tersedia sesuai status workflow; jangan menganggap request gagal berarti dokumen sudah terposting. Pembatasan server tetap berlaku walaupun opsi akun disaring di UI.

## Tes dan batas verifikasi

Jalankan dari root backend:

```text
python -m unittest discover -s tests -p test_manual_control_accounts.py -v
```

Tes menggunakan standard library, mengeksekusi fungsi asli melalui AST dan mensimulasikan lookup akun serta dependency lain. Decorator transaksi, HTTP, SQL, dan workflow approval end-to-end tidak dieksekusi.

Tujuh skenario:
1. Edit ke akun kontrol terlarang ditolak sebelum detail diganti.
2. Posting akun kontrol terlarang ditolak; status tetap DRAFT.
3. Kebijakan akun yang berubah setelah pembuatan draft diperiksa kembali saat post.
4. Akun biasa tetap bisa diedit dan diposting.
5. Akun kontrol dengan izin manual eksplisit tetap diterima.
6. Validator jurnal sistem tetap menerima akun kontrol yang sesuai.
7. Validator reversal tetap menerima akun kontrol nonaktif dengan pengecualian histori.

Hasil: kode ZIP asli gagal pada 3 tes penolakan yang relevan; kode perbaikan lulus 7/7. Perbandingan byte memastikan hanya dua file source asli yang berubah. File tests dan dokumen ini ditambahkan.

Dependency aplikasi (FastAPI/SQLAlchemy/driver PostgreSQL) belum tersedia pada runtime pengujian. Sebelum penerapan produksi, jalankan uji integrasi di lingkungan aplikasi dengan database uji: create draft, edit ke akun terlarang, approve/post, perubahan flag akun setelah draft, rollback kegagalan, serta transaksi sistem/reversal.

Tidak melakukan merge, deployment, atau perubahan data produksi.
