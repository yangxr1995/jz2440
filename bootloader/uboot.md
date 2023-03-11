# uboot 的基本功能

uboot最终：启动kernel

* 初始化时钟（一开始时钟频率太低）
* 初始化串口
* 从flash读出内核，放到SDRAM
* 启动内核

额外功能，方便调试：

* 写flash功能
* 网卡
* usb

# uboot1.1.6 分析

## Makefile分析

### make xxx\_config

* 创建几个软链接，以选择C头文件包含路径
* 创建include/config.mk
* 创建include/config.h，包含 include/configs/100ask24x0.h
  
```shell
  cd ./include
  rm -f asm
  ln -s asm-arm asm
  rm -f asm-arm/arch
  ln -s arch-s3c24x0 asm-arm/arch
  rm -f asm-arm/proc
  ln -s proc-armv asm-arm/proc
#
# Create include file for Make
#
echo "ARCH   = $2" >  config.mk
echo "CPU    = $3" >> config.mk
echo "BOARD  = $4" >> config.mk
echo "SOC    = s3c24x0" >> config.mk

> config.h      # Create new config file
> echo "/* Automatically generated - do not edit */" >>config.h
> echo "#include <configs/100ask24x0.h>" >>config.h
```

### make

u-boot.bin 二进制文件
u-boot elf文件
```shell
u-boot.bin: u-boot
    $(OBJCOPY) $(OBJCFLAGS) -O binary $< $@
```

链接生成 u-boot。
链接脚本 board/100ask24x0/u-boot.lds 
链接地址 0x33F80000

```shell
$(obj)u-boot:       depend version $(SUBDIRS) $(OBJS) $(LIBS) $(LDSCRIPT)
arm-linux-ld -Bstatic -T board/100ask24x0/u-boot.lds \
            -Ttext 0x33F80000 \
            cpu/arm920t/start.o lib_generic/libgeneric.a  ... \
            -o u-boot
```

.a 文件的生成
使用每个子模块的Makefile进行构造，
子模块的Makefile定义的链接，
顶层目录的config.mk定义了编译

```shell
$(LIBS):
    $(MAKE) -C $(dir $(subst $(obj),,$@))

-->
net/libnet.a disk/libdisk.a:
    $(MAKE) -C $(dir $(subst $(obj),,$@))

-->
make -C net
make -C disk

--> 比如net/Makefile
include $(TOPDIR)/config.mk @ 这里包含了编译规则

LIB = $(obj)libnet.a

COBJS   = net.o tftp.o bootp.o rarp.o eth.o nfs.o sntp.o

SRCS    := $(COBJS:.o=.c)
OBJS    := $(addprefix $(obj),$(COBJS))

all:    $(LIB)

$(LIB): $(obj).depend $(OBJS)
    $(AR) $(ARFLAGS) $@ $(OBJS)

--> $(TOPDIR)/config.mk
ifndef REMOTE_BUILD

%.s:    %.S
    $(CPP) $(AFLAGS) -o $@ $<
%.o:    %.S
    $(CC) $(AFLAGS) -c -o $@ $<
%.o:    %.c
    $(CC) $(CFLAGS) -c -o $@ $<

else

$(obj)%.s:  %.S
    $(CPP) $(AFLAGS) -o $@ $<
$(obj)%.o:  %.S
    $(CC) $(AFLAGS) -c -o $@ $<
$(obj)%.o:  %.c
    $(CC) $(CFLAGS) -c -o $@ $<
endif
```

### 分析链接脚本

```shell
arm-linux-ld -Bstatic -T board/100ask24x0/u-boot.lds \
            -Ttext 0x33F80000 \
```

```shell
OUTPUT_FORMAT("elf32-littlearm", "elf32-littlearm", "elf32-littlearm")
/*OUTPUT_FORMAT("elf32-arm", "elf32-arm", "elf32-arm")*/
OUTPUT_ARCH(arm)
ENTRY(_start)
SECTIONS
{
    . = 0x00000000;   @ 由于使用了 -Ttext 所以最终的运行地址为 0x33F80000

    . = ALIGN(4);
    .text      :
    {
      cpu/arm920t/start.o   (.text)       @ 入口为 start.o
          board/100ask24x0/boot_init.o (.text)
      *(.text)
    }

    . = ALIGN(4);
    .rodata : { *(.rodata) }

    . = ALIGN(4);
    .data : { *(.data) }

    . = ALIGN(4);
    .got : { *(.got) }

    . = .;
    __u_boot_cmd_start = .;          @ 自定义段，分析命令时有用
    .u_boot_cmd : { *(.u_boot_cmd) }
    __u_boot_cmd_end = .;

    . = ALIGN(4);
    __bss_start = .;
    .bss : { *(.bss) }
    _end = .;
}
```

