---
title: "Raspberry Pi Secure Boot Enablement"
date: 2022-08-15
categories:
  - blog
tags:
  - Secure Boot
---

這篇主要是記錄RPi CM4上，如何開啟secure boot的過程。目前找得到的資料中，只有Compute Module系列可以有機會啟用secure
boot。是因為在CM系列上有預留`nRPIBOOT`的PIN，可以打開`nRPIBOOT`
mode，進入`nRPIBOOT`才可以更新EEPROM上bootloader config。RPI4上沒有這個PIN，但好像也可以透過`config.txt`來設定這個`nRPIBOOT` PIN。
所以這裡是以CM4為範例, 一步一步記錄開啟secure boot的過程以及比較secure boot對於boot time的影響。

開始之前，有下列的東西需要準備
1. CM4 + Carrier Board
2. Host PC running Ubuntu 20.04

Ubuntu 20.04需要安裝的套件及工具
- Python Crypto Support
Secure boot require a 2048 bit RSA asymmetric keypair and the Python `pycrytodomex` module to sign the EEPROM config and boot image.
```shell
$python3 -m pip install pycryptodomex
```
- USB Device Boot Code
A special firmware to emulate CM4 as a USB mass storage device.
```shell
$sudo apt install git libusb-1.0-0-dev pkg-config
$git clone https://github.com/raspberrypi/usbboot.git
$cd usbboot
$make
```

## Generate RSA Key-Pair
The RSA key pair is required to sign EEPROM bootloader and boot image.
```shell
$cd usbboot
$mkdir keypair
$openssl genrsa 2048 > private.pem
$openssl rsa -in private.pem -out public.pem -pubout -outform PEM
```

## Signed Boot Image
Secure boot需要產生一個`boot.img` FAT image以及`boot.sig` signature file。
這裡所使用的boot firmware是從rpi github上clone下來的，或是可以從boot partition上把firmware copy出來。
```shell
$git clone --depth 1 --branch stable https://github.com/raspberrypi/firmware.git
```

### Creating a signed boot image
- Copy boot firmware
```shell
$cd usbboot
$mkdir secure-boot-files
$cp -R ../firmware/boot/* secure-boot-files
```
- make-boot-image
```shell
$sudo tools/make-boot-image -d secure-boot-files -o boot.img -b cm4 -a 64
```
- Sign boot image
```shell
$sudo tools/rpi-eeprom-digest -i boot.img -o boot.sig -k keypair/private.pem
```
到這一步後，會有兩個檔案產生，`boot.img` 和 `boot.sig`, 把這兩個檔案copy到CM4 boot partitation.
Copy前要先進到`nRPIBOOT` mode，然後把`/dev/sda2`mount到PC上，在把`boot.img`和`boot.sig`copy進去。

## 更新EEPROM bootloader config
這一步是要更新EEPROM bootloader config，把secure boot打開。
主要是要修改`boot.conf`這個檔案，把`SIGNED_BOOT=1`加進去。設定的部份就這樣而以。
```shell
$cd usbboot/secure-boot-recovery
# modified boot.conf "SIGNED_BOOT=1" to enable secure boot
```
以下是完整的`boot.conf`
```shell
[all]
BOOT_UART=1
WAKE_ON_GPIO=0
POWER_OFF_ON_HALT=1
HDMI_DELAY=0

# SD, USB-MSD, BCM-USB-MSD, Network
BOOT_ORDER=0xf2541

# Disable self-update mode
ENABLE_SELF_UPDATE=0

# Select signed-boot mode in the EEPROM. This can be used to during development
# to test the signed boot image. Once secure boot is enabled via OTP this setting
# has no effect i.e. it is always 1.
SIGNED_BOOT=1
```
修改完後，就可以用`update-pieeprom.sh`這個script來更新bootloader config及用之前生成的key-pair來sign
```shell
$../tools/update-pieeprom.sh -k ../keypair/private.pem
```
在來就可以將新的bootloader寫到EEPROM。一樣還是需要先進入`nRPIBOOT`mode，在執行下面的指令。
```shell
$sudo ../rpiboot -d .
```
預期會看到以下的訊息
```shell
SIG pieeprom.sig 33ca8b2a63b1f91e5259d3232637b7a0c83ee1696545e94f22e92988f1ea3f47 1660530610
Reading EEPROM: 524288
EEPROM CWP: 00
Writing EEPROM
................................................................................................++.............................+
Verify BOOT EEPROM
Reading EEPROM: 524288
BOOT-EEPROM: UPDATED
EEPROM WP: bc
EEPROM WP: bc
EEPROM set WP 1
```
接著把`nRPIBOOT` mode disable，然後重新插拔電，應該就可以看到CM4從signed `boot.img`啟動。

