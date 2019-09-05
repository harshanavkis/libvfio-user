Mediated User space device
==========================

Overview
--------

muser is a framework that allows mediated device drivers to be implemented in
user space. The device driver can by a completely virtual one without driving
an actual device of that type. This can greatly simplify the initial
development and prototyping of kernel drivers as no kernel code needs to be
written, and failures result in the user space process crashing in the worst
case. The mediated device can be passed to a virtual machine for proper
testing. Device drivers are typically implemented entirely in kernel space for
various reasons, however in early development stages it's acceptable to do it
in user space.

muser is implemented by a small kernel module, muser.ko, that registers itself
with mdev. Every request is forwarded to a user space application via a small,
custom ioctl interface on a control device. The application must be externally
provided and needs to contain the actual device implementation by using the API
of libmuser. See src/samples on how to build such an application.  Currently
there is a one, single-threaded application instance per device, however the
application can employ any form of concurrency needed. In the future we plan to
make libmuser multi-threaded.  The application can be implemented in whatever
way is convenient, e.g. as a Python script using bindings, on the cloud, etc.


Memory Mapping the Device
-------------------------

The device driver can allow parts of the virtual device to be memory mapped by
the virtual machine (e.g. the PCI BARs). The business logic needs to implement
the mmap callback and reply to the request passing the memory address whose
backing pages are then used to satisfy the original mmap call. Currently
reading and writing of the memory mapped memory by the client goes undetected
by libmuser, the business logic needs to poll. In the future we plan to
implement a mechanism in order to provide notifications to libmuser whenever a
page is written to.


Interrupts
----------

Interrupts are implemented by installing the event file descriptor in libmuser
and then notifying it about it. libmuser can then trigger interrupts simply by
writing to it. This can be much more expensive compared to triggering interrupts
from the kernel, however this performance penalty is perfectly acceptable when
prototyping the functional aspect of a device driver.


System Architecture
-------------------

muser.ko and libmuser communicate via ioctl on a control device. This control
device is create when the mediated device is created and appears as
/dev/muser/<UUID>. libmuser opens this device and then executes a "wait
command" ioctl. Whenever a callback of muser.ko is executed, it fills a struct
with the command details and then completes the ioctl, unblocking libmuser. It
then waits to receive another ioctl from libmuser with the result. Currently
there can be only one command pending, we plan to allow multiple commands to be
executed in parallel.


Building muser
==============

vfio/mdev needs to be patched. To generate the patch run:

	git diff 869e3305f23dfeacdaa234717c92ccb237815d90 --diff-filter=M > vfio.patch

Apply the patch and rebuild the vfio/mdev modules:

	make SUBDIRS=drivers/vfio/ modules

Reload the relevant kernel modules:

	drivers/vfio/vfio_iommu_type1.ko
	drivers/vfio/vfio.ko
	drivers/vfio/mdev/mdev.ko
	drivers/vfio/mdev/vfio_mdev.ko

Build the kernel module:

	cd src/kmod
	make

Build the library:

	mkdir build
	cd build
	cmake ..
	make
	make install

Finally build your program and link it to libmuser.so.

Running QEMU
============

To pass the device to QEMU add the following options:

		-device vfio-pci,sysfsdev=/sys/bus/mdev/devices/<UUID>
		-object memory-backend-file,id=ram-node0,prealloc=yes,mem-path=mem,share=yes,size=1073741824 -numa node,nodeid=0,cpus=0,memdev=ram-node0

Guest RAM must be shared (share=yes) otherwise libmuser won't be able to do DMA
transfers from/to it. If you're not using QEMU then any memory that must be
accessed by libmuser must be allocate MAP_SHARED. Registering memory for DMA
that has not been allocated with MAP_SHARED is ignored and any attempts to
access that memory will result in an error.

Example
=======

samples/gpio-pci-idio-16.c implements a tiny part of the PCI-IDIO-16 GPIO
(https://www.accesio.com/?p=/pci/pci_idio_16.html). In this sample it's a simple
device that toggles the input every 3 times it's read.

Running gpio-pci-idio-16
------------------------

1. First, follow the instructions to build and load muser.
2. Then, start the gpio-pci-idio-16 device emulation:
```
# echo 00000000-0000-0000-0000-000000000000 > /sys/class/muser/muser/mdev_supported_types/muser-1/create
# build/dbg/samples/gpio-pci-idio-16 00000000-0000-0000-0000-000000000000
```
3. Finally, start the VM adding the command line explained earlier and then
execute:
```
# insmod gpio-pci-idio-16.ko
# cat /sys/class/gpio/gpiochip480/base > /sys/class/gpio/export
# for ((i=0;i<12;i++)); do cat /sys/class/gpio/OUT0/value; done
0
0
0
1
1
1
0
0
0
1
1
1
```

Future Work
===========

Making libmuser Restartable
----------------------------

muser can be made restartable so that (a) it can recover from failures, and
(b) upgrades are less disrupting. This is something we plan to implement in the
future. To make it restarable muser needs to reconfigure eventfds and DMA
region mmaps first thing when the device is re-opened by libmuser. After muser
has finished reconfiguring it will send a "ready" command, after which normal
operation will be resumed. This "ready" command will always be sent when the
device is opened, even if this is the first time, as this way we don't need to
differentiate between normal operation and restarted operation. libmuser will
store the PCI BAR on /dev/shm (named after e.g. the device UUID) so that it can
easily find them on restart.


Making libmuser Multi-threaded
-------------------------------

libmuser can be made multi-threaded in order to improve performance. To
implement this we'll have to maintain a private context in struct file.
