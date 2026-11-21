# Studi Kasus: Website & Server Tidak Bisa Diakses — Kabel LAN Server Kendor

**Kategori:** Troubleshooting Web / Infrastruktur
**Tanggal:** 21 November 2026
**Lingkungan:** Website WordPress (lspsmkp.id) yang di-hosting di VPS, dikelola lewat phpMyAdmin
**Tools yang digunakan:** Google Chrome, Windows PowerShell (`Test-NetConnection`), phpMyAdmin, pemeriksaan fisik server

---

## 1. Ringkasan

Sebuah website WordPress milik klien (`lspsmkp.id`) tiba-tiba tidak bisa diakses, dan server hosting-nya juga tidak bisa dibuka lewat panel manajemen (port 51623) maupun diverifikasi di level jaringan. Pengecekan awal lewat browser dan PowerShell menunjukkan gejala yang sama: server sama sekali tidak memberikan respons, bukan menolak atau memblokir koneksi. Setelah ditelusuri, akar masalahnya ternyata **kabel LAN fisik di server yang kendor/lepas**. Setelah kabel dipasang ulang dengan benar, akses ke panel server maupun website kembali normal.

---

## 2. Gejala Awal

### 2.1 Website tidak bisa diakses
Mengunjungi `lspsmkp.id/dashboard` menampilkan halaman 404 "Nothing Found" bawaan tema WordPress — menandakan aplikasi web-nya sebenarnya masih berjalan dan melayani halaman, tetapi URL/route yang dituju tidak ditemukan.

![Website menampilkan Nothing Found](assets/01-website-down.png)

### 2.2 Panel manajemen server tidak bisa diakses
Saat mencoba membuka panel server langsung lewat IP dan port-nya (`194.233.78.61:51623/index.php`), muncul pesan **"Hmmm... can't reach this page"** dengan error `ERR_CONNECTION_TIMED_OUT`.

![Browser timeout saat akses IP server](assets/02-server-login-fail.png)

### 2.3 Pengujian level jaringan memastikan tidak ada respons
Untuk memastikan apakah ini masalah di level aplikasi atau di level jaringan/konektivitas, dijalankan `Test-NetConnection` dari PowerShell ke IP server pada beberapa port:

```powershell
Test-NetConnection 194.233.78.61 -Port 51623
Test-NetConnection 194.233.78.61 -Port 443
```

Kedua pengujian gagal dengan hasil yang sama:
```
WARNING: TCP connect to (194.233.78.61 : 51623) failed
WARNING: Ping to 194.233.78.61 failed with status: TimedOut
...
WARNING: TCP connect to (194.233.78.61 : 443) failed
WARNING: Ping to 194.233.78.61 failed with status: DestinationHostUnreachable
```

![Hasil Test-NetConnection PowerShell gagal](assets/03-powershell-test-fail.png)

**Analisis:** Karena *semua* port yang diuji gagal — bukan hanya port panel saja — dan bahkan ICMP (ping) menghasilkan `DestinationHostUnreachable`, ini menyingkirkan kemungkinan masalah di satu aplikasi/service tertentu (misalnya aturan firewall yang salah untuk satu port saja). Server yang masih bisa dijangkau tapi memfilter trafik biasanya tetap merespons ping meski TCP-nya gagal, atau hanya memblokir port tertentu. Di sini, tidak adanya respons sama sekali di semua protokol dan port mengarah ke **kegagalan konektivitas di layer yang lebih rendah** — yaitu server memang tidak terjangkau sama sekali di jaringan, sesuai dengan ciri masalah kabel/koneksi fisik, bukan kesalahan konfigurasi software atau firewall.

---

## 3. Tindakan yang Dilakukan

Karena akses lewat software/remote sudah mentok, dilakukan pemeriksaan fisik langsung ke server. Ditemukan bahwa **kabel LAN** yang menghubungkan server ke switch/router dalam kondisi **kendor**. Kabel tersebut kemudian:

1. Dicabut dari kedua ujungnya (port NIC server dan port switch/router).
2. Diperiksa apakah ada kerusakan fisik yang terlihat (tidak ditemukan kerusakan).
3. Dipasang kembali dengan kencang ke kedua port.

---

## 4. Verifikasi Perbaikan

### 4.1 Akses server kembali normal
phpMyAdmin, yang dilayani lewat panel server, kembali bisa diakses, memastikan koneksi jaringan sudah tersambung lagi.

![phpMyAdmin bisa diakses kembali](assets/04-phpmyadmin-access-restored.png)

### 4.2 Akses website kembali normal
`lspsmkp.id` kembali bisa dibuka dan menampilkan seluruh konten halaman utamanya dengan baik.

![Website kembali normal](assets/05-website-restored.png)

---

## 5. Akar Masalah (Root Cause)

Kabel LAN yang kendor/lepas di interface jaringan server menyebabkan gangguan total di level jaringan: server dalam keadaan menyala dan service di dalamnya kemungkinan tetap berjalan, tetapi tidak bisa dijangkau dari luar jaringan pada port maupun protokol apa pun (baik TCP maupun ICMP). Hal ini awalnya terlihat seperti blokir firewall atau filtering di level IP, tetapi pola kegagalannya (total, di semua port, di semua protokol) justru menjadi indikator kuat yang mengarah ke masalah fisik, bukan software/firewall.

---

## 6. Pelajaran yang Didapat

- **Bedakan antara "diblokir" dan "tidak terjangkau."** Satu port timeout sementara port lain tetap merespons biasanya menandakan aturan firewall. Timeout *total* di semua port dan protokol (termasuk ICMP) adalah indikasi yang lebih kuat bahwa masalahnya ada di jaringan/fisik.
- **`Test-NetConnection` adalah alat triase yang cepat** untuk memisahkan apakah masalah ada di level aplikasi (misalnya konfigurasi WordPress) atau lebih dalam lagi di level jaringan/fisik.
- **Jangan lupakan infrastruktur fisik** saat troubleshooting "server tidak bisa diakses" — terutama ketika semua pengecekan di level software (resolusi DNS, IP yang benar, browser sudah dicoba ulang) sudah benar tapi koneksi tetap gagal total.
- **Dokumentasikan proses eliminasinya**, bukan cuma hasil akhirnya — menunjukkan bahwa kemungkinan masalah di level aplikasi dan firewall sudah disingkirkan dulu sebelum naik ke pemeriksaan fisik menunjukkan metodologi troubleshooting yang terstruktur.