### 总结

分析完Makefile得到如下信息:

* 入口文件为 start.o ，入口函数为 \_start
* 运行地址为 0x33F80000

## uboot启动
### start.S
* 硬件初始化：设为管理模式，关闭看门狗，设置时钟，关闭MMU
* 重定位bootloader到SDRAM：初始化芯片和重定位，设置栈
* 跳到 start\_armboot

### 从初始化到main\_loop
定义并初始化 gd\_t gd
```c
gd = (gd_t*)(_armboot_start - CFG_MALLOC_LEN - sizeof(gd_t)); 
gd->bd = (bd_t*)((char*)gd - sizeof(bd_t));
```

初始化完成后应该打印ok 和 banner
```c
  init_fnc_t *init_sequence[] = {
      cpu_init,       /* basic cpu dependent setup */
      board_init,     /* basic board dependent setup */
      interrupt_init,     /* set up exceptions */
      env_init,       /* initialize environment */
      init_baudrate,      /* initialze baudrate settings */
      serial_init,        /* serial communications setup */
      console_init_f,     /* stage 1 init of console */
      display_banner,     /* say that we are here */
  #if defined(CONFIG_DISPLAY_CPUINFO)
      print_cpuinfo,      /* display cpu info (and speed) */
  #endif
  #if defined(CONFIG_DISPLAY_BOARDINFO)
      checkboard,     /* display board info */
  #endif
      dram_init,      /* configure available RAM banks */
      display_dram_config,
      NULL,
  };


for (init_fnc_ptr = init_sequence; *init_fnc_ptr; ++init_fnc_ptr) {
  if ((*init_fnc_ptr)() != 0) {
      hang ();
  }
}

```

准备堆内存
```c
mem_malloc_init (_armboot_start - CFG_MALLOC_LEN);
```

构造环境变量
```c
env_relocate
   env_ptr = (env_t *)malloc (CFG_ENV_SIZE); // 分配内存给环境变量
   env_get_char = env_get_char_memory;       // 变量环境变量的函数
   if (gd->env_valid == 0) {                 // 如果以前没有设置过环境变量，则用默认的
      memset (env_ptr, 0, sizeof(env_t));
      memcpy (env_ptr->data,
              default_environment,
              sizeof(default_environment));
   }
   else {
      env_relocate_spec ();              // 否则从flash中读取环境变量
   }
   gd->env_addr = (ulong)&(env_ptr->data);
```

main\_loop
```c
main_loop
   s = getenv ("bootcmd");
   if (bootdelay >= 0 && s && !abortboot (bootdelay)) { // 如果使用了bootdelay，并且有bootcmd环境变量，并且 abortboot 期间没有用户输入
      printf("Booting Linux ...\n");
      run_command (s, 0);     // 执行 bootcmd
   }
   for (;;) {  // 读取输入，运行命令
      len = readline (CFG_PROMPT);
      rc = run_command (lastcommand, flag);
   }
```
### uboot的命令实现
```c
int run_command (const char *cmd, int flag)
   char *str = cmdbuf;  // 命令行输入，比如 "bootcmd 0x1111 - 0x222"
   strcpy (cmdbuf, cmd);
   while (*str) {
      ... // 解析str，得到token
      argc = parse_line (finaltoken, argv);
      cmdtp = find_cmd(argv[0]));
      (cmdtp->cmd) (cmdtp, flag, argc, argv);
   }
```

