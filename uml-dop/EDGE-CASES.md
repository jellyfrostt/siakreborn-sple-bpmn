# Edge case dan elemen UML-DOP

Pemetaan kasus khusus registrasi (pembayaran UKT dan pengisian IRS) ke elemen model UML-DOP sejak v10. Setiap baris merujuk sumber regulasi atau praktik yang dapat diakses publik.

| Kode | Kasus | Sumber | Elemen UML-DOP | Varian terkait |
|---|---|---|---|---|
| EC-01 | Mahasiswa dengan persetujuan cicilan boleh mengisi IRS setelah cicilan pertama dibayar | Pengumuman registrasi Fakultas Farmasi UI, Gasal 2026/2027 | MCicilan (`CicilanImpl.cicilanKe`, `status`, `cekCicilanPertama`), `TagihanImpl.skemaPembayaran`, `PembayaranImpl.cicilanimpl` | BayarDuluIRS |
| EC-02 | Cicilan yang belum lunas sampai akhir semester menghalangi Registrasi Akademik semester berikutnya | PR UI 24/2022 Pasal 11 ayat (6)–(7) | `CicilanImpl.status`, `CicilanImpl.jatuhTempo`, MRegistrasi (`statusAkademik`) | semua |
| EC-03 | Penerima beasiswa dan KIP-K diproses dengan penundaan pembayaran sehingga dapat mengisi IRS sebelum melunasi UKT | Pengumuman registrasi Fakultas Farmasi UI, Gasal 2026/2027 | `BeasiswaImpl.penundaanPembayaran` | Beasiswa |
| EC-04 | IRS dibatalkan apabila biaya pendidikan belum dibayar sampai akhir periode pembayaran | PR UI 24/2022 Pasal 11 ayat (5) | MPeriodeRegistrasi (`akhir`), `PengisianRencanaStudiImpl.batalkanIRS`, `alasanPembatalan` | IRSDuluBayar |
| EC-05 | Pembayaran dan pengisian IRS hanya dapat dilakukan pada periode yang dijadwalkan kalender akademik | PR UI 24/2022 Pasal 11 ayat (4); pengumuman Farmasi UI 2026 (bayar 13 Juli–6 Agustus, IRS 7–16 Agustus) | MPeriodeRegistrasi (`jenis`, `mulai`, `akhir`, `cekPeriodeAktif`) | semua |
| EC-06 | Mahasiswa yang tidak melakukan salah satu registrasi berstatus tidak aktif (kosong); dua semester kosong berturut-turut berujung pengunduran diri | PR UI 24/2022 Pasal 14 dan Pasal 19 | MRegistrasi (`statusAdministrasi`, `statusAkademik`, `statusSemester`, `semesterKosongBerturut`, `tetapkanStatusSemester`) | semua |
| EC-07 | Pembayaran kurang dari jumlah tagihan | Kasus turunan dari EC-01 | `TagihanImpl.jumlahTerbayar` | UKT |
| EC-08 | Besaran UKT ditentukan per kelompok | Permendikbudristek No. 2 Tahun 2024 | `TagihanImpl.golonganUkt` | UKT |
| EC-09 | Pembayaran sudah dilakukan di bank tetapi statusnya belum terverifikasi, sehingga tagihan belum lunas | Praktik registrasi (pengaduan pembayaran yang belum tercatat bank) | `PembayaranImpl.status` = `MENUNGGU`, `sumberStatus`, `tanggalVerifikasi`, `konfirmasiPembayaran`, `cekStatusPembayaran` | UKT, BayarDuluIRS |
| EC-13 | Verifikasi pembayaran ditolak (misalnya nomor referensi atau nominal tidak cocok) | Turunan dari EC-09 | `PembayaranImpl.status` = `DITOLAK`; tagihan tetap belum lunas dan proses ditolak | UKT |
| EC-14 | Konfirmasi dilakukan padahal belum ada pembayaran yang tercatat | Turunan dari EC-09 | `konfirmasiPembayaran` menolak permintaan | UKT |
| EC-10 | Kelas penuh | Aturan umum pengisian IRS | `KelasImpl.kapasitas`, `RuanganImpl.kapasitas`, `cekKapasitasKelas` | IsiRencanaStudi |
| EC-11 | Jadwal kelas bentrok | Aturan umum pengisian IRS | MJadwal, MRuangan, `cekBentrokJadwal` | IsiRencanaStudi |
| EC-12 | Jumlah SKS melebihi batas berdasarkan indeks prestasi | Aturan umum pengisian IRS | MIndeksPrestasi (`ips`, `ipk`, `sksLulus`), `cekBatasSks` | IsiRencanaStudi |

