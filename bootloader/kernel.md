本文描述内核移植到jz2440

# Kbuild分析
## menuconfig
生成 .config
内容，有y, m ,字符串，数字
```c
CONFIG_GENERIC_BUG=y
CONFIG_DEFCONFIG_LIST="/lib/modules/$UNAME_RELEASE/.config"
CONFIG_NETFILTER_XT_MATCH_MAC=m
CONFIG_BLK_DEV_RAM_SIZE=4096
```
## make
根据 .config 生成
* include 目录下 auto.conf，定义变量，给Makefile使用
```c
CONFIG_GENERIC_BUG=y
CONFIG_NETFILTER_XT_MATCH_MAC=m
CONFIG_DEFCONFIG_LIST="/lib/modules/$UNAME_RELEASE/.config"
CONFIG_BLK_DEV_RAM_SIZE=4096
```
* include 目录下 autoconf.h，定义宏，给c使用
```c
#define CONFIG_IP6_NF_MATCH_AH_MODULE 1
#define CONFIG_UEVENT_HELPER_PATH "/sbin/hotplug"
#define CONFIG_INPUT_MOUSEDEV_SCREEN_X 1024
```

### obj-m obj-y
auto.conf定义的y,m给各个子目录使用，如driver/leds/Makefile
```makefile
obj-$(CONFIG_NEW_LEDS)          += led-core.o
```
如此便定义了变量 obj-m obj-y
* obj-y 表示链接到vmlinux
* obj-m 表示链接成模块
如果要定义一个模块可以如下写
构造一个ab.ko
```makefile
obj-m = ab.o
ab-objs = a.o b.o
```
构造a.ko b.ko
```makefile
obj-m = a.o b.o
```
### uImage的生成
#### 相关函数
if\_changed 就是检查是否改变，如果改变执行 cmd_$(1)
```makefile
if_changed = $(if $(strip $(any-prereq) $(arg-check)),                       \
    @set -e;                                                             \
    $(echo-cmd) $(cmd_$(1));                                             \
    echo 'cmd_$@ := $(make-cmd)' > $(dot-target).cmd)
```

$(build)=xxx 也就是 
-f scripts/Makefile.build obj
```makefile
build := -f $(if $(KBUILD_SRC),$(srctree)/)scripts/Makefile.build obj
```
#### uImage
arch/arm/boot/Makefile
```makefile
$(obj)/uImage:  $(obj)/zImage FORCE
    @$(check_for_multiple_loadaddr)
    $(call if_changed,uimage)       // 继续追踪会发现调用 cmd_uimage命令，也就是用mkimage将 zImage -> uImage
    @echo '  Image $@ is ready'
```

#### zImage
使用objcopy 将compressed/vmlinux生成 zImage
```makefile
$(obj)/zImage:  $(obj)/compressed/vmlinux FORCE
    $(call if_changed,objcopy)
    @echo '  Kernel: $@ is ready'

cmd_objcopy = $(OBJCOPY) $(OBJCOPYFLAGS) $(OBJCOPYFLAGS_$(@F)) $< $@
```

#### compressed/vmlinux
Kbuild会生成两个vmlinux
首先生成Image, 未压缩的kernel
对Image进行压缩生成 piggy.gzip.o 
链接: head.o  piggy.gzip.o(已经压缩的kernlel程序) decompress.o -> vmlinux -- objcopy --> zImage

生成压缩的vmlinux
```shell
 arm-linux-ld 
 -T arch/arm/boot/compressed/vmlinux.lds   // 链接脚本
 arch/arm/boot/compressed/head.o           
 arch/arm/boot/compressed/piggy.gzip.o     // 压缩后的kernel的程序
 arch/arm/boot/compressed/misc.o 
 arch/arm/boot/compressed/decompress.o     // 解压程序
 arch/arm/boot/compressed/string.o 
 arch/arm/boot/compressed/lib1funcs.o 
 arch/arm/boot/compressed/ashldi3.o 
	 -o arch/arm/boot/compressed/vmlinux
```