find\_cmd 会遍历 \_\_u\_boot\_cmd\_start 开始的数组，根据命令名称找到命令
```c
struct cmd_tbl_s {
    char        *name;      /* Command Name         */
    int     maxargs;    /* maximum number of arguments  */
    int     (*cmd)(struct cmd_tbl_s *, int, int, char *[]);
    char        *usage;     /* Usage message    (short) */
    char        *help;      /* Help  message    (long)  */
    ...
};

typedef struct cmd_tbl_s    cmd_tbl_t;

cmd_tbl_t *find_cmd (const char *cmd)
{
   cmd_tbl_t *cmdtp;
   cmd_tbl_t *cmdtp_temp = &__u_boot_cmd_start;    /*Init value */
   const char *p;
   int len;
   int n_found = 0;
   
   for (cmdtp = &__u_boot_cmd_start;
      cmdtp != &__u_boot_cmd_end;
      cmdtp++) {
     if (strncmp (cmd, cmdtp->name, len) == 0) {
         if (len == strlen (cmdtp->name))
   	  return cmdtp;   /* full match */
   
         cmdtp_temp = cmdtp; /* abbreviated command ? */
         n_found++;
     }
   }
   if (n_found == 1) {         /* exactly one match */
     return cmdtp_temp;
   }
   
   return NULL;    /* not found or ambiguous command */
}
```

可见uboot将命令定义到特定段
```lds
    __u_boot_cmd_start = .;          @ 自定义段，分析命令时有用
    .u_boot_cmd : { *(.u_boot_cmd) }
    __u_boot_cmd_end = .;
```

```c
#define Struct_Section  __attribute__ ((unused,section (".u_boot_cmd")))

#define U_BOOT_CMD(name,maxargs,rep,cmd,usage,help) \
cmd_tbl_t __u_boot_cmd_##name Struct_Section = {#name, maxargs, rep, cmd, usage, help}

```

#### 命令的定义
对于bootm的定义
```c
  U_BOOT_CMD(
      bootm,  CFG_MAXARGS,    1,  do_bootm,
      "bootm   - boot application image from memory\n",
      "[addr [arg ...]]\n    - boot application image stored in memory\n"
      "\tpassing arguments 'arg ...'; when booting a Linux kernel,\n"
      "\t'arg' can be the address of an initrd image\n"
  #ifdef CONFIG_OF_FLAT_TREE
      "\tWhen booting a Linux kernel which requires a flat device-tree\n"
      "\ta third argument is required which is the address of the of the\n"
      "\tdevice-tree blob. To boot that kernel without an initrd image,\n"
      "\tuse a '-' for the second argument. If you do not pass a third\n"
      "\ta bd_info struct will be passed instead\n"
  #endif
  );

// 相当于定义了
cmd_tbl_t __u_boot_cmd_bootm __attribute__ ((unused,section (".u_boot_cmd"))) = {
	.name = "bootm",
	.maxargs = CFG_MAXARGS,
	.repeat = 1,
	.cmd = do_bootm,
	...
};

```
#### 添加一个命令
实现了一个hello命令测试
添加文件 common/hello.c
```c
#include <common.h>
#include <watchdog.h>
#include <command.h>
#include <image.h>
#include <malloc.h>
#include <zlib.h>
#include <bzlib.h>
#include <environment.h>
#include <asm/byteorder.h>


int do_hello (cmd_tbl_t *cmdtp, int flag, int argc, char *argv[])
{
        int i;
        printf("hello\n");
        for (i = 0; i < argc; i++) {
                printf("argv[%d] : %s\n", i, argv[i]);
        }
        printf("\n");
        return 0;
}

U_BOOT_CMD(
        hello,  CFG_MAXARGS,    1,      do_hello,
        "hello for test\n"
        "[[arg ...]]\n",
);
```
修改common/Makefile
```c
COBJS += hello.o
```

