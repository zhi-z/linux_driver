KERNELDIR := /home/nongjh/dev/rk356x/rk356x_linux_release_20210511/kernel

CURRENT_PATH := $(shell pwd)

obj-m := newchrled.o

build: kernel_modules

kernel_modules:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) modules

clean:
	$(MAKE) -C $(KERNELDIR) M=$(CURRENT_PATH) clean