arch/arm/boot/Makefile
```makefile
$(obj)/compressed/vmlinux: $(obj)/Image FORCE
    $(Q)$(MAKE) $(build)=$(obj)/compressed $@
-->
make -f scripts/Makefile.build obj=$(obj)/compressed $(obj)/compressed/vmlinux
obj = arch/arm/boot/
-->
make -f scripts/Makefile.build obj=arch/arm/boot/compressed arch/arm/boot/compressed/vmlinux
```

分析 scripts/Makefile.build，发现包含 arch/arm/boot/compressed/Makefile
scripts/Makefile.build
```makefile
obj=arch/arm/boot/compressed
src=arch/arm/boot/compressed/

kbuild-dir := $(if $(filter /%,$(src))   ,$(src),$(srctree)/$(src)) // 保证是绝对路径
kbuild-dir = /root/linux/arch/arm/boot/compressed

kbuild-file := $(if $(wildcard $(kbuild-dir)/Kbuild),$(kbuild-dir)/Kbuild,$(kbuild-dir)/Makefile)
如果有  $(kbuild-dir)/Kbuild 则用此文件，否则用 $(kbuild-dir)/Makefile

include $(kbuild-file)
```

arch/arm/boot/compressed/Makefile
```makefile
obj=arch/arm/boot/compressed

$(obj)/vmlinux: $(obj)/vmlinux.lds $(obj)/$(HEAD) $(obj)/piggy.$(suffix_y).o \
        $(addprefix $(obj)/, $(OBJS)) $(lib1funcs) $(ashldi3) FORCE
    @$(check_for_multiple_zreladdr)
    $(call if_changed,ld)  // cmd_ld
    @$(check_for_bad_syms)

链接方法
cmd_ld = $(LD) $(LDFLAGS) $(ldflags-y) $(LDFLAGS_$(@F)) \
           $(filter-out FORCE,$^) -o $@

链接脚本的输入文件为 arch/arm/boot/compressed/vmlinux.lds.in 
              输出为 arch/arm/boot/compressed/vmlinux.lds
$(obj)/vmlinux.lds: $(obj)/vmlinux.lds.in arch/arm/boot/Makefile $(KCONFIG_CONFIG)
    @sed "$(SEDFLAGS)" < $< > $@

linux的头
$(obj)/$(HEAD) 


这是前段解压程序, 使用Image得到
$(obj)/piggy.$(suffix_y).o

$(obj)/piggy.$(suffix_y): $(obj)/../Image FORCE
    $(call if_changed,$(suffix_y))


真正的kernel
$(addprefix $(obj)/, $(OBJS)) $(lib1funcs) $(ashldi3) FORCE

```

### 分析$(obj)/Image
arch/arm/boot/Makefile
```makefile
$(obj)/Image: vmlinux FORCE
    $(call if_changed,objcopy)  --> 用vmlinux objcopy得到Image
    @echo '  Kernel: $@ is ready'
```

#### 未压缩的vmlinux
${topdir}/makefile
```makefile
vmlinux-lds  := arch/$(SRCARCH)/kernel/vmlinux.lds

vmlinux: $(vmlinux-lds) $(vmlinux-init) $(vmlinux-main) vmlinux.o $(kallsyms.o) FORCE
    $(call vmlinux-modpost)
    $(call if_changed_rule,vmlinux__) --> rule_vmlinux__ 进行链接
    $(Q)rm -f .old_version
```

```makefile
if_changed_rule = $(if $(strip $(any-prereq) $(arg-check) ),                 \
    @set -e;                                                             \
    $(rule_$(1)))
```

