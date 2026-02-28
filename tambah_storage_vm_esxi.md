# Tambah Storage VM Ubuntu (ESXi + LVM)

Panduan ini menjelaskan cara memperluas storage di Ubuntu Server yang berjalan di atas hypervisor (ESXi/Proxmox/VMware) menggunakan LVM (Logical Volume Manager).

## Prasyarat

- Akses root atau sudo ke server
- VM menggunakan LVM (default pada Ubuntu Server)
- Disk sudah diperbesar melalui panel hypervisor (ESXi, Proxmox, dll.)
- VM sudah dinyalakan ulang setelah penambahan disk

---

## Step 1 — Cek Kondisi Disk Saat Ini

```bash
fdisk -l
```

Cari bagian **Disk /dev/sda** dan catat ukuran total. Pastikan ukurannya sudah sesuai dengan yang ditambahkan di panel hypervisor.

Cek juga penggunaan filesystem:

```bash
df -h
```

---

## Step 2 — Buat Partisi Baru

```bash
fdisk /dev/sda
```

Setelah masuk prompt `fdisk`, ikuti langkah berikut:

| Input | Keterangan |
|-------|------------|
| `p` | Tampilkan partisi yang ada |
| `n` | Buat partisi baru |
| `[Enter]` | Partition number (gunakan default) |
| `[Enter]` | First sector (gunakan default) |
| `[Enter]` | Last sector (gunakan semua sisa ruang) |
| `p` | Verifikasi partisi baru sudah muncul |
| `w` | Simpan perubahan dan keluar |

> **Catat nomor partisi baru yang terbentuk**, misalnya `/dev/sda3`, `/dev/sda7`, dst.

---

## Step 3 — Cek Nama Volume Group

```bash
vgdisplay
```

Catat nilai **VG Name**, biasanya `ubuntu-vg`.

---

## Step 4 — Tambahkan Partisi ke Volume Group

Ganti `sdaX` dengan nomor partisi baru dari Step 2, dan `<vg-name>` dengan nama VG dari Step 3.

```bash
vgextend <vg-name> /dev/sdaX
```

Contoh:

```bash
vgextend ubuntu-vg /dev/sda7
```

Verifikasi ruang VG sudah bertambah:

```bash
vgdisplay
```

Pastikan **Free PE / Size** menunjukkan angka yang lebih besar.

---

## Step 5 — Perbesar Logical Volume

Gunakan semua ruang yang tersedia di VG:

```bash
lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
```

Atau tentukan ukuran spesifik (ganti `10G` sesuai kebutuhan):

```bash
lvextend -L+10G /dev/ubuntu-vg/ubuntu-lv
```

---

## Step 6 — Resize Filesystem

```bash
resize2fs /dev/ubuntu-vg/ubuntu-lv
```

Tunggu proses selesai. Durasi bervariasi tergantung ukuran disk.

---

## Step 7 — Verifikasi

```bash
df -h
```

Pastikan kolom **Size** pada `/dev/mapper/ubuntu--vg-ubuntu--lv` sudah menunjukkan ukuran baru.

---

## Troubleshooting

| Masalah | Penyebab | Solusi |
|---------|----------|--------|
| `Volume group name has invalid characters` | Nama VG ditulis dengan `{kurung kurawal}` | Hapus `{}`, tulis langsung nama VG-nya |
| `Insufficient free space` | VG belum diperluas sebelum `lvextend` | Jalankan `vgextend` terlebih dahulu (Step 4) |
| Partisi baru tidak muncul setelah `fdisk` | Kernel belum membaca ulang tabel partisi | Jalankan `partprobe /dev/sda` atau reboot |
| Ukuran disk di `fdisk` tidak sesuai | VM belum direstart setelah penambahan disk di hypervisor | Restart VM, lalu ulangi dari Step 1 |

---

## Catatan

- Semua perintah di atas memerlukan hak akses **root** atau **sudo**
- Panduan ini menggunakan filesystem **ext4** (default Ubuntu). Jika menggunakan **XFS**, ganti `resize2fs` dengan `xfs_growfs /`
- Lakukan **backup data penting** sebelum melakukan perubahan partisi