## Secure Boot增加額外的開機時間
量測開機時間的區間是從Power On到看到U-Boot的這段期間，`boot.img`的驗証是在這段時間做的。

|              | Normal Boot | Secure Boot |
|--------------|-------------|-------------|
| Boot Time(s) | 5           | 15          |

開啟secure boot後，總共會對整體開機時間增加了10秒。

## CM4 Booting Logs

### Normal Boot
```shell
[2022-08-15 10:36:02] RPi: BOOTLOADER release VERSION:d8b208d3 DATE: 2022/02/22 TIME: 09:33:50 BOOTMODE: 0x00000006 part: 0 BUILD_TIMESTAMP=1645522430 0xb1cce3d6 0x00b03140 0x00065186
[2022-08-15 10:36:02] PM_RSTS: 0x00001000
[2022-08-15 10:36:02] part 00000000 reset_info 00000000
[2022-08-15 10:36:02] uSD voltage 3.3V
[2022-08-15 10:36:02] Initialising SDRAM 'Samsung' 16Gb x1 total-size: 16 Gbit 3200
[2022-08-15 10:36:03] 
[2022-08-15 10:36:03] Boot mode: SD (01) order f254
[2022-08-15 10:36:04] SD HOST: 250000000 CTL0: 0x00000000 BUS: 200000 Hz actual: 200000 HZ div: 1250 (625) status: 0x1fff0000 delay: 540
[2022-08-15 10:36:04] SD HOST: 250000000 CTL0: 0x00000f00 BUS: 200000 Hz actual: 200000 HZ div: 1250 (625) status: 0x1fff0000 delay: 540
[2022-08-15 10:36:04] EMMC
[2022-08-15 10:36:04] SD retry 1 oc 0
[2022-08-15 10:36:04] SD HOST: 250000000 CTL0: 0x00000000 BUS: 200000 Hz actual: 200000 HZ div: 1250 (625) status: 0x1fff0000 delay: 540
[2022-08-15 10:36:04] OCR c0ff8080 [0]
[2022-08-15 10:36:04] CID: 00150100424a54443452033047b6fcc7
[2022-08-15 10:36:04] SD HOST: 250000000 CTL0: 0x00000f00 BUS: 25000000 Hz actual: 25000000 HZ div: 10 (5) status: 0x1fff0000 delay: 4
[2022-08-15 10:36:04] SD HOST: 250000000 CTL0: 0x00000f04 BUS: 50000000 Hz actual: 41666666 HZ div: 6 (3) status: 0x1fff0000 delay: 2
[2022-08-15 10:36:04] MBR: 0x00000800,  524288 type: 0x0c
[2022-08-15 10:36:04] MBR: 0x00080800,60544991 type: 0x83
[2022-08-15 10:36:04] MBR: 0x00000000,       0 type: 0x00
[2022-08-15 10:36:04] MBR: 0x00000000,       0 type: 0x00
[2022-08-15 10:36:04] Trying partition: 0
[2022-08-15 10:36:04] type: 32 lba: 2048 oem: 'mkfs.fat' volume: ' system-boot'
[2022-08-15 10:36:04] rsc 32 fat-sectors 4033 c-count 516190 c-size 1
[2022-08-15 10:36:04] root dir cluster 2 sectors 0 entries 0
[2022-08-15 10:36:04] FAT32 clusters 516190
[2022-08-15 10:36:04] Trying partition: 0
[2022-08-15 10:36:04] type: 32 lba: 2048 oem: 'mkfs.fat' volume: ' system-boot'
[2022-08-15 10:36:04] rsc 32 fat-sectors 4033 c-count 516190 c-size 1
[2022-08-15 10:36:04] root dir cluster 2 sectors 0 entries 0
[2022-08-15 10:36:04] FAT32 clusters 516190
[2022-08-15 10:36:04] Read config.txt bytes     1914 hnd 0x00000000
[2022-08-15 10:36:04] Read start4.elf bytes  2241504 hnd 0x00000000
[2022-08-15 10:36:04] Read fixup4.dat bytes     5411 hnd 0x00000000
[2022-08-15 10:36:04] Firmware: d9b293558b4cef6aabedcc53c178e7604de90788 Nov 18 2021 16:16:49
[2022-08-15 10:36:04] 0x00b03140 0x00000000 0x00000fff
[2022-08-15 10:36:04] MEM GPU: 76 ARM: 948 TOTAL: 1024
[2022-08-15 10:36:05] Starting start4.elf @ 0xfec00200 partition 0
[2022-08-15 10:36:05] +
[2022-08-15 10:36:05] 
[2022-08-15 10:36:07] 
[2022-08-15 10:36:07] U-Boot 2021.01+dfsg-3ubuntu0~20.04.4 (Sep 21 2021 - 15:55:38 +0000)
[2022-08-15 10:36:07] 
[2022-08-15 10:36:07] DRAM:  1.9 GiB
[2022-08-15 10:36:07] RPI: Board rev 0x14 outside known range
[2022-08-15 10:36:07] RPI Unknown model (0xb03140)
[2022-08-15 10:36:07] MMC:   mmcnr@7e300000: 1, emmc2@7e340000: 0
[2022-08-15 10:36:07] Loading Environment from FAT... *** Warning - bad CRC, using default environment
```