### do\_bootm
#### 对头信息的检查和kernel重定位
do\_bootm 对uImage头部信息进行校验，并重定位kernel到加载地址
```c
do_bootm (cmd_tbl_t *cmdtp, int flag, int argc, char *argv[])
   image_header_t *hdr = &header;
   addr = simple_strtoul(argv[1], NULL, 16);
   printf ("## Booting image at %08lx ...\n", addr);
   memmove (&header, (char *)addr, sizeof(image_header_t));
   // 检查头部包括
   if (ntohl(hdr->ih_magic) != IH_MAGIC) {
     puts ("Bad Magic Number\n");
     return 1;
   }
   if (crc32 (0, (uchar *)data, len) != checksum) {
     puts ("Bad Header Checksum\n");
     SHOW_BOOT_PROGRESS (-2);
     return 1;
   }
   print_image_hdr ((image_header_t *)addr); // 打印头部信息，包含了Image Name, Created 时间...
   if (verify) {
     puts ("   Verifying Checksum ... ");
     if (crc32 (0, (uchar *)data, len) != ntohl(hdr->ih_dcrc)) {
         printf ("Bad Data CRC\n");
         SHOW_BOOT_PROGRESS (-3);
         return 1;
     }
     puts ("OK\n");
   }
   if (hdr->ih_arch != IH_CPU_ARM) {
     printf ("Unsupported Architecture 0x%x\n", hdr->ih_arch);
     SHOW_BOOT_PROGRESS (-4);
     return 1;
   }

          name = "Kernel Image";

   // 重定位kernel，将kernel复制到加载地址处
   switch (hdr->ih_comp) {
   case IH_COMP_NONE:
     if(ntohl(hdr->ih_load) == data) {   // 如果data(kernel)已经在加载地址处，则不重定位
         printf ("   XIP %s ... ", name);
     } else {
         // 否则进行重定位到加载地址
         memmove ((void *) ntohl(hdr->ih_load), (uchar *)data, len);
     }
     break;
     printf ("   Uncompressing %s ... ", name);
     // 使用了压缩则解压后复制
     if (gunzip ((void *)ntohl(hdr->ih_load), unc_len,
   	  (uchar *)data, &len) != 0) {
         puts ("GUNZIP ERROR - must RESET board to recover\n");
         SHOW_BOOT_PROGRESS (-6);
         do_reset (cmdtp, flag, argc, argv);
     }
     break;
     ...
   }
   puts ("OK\n");

   do_bootm_linux  (cmdtp, flag, argc, argv, addr, len_ptr, verify);
      char *commandline = getenv ("bootargs");
      theKernel = (void (*)(int, int, uint))ntohl(hdr->ih_ep); // kernel的入口地址
      if (argc >= 3) {  // 如果使用 bootm kernel_load_addr initrd dtbs
          addr = simple_strtoul (argv[2], NULL, 16);  // 得到inird地址
          printf ("## Loading Ramdisk Image at %08lx ...\n", addr);
          memcpy (&header, (char *) addr, sizeof (image_header_t));
          ... // 检查initrd image                  
      } 
      // 设置ATAG
      setup_start_tag (bd);
      setup_serial_tag (&params);
      setup_memory_tags (bd);
      setup_commandline_tag (bd, commandline);
      setup_initrd_tag (bd, initrd_start, initrd_end);
      setup_end_tag (bd);
      // 启动kernel      
      printf ("\nStarting kernel ...\n\n");
      theKernel (0, bd->bi_arch_number, bd->bi_boot_params); // 给内核传参 r0 : 0, r1 : 机器码， r3 : ATAGS地址
```
#### ATAGS 的处理
就是定义类似链表的数组，地址位 bd\-\>bi\_boot\_params
```c
  static void setup_start_tag (bd_t *bd)
  {
      params = (struct tag *) bd->bi_boot_params;

      params->hdr.tag = ATAG_CORE;
      params->hdr.size = tag_size (tag_core);

      params->u.core.flags = 0;
      params->u.core.pagesize = 0;
      params->u.core.rootdev = 0;

      params = tag_next (params);
  }

  static void setup_memory_tags (bd_t *bd)
  {
      int i;

      for (i = 0; i < CONFIG_NR_DRAM_BANKS; i++) {
          params->hdr.tag = ATAG_MEM;
          params->hdr.size = tag_size (tag_mem32);

          params->u.mem.start = bd->bi_dram[i].start;
          params->u.mem.size = bd->bi_dram[i].size;

          params = tag_next (params);
      }
  }

  static void setup_commandline_tag (bd_t *bd, char *commandline)
  {
      char *p;

      if (!commandline)
          return;

      /* eat leading white space */
      for (p = commandline; *p == ' '; p++);

      /* skip non-existent command lines so the kernel will still
       * use its default command line.
       */
      if (*p == '\0')
          return;

      params->hdr.tag = ATAG_CMDLINE;
      params->hdr.size =
          (sizeof (struct tag_header) + strlen (p) + 1 + 4) >> 2;

      strcpy (params->u.cmdline.cmdline, p);

      params = tag_next (params);
  }
```

