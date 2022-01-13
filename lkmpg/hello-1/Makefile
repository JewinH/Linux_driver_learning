# obj-m += hello-1.o
# obj-m += hello-2.o
obj-m += hello-3.o
obj-m += hello-4.o
obj-m += hello-5.o
obj-m += startstop.o 
startstop-objs := start.o stop.o
obj-m += chardev.o
obj-m += procfs1.o
obj-m += procfs2.o

CFLAGS_procfs3.o := -DDEBUG
obj-m += procfs3.o


CFLAGS_procfs4.o := -DDEBUG
obj-m += procfs4.o

CFLAGS_hello-sysfs.o := -DDEBUG
obj-m += hello-sysfs.o

obj-m += chardev2.o

obj-m += ioctl.o
obj-m += sleep.o

obj-m += completions.o

obj-m += example_mutex.o

obj-m += example_spinlock.o

obj-m += print_string.o


obj-m += kbleds.o

CFLAGS_example_tasklet.o := -DDECLARE_TASKLET_OLD
obj-m += example_tasklet.o

obj-m += sched.o

obj-m += cryptosha256.o

obj-m += cryptosk.o

obj-m += devicemodel.o

PWD := $(CURDIR)
 
all: 
	echo $(PWD)
	make -C  /lib/modules/$(shell uname -r)/build M=$(PWD) modules 

clean: 
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean