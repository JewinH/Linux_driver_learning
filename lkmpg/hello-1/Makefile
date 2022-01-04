obj-m += hello-1.o

PWD := $(CURDIR)
 
all: 
	echo $(PWD)
	make -C  /lib/modules/$(shell uname -r)/build M=$(PWD) modules 

clean: 
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean