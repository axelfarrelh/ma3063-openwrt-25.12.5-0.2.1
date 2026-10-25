# OpenWrt 25.12.5 untuk Ruijie RG-MA3063 (FIT `config@mp02.1`)

Port tidak resmi OpenWrt 25.12.5 untuk router **Ruijie RG-MA3063** (AX3000, umumnya versi China Mobile). Device ditambahkan ke target upstream `qualcommax/ipq50xx` (kernel 6.12). SoC IPQ5018 sudah didukung resmi sejak 25.12, jadi isi port ini hanya definisi board MA3063, dengan basis CMCC PZ-L8 yang hardware-nya hampir sama.

> [!IMPORTANT]
> **Repo ini khusus untuk unit yang U-Boot-nya mencari FIT config `config@mp02.1`.**
> Image di sini hanya membawa satu konfigurasi FIT, yaitu `config@mp02.1`. Kalau U-Boot unit Anda mencari nama lain (misalnya `config@mp03.5-c1`), image ini **tidak akan boot**. Untuk varian `config@mp03.5-c1`, gunakan repo [ma3063-openwrt-25.12.5](https://github.com/axelfarrelh/ma3063-openwrt-25.12.5). Cara mengeceknya ada di [Langkah 0](#langkah-0--pastikan-u-boot-memakai-configmp021).

> [!WARNING]
> Ini firmware tidak resmi dan belum masuk upstream OpenWrt. Flashing lewat UART selalu punya risiko bata. Simpan **backup MTD lengkap** sebelum mulai dan gunakan dengan risiko sendiri.

## Daftar isi

- [Spesifikasi](#spesifikasi)
- [Isi repo](#isi-repo)
- [Fitur dan perbedaan dengan port komunitas](#fitur-dan-perbedaan-dengan-port-komunitas)
- [Peta port Ethernet](#peta-port-ethernet)
- [Build](#build)
- [Instalasi](#instalasi-butuh-ttl-115200-8n1)
- [Upgrade dan kembali ke stock](#upgrade-berikutnya)
- [Catatan RAM 256 MB](#catatan-ram-256-mb)
- [Risiko yang sudah diketahui](#risiko-yang-sudah-diketahui)
- [Lisensi](#lisensi)
- [Kredit](#kredit)

## Spesifikasi

| | |
|---|---|
| SoC | Qualcomm IPQ5018 (board AP-MP02.1), 2x Cortex-A53 |
| RAM | 256 MB DDR3 |
| Flash | 128 MB SPI-NAND (Winbond / Macronix / FudanMicro FM25S01A) |
| Ethernet | 4x 1 GbE: 1 WAN (GE PHY internal IPQ5018) + 3 LAN (switch QCA8337 via SGMII) |
| WiFi 2.4 GHz | IPQ5018, 2x2 802.11ax |
| WiFi 5 GHz | QCN6102, 2x2 802.11ax (memakai firmware keluarga QCN6122) |
| LED | Status hijau / biru / merah |
| Tombol | Reset, WPS |
| UART | 115200 8N1 |
| FIT config | `config@mp02.1` |

## Isi repo

| File | Fungsi |
|---|---|
| `patches/0001-qualcommax-ipq50xx-add-support-for-Ruijie-RG-MA3063.patch` | Satu commit di atas tag `v25.12.5`: DTS, definisi image, BDF WiFi, MAC, sysupgrade A/B, `fw_env.config` |
| `patches-mac80211/990-ath11k-shrink-dp-rings-for-256M-RAM.patch` | Opsional. Mengecilkan ring DMA ath11k ke ukuran profil 256 MB milik QSDK agar RAM lebih lega |
| `.github/workflows/build.yml` | Build otomatis di GitHub Actions |
| `ma3063.config` | Seed `.config`: LuCI + HTTPS, zram-swap, image initramfs untuk uji di RAM |
| `LICENSE` | GPL-2.0 |

## Fitur dan perbedaan dengan port komunitas

1. **Sysupgrade sadar A/B.** MA3063 punya dua slot, `rootfs` dan `rootfs_1`, yang dipilih lewat `0:bootconfig`. Port ini memakai jalur upgrade yang sama dengan PZ-L8 / ELECOM / I-O DATA di upstream, bukan `nand_do_upgrade` polos yang bisa menulis ke slot yang salah.
2. **MAC dibaca langsung dari U-Boot env** (`0:appsblenv`, variabel `ethaddr`) lewat nvmem, tanpa init script tambahan.
3. **Data kalibrasi WiFi diambil dari partisi `0:art` milik unit sendiri** saat boot, lalu MAC radio di-patch dan regdomain bawaan dihapus agar pengaturan negara di OpenWrt yang berlaku.
4. **BDF dikemas dengan nama board satu string** (`...,variant=Ruijie-RG-MA3063`), sesuai `qcom,ath11k-calibration-variant` di DTS.
5. **`fw_printenv` / `fw_setenv` langsung jalan** karena `0:appsblenv` sudah didaftarkan di uboot-envtools.
6. **Kanal 5 GHz U-NII-3 tetap tersedia** (batas frekuensi 5150–5850 MHz). Atur kode negara di LuCI sesuai lokasi Anda.

### Susunan MAC

| Interface | MAC |
|---|---|
| LAN (`br-lan`) | `ethaddr` |
| WAN | `ethaddr` + 1 |
| WiFi 2.4 GHz | `ethaddr` + 2 |
| WiFi 5 GHz | `ethaddr` + 3 |

## Peta port Ethernet

> [!NOTE]
> Label di casing **tidak sama** dengan penamaan di OpenWrt. Port yang tercetak "WAN" di casing justru menjadi LAN.

| Port fisik | Nama di OpenWrt | Jalur hardware |
|---|---|---|
| Port 1, tepat di sebelah colokan power | `wan` | GE PHY internal IPQ5018 (`dp1`) |
| Port 2 | LAN (`lan1`–`lan3`) | Switch QCA8337 (`dp2`) |
| Port 3 | LAN (`lan1`–`lan3`) | Switch QCA8337 (`dp2`) |
| Port bertuliskan "WAN" di casing | LAN (`lan1`–`lan3`) | Switch QCA8337 (`dp2`) |

## Build

### Lewat GitHub Actions (disarankan)

1. **Fork** repo ini (atau salin seluruh isinya, termasuk folder `.github/`, ke repo Anda sendiri).
2. Buka tab **Actions** → **Build OpenWrt 25.12.5 - Ruijie RG-MA3063** → **Run workflow**.
3. Isi opsi build:

   | Opsi | Default | Keterangan |
   |---|---|---|
   | `openwrt_tag` | `v25.12.5` | Tag OpenWrt yang dipakai. Patch hanya diuji di `v25.12.5` |
   | `fw_memory_mode` | `2` | Mode memori firmware ath11k. `1` = hemat, `2` = paling hemat untuk RAM 256 MB. DTS di patch memakai `1`; workflow menimpanya dengan nilai ini |
   | `ath11k_small_rings` | aktif | Menerapkan patch `patches-mac80211`. Disarankan untuk RAM 256 MB |

4. Tunggu sekitar 1,5–3 jam. Hasilnya ada di bagian **Artifacts** (nama artifact mencantumkan mode memori dan `-smallrings` bila aktif):

   | File | Kegunaan |
   |---|---|
   | `...-initramfs-uImage.itb` | Uji coba di RAM, tidak menulis ke flash |
   | `...-squashfs-factory.ubi` | Flash pertama lewat U-Boot |
   | `...-squashfs-sysupgrade.bin` | Upgrade berikutnya dari OpenWrt |
   | `sha256sums`, `*.manifest` | Checksum dan daftar paket |

### Build lokal

```sh
git clone -b v25.12.5 https://github.com/openwrt/openwrt
cd openwrt
git am /path/ke/repo-ini/patches/*.patch

# Opsional, untuk RAM 256 MB:
cp /path/ke/repo-ini/patches-mac80211/*.patch package/kernel/mac80211/patches/ath11k/

./scripts/feeds update -a && ./scripts/feeds install -a
cp /path/ke/repo-ini/ma3063.config .config
make defconfig
make -j$(nproc)
```

Hasil build ada di `bin/targets/qualcommax/ipq50xx/`.

## Instalasi (butuh TTL 115200 8N1)

Yang dibutuhkan:

- Adaptor USB-TTL 3.3 V yang tersambung ke header UART di board.
- PC dengan server TFTP di alamat `192.168.1.123`.
- Kabel LAN ke **port 1, di sebelah colokan power**.
- **Backup MTD lengkap** semua partisi, disimpan di luar router.

### Langkah 0 — pastikan U-Boot memakai `config@mp02.1`

Langkah ini menentukan apakah repo ini cocok untuk unit Anda. Dari firmware stock lewat SSH:

```sh
cat /proc/mtd                                         # cari nomor mtd "0:APPSBL"
strings /dev/mtdX | grep -o 'config@[a-z0-9.-]*' | sort -u
```

- Muncul `config@mp02.1` → lanjut ke Langkah 1.
- Muncul nama lain, misalnya `config@mp03.5-c1` → **berhenti**, gunakan repo [ma3063-openwrt-25.12.5](https://github.com/axelfarrelh/ma3063-openwrt-25.12.5) atau ubah `DEVICE_DTS_CONFIG` di `target/linux/qualcommax/image/ipq50xx.mk` (di dalam patch) lalu build ulang.

Kalau tidak bisa masuk SSH, Langkah 1 tetap aman dicoba: image yang salah config hanya gagal boot dan tidak menulis apa pun ke flash.

### Langkah 1 — uji di RAM (tidak menulis apa pun ke flash)

Nyalakan router dan tekan Enter berulang sampai muncul prompt `IPQ5018#`.

```
setenv ipaddr 192.168.1.1
setenv serverip 192.168.1.123
tftpboot 0x44000000 openwrt-25.12.5-...-ruijie_rg-ma3063-initramfs-uImage.itb
bootm 0x44000000
```

Setelah OpenWrt boot, periksa:

```sh
ip link                            # lan1 lan2 lan3 wan harus ada
dmesg | grep -i ath11k             # kedua radio terdeteksi tanpa error
iw dev                             # phy0 (2.4 GHz) dan phy1 (5 GHz) muncul
free -m                            # catat sisa RAM setelah WiFi aktif
cat /sys/class/net/br-lan/address  # harus sama dengan ethaddr di U-Boot
```

Kalau ada yang tidak beres, cukup cabut power. Firmware stock masih utuh.

### Langkah 2 — flash permanen

Lakukan hanya kalau Langkah 0 dan 1 berhasil. Tulis ke **kedua** slot supaya ke mana pun `bootconfig` menunjuk, router tetap boot OpenWrt:

```
tftpboot 0x44000000 openwrt-25.12.5-...-ruijie_rg-ma3063-squashfs-factory.ubi
flash rootfs
flash rootfs_1
reset
```

> [!CAUTION]
> Jangan menyentuh partisi U-Boot (`0:APPSBL`), terutama pada unit dengan flash FudanMicro. U-Boot yang salah atau rusak adalah penyebab utama bata yang hanya bisa dipulihkan dengan programmer SPI.

Setelah boot, LuCI tersedia di `https://192.168.1.1` (user `root`, belum ada password; segera buat password).

### Upgrade berikutnya

Gunakan `sysupgrade` biasa dari LuCI (**System → Backup / Flash Firmware**) atau dari shell, dengan file `...-squashfs-sysupgrade.bin`. Skrip upgrade otomatis menulis ke slot yang benar sesuai `bootconfig`.

### Kembali ke stock

Dari U-Boot, TFTP backup partisi `rootfs` Anda lalu jalankan `flash rootfs`. Lakukan hal yang sama untuk `rootfs_1` dari backup-nya bila ada. Sebaiknya kedua slot berisi versi firmware stock yang sama.

## Catatan RAM 256 MB

Upstream sengaja mematikan WiFi di PZ-L8 karena kehabisan memori (OOM) saat LuCI dipakai. Di port ini WiFi tetap aktif dengan kombinasi:

- `qcom,ath11k-fw-memory-mode` = 1 di DTS, atau 2 lewat opsi workflow.
- Patch ring DMA kecil (`patches-mac80211`). Tanpa patch ini, ath11k mengisi ring RX untuk kedua radio saat probe dan memakan sekitar 97 MB walau interface belum aktif. Ukuran ring diambil dari profil 256 MB firmware QSDK Ruijie (`QCA5018_i.ini` dan `QCN6122_i.ini`).
- `zram-swap`.

Untuk pemakaian sebagai **AP / bridge** seharusnya cukup. Untuk router utama (NAT, PPPoE, SQM), hindari paket berat dan pantau `free -m`.

## Risiko yang sudah diketahui

- **Hanya `config@mp02.1`.** Lihat [Langkah 0](#langkah-0--pastikan-u-boot-memakai-configmp021).
- **Asal BDF.** File board data WiFi diambil dari port komunitas, yang aslinya berasal dari firmware stock Ruijie. Bandingkan dengan unit Anda: payload 128 KB-nya harus ber-MD5 `1dc888c099cde2c29c6e25641b32f03b` (IPQ5018) dan `f20ca4e08a5c71d56e5a080c42a559a2` (QCN6122). Kalau berbeda, BDF bisa diganti dengan file dari `/lib/firmware/` di firmware stock unit Anda.
- **`qmi-board-id`.** BDF di-encode untuk `qmi-board-id=255`. Kalau WiFi gagal dengan pesan `failed to fetch board data for bus=ahb,...`, nilai board-id yang sebenarnya tercantum di baris pesan itu; BDF tinggal di-encode ulang dengan nilai tersebut.
- **Belum upstream.** Patch ini belum dikirim ke OpenWrt, jadi tidak ada build resmi dan tidak ada jaminan kompatibilitas dengan versi selain `v25.12.5`.

## Lisensi

Repo ini berlisensi **GNU General Public License v2.0**, mengikuti OpenWrt dan kernel Linux. Lihat [LICENSE](LICENSE).

- File DTS baru berlisensi ganda `GPL-2.0-or-later OR MIT`, mengikuti konvensi DTS ipq50xx di OpenWrt.
- **File board data WiFi** (`board-ruijie_rg-ma3063.ipq5018` dan `board-ruijie_rg-ma3063.qcn6122`) **tidak termasuk lisensi GPL**. File tersebut berasal dari firmware stock Ruijie RG-MA3063, hak ciptanya tetap milik pemiliknya (Ruijie / Qualcomm), dan disertakan apa adanya hanya agar WiFi dapat berfungsi di OpenWrt, sesuai praktik paket `ipq-wifi` di OpenWrt.
- Firmware hasil build mengandung perangkat lunak GPL. Kode sumber lengkapnya adalah OpenWrt tag `v25.12.5` ditambah patch di repo ini.

Merek Ruijie, China Mobile, Qualcomm, dan OpenWrt adalah milik masing-masing pemiliknya. Proyek ini tidak berafiliasi dengan mereka.

## Kredit

- [OpenWrt](https://github.com/openwrt/openwrt) dan para kontributornya: basis firmware dan dukungan target `qualcommax/ipq50xx`.
- Kontributor dukungan **CMCC PZ-L8** di upstream OpenWrt: basis definisi board dan jalur sysupgrade A/B.
- Port komunitas MA3063 (Nwrt) oleh `<nama pembuat>` — `<tautan>`: sumber file BDF dan referensi awal.
- Ruijie Networks: data kalibrasi dan profil memori QSDK dari firmware stock.
- Patch disusun dengan bantuan Claude (Anthropic), sebagaimana tercatat di baris `Co-Authored-By` pada masing-masing patch.
