---
title: "Raspberry Pi 4B Secure Boot Enablement"
date: 2022-09-12
categories:
  - Technical
tags:
  - Secure Boot
  - RaspberryPi
---

RPi4上並沒有像CM4上有額外的Pin可以進到nRPIBOOT mode，這個如果沒有RaspberryPi原廠的support就沒有機會可以在Pi4上打開secure
boot了。好在[usbboot](https://github.com/raspberrypi/usbboot)有留一手，讓有心人可以找到pi4上進nRPIBOOT的方式。

在usbboot
[secure-boot-recovery](https://github.com/raspberrypi/usbboot/tree/master/secure-boot-recovery)這個目錄下有README可以參考。只不過描述的不是很詳細，這邊就在記錄在Pi4上進入nRPIBOOT的方式。

## Step1 - Erase EEPROM
```
$cd usbboot/secure-boot-recovery
$cp bootcode4.bin [SD-CARD-BOOT-PARTITION]/recovery.bin
# Added `erase_eeprom=1` to `config.txt`
# Then insert the SD card and boot the Pi and wait at least 10 seconds for the green LED to flash rapidly.
```
Once the EEPROM has been erased, remove `recovery.bin` and `erase_eeprom=1` from `config.txt` in the SD-CARD boot partition.

## Step2 - Set nRPIBOOT GPIO in the OTP
```
$cd usbboot/secure-boot-recovery
# Modified `config.txt` to specify the GPIO to use for nRPIBOOT.
# program_rpiboot_gpio=5
```

## Step3 - Program EEPROM
Plugged a USB-C cable with RPi4 to a Host PC(usbboot).
```
$cd usbboot/secure-boot-recovery
$sudo ../rpiboot -d .
```
如果都沒錯的話，在console上可以看到如下的logs
```
SIG pieeprom.sig 33ca8b2a63b1f91e5259d3232637b7a0c83ee1696545e94f22e92988f1ea3f47 1660530610
Reading EEPROM: 524288
Bootloader EEPROM is up to date
EEPROM WP: bc
EEPROM set WP 1
```

至此就算完成了。接下來就找個Jumper插在GPIO5，就可以進到nRPIBOOT mode了。
Signed image和更新EEPROM的方式和CM4一樣，可以參考這篇[
Raspberry Pi Secure Boot Enablement ](https://kuohsianglu.github.io/technical/rpi-sb-enablement/)

