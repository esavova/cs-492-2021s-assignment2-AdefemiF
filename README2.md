cs-492-2021s-assignment2-AdefemiF/driver/Makefile                                                   0000644 0000000 0000000 00000001306 14027413305 020062  0                                                                                                    ustar   root                            root                                                                                                                                                                                                                   # Comment/uncomment the following line to disable/enable debugging
#DEBUG = y


# Add your debugging flag (or not) to CFLAGS
ifeq ($(DEBUG),y)
  DEBFLAGS = -O0 -g -DSCULL_DEBUG # "-O" is needed to expand inlines
else
  DEBFLAGS = -O2
endif

EXTRA_CFLAGS += $(DEBFLAGS)

ifneq ($(KERNELRELEASE),)
# call from kernel build system

obj-m	:= scull.o

else

KERNELDIR ?= /lib/modules/$(shell uname -r)/build
PWD       := $(shell pwd)

modules:
	$(MAKE) -C $(KERNELDIR) M=$(PWD) modules

endif



clean:
	rm -rf *.o *~ core .depend .*.cmd *.ko *.mod.c .tmp_versions *.mod modules.order *.symvers

depend .depend dep:
	$(CC) $(EXTRA_CFLAGS) -M *.c > .depend


ifeq (.depend,$(wildcard .depend))
include .depend
endif
                                                                                                                                                                                                                                                                                                                          cs-492-2021s-assignment2-AdefemiF/src/scull.c                                                       0000644 0000000 0000000 00000010325 14027525361 017213  0                                                                                                    ustar   root                            root                                                                                                                                                                                                                   #include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <wait.h>
#include <pthread.h>
#include "scull.h"

#define CDEV_NAME "/dev/scull"

/* Quantum command line option */
static int g_quantum;

static void usage(const char *cmd)
{
	printf("Usage: %s <command>\n"
	       "Commands:\n"
	       "  R          Reset quantum\n"
	       "  S <int>    Set quantum\n"
	       "  T <int>    Tell quantum\n"
	       "  G          Get quantum\n"
	       "  Q          Qeuery quantum\n"
	       "  X <int>    Exchange quantum\n"
	       "  H <int>    Shift quantum\n"
	       "  h          Print this message\n",
	       cmd);
}

typedef int cmd_t;

static cmd_t parse_arguments(int argc, const char **argv)
{
	cmd_t cmd;

	if (argc < 2) {
		fprintf(stderr, "%s: Invalid number of arguments\n", argv[0]);
		cmd = -1;
		goto ret;
	}

	/* Parse command and optional int argument */
	cmd = argv[1][0];
	switch (cmd) {
	case 'S':
	case 'T':
	case 'H':
	case 'X':
	case 'p':
	case 't':
		if (argc < 3) {
			fprintf(stderr, "%s: Missing quantum\n", argv[0]);
			cmd = -1;
			break;
		}
		g_quantum = atoi(argv[2]);
		break;
	case 'R':
	case 'G':
	case 'Q':
	case 'h':
		break;
	default:
		fprintf(stderr, "%s: Invalid command\n", argv[0]);
		cmd = -1;
	}

ret:
	if (cmd < 0 || cmd == 'h') {
		usage(argv[0]);
		exit((cmd == 'h')? EXIT_SUCCESS : EXIT_FAILURE);
	}
	return cmd;
}
static void* helper(void* ptr){
	struct task_info info;
	ioctl(*(int*)ptr, SCULL_IOCKQUANTUM, &info);
	printf("state %ld, stack %lx, cpu %u, prio %d, sprio %d, nprio %d, rtprio %u, pid %d, tgid %d, nv %lu, niv %lu\n", info.state, (unsigned long) info.stack, info.cpu, info.prio, info.static_prio, info.normal_prio, info.rt_priority, info.pid, info.tgid, info.nvcsw, info.nivcsw);
	pthread_exit(NULL);
	
}
static int do_op(int fd, cmd_t cmd)
{
	int ret, q;
	struct task_info info;
	pthread_t yeah[10];
	int num = 0;
	int split;
	switch (cmd) {
	case 'R':
		ret = ioctl(fd, SCULL_IOCRESET);
		if (ret == 0)
			printf("Quantum reset\n");
		break;
	case 'Q':
		q = ioctl(fd, SCULL_IOCQQUANTUM);
		printf("Quantum: %d\n", q);
		ret = 0;
		break;
	case 'G':
		ret = ioctl(fd, SCULL_IOCGQUANTUM, &q);
		if (ret == 0)
			printf("Quantum: %d\n", q);
		break;
	case 'T':
		ret = ioctl(fd, SCULL_IOCTQUANTUM, g_quantum);
		if (ret == 0)
			printf("Quantum set\n");
		break;
	case 'S':
		q = g_quantum;
		ret = ioctl(fd, SCULL_IOCSQUANTUM, &q);
		if (ret == 0)
			printf("Quantum set\n");
		break;
	case 'X':
		q = g_quantum;
		ret = ioctl(fd, SCULL_IOCXQUANTUM, &q);
		if (ret == 0)
			printf("Quantum exchanged, old quantum: %d\n", q);
		break;
	case 'H':
		q = ioctl(fd, SCULL_IOCHQUANTUM, g_quantum);
		printf("Quantum shifted, old quantum: %d\n", q);
		ret = 0;
		break;
	case 'p':
		if (g_quantum <11){
			if (g_quantum >0){
				num = 0;
				while (num<g_quantum){
					split = fork();
					if (split && split ==0){
						ret = ioctl(fd, SCULL_IOCKQUANTUM, &info);
						printf("state %ld, stack %lx, cpu %u, prio %d, sprio %d, nprio %d, rtprio %u, pid %d, tgid %d, nv %lu, niv %lu\n", info.state, (unsigned long) info.stack, info.cpu, info.prio, info.static_prio, info.normal_prio, info.rt_priority, info.pid, info.tgid, info.nvcsw, info.nivcsw);
						exit(0);
					}
					num++;
				}
				num=0;
				while (num<g_quantum){
					wait(NULL);
					num++;
			}
		}
	}
	ret = 0;
	break;
	case 't':
		if (g_quantum <11){
			if (g_quantum >0){
				num=0;
				while(num<g_quantum){
					pthread_create(&yeah[num], NULL, helper, &fd);
					num++;
				}
				num = 0;
				while(num<g_quantum){
					pthread_join(yeah[num], NULL);
					num++;
				}
			}
		}
		ret = 0;
		break;
	default:
		/* Should never occur */
		abort();
		ret = -1; /* Keep the compiler happy */
	}

	if (ret != 0)
		perror("ioctl");
	return ret;
}