## Batas sistem: pembayaran di luar SLCM

Pembayaran dana UKT tidak terjadi di sistem akademik. Mahasiswa membayar melalui bank atau sistem keuangan universitas, dan sistem akademik hanya menerima statusnya untuk dipakai sebagai syarat pengisian IRS. Model ini mengikuti batas tersebut tanpa mengubah struktur modul, karena aktivitas Bayar UKT, operasi `bayarUkt`, dan delta DIRSDuluBayar tetap dibutuhkan oleh alur proses.

| Elemen | Peran pada model |
|---|---|
| MTagihan | Tagihan yang diterbitkan sistem keuangan. Sistem akademik membaca jumlah, golongan, skema pembayaran, dan status lunasnya. |
| MPembayaran | Catatan status pembayaran yang dilaporkan bank, bukan transaksi pembayaran. `paymentId` berisi nomor referensi bank, `metodePembayaran` berisi bank atau kanal pembayaran, `sumberStatus` berisi `BANK` atau `MANUAL`, dan `tanggalVerifikasi` diisi saat status diverifikasi. |
| `bayarUkt` | Mencatat pembayaran yang dilaporkan dengan status `MENUNGGU`. Nomor referensi dikirim melalui `nomorReferensi`. Dana tidak diproses oleh sistem. |
| `konfirmasiPembayaran` | Memverifikasi status dari bank. Hasil `TERVERIFIKASI` membuat tagihan LUNAS; hasil `DITOLAK` (dikirim melalui `hasil`) menghentikan proses dan tagihan tetap belum lunas. |
| `cekStatusPembayaran` | Mengembalikan status lunas tagihan yang menjadi syarat IRS pada BayarDuluIRS dan syarat pembatalan IRS pada IRSDuluBayar. |
| MCicilan | Jadwal dan status cicilan dari sistem keuangan. Persetujuan cicilan terjadi di luar sistem akademik. |
| `BeasiswaImpl.penundaanPembayaran` | Status penundaan pembayaran yang dipakai sebagai pengecualian syarat IRS. |

Dalam pengujian, pemanggilan `bayarUkt` berperan sebagai simulasi notifikasi pembayaran dari bank, dan `konfirmasiPembayaran` berperan sebagai verifikasi status tersebut.

## Catatan

- Semua operasi baru (`cekCicilanPertama`, `cekPeriodeAktif`, `tetapkanStatusSemester`, `batalkanIRS`) bukan aktivitas pada varBPMN. Operasi tersebut dipanggil oleh logika aktivitas yang sudah ada, sama seperti `cekStatusPembayaran`, sehingga tidak menambah *endpoint* proses.
- Status disimpan sebagai `String` karena pembangkit kode UML-DOP hanya memetakan tipe `Boolean`, `Date`, `Integer`, `String`, dan `UUID`.
- Nilai status yang dipakai: `statusSemester` = `AKTIF` | `KOSONG` | `MENGUNDURKAN_DIRI`; `CicilanImpl.status` = `BELUM_BAYAR` | `LUNAS` | `TERLAMBAT`; `PembayaranImpl.status` = `MENUNGGU` | `TERVERIFIKASI` | `DITOLAK`; `PengisianRencanaStudiImpl.status` ditambah `DIBATALKAN`; `PeriodeRegistrasiImpl.jenis` = `PEMBAYARAN` | `IRS`; `TagihanImpl.skemaPembayaran` = `LUNAS` | `CICILAN`.
