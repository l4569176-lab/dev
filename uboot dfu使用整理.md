# uboot dfu使用整理

参考[uboot dfu](https://docs.u-boot.org/en/latest/usage/dfu.html#device-firmware-upgrade-dfu)

`uboo`t的`dfu`工具支持`pc`端通过`usb`传递数据给存储器，`ram`以及`otp`等存储设备。

- 设备端

```shell
> help dfu
dfu - Device Firmware Upgrade

Usage:
dfu <USB_controller> [<interface> <dev>] [list]
  - device firmware upgrade via <USB_controller>
    on device <dev>, attached to interface
    <interface>
    [list] - list available alt settings
dfu tftp [<interface> <dev>] [<addr>]
  - device firmware upgrade via TFTP
    on device <dev>, attached to interface
    <interface>
    [<addr>] - address where FIT image has been stored

###########################################################################
#
# 目前dfu tftp只支持XXXX.itb格式的文件。所以本文档不涉及该部分
# 使用方式如下：
# > dfu 0 mmc 0
#
###########################################################################
```

- PC端

```shell
$ dfu-util --help
dfu-util 0.11

Copyright 2005-2009 Weston Schmidt, Harald Welte and OpenMoko Inc.
Copyright 2010-2021 Tormod Volden and Stefan Schmidt
This program is Free Software and has ABSOLUTELY NO WARRANTY
Please report bugs to http://sourceforge.net/p/dfu-util/tickets/

You need to specify one of -D or -U
Usage: dfu-util [options] ...
  -h --help			Print this help message
  -V --version			Print the version number
  -v --verbose			Print verbose debug statements
  -l --list			List currently attached DFU capable devices
  -e --detach			Detach currently attached DFU capable devices
  -E --detach-delay seconds	Time to wait before reopening a device after detach
  -d --device <vendor>:<product>[,<vendor_dfu>:<product_dfu>]
				Specify Vendor/Product ID(s) of DFU device
  -n --devnum <dnum>		Match given device number (devnum from --list)
  -p --path <bus-port. ... .port>	Specify path to DFU device
  -c --cfg <config_nr>		Specify the Configuration of DFU device
  -i --intf <intf_nr>		Specify the DFU Interface number
  -S --serial <serial_string>[,<serial_string_dfu>]
				Specify Serial String of DFU device
  -a --alt <alt>		Specify the Altsetting of the DFU Interface
				by name or by number
  -t --transfer-size <size>	Specify the number of bytes per USB Transfer
  -U --upload <file>		Read firmware from device into <file>
  -Z --upload-size <bytes>	Specify the expected upload size in bytes
  -D --download <file>		Write firmware from <file> into device
  -R --reset			Issue USB Reset signalling once we're finished
  -w --wait			Wait for device to appear
  -s --dfuse-address address<:...>	ST DfuSe mode string, specifying target
				address for raw file download or upload (not
				applicable for DfuSe file (.dfu) downloads).
				Add more DfuSe options separated with ':'
		leave		Leave DFU mode (jump to application)
		mass-erase	Erase the whole device (requires "force")
		unprotect	Erase read protected device (requires "force")
		will-reset	Expect device to reset (e.g. option bytes write)
		force		You really know what you are doing!


#########################################################################
#
# dfu list可以查看映射到pc端的存储设备
# dfu -a 可以操作的存储设备，可以通过名字或者序列号指定
# dfu -U 读取存储设备数据
# dfu -D 下载存储设备数据
# 以下是一个例子
#
#########################################################################
$ dfu-util -l
dfu-util 0.11

Copyright 2005-2009 Weston Schmidt, Harald Welte and OpenMoko Inc.
Copyright 2010-2021 Tormod Volden and Stefan Schmidt
This program is Free Software and has ABSOLUTELY NO WARRANTY
Please report bugs to http://sourceforge.net/p/dfu-util/tickets/

Found DFU: [0483:df11] ver=0200, devnum=11, cfg=1, intf=0, path="1-10.1", alt=2, name="mmc_partiton", serial="00112233445"
Found DFU: [0483:df11] ver=0200, devnum=11, cfg=1, intf=0, path="1-10.1", alt=1, name="mmc0_rootfs", serial="00112233445"
Found DFU: [0483:df11] ver=0200, devnum=11, cfg=1, intf=0, path="1-10.1", alt=0, name="mmc0_boot", serial="00112233445"

$ dfu-util -a 2 -D boot.src.cmd 
dfu-util 0.11

Copyright 2005-2009 Weston Schmidt, Harald Welte and OpenMoko Inc.
Copyright 2010-2021 Tormod Volden and Stefan Schmidt
This program is Free Software and has ABSOLUTELY NO WARRANTY
Please report bugs to http://sourceforge.net/p/dfu-util/tickets/

dfu-util: Warning: Invalid DFU suffix signature
dfu-util: A valid DFU suffix will be required in a future dfu-util release
Opening DFU capable USB device...
Device ID 0483:df11
Device DFU version 0110
Claiming USB DFU Interface...
Setting Alternate Interface #2 ...
Determining device status...
DFU state(2) = dfuIDLE, status(0) = No error condition is present
DFU mode device DFU version 0110
Device returned transfer size 4096
Copying data from PC to DFU device
Download	[=========================] 100%          119 bytes
Download done.

```

## device端

`uboot`中使用`dfu`功能，设备`usb`接口作为`device`设备，主要有以下3个步骤

- 连接`usb`
- 获取`dfu alt`
- 启动`dfu`，等待`pc`端指令

### dfu alt

`dfu alt`在`dfu`中是最重要的一部分，他决定将哪些设备映射到`pc`端。

类似于

```shell
Found DFU: [0483:df11] ver=0200, devnum=11, cfg=1, intf=0, path="1-10.1", alt=2, name="mmc_partiton", serial="00112233445"
Found DFU: [0483:df11] ver=0200, devnum=11, cfg=1, intf=0, path="1-10.1", alt=1, name="mmc0_rootfs", serial="00112233445"
Found DFU: [0483:df11] ver=0200, devnum=11, cfg=1, intf=0, path="1-10.1", alt=0, name="mmc0_boot", serial="00112233445"

```

`dfu alt`可以通过环境变量设置，也可以通过函数直接设置

- 环境变量方式：设置`dfu_alt_info`，然后调用`dfu_init_env_entities`函数，即可为`dfu`设置`dfu alt`。
    - 类似`setenv dfu_alt_info "mmc0_boot part 0 1;mmc0_rootfs part 0 2;mmc_partiton script 0 3`。这里介绍只`mmc`设备，`raw`设备和`mmc`的`raw`模式一样，`mmc`设备支持以下几种类型
        - `part`: 分区模式
        - `raw`:原始地址+偏移地址模式，以`LBA`方式管理
        - `FAT / EXT2 / EXT3 / EXT4`：文件系统模式
        - `SKIP`：不操作
        - `SCRIPT`：脚本模式，明文方式，device读取到数据后，直接运行指令
    - 
- 函数调用：通过`dfu_alt_add`添加一组一组设备，
    - `dfu_alt_init` 初始化
    - `dfu_alt_add`添加
    - `dfu_free_entities`：释放

`dfu`指令采用环境变量的方式，当操作`SCRIPT`类型的时候，会重新读取一次环境变量，并重新启动一次`dfu`。

### 启动dfu

- `dfu 0`： 多个存储设备

```shell
> printenv dfu_alt_info 
dfu_alt_info=ram 0=uImage ram 0xc2000000 0x2000000;devicetree.dtb ram 0xc4000000 0x100000;uramdisk.image.gz ram 0xc4400000 0x10000000&mmc 0=mmc0_boot part 0 1;mmc0_rootfs part 0 2;mmc0_system-data part 0 3;mmc0_user part 0 4;mmc0_modules 
part 0 5;mmc0_ramdisk part 0 6&mmc 1=mmc1_boot1 raw 0x0 0x400000 mmcpart 1;mmc1_boot2 raw 0x0 0x400000 mmcpart 2;mmc1_metadata1 part 1 1;mmc1_metadata2 part 1 2;mmc1_fip-a part 1 3;mmc1_fip-b part 1 4;mmc1_bootfs1 part 1 5;mmc1_rootfs1 pa
rt 1 6;mmc1_bootfs2 part 1 7;mmc1_rootfs2 part 1 8;mmc1_rootfs_rw part 1 9;mmc1_rootfs_data part 1 10;mmc1_vendor part 1 11;mmc1_vendordata part 1 12;mmc1_userapp part 1 13;mmc1_userdata part 1 14;mmc1_odm part 1 15;mmc1_misc part 1 16;mm
c1_upgrade part 1 17&virt 0=OTP
```

- `dfu 0 mmc 0`：单个存储设备

```
> printenv dfu_alt_info 
dfu_alt_info=mmc0_boot part 0 1;mmc0_rootfs part 0 2;mmc_partiton script 0 3
```

如上，两种启动方式，环境变量的格式不一样。然后调用`run_usb_dnl_gadget(controller_index, "usb_dnl_dfu");`函数即可启动`dfu`功能

## PC端

采用`dfu-utils`指令对`device`存储设备进行操作



| 1   | 2   |
| --- | --- |
|     |     |