int main(int argc, const char **argv)
{
	int fd, ret;
	cmd_t cmd;

	cmd = parse_arguments(argc, argv);

	fd = open(CDEV_NAME, O_RDONLY);
	if (fd < 0) {
		perror("cdev open");
		return EXIT_FAILURE;
	}

	printf("Device (%s) opened\n", CDEV_NAME);

	ret = do_op(fd, cmd);

	if (close(fd) != 0) {
		perror("cdev close");
		return EXIT_FAILURE;
	}

	printf("Device (%s) closed\n", CDEV_NAME);

	return (ret != 0)? EXIT_FAILURE : EXIT_SUCCESS;
}
                                                                                                                                                                                                                                                                                                           cs-492-2021s-assignment2-AdefemiF/driver/debug.h                                                    0000644 0000000 0000000 00000001272 14027413305 017663  0                                                                                                    ustar   root                            root                                                                                                                                                                                                                   #ifndef _DEBUG_H_
#define _DEBUG_H_

/*
 * Macros to help debugging
 */

#undef PDEBUG             /* undef it, just in case */
#ifdef SCULL_DEBUG
#  ifdef __KERNEL__
     /* This one if debugging is on, and kernel space */
#    define PDDEBUG(fmt, args...) printk( KERN_DEBUG "scull: " fmt, ## args)
#    define PDEBUG(fmt, args...) printk( KERN_INFO  "scull: " fmt, ## args)
#  else
     /* This one for user space */
#    define PDDEBUG(fmt, args...) fprintf(stderr, fmt, ## args)
#    define PDEBUG(fmt, args...) fprintf(stderr, fmt, ## args)
#  endif
#else
#  define PDDEBUG(fmt, args...) /* not debugging: nothing */
#  define PDEBUG(fmt, args...) /* not debugging: nothing */
#endif

#endif
                                                                                                                                                                                                                                                                                                                                      cs-492-2021s-assignment2-AdefemiF/driver/scull.init                                                 0000755 0000000 0000000 00000004312 14027413305 020434  0                                                                                                    ustar   root                            root                                                                                                                                                                                                                   #!/bin/bash
# Sample init script for the scull driver module <rubini@linux.it>

DEVICE="scull"

# The list of filenames and minor numbers: $PREFIX is prefixed to all names
PREFIX="scull"

INSMOD=/sbin/insmod; # use /sbin/modprobe if you prefer

function device_specific_post_load () {
    true; # fill at will
}
function device_specific_pre_unload () {
    true; # fill at will
}

# Everything below this line should work unchanged for any char device.
# Obviously, however, no options on the command line: either in
# /etc/${DEVICE}.conf or /etc/modules.conf (if modprobe is used)

# kernel version, used to look for modules
KERNEL=`uname -r`

# Root or die
if [ "$(id -u)" != "0" ]
then
  echo "You must be root to load or unload kernel modules"
  exit 1
fi

MODE=0664

# Create device files
function create_files () {
    cd /dev
    local devlist=$DEVICE
    local file=$DEVICE
    mknod $file c $MAJOR 0
    if [ -n "$OWNER" ]; then chown $OWNER $devlist; fi
    if [ -n "$GROUP" ]; then chgrp $GROUP $devlist; fi
    if [ -n "$MODE"  ]; then chmod $MODE  $devlist; fi
}

