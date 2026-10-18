# OpenWrt 25.12.5 untuk Ruijie RG-MA3063

Port device baru ke target upstream `qualcommax/ipq50xx` (kernel 6.12.94).
SoC IPQ5018 sudah didukung resmi di 25.12, jadi yang ditambahkan hanya
definisi board MA3063 — basisnya CMCC PZ-L8 yang hardware-nya hampir sama.

| | |
|---|---|
| SoC | IPQ5018 (MP03.5-C1), 2x A53 |
| RAM | 256 MB |
| Flash | 128 MB SPI-NAND (Winbond / Macronix / Fudan FM25S01A) |
| Ethernet | 1 WAN + 3 LAN, QCA8337 via SGMII |
| WiFi | 2.4 GHz IPQ5018 2x2 ax, 5 GHz QCN6102 2x2 ax |

## Isi folder

- `patches/0001-...patch` — satu commit, berlaku di atas tag `v25.12.5`
- `.github/workflows/build.yml` — build otomatis di GitHub Actions
- `ma3063.config` — seed `.config` (LuCI + HTTPS, zram-swap, image initramfs)

## Bedanya dengan port komunitas yang sudah ada

1. **Sysupgrade sadar A/B.** MA3063 punya `rootfs` dan `rootfs_1` yang dipilih
   lewat `0:bootconfig`. Port ini memakai jalur upgrade yang sama dengan PZ-L8 /
   ELECOM di upstream, bukan `nand_do_upgrade` polos yang bisa menulis ke slot
   yang salah.
2. **MAC dibaca langsung dari U-Boot env** lewat nvmem, tanpa init script
   tambalan. Susunannya: LAN = `ethaddr`, WAN = +1, 2.4 GHz = +2, 5 GHz = +3.
3. **BDF di-encode ulang dengan benar.** Data kalibrasinya identik dengan yang
   beredar, tapi nama board-nya sekarang satu string lengkap
   (`...,variant=Ruijie-RG-MA3063`), bukan dua entri terpisah dengan typo
   "RUJIE".
4. `fw_env.config` terdaftar, jadi `fw_printenv`/`fw_setenv` langsung jalan.

## Build

1. Buat repo GitHub baru, masukkan seluruh isi folder ini (termasuk `.github/`).
2. Tab **Actions** → *Build OpenWrt 25.12.5 - Ruijie RG-MA3063* → **Run workflow**.
3. Tunggu sekitar 1,5–3 jam. Hasilnya ada di bagian *Artifacts*:
   - `...-initramfs-uImage.itb` — untuk uji coba di RAM
   - `...-squashfs-factory.ubi` — untuk flash pertama lewat U-Boot
   - `...-squashfs-sysupgrade.bin` — untuk upgrade berikutnya dari OpenWrt

Build lokal: `git clone -b v25.12.5 https://github.com/openwrt/openwrt`,
`git am patches/*.patch`, `./scripts/feeds update -a && ./scripts/feeds install -a`,
`cp ma3063.config .config && make defconfig && make -j$(nproc)`.

## Instalasi (butuh TTL 115200 8N1)

Pastikan backup MTD lengkap tersimpan aman sebelum mulai.

### Langkah 0 — cek nama config FIT di U-Boot

Image memakai `config@mp03.5-c1` (sama seperti PZ-L8 dan port komunitas). U-Boot
QSDK memilih config berdasarkan nama, jadi cocokkan dulu. Dari firmware stock
via SSH:

```sh
cat /proc/mtd                      # cari nomor mtd "0:APPSBL"
strings /dev/mtdX | grep -o 'config@[a-z0-9.-]*' | sort -u
```

Kalau yang muncul bukan `config@mp03.5-c1` (misalnya `config@mp02.1`), ubah
`DEVICE_DTS_CONFIG` di patch (`target/linux/qualcommax/image/ipq50xx.mk`) lalu
build ulang. Langkah 1 tetap aman walau nama ini salah.

### Langkah 1 — uji di RAM (tidak menulis apa pun ke flash)

Hentikan boot dengan Enter sampai muncul `IPQ5018#`. Kabel LAN ke port dekat
colokan power, PC di 192.168.1.123 dengan server TFTP.

```
setenv ipaddr 192.168.1.1
setenv serverip 192.168.1.123
tftpboot 0x44000000 openwrt-25.12.5-...-ruijie_rg-ma3063-initramfs-uImage.itb
bootm 0x44000000
```

Setelah boot, cek:

```sh
ip link                         # lan1 lan2 lan3 wan harus ada
dmesg | grep -i ath11k          # kedua radio harus terdeteksi
iw dev                          # phy0 (2.4) dan phy1 (5) muncul
free -m                         # catat sisa RAM setelah WiFi aktif
cat /sys/class/net/br-lan/address
```

Kalau ada yang tidak beres, cukup cabut power — stock firmware masih utuh.

### Langkah 2 — flash permanen

Hanya kalau Langkah 0 dan 1 beres. Tulis ke **kedua** slot supaya bootconfig
menunjuk ke mana pun tetap boot OpenWrt:

```
tftpboot 0x44000000 openwrt-25.12.5-...-ruijie_rg-ma3063-squashfs-factory.ubi
flash rootfs
flash rootfs_1
reset
```

Jangan menyentuh partisi U-Boot (`0:APPSBL`), apalagi pada unit dengan flash
Fudan — U-Boot yang salah di sana yang bikin bata.

### Upgrade berikutnya

`sysupgrade` biasa dari LuCI atau shell memakai `...-squashfs-sysupgrade.bin`.

### Kembali ke stock

Dari U-Boot, TFTP backup partisi `rootfs` Anda lalu `flash rootfs` (dan
`rootfs_1` dari backup-nya bila ada).

## Risiko yang sudah diketahui

- **RAM 256 MB.** Upstream sengaja mematikan WiFi di PZ-L8 karena OOM saat
  LuCI dipakai. Di port ini WiFi aktif dengan firmware memory mode 1 +
  `kmod-ath11k-smallbuffers` + zram-swap. Untuk dipakai sebagai AP mode
  bridge (seperti sekarang) seharusnya cukup; jangan tambah paket berat.
- **Asal BDF.** File kalibrasi diambil dari port komunitas. Bandingkan dengan
  milik unit Anda: payload 128 KB-nya harus ber-MD5
  `1dc888c099cde2c29c6e25641b32f03b` (IPQ5018) dan
  `f20ca4e08a5c71d56e5a080c42a559a2` (QCN6122). Kalau stock Anda beda,
  BDF bisa diganti dari `/lib/firmware/` firmware stock.
- **qmi-board-id.** BDF di-encode untuk `qmi-board-id=255`. Jika WiFi gagal
  dengan pesan `failed to fetch board data for bus=ahb,...`, nilai board-id
  sebenarnya ada di baris itu — tinggal encode ulang.