rule\_vmlinux\_\_
```makefile
define rule_vmlinux__
    :
    $(if $(CONFIG_KALLSYMS),,+$(call cmd,vmlinux_version))

    $(call cmd,vmlinux__) --> cmd_vmlinux__
    $(Q)echo 'cmd_$@ := $(cmd_vmlinux__)' > $(@D)/.$(@F).cmd

    $(Q)$(if $($(quiet)cmd_sysmap),                                      \
      echo '  $($(quiet)cmd_sysmap)  System.map' &&)                     \
    $(cmd_sysmap) $@ System.map;                                         \
    if [ $$? -ne 0 ]; then                                               \
        rm -f $@;                                                    \
        /bin/false;                                                  \
    fi;
    $(verify_kallsyms)
endef
```

cmd\_vmlinux\_\_
```makefile
      cmd_vmlinux__ ?= $(LD) $(LDFLAGS) $(LDFLAGS_vmlinux) -o $@ \
      -T $(vmlinux-lds) $(vmlinux-init)                          \
      --start-group $(vmlinux-main) --end-group                  \
      $(filter-out $(vmlinux-lds) $(vmlinux-init) $(vmlinux-main) vmlinux.o FORCE ,$^)
```

#### 构建每个模块下的 built-in.o  ---- vmlinux-dirs
要生成 vmlinux依赖 $(vmlinux-dirs)
$(sort $(vmlinux-init) $(vmlinux-main)) $(vmlinux-lds): $(vmlinux-dirs) ;

```makefile
init-y      := init/
drivers-y   := drivers/ sound/ firmware/
net-y       := net/
libs-y      := lib/
core-y      := usr/

core-y      += kernel/ mm/ fs/ ipc/ security/ crypto/ block/

vmlinux-dirs    := $(patsubst %/,%,$(filter %/, $(init-y) $(init-m) \
             $(core-y) $(core-m) $(drivers-y) $(drivers-m) \
             $(net-y) $(net-m) $(libs-y) $(libs-m)))

vmlinux-alldirs := $(sort $(vmlinux-dirs) $(patsubst %/,%,$(filter %/, \
             $(init-n) $(init-) \
             $(core-n) $(core-) $(drivers-n) $(drivers-) \
             $(net-n)  $(net-)  $(libs-n)    $(libs-))))

init-y      := $(patsubst %/, %/built-in.o, $(init-y))      --> init/built-in.o
core-y      := $(patsubst %/, %/built-in.o, $(core-y))      --> usr/built-in.o
drivers-y   := $(patsubst %/, %/built-in.o, $(drivers-y))   --> drivers/built-in.o sound/built-in.o firmware/built-in.o
net-y       := $(patsubst %/, %/built-in.o, $(net-y))       --> net/built-in.o
libs-y1     := $(patsubst %/, %/lib.a, $(libs-y))           --> lib/lib.a
libs-y2     := $(patsubst %/, %/built-in.o, $(libs-y))      --> lib/built-in.o
libs-y      := $(libs-y1) $(libs-y2)


# vmlinux-dirs 为 init drivers sound firmware ...
vmlinux-dirs    := $(patsubst %/,%,$(filter %/, $(init-y) $(init-m) \
             $(core-y) $(core-m) $(drivers-y) $(drivers-m) \
             $(net-y) $(net-m) $(libs-y) $(libs-m)))
```

vmlinux-dirs的规则
```makefile
PHONY += $(vmlinux-dirs)
$(vmlinux-dirs): prepare scripts
    $(Q)$(MAKE) $(build)=$@

 -->
make -f scripts/Makefile.build obj=init

```

obj = init
scripts/Makefile.build
```makefile
src = $(obj)

kbuild-dir := $(if $(filter /%,$(src)),$(src),$(srctree)/$(src))  # 确保绝对路径
kbuild-dir = /root/linux/init

kbuild-file := $(if $(wildcard $(kbuild-dir)/Kbuild),$(kbuild-dir)/Kbuild,$(kbuild-dir)/Makefile)
kbuild-file = /root/linux/init/Makefile

include $(kbuild-file)

// 编译内核时， KBUILD_BUILTIN = 1
// 

lib-target := $(obj)/lib.a
builtin-target := $(obj)/built-in.o
always := 

__build: $(if $(KBUILD_BUILTIN),$(builtin-target) $(lib-target) $(extra-y)) \
     $(if $(KBUILD_MODULES),$(obj-m) $(modorder-target)) \
     $(subdir-ym) $(always)
    @:              // 这个规则没有任何执行的命令，只是定义依赖关系

-->
构建依赖 init/builtin-in.o
$(builtin-target): $(obj-y) FORCE
    $(call if_changed,link_o_target)  // 链接$(obj-y)定义的.o文件

-->
编译
$(obj)/%.o: $(src)/%.c $(recordmcount_source) FORCE
    $(call cmd,force_checksrc)
    $(call if_changed_rule,cc_o_c)

```