# Remove device files
function remove_files () {
    cd /dev
    local devlist=${DEVICE}
    rm -f $devlist
    cd - > /dev/null
}

MY_PATH="`dirname $0`"

# Load and create files
function load_device () {
    local devpath=$DEVICE

    if [ -f ./$DEVICE.ko ]; then
	devpath=./$DEVICE.ko
    elif [ -f $MY_PATH/$DEVICE.ko ]; then
	devpath=$MY_PATH/$DEVICE.ko
    else
	devpath=$DEVICE
    fi

    if [ "$devpath" != "$DEVICE" ]; then
	echo " (loading file $devpath)"
    fi

    if $INSMOD $devpath $OPTIONS; then
	MAJOR=`awk "\\$2==\"$DEVICE\" {print \\$1}" /proc/devices`
	remove_files
	create_files
	device_specific_post_load
    else
	echo " FAILED!"
     fi
}

# Unload and remove files
function unload_device () {
    device_specific_pre_unload 
    /sbin/rmmod $DEVICE
    remove_files 
}


case "$1" in
  start)
     echo -n "Loading $DEVICE"
     load_device
     echo "."
     ;;
  stop)
     echo -n "Unloading $DEVICE"
     unload_device
     echo "."
     ;;
  force-reload|restart)
     echo -n "Reloading $DEVICE"
     unload_device
     load_device
     echo "."
     ;;
  *)
     echo "Usage: $0 {start|stop|restart|force-reload}"
     exit 1
esac

exit 0
                                                                                                                                                                                                                                                                                                                      cs-492-2021s-assignment2-AdefemiF/driver/Makefile                                                   0000644 0000000 0000000 00000000000 14027413305 030356  1cs-492-2021s-assignment2-AdefemiF/driver/Makefile                                                   ustar   root                            root                                                                                                                                                                                                                   cs-492-2021s-assignment2-AdefemiF/driver/scull.h                                                    0000644 0000000 0000000 00000004074 14027517640 017731  0                                                                                                    ustar   root                            root                                                                                                                                                                                                                   /*
 * scull.h -- definitions for the char module
 *
 * Copyright (C) 2001 Alessandro Rubini and Jonathan Corbet
 * Copyright (C) 2001 O'Reilly & Associates
 *
 * The source code in this file can be freely used, adapted,
 * and redistributed in source or binary form, so long as an
 * acknowledgment appears in derived source files.  The citation
 * should list that the code comes from the book "Linux Device
 * Drivers" by Alessandro Rubini and Jonathan Corbet, published
 * by O'Reilly & Associates.   No warranty is attached;
 * we cannot take responsibility for errors or fitness for use.
 *
 * $Id: scull.h,v 1.15 2004/11/04 17:51:18 rubini Exp $
 */

#ifndef _SCULL_H_
#define _SCULL_H_

#include <linux/ioctl.h> /* needed for the _IOW etc stuff used later */


#ifndef SCULL_MAJOR
#define SCULL_MAJOR 0   /* dynamic major by default */
#endif


/*
 * SCULL_QUANTUM
 */
#ifndef SCULL_QUANTUM
#define SCULL_QUANTUM 4000
#endif


/*
 * Ioctl definitions
 */

/* Use 'k' as magic number */
#define SCULL_IOC_MAGIC  'k'
/* Please use a different 8-bit number in your code */

#define SCULL_IOCRESET    _IO(SCULL_IOC_MAGIC, 0)

/*
 * S means "Set" through a ptr,
 * T means "Tell" directly with the argument value
 * G means "Get": reply by setting through a pointer
 * Q means "Query": response is on the return value
 * X means "eXchange": switch G and S atomically
 * H means "sHift": switch T and Q atomically
 */
#define SCULL_IOCSQUANTUM _IOW(SCULL_IOC_MAGIC,  1, int)
#define SCULL_IOCTQUANTUM _IO(SCULL_IOC_MAGIC,   2)
#define SCULL_IOCGQUANTUM _IOR(SCULL_IOC_MAGIC,  3, int)
#define SCULL_IOCQQUANTUM _IO(SCULL_IOC_MAGIC,   4)
#define SCULL_IOCXQUANTUM _IOWR(SCULL_IOC_MAGIC, 5, int)
#define SCULL_IOCHQUANTUM _IO(SCULL_IOC_MAGIC,   6)



 struct task_info{ 

	long state;
	void *stack;
	unsigned int cpu;
	int prio;
	int static_prio;
	int normal_prio;
	unsigned int rt_priority;
	pid_t pid;
	pid_t tgid;
	unsigned long nvcsw;
	unsigned long nivcsw;
 };

#define SCULL_IOCKQUANTUM _IOR(SCULL_IOC_MAGIC, 7, struct task_info)
/* ... more to come */

#define SCULL_IOC_MAXNR 7

#endif /* _SCULL_H_ */
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    