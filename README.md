# Dokumentasi Pengerjaan POC Validasi Penerimaan Produksi adhira ganteng

**Tanggal pembaruan: 29 Oktober 2026**  
**dari: Odoo Enterprise saas-18.2 lokal, menggunakan Docker**  
**Status: addon inti terpasang dan konfigurasi POC aktif; integrasi addon kantor menyusul.**

## 1. Tujuan pekerjaan

Proyek ini dibuat untuk memisahkan proses pengiriman barang dari gudang dengan proses penerimaan oleh produksi

Dalam alur yang ingin diperbaiki, validasi pengiriman langsung menambah stok produksi. Padahal, barang mungkin masih dalam perjalanan atau belum diperiksa penerima. POC menambahkan lokasi penampungan bernama Virtual WH. Stok baru masuk ke lokasi produksi setelah penerima melakukan konfirmasi.

Rujukan pekerjaan adalah PDF **POC VALIDATE II DARI PRODUKSI** dan [panduan rm_validate di GitHub](https://github.com/rahmansaleh7/training_metindo/blob/0906a5fa2d0ce17e35da8a83c0ca2e2a5170b80a/rm_validate/README.md). Folder GitHub tersebut berisi petunjuk dan potongan kode; implementasi addon dilengkapi berdasarkan panduan itu.

## 2. Persiapan lingkungan lokal

Pekerjaan dimulai dengan memeriksa laptop dan perangkat lunak yang sudah ada. Docker Desktop, Docker Compose, dan WSL 2 sudah terpasang sehingga tidak dipasang ulang.

Docker sempat gagal menyala karena socket sementara yang tertinggal. Folder socket tersebut disimpan sebagai backup, lalu Docker dijalankan kembali. Pemulihan ini tidak menggunakan factory reset dan tidak menghapus volume database.

Selanjutnya, server dibangun dari source Community cabang saas-18.2 agar sesuai dengan folder addon Enterprise saas-18.2 yang tersedia. Image `odoo:18.0` dipakai sebagai dasar runtime, tetapi executable Odoo diarahkan ke source saas-18.2. Versi server telah diperiksa sebagai `saas~18.2`. PostgreSQL menggunakan image versi 16.

Proyek ditempatkan di **Desktop/validate project** agar dapat dibuka langsung melalui PyCharm. Folder `addons` dipasang ke container melalui bind mount, sehingga perubahan file lokal dapat dibaca Odoo. Shortcut **Buka Odoo Lokal** juga dibuat untuk menyalakan Docker, menunggu engine siap, menjalankan Odoo, dan membuka browser.

Alamat Odoo lokal adalah `http://localhost:8071` dan port hanya diikat ke loopback. Perangkat lain tidak otomatis dapat mengaksesnya melalui jaringan.

## 3. Pemisahan database

| Database | Kegunaan |
|---|---|
| `odoo182_enterprise` | Instalasi Enterprise awal |
| `validate_poc` | Konfigurasi dan latihan proses validasi produksi |
| `validate_code_test` | Pengujian otomatis addon inti |

Database dan kode POC dicadangkan sebelum implementasi addon. Cadangan disimpan di folder `backups` pada proyek lokal. File cadangan database, konfigurasi rahasia, dan password tidak menjadi bagian dokumentasi ini.

## 4. Persiapan proses di Odoo

Database `validate_poc` disiapkan dengan Inventory, lokasi penyimpanan, dan operasi internal transfer.

| Komponen | Konfigurasi akhir |
|---|---|
| Gudang pengirim | Warehouse Raw Material, kode `WRM` |
| Lokasi asal | `WRM/Stock` |
| Lokasi penampungan | `WRM/VIR WH` |
| Lokasi produksi | `PDM/Stock` |
| Operasi pengiriman | `POC - Kirim ke VIR WH` |
| Operasi penerimaan | `POC - Terima di Produksi` |

VIR WH menggunakan tipe **Internal** dan berada di bawah view gudang, bukan di bawah `WRM/Stock`. Pemisahan ini membuat barang yang sudah dikirim tidak lagi berada dalam lokasi stok asal untuk reservasi normal.

Produk latihan `POC-VALIDATE-001` dibuat dengan satuan Units dan stok awal 100 unit. Produk awal ini tidak memakai lot/serial agar alur dasar mudah diperiksa.

## 5. Simulasi manual sebelum membuat addon

Sebelum otomasi ditambahkan, perpindahan diuji menggunakan dua transfer standar Odoo.

| Tahap | WRM/Stock | VIR WH | PDM/Stock |
|---|---:|---:|---:|
| Awal | 100 | 0 | 0 |
| Pengiriman 10 unit selesai | 90 | 10 | 0 |
| Produksi menerima 10 unit | 90 | 0 | 10 |

Dokumen pengiriman yang sudah selesai adalah `WRM/INT/00001`. Penerimaan dilanjutkan melalui `WRM/POCIN/00001`. Keduanya berstatus Done.

Simulasi ini membuktikan alur stok dua tahap. Pada saat simulasi manual, tombol khusus penerimaan dan stage Confirmed belum dibuat. Draft awal `WRM/POCOUT/00001` tidak ikut divalidasi karena akan menggandakan pengiriman.

## 6. Addon yang dibuat

Implementasi utama berada di:

```text
validate project/addons/vit_internal_transfer_virtual/
├── __init__.py
├── __manifest__.py
├── models/
│   ├── __init__.py
│   ├── stock_picking_type.py
│   ├── stock_picking.py
│   ├── stock_move.py
│   └── stock_return.py
├── security/
│   └── security.xml
├── view/
│   ├── stock_picking_type.xml
│   └── stock_picking.xml
└── tests/
    ├── __init__.py
    └── test_production_confirm.py
```

Pada awal pemeriksaan, beberapa bagian masih berupa kerangka: import kosong, kode Python berada dalam file XML, dan XML keamanan belum mempunyai tag pembuka. Struktur tersebut diperbaiki, kemudian model, view, keamanan, dan tes dilengkapi.

### Pengaturan pada Operation Type

Ditambahkan tiga field:

- **Gunakan Konfirmasi Produksi** untuk mengaktifkan alur khusus hanya pada operasi yang dipilih.
- **Lokasi Produksi Default** untuk menentukan tujuan akhir.
- **Operation Type Penerimaan** untuk menentukan jenis transfer kedua.

Konfigurasi diperiksa agar lokasi asal, virtual, dan produksi berbeda, bertipe Internal, serta berada dalam company yang sama. Operation penerimaan tidak boleh mengaktifkan konfirmasi produksi lagi, sehingga tidak membentuk rangkaian penerimaan berulang.

### Status dan tombol pada transfer

Tombol **Confirm Penerimaan** ditambahkan pada form transfer. Tombol tersedia setelah pengiriman Done, masih menunggu penerimaan, dan pengguna mempunyai grup penerima.

Status teknis `stock.picking.state` tidak ditambah atau diganti. Program menggunakan:

| Field | Fungsi |
|---|---|
| `receive_state` | Menyimpan apakah barang masih menunggu atau sudah diterima |
| `display_state` | Menampilkan urutan status, termasuk Confirmed |
| `final_location_dest_id` | Menyimpan tujuan akhir produksi |
| `receive_picking_id` | Menghubungkan pengiriman dengan dokumen penerimaan |
| `production_source_picking_id` | Menghubungkan penerimaan kembali ke pengiriman asal |
| `received_by_id` | Mencatat penerima |
| `received_date` | Mencatat waktu penerimaan |

Dengan cara ini, layar mengikuti POC sementara status perpindahan stok bawaan Odoo tetap konsisten.

### Proses saat konfirmasi

Sesudah pengiriman selesai, barang berada di Virtual. Saat pengguna menekan Confirm Penerimaan, sistem membuat dan memvalidasi transfer kedua dari Virtual ke lokasi produksi.

Transfer kedua menggunakan jumlah yang benar-benar dikirim, bukan sekadar jumlah permintaan. Lot/serial, satuan, pemilik barang, dan paket pada tujuan pengiriman pertama dipertahankan. Reservasi otomatis tidak dibiarkan mengganti lot dengan lot lain.

Status Confirmed baru disimpan setelah transfer kedua benar-benar Done. Penerima dan waktunya dicatat, serta catatan audit ditambahkan ke chatter tanpa perlu mengirim email.

## 7. Hak akses dan pengaman

Grup **Internal Transfer / Penerima Produksi** dibuat. Pada konfigurasi POC akhir, akun admin diberi grup ini atas persetujuan pengguna. Tidak dibuat akun gudang/produksi permanen baru dan password admin tidak diubah pada aktivasi tersebut.

Pengaman yang diterapkan meliputi:

- Pemeriksaan hak penerima di server, bukan hanya menyembunyikan tombol.
- Penolakan perubahan langsung terhadap field hasil konfirmasi melalui create/write.
- Pemeriksaan status agar transaksi belum selesai atau sudah diterima tidak dikonfirmasi lagi.
- Penguncian baris database dan batas unik hubungan pengiriman–penerimaan untuk mencegah penerimaan ganda.
- Pemeriksaan ketersediaan stok lot/paket di Virtual sebelum diterima.
- Pembatalan transaksi konfirmasi jika penerimaan kedua tidak selesai.
- Pembatasan perubahan kuantitas dan unlock pada transfer yang sudah diproses oleh alur ini.
- Setelah Confirmed, retur diarahkan melalui dokumen penerimaan; retur langsung dari pengiriman asal ditolak.
- Perubahan konfigurasi rute ditolak ketika masih ada penerimaan tertunda.

## 8. Penyesuaian terhadap panduan GitHub

Panduan menyebut dependensi kantor `vit_validate_int` dan `vit_custom_stock_barcode`. Source keduanya belum tersedia di laptop. Pengguna menyetujui agar dependensi tersebut menyusul.

Karena itu, implementasi dipisah menjadi:

| Addon | Status dan fungsi |
|---|---|
| `vit_internal_transfer_virtual` | Inti alur POC, bergantung pada `stock`, sudah terpasang |
| `vit_internal_transfer_metindo` | Penghubung dependensi kantor, belum diinstal |
| `validate_produksi` | Penghubung kompatibilitas untuk nama modul latihan lama yang sudah tercatat di database |

Tidak dibuat addon palsu untuk menggantikan kode kantor. Context `skip_internal_validation=True` disiapkan pada validasi transfer penerimaan sesuai petunjuk, tetapi perilakunya terhadap addon asli kantor belum dapat dinyatakan teruji.

Nama folder `models/` dipakai konsisten dalam kode lokal. Perbedaan dengan `model/` pada panduan tidak mengubah perilaku selama import benar.

## 9. Hasil pengujian

Sembilan skenario otomatis pada database terpisah lulus, dengan hasil **0 failed, 0 errors**:

1. Alur penuh, hak penerima, dan penolakan konfirmasi ulang secara berurutan.
2. Lot yang diterima sama dengan lot yang dikirim meskipun ada lot lain di Virtual.
3. Pengiriman sebagian: hanya jumlah yang selesai dikirim diterima; sisanya menjadi backorder.
4. Penolakan penulisan status langsung, unlock, perubahan jumlah, dan perubahan rute saat penerimaan tertunda.
5. Internal transfer biasa tetap berjalan tanpa alur konfirmasi produksi.
6. Penolakan konfigurasi lokasi yang tidak valid.
7. Retur melalui dokumen penerimaan, bukan pengiriman yang sudah Confirmed.
8. Stok tidak cukup: konfirmasi gagal tanpa membuat penerimaan sukses atau mengganti status.
9. Serial, paket hasil pengepakan ulang, dan pemilik barang tetap sesuai.

Setelah instalasi pada `validate_poc`, keberadaan field, status modul, konfigurasi operation type, hak admin, dan view gabungan juga diperiksa. Halaman login merespons HTTP 200. Kode lokal dicocokkan dengan versi yang menjalani tes.

Penguncian untuk permintaan bersamaan sudah diterapkan, tetapi belum dilakukan tes dua transaksi paralel. Valuasi dan jurnal akuntansi juga belum diuji pada konfigurasi perusahaan. Hasil tes inti tidak menggantikan uji penerimaan pengguna di lingkungan kantor.

## 10. Cara mencoba fitur

1. Jalankan shortcut Buka Odoo Lokal.
2. Buka `http://localhost:8071/web/login?db=validate_poc` dan masuk dengan akun yang sudah tersedia.
3. Di Inventory, buat transfer dengan operation **POC - Kirim ke VIR WH**.
4. Periksa lokasi Virtual, tujuan akhir produksi, produk, serta kuantitas.
5. Konfirmasi transfer dan Validate pengiriman. Stok masuk Virtual; dokumen menampilkan Done.
6. Dengan akun yang memiliki grup penerima, klik **Confirm Penerimaan**.
7. Periksa stage Confirmed, dokumen penerimaan, nama penerima, waktu, dan stok per lokasi.

Dokumen lama yang selesai sebelum addon dipasang tidak otomatis dijadikan transaksi menunggu penerimaan. Ini menghindari pembuatan penerimaan kedua atas transaksi lama yang sudah diproses manual.

## 11. Lokasi kode yang biasa diedit

| Perubahan | File dalam addon inti |
|---|---|
| Aturan dan proses konfirmasi | `models/stock_picking.py` |
| Tulisan stage Confirmed | Pilihan label pada `display_state` dan `receive_state` di `models/stock_picking.py` |
| Tombol, statusbar, kolom, dan filter | `view/stock_picking.xml` |
| Konfigurasi jenis operasi | `models/stock_picking_type.py` dan `view/stock_picking_type.xml` |
| Grup penerima | `security/security.xml` |
| Pengaman perubahan stok | `models/stock_move.py` |
| Pengaman retur | `models/stock_return.py` |

Untuk mengganti tulisan Confirmed, ubah labelnya, bukan nilai teknis `received`. Setelah perubahan Python, restart service web. Setelah perubahan schema atau XML, upgrade addon pada database yang dituju dan refresh halaman.

## 12. Pekerjaan yang masih menyusul

- Mendapatkan dan meninjau source asli dua addon kantor, kemudian memasang penghubung dan menguji integrasinya.
- Membandingkan versi, view, hak akses, dan konfigurasi dengan `metindo_dev`. Instalasi lokal belum merupakan salinan lengkap sistem kantor.
- Menguji validasi melalui Barcode kantor; tombol baru saat ini berada pada form Inventory backend.
- Menentukan kebutuhan penerimaan sebagian, barang rusak, selisih, pembatasan per plant, dan reminder barang tertahan.
- Menguji jurnal/valuasi serta konfirmasi paralel pada lingkungan integrasi.
- Melakukan UAT bersama gudang dan produksi sebelum dipasang di server kantor.

Penerimaan saat ini adalah penerimaan penuh atas jumlah aktual yang dikirim. Ini berbeda dari pengiriman sebagian dengan backorder, yang sudah diuji. Program belum menyediakan wizard untuk menerima hanya sebagian dari barang yang sudah berada di Virtual.

## Kesimpulan pengerjaan

Lingkungan lokal, konfigurasi POC, addon inti, hak admin untuk latihan, dan pengujian inti sudah disiapkan. Alur dua tahap dapat dijalankan pada `validate_poc`. Tahap berikutnya adalah menghubungkan kode kantor dan melakukan pengujian bersama pengguna agar hasil lokal sesuai kebutuhan operasional sebenarnya.
