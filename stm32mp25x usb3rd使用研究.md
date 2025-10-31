![stm32mp25x usb 框架](http://photo.xieyongbin.press:9000/system/lll/stm32mp25x_usb.png)

在[stm32mpu usb](https://wiki.st.com/stm32mpu/wiki/USB_overview#USB_gadget_monitoring_with_sysfs)中介绍，stm32mp25x系列支持

* USBH
* USB3RD
* UCPD

本次主要研究的是，使用USB3RD的typec接口的串行使用方式。

参考[stm32mpu usb3rd设备树配置](https://wiki.st.com/stm32mpu/wiki/USB3DR_device_tree_configuration#DT_configuration_example_as_USB2-speed_only_USB3DR_in_Peripheral_mode--with_micro-B-28ID_left_unconnected-29_or_Type-C_connector)，串行的配置如下

```shell
&usb3dr {
	status = "okay";	/* enable USB3DR */

	dwc3: usb@48300000 {
		maximum-speed = "high-speed";
		role-switch-default-mode = "peripheral";
		usb-role-switch;
	};
};
```

根据模型图，当usb3rd使用串行的时候， usb user driver使用usb gadget，usb gadget配合libcomposite驱动，可以将typec映射成各种设备。

1. `ACM function`: 串口线
2. `ECM function`: 网络设备
3. `ECM subset function`：网络设备
4. `EEM function`：网络设备
5. `FFS function`：文件系统
6. `HID function`：keyboard...
7. `LOOPBACK function`:
8. `MASS STORAGE function`: 存储器
9. `MIDI function`: 声卡
10. `NCM function`： 网络设备
11. `OBEX function`
12. `PHONET function`
13. `RNDIS function`：网络
14. `SERIAL function`：串口
15. `SOURCESINK function`
16. `UAC1 function (legacy implementation)`： 音频
17. `UAC2 function`：音频
18. `UVC function`: 摄像头
19. `PRINTER function`
20. `UAC1 function (new API)`
21. `MIDI2 function`：声卡

## usb gadget使用

```shell
configfs="/sys/kernel/config/usb_gadget"
g=g1
c=c.1
d="${configfs}/${g}"

VENDOR_ID="0x1d6b"
PRODUCT_ID="0x0104"

do_start() {
    if [ ! -d ${configfs} ]; then
        modprobe libcomposite
        if [ ! -d ${configfs} ]; then
        exit 1
        fi
    fi

    if [ -d ${d} ]; then
        exit 0
    fi

    udc=$(ls -1 /sys/class/udc/)
    if [ -z $udc ]; then
        echo "No UDC driver registered"
        exit 1
    fi

    mkdir "${d}"
    echo ${VENDOR_ID} > "${d}/idVendor"
    echo ${PRODUCT_ID} > "${d}/idProduct"
    echo 0x0200 > "${d}/bcdUSB"
    # Windows extension to use IAD (Interface Association Descriptor)
    # https://learn.microsoft.com/en-us/windows-hardware/drivers/usbcon/usb-interface-association-descriptor
    echo "0xEF" > "${d}/bDeviceClass"
    echo "0x02" > "${d}/bDeviceSubClass"
    echo "0x01" > "${d}/bDeviceProtocol"
    echo "0x0100" > "${d}/bcdDevice"

    mkdir -p "${d}/strings/0x409"
    tr -d '\0' < /proc/device-tree/serial-number > "${d}/strings/0x409/serialnumber"
    echo "STMicroelectronics" > "${d}/strings/0x409/manufacturer"
    echo "STM32MP1" > "${d}/strings/0x409/product"

    # Config
    mkdir -p "${d}/configs/${c}"
    mkdir -p "${d}/configs/${c}/strings/0x409"
    echo "Config 1: NCM" > "${d}/configs/${c}/strings/0x409/configuration"

    if $(cat /proc/device-tree/compatible | grep -q "stm32mp215f-dk") ; then
        echo 500 > "${d}/configs/${c}/MaxPower"
        echo 0x80 > "${d}/configs/${c}/bmAttributes" # Bus powered device
    else
        echo 0 > "${d}/configs/${c}/MaxPower"
        echo 0xC0 > "${d}/configs/${c}/bmAttributes" # self powered device
    fi

    # Enable use of OS descriptor (for windows to bind drivers like NCM, RNDIS...
    # without additional .inf file)
    mkdir -p "${d}/os_desc"
    echo "1" > "${d}/os_desc/use"
    echo "0xbc" > "${d}/os_desc/b_vendor_code"
    echo "MSFT100" > "${d}/os_desc/qw_sign"
}

do_stop() {
    interfacename=$(cat ${d}/functions/${func_eth}/ifname 2> /dev/null)
    if [ -z "${interfacename}" ];
    then
        echo "Nothing to do"
        return
    fi
    ifconfig ${interfacename} down

    sleep 0.2

    echo "" > "${d}/UDC"

    rm -f "${d}/os_desc/${c}"
    [ -d "${d}/configs/${c}/${func_eth}" ] &&rm -f "${d}/configs/${c}/${func_eth}"

    [ -d "${d}/strings/0x409/" ] && rmdir "${d}/strings/0x409/"
    [ -d "${d}/configs/${c}/strings/0x409" ] && rmdir "${d}/configs/${c}/strings/0x409"
    [ -d "${d}/configs/${c}" ] && rmdir "${d}/configs/${c}"
}
```

运行以上代码可以使用usb gadget接口。以下使用网口的方式来讲解如何映射一个设备节点

```shell
    ####################开启##################################
    #
    # functions目录即是选择映射对应的设备。（必须）
    #
    mkdir -p "${d}/functions/${func_eth}" 

    #
    # 不同的设备，会有不同的文件，根据需要编辑对应的文件（可选）
    #
    mkdir -p "${d}/functions/${func_eth}/os_desc/interface.ncm"
    echo "WINNCM" > "${d}/functions/${func_eth}/os_desc/interface.ncm/compatible_id"

    if [ "$MAC_HOST_CUST" != "" ]; then
        echo $MAC_HOST_CUST > "${d}/functions/${func_eth}/host_addr"
    else
        mac_host=$(get_mac_address_from_serial_number)
        echo $mac_host > "${d}/functions/${func_eth}/host_addr"
    fi
    if [ "$MAC_DEV_CUST" != "" ]; then
        echo $MAC_DEV_CUST > "${d}/functions/${func_eth}/dev_addr"
    fi

    #
    # 产生对应的软链接（必须）
    #
    # Set up the rndis device only first
    ln -s "${d}/functions/${func_eth}" "${d}/configs/${c}"
    ln -s "${d}/configs/${c}" "${d}/os_desc"
    #
    # 使能usb gadget，host可识别到设备（必须）
    #
    echo "${udc}" > "${d}/UDC"

    ####################关闭##################################
    #
    # 关闭usb gadget，host无法读取到设备（必须）
    #
    echo "" > "${d}/UDC"
    #
    # 关闭对应的功能(必须)
    #
    rm -f "${d}/os_desc/c.1"
    rm -rf "${d}/configs/c.1/${func_eth}"
    rmdir "${d}/functions/${func_eth}"

```

以下是对应的设备使用例子

### NCM网口设备

```shell
    mkdir -p "${d}/functions/ncm.0"    #ncm.X,产生对应的网口配置接口，自动加载usb_f_ncm.ko模块
  
    mkdir -p "${d}/functions/${func_eth}/os_desc/interface.ncm"
    echo "WINNCM" > "${d}/functions/${func_eth}/os_desc/interface.ncm/compatible_id"

    if [ "$MAC_HOST_CUST" != "" ]; then
        echo $MAC_HOST_CUST > "${d}/functions/${func_eth}/host_addr"
    else
        mac_host=$(get_mac_address_from_serial_number)
        echo $mac_host > "${d}/functions/${func_eth}/host_addr"
    fi
    if [ "$MAC_DEV_CUST" != "" ]; then
        echo $MAC_DEV_CUST > "${d}/functions/${func_eth}/dev_addr"
    fi


    # Set up the rndis device only first
    ln -s "${d}/functions/${func_eth}" "${d}/configs/${c}"
    ln -s "${d}/configs/${c}" "${d}/os_desc"

    echo "${udc}" > "${d}/UDC"
    # 	ifname		network device interface name associated with this
    #    		function instance
    #	qmult		queue length multiplier for high and super speed
    #	host_addr	MAC address of host's end of this
    #			Ethernet over USB link
    # 	dev_addr	MAC address of device's end of this
    #			Ethernet over USB link

```

运行结果如下

```shell
##device
$ ip a show
11: usb0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state
UP group default qlen 1000
link/ether be:26:e8:11:a4:3a brd ff:ff:ff:ff:ff:ff
inet6 fe80::bc26:e8ff:fe11:a43a/64 scope link
valid_lft forever preferred_lft forever
## host
$ ip a show
7: enxdc2b4f5dafae: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc
fq_codel state UP group default qlen 1000
link/ether dc:2b:4f:5d:af:ae brd ff:ff:ff:ff:ff:ffd
$ dmesg
[82225.955586] usb 1-10.2.3: new high-speed USB device number 13 using
xhci_hcd
#插入typec线
[82226.033067] usb 1-10.2.3: New USB device found, idVendor=1d6b,
idProduct=0104, bcdDevice= 1.00
[82226.033081] usb 1-10.2.3: New USB device strings: Mfr=1, Product=2,
SerialNumber=3
[82226.033086] usb 1-10.2.3: Product: STM32MP1
[82226.033090] usb 1-10.2.3: Manufacturer: STMicroelectronics
[82226.033094] usb 1-10.2.3: SerialNumber: 00112233445
[82226.101715] usbcore: registered new interface driver cdc_ether
[82226.129289] cdc_ncm 1-10.2.3:1.0: MAC-Address: dc:2b:4f:5d:af:ae
[82226.129790] cdc_ncm 1-10.2.3:1.0 eth0: register 'cdc_ncm' at usb-
0000:00:14.0-10.2.3, CDC NCM (NO ZLP), dc:2b:4f:5d:af:ae
[82226.129933] usbcore: registered new interface driver cdc_ncm
[82226.135338] usbcore: registered new interface driver cdc_wdm
[82226.137454] usbcore: registered new interface driver cdc_mbim
[82226.140955] cdc_ncm 1-10.2.3:1.0 enxdc2b4f5dafae: renamed from eth0
[82631.811765] audit: type=1400 audit(1757662377.006:138): apparmor="DENIED"
operation="capable" class="cap" profile="tcpdump" pid=67760 comm="tcpdump"
capability=16
capname="sys_module"
[82636.808181] r8169 0000:04:00.0 enp4s0: entered promiscuous mode
[82650.555637] r8169 0000:04:00.0 enp4s0: left promiscuous mode
[82655.35
```

### mass storage存储设备接口

```shell
    mkdir -p "${d}/functions/mass_storage.0"    # mass_storage.X,产生对应的网口配置接口，自动加载usb_f_mass_storage.ko模块
    
    echo /dev/mmcblk0 > "${d}/functions/mass_storage.0/lun.0/file"    # 映射对应的设备
    # Set up the rndis device only first
    ln -s "${d}/functions/${func_eth}" "${d}/configs/${c}"
    ln -s "${d}/configs/${c}" "${d}/os_desc"

    echo "${udc}" > "${d}/UDC"
    # 		file	The path to the backing file for the LUN.
	#		    Required if LUN is not marked as removable.
	#       ro		Flag specifying access to the LUN shall be
    #			read-only. This is implied if CD-ROM emulation
    #			is enabled as well as when it was impossible
    #    		to open "filename" in R/W mode.
	#       removable	Flag specifying that LUN shall be indicated as
	#    		being removable.
	#       cdrom		Flag specifying that LUN shall be reported as
	#    		being a CD-ROM.
	#       nofua		Flag specifying that FUA flag
	#    		in SCSI WRITE(10,12)
	#       forced_eject	This write-only file is useful only when
	#		    the function is active. It causes the backing
	#		    file to be forcibly detached from the LUN,
	#		    regardless of whether the host has allowed it.
	#		    Any non-zero number of bytes written will
	#		    result in ejection.

```

运行结果去如下

```shell
## device
$ dmesg
[  348.269376] Mass Storage Function, version: 2009/09/11
[  348.269393] LUN: removable file: (no medium)

## host
$ dmesg
236884.307814] usb 1-10.2.3: new high-speed USB device number 11 using xhci_hcd
[236884.386347] usb 1-10.2.3: New USB device found, idVendor=1d6b, idProduct=0104, bcdDevice= 1.00
[236884.386359] usb 1-10.2.3: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[236884.386364] usb 1-10.2.3: Product: STM32MP1
[236884.386367] usb 1-10.2.3: Manufacturer: STMicroelectronics
[236884.386370] usb 1-10.2.3: SerialNumber: 00112233445
[236884.391198] usb-storage 1-10.2.3:1.0: USB Mass Storage device detected
[236884.391591] scsi host6: usb-storage 1-10.2.3:1.0
[236885.423553] scsi 6:0:0:0: Direct-Access     Linux    File-Stor Gadget 0606 PQ: 0 ANSI: 2
[236885.424245] sd 6:0:0:0: Attached scsi generic sg1 type 0
[236885.424601] sd 6:0:0:0: Power-on or device reset occurred
[236885.425044] sd 6:0:0:0: [sdb] 15269888 512-byte logical blocks: (7.82 GB/7.28 GiB)
[236885.425251] sd 6:0:0:0: [sdb] Write Protect is off
[236885.425262] sd 6:0:0:0: [sdb] Mode Sense: 0f 00 00 00
[236885.425407] sd 6:0:0:0: [sdb] Write cache: enabled, read cache: enabled, doesn't support DPO or FUA
[236885.430201]  sdb: sdb1 sdb2 sdb3 sdb4 sdb5 sdb6 sdb7 sdb8 sdb9 sdb10 sdb11 sdb12 sdb13 sdb14 sdb15 sdb16 sdb17
[236885.436151] sd 6:0:0:0: [sdb] Attached SCSI removable disk
[236885.792908] EXT4-fs (sdb5): mounted filesystem 4b626fea-c4b8-4edf-bbbb-2edd03f89f1e r/w with ordered data mode. Quota mode: none.
[236885.792916] ext4 filesystem being mounted at /media/lll/bootfs supports timestamps until 2038-01-19 (0x7fffffff)
[236886.075456] erofs: (device sdb6): mounted with root inode @ nid 36.
[236886.090703] EXT4-fs (sdb11): mounted filesystem ad0144bf-7606-4db4-b4c7-06c9b9d18dec r/w with ordered data mode. Quota mode: none.
[236886.090714] ext4 filesystem being mounted at /media/lll/vendorfs supports timestamps until 2038-01-19 (0x7fffffff)
[236886.095883] EXT4-fs (sdb9): recovery complete
[236886.095885] EXT4-fs (sdb7): mounted filesystem 665e00e2-1b86-4581-86e6-f40b76b48e3f r/w with ordered data mode. Quota mode: none.
[236886.095923] EXT4-fs (sdb9): mounted filesystem d492e3d4-c6bc-4aed-9c65-754a7322066a r/w with ordered data mode. Quota mode: none.
[236886.103097] EXT4-fs (sdb14): recovery complete
[236886.103130] EXT4-fs (sdb14): mounted filesystem b8f7f067-e78d-4e00-ae88-7a4c8bd04f2a r/w with ordered data mode. Quota mode: none.
[236886.166215] ext4 filesystem being mounted at /media/lll/bootfs1 supports timestamps until 2038-01-19 (0x7fffffff)
[236886.478153] F2FS-fs (sdb10): Mounted with checkpoint version = 3624ac8a
[236886.605443] EXT4-fs (sdb16): recovery complete
[236886.605482] EXT4-fs (sdb16): mounted filesystem 4bb2c799-d4fa-48a7-aef0-d6791c32d638 r/w with ordered data mode. Quota mode: none.
[236886.613817] erofs: (device sdb8): mounted with root inode @ nid 36.
[236886.639702] EXT4-fs (sdb15): mounted filesystem a8a09221-a056-4d98-92f5-85a187a14a49 r/w with ordered data mode. Quota mode: none.
[236886.644730] EXT4-fs (sdb13): recovery complete
[236886.647114] EXT4-fs (sdb13): mounted filesystem c24f4dd6-507e-4795-8834-c7772b6994b5 r/w with ordered data mode. Quota mode: none.
[236886.670825] EXT4-fs (sdb12): recovery complete
[236886.673140] EXT4-fs (sdb17): recovery complete
[236886.675640] EXT4-fs (sdb17): mounted filesystem d580543d-49fd-4ed1-bd70-c0b000967716 r/w with ordered data mode. Quota mode: none.
[236886.676474] EXT4-fs (sdb12): mounted filesystem 24480d36-5fad-4848-8fbd-3b2ca4d988ef r/w with ordered data mode. Quota mode: none.

$ mount
/dev/sdb5 on /media/lll/bootfs type ext4 (rw,nosuid,nodev,relatime,errors=remount-ro,uhelper=udisks2)
/dev/sdb6 on /media/lll/367dbeda-72fd-4862-9856-41bef2e27b92 type erofs (ro,nosuid,nodev,relatime,user_xattr,acl,cache_strategy=readaround,uhelper=udisks2)
/dev/sdb11 on /media/lll/vendorfs type ext4 (rw,nosuid,nodev,relatime,errors=remount-ro,uhelper=udisks2)
/dev/sdb9 on /media/lll/rootfs1 type ext4 (rw,nosuid,nodev,relatime,errors=remount-ro,uhelper=udisks2)
/dev/sdb14 on /media/lll/b8f7f067-e78d-4e00-ae88-7a4c8bd04f2a type ext4 (rw,nosuid,nodev,relatime,errors=remount-ro,uhelper=udisks2)
/dev/sdb7 on /media/lll/bootfs1 type ext4 (rw,nosuid,nodev,relatime,errors=remount-ro,uhelper=udisks2)
/dev/sdb10 on /media/lll/rootfs_data type f2fs (rw,nosuid,nodev,relatime,lazytime,background_gc=on,nogc_merge,nodiscard,user_xattr,inline_xattr,acl,inline_data,inline_dentry,flush_merge,barrier,extent_cache,mode=adaptive,active_logs=6,alloc_mode=reuse,checkpoint_merge,fsync_mode=posix,memory=normal,errors=continue,uhelper=udisks2)
/dev/sdb16 on /media/lll/4bb2c799-d4fa-48a7-aef0-d6791c32d638 type ext4 (rw,nosuid,nodev,relatime,errors=remount-ro,uhelper=udisks2)
/dev/sdb8 on /media/lll/3e62423e-9797-4e89-92fe-fe838ab37a42 type erofs (ro,nosuid,nodev,relatime,user_xattr,acl,cache_strategy=readaround,uhelper=udisks2)
/dev/sdb15 on /media/lll/a8a09221-a056-4d98-92f5-85a187a14a49 type ext4 (rw,nosuid,nodev,relatime,errors=remount-ro,uhelper=udisks2)
/dev/sdb13 on /media/lll/c24f4dd6-507e-4795-8834-c7772b6994b5 type ext4 (rw,nosuid,nodev,relatime,errors=remount-ro,uhelper=udisks2)
/dev/sdb17 on /media/lll/d580543d-49fd-4ed1-bd70-c0b000967716 type ext4 (rw,nosuid,nodev,relatime,errors=remount-ro,uhelper=udisks2)
/dev/sdb12 on /media/lll/24480d36-5fad-4848-8fbd-3b2ca4d988ef type ext4 (rw,nosuid,nodev,relatime,errors=remount-ro,uhelper=udisks2)
```

### ACM串口接口

```shell
    mkdir -p "${d}/functions/acm.0"    # acm.X,产生对应的网口配置接口，自动加载usb_f_acm.ko和usb_f_serial.ko模块
    
    # Set up the rndis device only first
    ln -s "${d}/functions/acm.0" "${d}/configs/${c}"
    ln -s "${d}/configs/${c}" "${d}/os_desc"

    echo "${udc}" > "${d}/UDC"
    # 		file	The path to the backing file for the LUN.
	#		    Required if LUN is not marked as removable.
	#       ro		Flag specifying access to the LUN shall be
    #			read-only. This is implied if CD-ROM emulation
    #			is enabled as well as when it was impossible
    #    		to open "filename" in R/W mode.
	#       removable	Flag specifying that LUN shall be indicated as
	#    		being removable.
	#       cdrom		Flag specifying that LUN shall be reported as
	#    		being a CD-ROM.
	#       nofua		Flag specifying that FUA flag
	#    		in SCSI WRITE(10,12)
	#       forced_eject	This write-only file is useful only when
	#		    the function is active. It causes the backing
	#		    file to be forcibly detached from the LUN,
	#		    regardless of whether the host has allowed it.
	#		    Any non-zero number of bytes written will
	#		    result in ejection.

```

运行结果如下

```shell
## device
$ ls /dev/ttyGS* 
/dev/ttyGS1
## host
$ dmesg
[248140.884503] usb 1-10.2.3: new high-speed USB device number 14 using xhci_hcd
[248140.962790] usb 1-10.2.3: New USB device found, idVendor=1d6b, idProduct=0104, bcdDevice= 1.00
[248140.962805] usb 1-10.2.3: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[248140.962810] usb 1-10.2.3: Product: STM32MP1
[248140.962814] usb 1-10.2.3: Manufacturer: STMicroelectronics
[248140.962818] usb 1-10.2.3: SerialNumber: 00112233445
[248140.966140] cdc_acm 1-10.2.3:1.0: ttyACM2: USB ACM device

$ ls /dev/ttyACM*
/dev/ttyACM0  /dev/ttyACM1  /dev/ttyACM2
```