### 从未压缩的vmlinux到压缩的piggy.gzip.o
```shell
arm-linux-gcc      -c -o arch/arm/boot/compressed/piggy.gzip.o arch/arm/boot/compressed/piggy.gzip.S
```

arch/arm/boot/compressed/piggy.gzip.S
```asm
      .section .piggydata,#alloc
      .globl  input_data
  input_data:
      .incbin "arch/arm/boot/compressed/piggy.gzip"
      .globl  input_data_end
  input_data_end:
```

cat Image |  gzip -n -f -9 > arch/arm/boot/compressed/piggy.gzip
```shell
Kernel: arch/arm/boot/Image is ready
make -f scripts/Makefile.build obj=arch/arm/boot/compressed arch/arm/boot/compressed/vmlinux
  (cat arch/arm/boot/compressed/../Image | gzip -n -f -9 > arch/arm/boot/compressed/piggy.gzip) || (rm -f arch/arm/boot/compressed/piggy.gzip ; false)
```


## 链接脚本
链接脚本有两个
生成未压缩的vmlinux : arch/arm/boot/kernel/vmlinux.lds
生成已压缩的vmlinux : arch/arm/boot/compressed/vmlinux.lds

uboot启动kernel，对应的是zImage，所以查看 arch/arm/boot/compressed/vmlinux.lds
查看vmlinux.lds，定义了入口符号 \_start, 没有定义哪个.o文件放在前面
```lds
ENTRY(_start)
SECTIONS
{

  .text : {
    _start = .;
    *(.start)
    *(.text)
    *(.text.*)
    *(.fixup)
    *(.gnu.warning)
    *(.glue_7t)
    *(.glue_7)
  }
```
zImage 根据链接顺序，先执行的文件为 arch/arm/boot/compressed/head.S


Image,根据链接顺序获得前面的.o文件, 为 head.o ，对应源文件为 arch/arm/kernel/head.S
cmd\_vmlinux\_\_
```makefile
      cmd_vmlinux__ ?= $(LD) $(LDFLAGS) $(LDFLAGS_vmlinux) -o $@ \
      -T $(vmlinux-lds) $(vmlinux-init)                          \
      --start-group $(vmlinux-main) --end-group                  \
      $(filter-out $(vmlinux-lds) $(vmlinux-init) $(vmlinux-main) vmlinux.o FORCE ,$^)

vmlinux-init := $(head-y) $(init-y)

head-y      := arch/arm/kernel/head.o arch/arm/kernel/init_task.o
```

## 总结
kernel的编译过程
各个目录下生成built-in.o ---链接--> vmlinux --objcopy--> Image --压缩--> piggy.gzip.o --链接上解压程序--> vmlinux --objcopy--> zImage --mkimage加头部--> uImage

入口函数为 \_start
第一个源文件为 arch/arm/kernel/head.S

