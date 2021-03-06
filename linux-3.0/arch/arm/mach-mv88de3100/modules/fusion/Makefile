# for native builds:
# make modules modules_install
# make KERNELDIR=<not currently running kernel's build tree> modules modules_install
# make KERNEL_VERSION=<uname -r of the not currently running kernel> modules modules_install
#
# for cross builds, using standard kernel make environment, i.e.
# make KERNELDIR=<linux build tree> INSTALL_MOD_PATH=<target root fs> modules modules_install

KERNEL_VERSION   ?= $(shell uname -r)
INSTALL_MOD_PATH ?= /
KERNELDIR        ?= $(INSTALL_MOD_PATH)/lib/modules/$(KERNEL_VERSION)/build
FUSIONCORE       ?= single

export FUSIONCORE

CPPFLAGS += $(FUSION_CPPFLAGS) -DFUSION_CALL_INTERRUPTIBLE

# If KERNELRELEASE is defined, it is invoked from the kernel build system.
ifneq ($(KERNELRELEASE),)
TOP := $(srctree)/arch/arm/mach-mv88de3100/modules/fusion
else
TOP := $(shell pwd)
endif
CPPFLAGS += \
	-I$(TOP)/linux/drivers/char/fusion \
	-I$(TOP)/linux/drivers/char/fusion/multi \
	-I$(TOP)/linux/drivers/char/fusion/$(FUSIONCORE) \
	-I$(TOP)/linux/include

KERNEL_BUILD  = $(KERNELDIR)
# works for 2.6.23
KERNEL_SOURCE = $(shell grep ^KERNELSRC $(KERNEL_BUILD)/Makefile | cut -d ' ' -f 6)
ifeq ($(KERNEL_SOURCE), )
  # works for 2.6.32
  KERNEL_SOURCE = $(shell grep '^MAKEARGS := -C ' $(KERNEL_BUILD)/Makefile | cut -d ' ' -f 4)
endif
ifneq ($(KERNEL_SOURCE), )
  K_VERSION    = $(shell grep '^VERSION =' $(KERNEL_SOURCE)/Makefile | cut -d ' ' -f 3)
  K_PATCHLEVEL = $(shell grep '^PATCHLEVEL =' $(KERNEL_SOURCE)/Makefile | cut -d ' ' -f 3)
  K_SUBLEVEL   = $(shell grep '^SUBLEVEL =' $(KERNEL_SOURCE)/Makefile | cut -d ' ' -f 3)
else
  K_VERSION    = $(shell grep '^VERSION =' $(KERNEL_BUILD)/Makefile | cut -d ' ' -f 3)
  K_PATCHLEVEL = $(shell grep '^PATCHLEVEL =' $(KERNEL_BUILD)/Makefile | cut -d ' ' -f 3)
  K_SUBLEVEL   = $(shell grep '^SUBLEVEL =' $(KERNEL_BUILD)/Makefile | cut -d ' ' -f 3)
endif

SUB    = linux/drivers/char/fusion
SUBMOD = drivers/char/fusion

export CONFIG_FUSION_DEVICE=m


ifeq ($(DEBUG),yes)
  CPPFLAGS += -DFUSION_DEBUG_SKIRMISH_DEADLOCK -DFUSION_ENABLE_DEBUG
endif

ifeq ($(shell test -e $(KERNEL_BUILD)/include/linux/autoconf.h && echo yes),yes)
  AUTOCONF_H = -include $(KERNEL_BUILD)/include/linux/autoconf.h
endif

ifeq ($(shell test -e $(KERNEL_BUILD)/include/linux/config.h && echo yes),yes)
  CPPFLAGS += -DHAVE_LINUX_CONFIG_H
endif

ifeq ($(K_VERSION),3)
  KMAKEFILE = Makefile-2.6
else
  KMAKEFILE = Makefile-2.$(K_PATCHLEVEL)
endif

check-version = $(shell expr \( $(K_VERSION) \* 65536 + $(K_PATCHLEVEL) \* 256 + $(K_SUBLEVEL) \) \>= \( $(1) \* 65536 + $(2) \* 256 + $(3) \))