### Secure boot
```shell
[2022-08-15 10:06:23] RPi: BOOTLOADER release VERSION:d8b208d3 DATE: 2022/02/22 TIME: 09:33:50 BOOTMODE: 0x00000006 part: 0 BUILD_TIMESTAMP=1645522430 0xb1cce3d6 0x00b03140 0x00065186
[2022-08-15 10:06:23] PM_RSTS: 0x00001000
[2022-08-15 10:06:23] part 00000000 reset_info 00000000
[2022-08-15 10:06:23] uSD voltage 3.3V
[2022-08-15 10:06:23] Initialising SDRAM 'Samsung' 16Gb x1 total-size: 16 Gbit 3200
[2022-08-15 10:06:24] 
[2022-08-15 10:06:24] Boot mode: SD (01) order f254
[2022-08-15 10:06:25] SD HOST: 250000000 CTL0: 0x00000000 BUS: 200000 Hz actual: 200000 HZ div: 1250 (625) status: 0x1fff0000 delay: 540
[2022-08-15 10:06:25] SD HOST: 250000000 CTL0: 0x00000f00 BUS: 200000 Hz actual: 200000 HZ div: 1250 (625) status: 0x1fff0000 delay: 540
[2022-08-15 10:06:25] EMMC
[2022-08-15 10:06:25] SD retry 1 oc 0
[2022-08-15 10:06:25] SD HOST: 250000000 CTL0: 0x00000000 BUS: 200000 Hz actual: 200000 HZ div: 1250 (625) status: 0x1fff0000 delay: 540
[2022-08-15 10:06:25] OCR c0ff8080 [0]
[2022-08-15 10:06:25] CID: 00150100424a54443452033047b6fcc7
[2022-08-15 10:06:25] SD HOST: 250000000 CTL0: 0x00000f00 BUS: 25000000 Hz actual: 25000000 HZ div: 10 (5) status: 0x1fff0000 delay: 4
[2022-08-15 10:06:25] SD HOST: 250000000 CTL0: 0x00000f04 BUS: 50000000 Hz actual: 41666666 HZ div: 6 (3) status: 0x1fff0000 delay: 2
[2022-08-15 10:06:25] MBR: 0x00000800,  524288 type: 0x0c
[2022-08-15 10:06:25] MBR: 0x00080800,60544991 type: 0x83
[2022-08-15 10:06:25] MBR: 0x00000000,       0 type: 0x00
[2022-08-15 10:06:25] MBR: 0x00000000,       0 type: 0x00
[2022-08-15 10:06:25] Trying partition: 0
[2022-08-15 10:06:25] type: 32 lba: 2048 oem: 'mkfs.fat' volume: ' system-boot'
[2022-08-15 10:06:25] rsc 32 fat-sectors 4033 c-count 516190 c-size 1
[2022-08-15 10:06:25] root dir cluster 2 sectors 0 entries 0
[2022-08-15 10:06:25] FAT32 clusters 516190
[2022-08-15 10:06:25] secure-boot
[2022-08-15 10:06:25] Loading boot.img ...
[2022-08-15 10:06:25] SIG boot.sig 2879b82e7474a5384db860d2a257e570120c2cf39c545f04b1a87e99daa0cf85 1659519242
[2022-08-15 10:06:31] Verifying
[2022-08-15 10:06:36] RSA verify
[2022-08-15 10:06:36] rsa-verify pass (0x0)
[2022-08-15 10:06:36] MBR: 0x00000000,       0 type: 0x00
[2022-08-15 10:06:36] MBR: 0x00000000,       0 type: 0x00
[2022-08-15 10:06:36] MBR: 0x00000000,       0 type: 0x00
[2022-08-15 10:06:36] MBR: 0x00000000,       0 type: 0x00
[2022-08-15 10:06:36] Trying partition: 0
[2022-08-15 10:06:36] type: 16 lba: 0 oem: 'mkfs.fat' volume: '  V       ^ '
[2022-08-15 10:06:36] rsc 4 fat-sectors 92 c-count 23012 c-size 4
[2022-08-15 10:06:36] root dir cluster 1 sectors 16 entries 256
[2022-08-15 10:06:36] FAT16 clusters 23012
[2022-08-15 10:06:36] Read config.txt bytes     1914 hnd 0x00000000
[2022-08-15 10:06:36] Read start4.elf bytes  2241504 hnd 0x00000000
[2022-08-15 10:06:36] Read fixup4.dat bytes     5411 hnd 0x00000000
[2022-08-15 10:06:36] Firmware: d9b293558b4cef6aabedcc53c178e7604de90788 Nov 18 2021 16:16:49
[2022-08-15 10:06:36] 0x00b03140 0x00000000 0x00000fff
[2022-08-15 10:06:36] MEM GPU: 76 ARM: 948 TOTAL: 1024
[2022-08-15 10:06:36] Starting start4.elf @ 0xfec00200 partition 0
[2022-08-15 10:06:36] +
[2022-08-15 10:06:36] 
[2022-08-15 10:06:38] 
[2022-08-15 10:06:38] U-Boot 2021.01+dfsg-3ubuntu0~20.04.4 (Sep 21 2021 - 15:55:38 +0000)
[2022-08-15 10:06:38] 
[2022-08-15 10:06:38] DRAM:  1.9 GiB
[2022-08-15 10:06:38] RPI: Board rev 0x14 outside known range
[2022-08-15 10:06:38] RPI Unknown model (0xb03140)
[2022-08-15 10:06:38] MMC:   mmcnr@7e300000: 1, emmc2@7e340000: 0
[2022-08-15 10:06:38] Loading Environment from FAT... *** Warning - bad CRC, using default environment
```