```shell
  arm-linux-ld -EL    --defsym _kernel_bss_size=167580 --defsym zreladdr=0x30108000 -p --no-undefined -X 
  -T arch/arm/boot/compressed/vmlinux.lds 
  arch/arm/boot/compressed/head.o 
  arch/arm/boot/compressed/piggy.gzip.o 
  arch/arm/boot/compressed/misc.o 
  arch/arm/boot/compressed/decompress.o 
  arch/arm/boot/compressed/string.o 
  arch/arm/boot/compressed/lib1funcs.o 
  arch/arm/boot/compressed/ashldi3.o 
  -o arch/arm/boot/compressed/vmlinux
```
需要注意的是 zImage 是地址无关的程序，所以链接地址从0开始，而Image是地址相关的，Image由 piggy.gzip.o 解压得到，放到 zreladdr 指定的物理地址处

 arch/arm/mach-s3c24xx/Makefile.boot
```shell
ifeq ($(CONFIG_PM_H1940),y)         // yes
    zreladdr-y  += 0x30108000       // zImage解压后将内核放到的地址
    params_phys-y   := 0x30100100   // 内核启动参数 struct boot_params 的物理地址
else
    zreladdr-y  += 0x3000800
    params_phys-y   := 0x30000100
endif
```

在zImage解压内核后会跳转
arch/arm/boot/compressed/head.S
```asm
        ldr r4, =zreladdr
```

# 内核的启动过程
## zImage
### head.S
* 解压Image
* 跳转到 zreladdr

arch/arm/boot/compressed/head.S
```asm
mov r7, r1          @ save architecture ID
mov r8, r2          @ save atags pointer
...
ldr r4, =zreladdr
...
bl  decompress_kernel
bl  cache_clean_flush
bl  cache_off
mov r0, #0          @ must be zero
mov r1, r7          @ restore architecture number
mov r2, r8          @ restore atags pointer

ARM(       mov pc, r4  )       @ call kernel
```

## Image
### head.S
arch/arm/kernel/head.S
```asm
   adr r3, __mmap_switched_data  @ r3 指向一个内存
 
   ldmia   r3!, {r4, r5, r6, r7}
   cmp r4, r5              @ Copy data segment if needed
   cmpne   r5, r6
   ldrne   fp, [r4], #4
   strne   fp, [r5], #4
   bne 1b
 
   mov fp, #0              @ Clear BSS (and zero fp)
   cmp r6, r7
   strcc   fp, [r6],#4
   bcc 1b


 ARM(   ldmia   r3, {r4, r5, r6, r7, sp})
 THUMB( ldmia   r3, {r4, r5, r6, r7}    )
 THUMB( ldr sp, [r3, #16]       )
    str r9, [r4]            @ Save processor ID
    str r1, [r5]            @ Save machine type
    str r2, [r6]            @ Save atags pointer
    bic r4, r0, #CR_A           @ Clear 'A' bit
    stmia   r7, {r0, r4}            @ Save control register values
    b   start_kernel
```
将r1, r2 保存到内存，r3指定的内存

r3指向的内存定义如下，定义了很多全局变量
```asm
      .align  2
      .type   __mmap_switched_data, %object
  __mmap_switched_data:
    .long   __data_loc          @ r4
    .long   _sdata              @ r5
    .long   __bss_start         @ r6
    .long   _end                @ r7
    .long   processor_id            @ r4
    .long   __machine_arch_type     @ r5
    .long   __atags_pointer         @ r6
    .long   cr_alignment            @ r7
    .long   init_thread_union + THREAD_START_SP @ sp
    .size   __mmap_switched_data, . - __mmap_switched_data
```
### start\_kernel