# If KERNELRELEASE is defined, it is invoked from the kernel build system.
ifneq ($(KERNELRELEASE),)
ccflags-y = $(CPPFLAGS)

obj-$(CONFIG_MV88DE3100_FUSION) = fusion.o
fusion-y += \
	linux/drivers/char/fusion/single/fusioncore_impl.o \
	linux/drivers/char/fusion/call.o \
	linux/drivers/char/fusion/debug.o \
	linux/drivers/char/fusion/entries.o \
	linux/drivers/char/fusion/fifo.o \
	linux/drivers/char/fusion/fusiondev.o \
	linux/drivers/char/fusion/fusionee.o \
	linux/drivers/char/fusion/hash.o \
	linux/drivers/char/fusion/list.o \
	linux/drivers/char/fusion/property.o \
	linux/drivers/char/fusion/reactor.o \
	linux/drivers/char/fusion/ref.o \
	linux/drivers/char/fusion/skirmish.o \
	linux/drivers/char/fusion/shmpool.o
else
.PHONY: all modules modules_install install clean

all: modules
install: modules_install headers_install

modules:
	rm -f $(SUB)/Makefile
	cp $(SUB)/$(KMAKEFILE) $(SUB)/Makefile
	echo kernel is in $(KERNEL_SOURCE) and version is $(K_SUBLEVEL)
ifeq ($(call check-version,2,6,24),1)
	$(MAKE) -C $(KERNEL_BUILD) \
		KCPPFLAGS="$(CPPFLAGS) -I`pwd`/linux/include" \
		SUBDIRS=`pwd`/$(SUB) modules
else
	$(MAKE) -C $(KERNEL_BUILD) \
		CPPFLAGS="$(CPPFLAGS) -D__KERNEL__ -I`pwd`/linux/include -I$(KERNEL_BUILD)/include -I$(KERNEL_BUILD)/include2 -I$(KERNEL_SOURCE)/include $(AUTOCONF_H)" \
		SUBDIRS=`pwd`/$(SUB) modules
endif

modules_install: modules
ifeq ($(K_PATCHLEVEL),4)
	install -d $(INSTALL_MOD_PATH)/lib/modules/$(KERNEL_VERSION)/drivers/char/fusion
	install -m 644 $(SUB)/fusion.o $(INSTALL_MOD_PATH)/lib/modules/$(KERNEL_VERSION)/drivers/char/fusion
	rm -f $(INSTALL_MOD_PATH)/lib/modules/$(KERNEL_VERSION)/fusion.o
	/sbin/depmod -ae -b $(INSTALL_MOD_PATH) $(KERNEL_VERSION)
else
ifeq ($(call check-version,2,6,24),1)
	$(MAKE) -C $(KERNEL_BUILD) \
		KCPPFLAGS="$(CPPFLAGS) -I`pwd`/linux/include" \
		INSTALL_MOD_DIR="$(SUBMOD)" \
		SUBDIRS=`pwd`/$(SUB) modules_install
else
	$(MAKE) -C $(KERNEL_BUILD) \
		CPPFLAGS="$(CPPFLAGS) -D__KERNEL__ -I`pwd`/linux/include -I$(KERNEL_BUILD)/include -I$(KERNEL_BUILD)/include2 -I$(KERNEL_SOURCE)/include $(AUTOCONF_H)" \
		SUBDIRS=`pwd`/$(SUB) modules_install
endif
endif

headers_install:
	install -d $(INSTALL_MOD_PATH)/usr/include/linux
	install -m 644 linux/include/linux/fusion.h $(INSTALL_MOD_PATH)/usr/include/linux



clean:
	find $(SUB) \( -name .git -prune \
		-o -name *.o -o -name *.ko -o -name .*.o.cmd \
		-o -name fusion.mod.* -o -name .fusion.* \
		-o -name Module.symvers -o -name modules.order \) \
		-type f -print | xargs rm -f
	find $(SUB) -name .tmp_versions -type d -print |  xargs rm -rf
	rm -f $(SUB)/Makefile
endif #KERNELRELEASE