#### MACHINE\_ID 和 ATAGS 的地址什么时候被设置 ? 
```c
for (init_fnc_ptr = init_sequence; *init_fnc_ptr; ++init_fnc_ptr) {
  if ((*init_fnc_ptr)() != 0) {
      hang ();
  }
}
调用  board_init, board_init定义在对应开发板中如board/100ask24x0/100ask24x0.c
board_init
   gd->bd->bi_arch_number = MACH_TYPE_S3C2440;
   gd->bd->bi_boot_params = 0x30000100;
```

### 总结
uboot 启动kernel最关键的是
* 重定位kernel到uImage中设置的加载地址
* 设置ATAGS, 里面有 bootags，内存信息 等参数
* 根据uImage中kernel的入口地址，调用启动kernel
* 传参 r0:0, r1:machine\_d r2:atags\_addr


## uboot和分区
使用mtd可以查看分区
```shell
OpenJTAG> mtdparts

device nand0 <nandflash0>, # parts = 4
 #: name                        size            offset          mask_flags
 0: bootloader          0x00040000      0x00000000      0
 1: params              0x00020000      0x00040000      0
 2: kernel              0x00200000      0x00060000      0
 3: root                0x0fda0000      0x00260000      0

active partition: nand0,0 - (bootloader) 0x00040000 @ 0x00000000

defaults:
mtdids  : nand0=nandflash0
mtdparts: mtdparts=nandflash0:256k@0(bootloader),128k(params),2m(kernel),-(root)
```
磁盘本身没有分区表，分区信息是uboot或kernel软件定义的。

```c
  int mtdparts_init(void)
      /* get variables */
      ids = getenv("mtdids");
      parts = getenv("mtdparts");
      current_partition = getenv("partition");
  

```

uboot中使用宏定义了默认分区信息，以后nand write/read 操作都按照此信息对flash的划分进行读写
```c
  #define MTDIDS_DEFAULT "nand0=nandflash0"
  #define MTDPARTS_DEFAULT "mtdparts=nandflash0:256k@0(bootloader)," \
                              "128k(params)," \
                              "2m(kernel)," \
                              "-(root)"
```
详细分析分区环境变量格式
```c
"mtdparts=nandflash0:"\     // 下面定义了4个分区
   "256k@0(bootloader)," \  // mtd0 : 256KB大小@起始地址为0 (分区别名为bootloader)
   "128k(params)," \        // mtd1 : 128KB大小，起始地址为紧接着bootloader，分区别名为params
   "2m(kernel)," \          // mtd2 : 2MB大小，起始地址紧接着params，别名为kernel
   "-(root)"                // mtd3 : 剩下的空间都为root分区
```

## uImage头部分析
```c
  typedef struct image_header {
    uint32_t    ih_magic;   /* Image Header Magic Number    */
    uint32_t    ih_hcrc;    /* Image Header CRC Checksum    */
    uint32_t    ih_time;    /* Image Creation Timestamp */
    uint32_t    ih_size;    /* Image Data Size      */
    uint32_t    ih_load;    /* Data  Load  Address      */
    uint32_t    ih_ep;      /* Entry Point Address      */
    uint32_t    ih_dcrc;    /* Image Data CRC Checksum  */

    uint8_t     ih_os;      /* Operating System     */
    uint8_t     ih_arch;    /* CPU architecture     */
    uint8_t     ih_type;    /* Image Type           */
    uint8_t     ih_comp;    /* Compression Type     */
    uint8_t     ih_name[IH_NMLEN];  /* Image Name       */
  } image_header_t;
```
```shell
hexdump -n64 -C uImage

          0x27051956
          ih_maigc      hi_crc         ih_time       ih_size
00000000  27 05 19 56   7d 88 19 07    64 0c 27 0d   00 24 9d d0                                   |'..V}...d.'..$..|
           ih_load      ih_ep          ih_dcrc       ih_os      ih_arch    ih_type     ih_comp 
00000010  30 00 50 00   30 00 50 00    b6 c3 d6 38   05         02         02          00          |0.P.0.P....8....|
          ih_name
00000020  4c 69 6e 75 78 2d 33 2e 34 2e 32 00 00 00 00 00                                          |Linux-3.4.2.....|

00000030  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|

```