#### 整体启动流程, 和 MID DTB ATAGS
注意head.S 定义的全局变量: \_\_atags\_pointer, \_\_machine\_arch\_type
```c

const char linux_banner[] =
   "Linux version " UTS_RELEASE " (" LINUX_COMPILE_BY "@"
   LINUX_COMPILE_HOST ") (" LINUX_COMPILER ") " UTS_VERSION "\n";


start_kernel
   printk(KERN_NOTICE "%s", linux_banner); // 串口打印  Linux version ...
   setup_arch(&command_line); // mid , ATAGS或dtb 安装
      struct machine_desc *mdesc;
      mdesc = setup_machine_fdt(__atags_pointer); // 优先假设传输的是设备树
         devtree = phys_to_virt(dt_phys);  // 先将物理地址转为虚拟地址
         if (be32_to_cpu(devtree->magic) != OF_DT_HEADER) // 检查是否是DTB
            return NULL;
         initial_boot_params = devtree;
         dt_root = of_get_flat_dt_root();
         for_each_machine_desc(mdesc) {  // 根据根节点compatible属性，和mdesc->dt_compat比较，找出最匹配的machine_desc
               extern struct machine_desc __arch_info_begin[], __arch_info_end[]; // 所有machine_desc放到arch_info段
               #define for_each_machine_desc(p)            \                      // 所以可以当数组遍历
                  for (p = __arch_info_begin; p < __arch_info_end; p++)


            score = of_flat_dt_match(dt_root, mdesc->dt_compat);
            if (score > 0 && score < mdesc_score) {
               mdesc_best = mdesc;
               mdesc_score = score;
            }
         }
	 // mid匹配后，从设备树获得 bootargs ， 等信息
	 // bootargs 放到 boot_command_line , __machine_arch_type 记录mid
         of_scan_flat_dt(early_init_dt_scan_chosen, boot_command_line);
         of_scan_flat_dt(early_init_dt_scan_root, NULL);
         of_scan_flat_dt(early_init_dt_scan_memory, NULL);
         __machine_arch_type = mdesc_best->nr;
	 -----

      if (!mdesc)
         mdesc = setup_machine_tags(machine_arch_type); // 如果不是设备树则使用ATAGS
	                                                // #  define machine_arch_type __machine_arch_type 
							// machine_arch_type 就是 __machine_arch_type，也就是head.S传入的

            for_each_machine_desc(p)  // 根据mid找machine_desc
              if (nr == p->nr) {
                  mdesc = p;
                  break;
              }
            
            if (!mdesc) {
              early_print("\nError: unrecognized/unsupported machine ID"
                  " (r1 = 0x%08x).\n\n", nr);
              dump_machine_table(); /* does not return */
            }
            // 如果找到了machine_desc ，则设置ATAGS 
            if (__atags_pointer)
              tags = phys_to_virt(__atags_pointer);   // 如果传入了ATAGS地址则使用传入的
            else if (mdesc->atag_offset)
              tags = (void *)(PAGE_OFFSET + mdesc->atag_offset); // 否则使用默认的

	    parse_tags(tags); // 解析ATAGS, 如果有bootargs，则将bootargs的值复制给 default_command_line

            // 如果没有设置bootargs，则会使用default_command_line 原理的默认值
            #define CONFIG_CMDLINE "root=/dev/hda1 ro init=/bin/bash console=ttySAC0"
            static char default_command_line[COMMAND_LINE_SIZE] __initdata = CONFIG_CMDLINE;
            char *from = default_command_line; //
            strlcpy(boot_command_line, from, COMMAND_LINE_SIZE);
	    -----

   reset_init
      kernel_init
         do_basic_setup
	    do_initcalls  // 安装模块
               for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++)
                  do_initcall_level(level);

         prepare_namespace // 挂载文件系统
	 init_post
            run_init_process(execute_command); // 运行init应用程序
```

#### execute\_command 
内核怎么从bootargs怎么获得需要的参数, 以execute\_command 为例，
```c

static int __init init_setup(char *str)
{
   unsigned int i;
   
   execute_command = str;
   for (i = 1; i < MAX_INIT_ARGS; i++)
     argv_init[i] = NULL;
   return 1;
}
__setup("init=", init_setup);
```

