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

## 总结
得到链接脚本 arch/arm/boot/compressed/vmlinux.lds
得到第一个文件 head.S
入口函数 \_start


# 内核的启动过程



# 内核分区

Creating 4 MTD partitions on "NAND":
0x000000000000-0x000000040000 : "bootloader"
0x000000040000-0x000000060000 : "params"
0x000000060000-0x000000260000 : "kernel"
0x000000260000-0x000010000000 : "rootfs"

