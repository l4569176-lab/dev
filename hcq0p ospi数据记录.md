# hcq0p ospi 数据整理记录

# [3.1.3](http://os.hcfa.cn:3000/hcfa/hcq0p_rootfs/releases/tag/v3.1.3)

ospi的数据数据如下

```dts
spi@40440000 {
    #address-cells = <1>;
    #size-cells = <0>;
    memory-region = <&mm_ospi2>;
    dma-names = "rx";
    status = "okay";

    flash0: flash@0 {
        compatible = "jedec,spi-nor";
        reg = <0>;
        spi-rx-bus-width = <8>;
        spi-tx-bus-width = <8>;
        spi-max-frequency = <33250000>;
    };
};
```

接收波形如下

```shell
dd if=/dev/mtd0 of=file bs=1 count=1  # 奇数个
```