\_\_setup 定义变量到特定段
```c
  #define __setup_param(str, unique_id, fn, early)            \
      static const char __setup_str_##unique_id[] __initconst \
          __aligned(1) = str; \
      static struct obs_kernel_param __setup_##unique_id  \
          __used __section(.init.setup)           \
          __attribute__((aligned((sizeof(long)))))    \
          = { __setup_str_##unique_id, fn, early }

  #define __setup(str, fn)                    \
      __setup_param(str, fn, fn, 0)

__setup("init=", init_setup);
-->
   __setup_param(str,     unique_id,  fn,         early)            \
   __setup_param("init=", init_setup, init_setup, 0)
--->
static const char __setup_str_init_setup[] __initconst \
  __aligned(1) = "init="; \
static struct obs_kernel_param __setup_init_setup  \
  __used __section(.init.setup)           \
  __attribute__((aligned((sizeof(long)))))    \
  = { "__setup_str_init_setup", init_setup, 0 }

// 所以定义了 struct obs_kernel_param 变量到 .init.setup 段
  struct obs_kernel_param {
      const char *str;
      int (*setup_func)(char *);
      int early;
  };


// 根据链接脚本
 __setup_start = .; *(.init.setup) __setup_end = .;

两个函数会使用 __setup_start __setup_end
start_kernel
   parse_args
      unknown_bootoption
         static int __init obsolete_checksetup(char *line) // 老式的检查setup
         {
             const struct obs_kernel_param *p;
             int had_early_param = 0;
       
             p = __setup_start;
             do {
                 int n = strlen(p->str);
                 if (parameqn(line, p->str, n)) {
                     if (p->early) {
                         /* Already done in parse_early_param?
                          * (Needs exact match on param part).
                          * Keep iterating, as we can have early
                          * params and __setups of same names 8( */
                         if (line[n] == '\0' || line[n] == '=')
                             had_early_param = 1;
                     } else if (!p->setup_func) {
                         printk(KERN_WARNING "Parameter %s is obsolete,"
                                " ignored\n", p->str);
                         return 1;
                     } else if (p->setup_func(line + n))
                         return 1;
                 }
                 p++;
             } while (p < __setup_end);
       
             return had_early_param;
         }

start_kernel
   setup_arch
      parse_early_param
         parse_early_options
            static int __init do_early_param(char *param, char *val)
            {
               const struct obs_kernel_param *p;
               
               for (p = __setup_start; p < __setup_end; p++) {
                 if ((p->early && parameq(param, p->str)) ||
                     (strcmp(param, "console") == 0 &&
                      strcmp(p->str, "earlycon") == 0)
                 ) {
                     if (p->setup_func(val) != 0)
                         printk(KERN_WARNING
                                "Malformed early option '%s'\n", param);
                 }
               }
               /* We accept everything at this stage. */
               return 0;
            }
```
可以看出内核解析bootargs是：
首先预定义了一个obs\_kernel\_param表，当遇到一个bootargs则遍历表，执行对应的处理函数.

#### 从ATAGS 到 command\_line
如何从ATAGS从获得bootargs字符串？ 
```c
setup_machine_tags
   parse_tags
      for (; t->hdr.size; t = tag_next(t))
         parse_tag(t);
            for (t = &__tagtable_begin; t < &__tagtable_end; t++)
             if (tag->hdr.tag == t->tag) {
                 t->parse(tag);
                 break;
             }

vmlinux.lds
  __tagtable_begin = .;
  *(.taglist.init)
  __tagtable_end = .;


#define __tag __used __attribute__((__section__(".taglist.init")))
#define __tagtable(tag, fn) \
static const struct tagtable __tagtable_##fn __tag = { tag, fn }

// 可以找到很多，使用__tagtable 定义的 struct tagtable, 只分析关心的 bootargs
bootargs 的tag 为 ATAG_CMDLINE

  static int __init parse_tag_cmdline(const struct tag *tag)
  {
      strlcpy(default_command_line, tag->u.cmdline.cmdline,
          COMMAND_LINE_SIZE);
      return 0;
  }

  __tagtable(ATAG_CMDLINE, parse_tag_cmdline);

```


# 内核分区

Creating 4 MTD partitions on "NAND":
0x000000000000-0x000000040000 : "bootloader"
0x000000040000-0x000000060000 : "params"
0x000000060000-0x000000260000 : "kernel"
0x000000260000-0x000010000000 : "rootfs"

