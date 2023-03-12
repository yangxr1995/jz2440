此文记录系统移植的错误

# uboot分区设置错误
常见报错：
uboot : CRC错误

原因:
uImage 大小 2.3MB
uboot的 mtdparts为：
```shell
OpenJTAG> printenv mtdparts
mtdparts=mtdparts=nandflash0:256k@0(bootloader),128k(params),2m(kernel),-(root)
```
导致读写kernel都错误
修改mtd
```shell
OpenJTAG> setenv mtdparts mtdparts=nandflash0:256k@0(bootloader),128k(params),3m(kernel),-(root)
```
重新烧写uImage 和 rootfs

# kernel启动 uncompressed error
常见报错:
kernel:uncompressed error

可能原因：
1. 镜像损坏
   重新编译
2. kernel不支持板子的CPU 
   修改支持的arch, 只选择一个SoC
```txt
System Type  --->
  ARM system type (Samsung S3C24XX SoCs)  --->
  SAMSUNG S3C24XX SoCs Support  --->
    [*] SAMSUNG S3C2440
```

# kernel报错 No filesystem could mount root, tried:  ext3 ext2 cramfs vfat msdos iso9660 romfs
常见报错
kernel : No filesystem could mount root, tried:  ext3 ext2 cramfs vfat msdos iso9660 romfs

原因：
内核不支持rootfs的文件系统


