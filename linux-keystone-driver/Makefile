DRIVER = keystone-driver.ko

LINUXSRC ?= $(CURDIR)/../linux
KEYSTONE_SDK_DIR ?= $(CURDIR)/../sdk

ifneq ($(KERNELRELEASE),)
	keystone-driver-y := \
		keystone.o \
		keystone-page.o \
		keystone-ioctl.o \
		keystone-enclave.o \
	  keystone-sbi.o
	obj-m += keystone-driver.o

PWD := $(shell pwd)

ifeq ($(KEYSTONE_SDK_DIR),)
    $(error KEYSTONE_SDK_DIR not defined)
endif

	ccflags-y := -I$(KEYSTONE_SDK_DIR)/include/shared
else

default:
	$(MAKE) -C $(LINUXSRC) ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- M=$(PWD) KEYSTONE_SDK_DIR=$(KEYSTONE_SDK_DIR)

endif

clean:
	rm -rvf *.o *.ko *.order *.symvers *.mod.c .tmp_versions .*o.cmd
