# VIRTual BLocK IO SIMulating (virtblkiosim)
**Virtual Linux block device driver for simulating and performing I/O**

Despite the fact of existence of various tutorial and referential sources on the Net on how to write a custom block device driver in the form of a loadable kernel module (LKM) for the Linux kernel, they are mostly quite outdated and referred back to somewhat old versions of the Linux kernel. So that the decision to create a simple block device driver LKM suitable (properly working) for recent Linux versions is the goal of this project.

It is created using the way of aggressively utilizing source code comments, even in those places of code where actually there are no needs to emphasize or describe what a particular construct or expression does. But on the other hand, such approach allows one to use these LKM sources as a some kind of tutorial to learn the sequence and calling techniques of the modern Linux LKM API and particularly block device driver API.

The current implementation of a block device driver actually does nothing except it has methods to perform I/O (read/write operations) with blocks of memory of strictly predefined sizes. The process of moving blocks of data to/from a virtual device is simulating the data exchange between a physical storage device and a userland program. This is possible (and is implemented) through the use of the `ioctl()` system call.

## Building

The building process is straightforward: simply `cd` to the `src` directory and `make` the module:

```
$ make
make -C/lib/modules/`uname -r`/build M=$PWD
make[1]: Entering directory '/usr/lib/modules/4.7.6-1-ARCH/build'
  LD      /home/<username>/virtblkiosim/src/built-in.o
  CC [M]  /home/<username>/virtblkiosim/src/virtblkiosim.o
  Building modules, stage 2.
  MODPOST 1 modules
  CC      /home/<username>/virtblkiosim/src/virtblkiosim.mod.o
  LD [M]  /home/<username>/virtblkiosim/src/virtblkiosim.ko
make[1]: Leaving directory '/usr/lib/modules/4.7.6-1-ARCH/build'
```

The output above demonstrates building the module using the Linux kernel headers version 4.7.6 (on Arch Linux system). It may vary depending on build tools and/or Linux distribution and kernel version used.

The finally built module, amongst other files produced is `virtblkiosim.ko`. To see what is it, check it:

```
$ file virtblkiosim.ko
virtblkiosim.ko: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), BuildID[sha1]=88b3fb4e28410614db58548edbe2e7b48bbcf51c, not stripped
```

This module then should be inserted (loaded) into the running kernel through usual `insmod` or `modprobe` commands (see the **Running** section).

To cleanup the working directory (`src`), run `make` with the `clean` target:

```
$ make clean
rm -f -vR virtblkiosim.ko virtblkiosim.o virtblkiosim.mod.* .virtblkiosim.*.cmd built-in.o .built-in.* modules.order Module.symvers .tmp_versions
removed 'virtblkiosim.ko'
removed 'virtblkiosim.o'
removed 'virtblkiosim.mod.c'
removed 'virtblkiosim.mod.o'
removed '.virtblkiosim.ko.cmd'
removed '.virtblkiosim.mod.o.cmd'
removed '.virtblkiosim.o.cmd'
removed 'built-in.o'
removed '.built-in.o.cmd'
removed 'modules.order'
removed 'Module.symvers'
removed '.tmp_versions/virtblkiosim.mod'
removed directory '.tmp_versions'
```

Make changes and build again :-))).

## Dependencies

To build the module one needs to have installed build tools and Linux kernel headers along with their respective dependencies. (As for the example above, the required package containing Linux kernel headers is `linux-headers 4.7.6-1`).

## Running

It is common and usual decision to play with the block device driver inside a virtual machine. It allows to keep the current development host system in a safe state when something will go wrong with the testing driver, and especially, if it will erroneously touch the running operating system, turning it into an unusable state, freeze or hang it up (worst case). So, first of all it needs to install, run, and set up a Linux distribution inside a VM. Then it needs to build the kernel module for the block device driver there. And finally, insert the module into the running kernel... of course inside a VM.

The following example practically demonstrates how to do all these steps. Chosen VM is QEMU (with KVM), chosen Linux distribution is Ubuntu Server 16.04 LTS x86-64 (Linux kernel 4.4.0).

### Creating QEMU HDD

**Note:** All the commands (and related output) shown below were executed under the aforementioned Arch Linux system (x86-64) used as a host OS.

```
$ qemu-img create -f raw <hdd-image-file> 20G
Formatting '<hdd-image-file>', fmt=raw size=21474836480
$
$ file <hdd-image-file>
<hdd-image-file>: UNIF v0 format NES ROM image
$
$ chmod -v 600 <hdd-image-file>
mode of '<hdd-image-file>' changed from 0644 (rw-r--r--) to 0600 (rw-------)
$
$ ls -al <hdd-image-file>
-rw------- 1 <username> <usergroup> 21474836480 Jun 14 17:37 <hdd-image-file>
```

### Installing Ubuntu Server

Run VM, install OS:

```
$ qemu-system-x86_64 -m 1G -enable-kvm -cdrom ubuntu-16.04.1-server-amd64.iso -boot order=d -drive file=<hdd-image-file>,format=raw
$
$ file <hdd-image-file>
<hdd-image-file>: DOS/MBR boot sector
```

### Starting up Ubuntu Server

In order to get access to the VM via SSH, first it needs to properly set up host-guest networking:

```
$ sudo usermod -a -G kvm <username>
$ su <username> -
```

Or simply do relogin. Next:

```
$ groups
wheel kvm <usergroup>
$
$ sudo ip tuntap add tap0 mode tap user <username> group <usergroup>
$ sudo ip addr add 10.0.2.1/24 dev tap0
$
$ sudo sysctl -w net.ipv4.ip_forward=1
net.ipv4.ip_forward = 1
$
$ sudo iptables -t nat -A POSTROUTING -s 10.0.2.0/24 -j MASQUERADE
$
$ sudo iptables -t nat -L
...
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination
MASQUERADE  all  --  10.0.2.0/24          anywhere
$
$ sudo vde_switch -mod 660 -group <usergroup> -tap tap0 -daemon
```

Finally, start up Ubuntu Server:

```
$ qemu-system-x86_64 -m 512M -enable-kvm -show-cursor -cpu host -smp 2 -net nic,model=virtio -net vde -drive file=<hdd-image-file>,format=raw
```

When the guest OS (Ubuntu Server) is up and running, login into it and do the following:

```
$ sudo ip addr add 10.0.2.100/24 dev eth0
$ sudo ip route add default via 10.0.2.1
$ sudo su -
# echo 'nameserver 8.8.8.8' >> /etc/resolv.conf
```

Logout from root, logout from the current user. The guest OS is now ready to accept connections from the host OS via SSH.

### Logging into Ubuntu Server through SSH

```
$ ssh -C <vmusername>@10.0.2.100
Welcome to Ubuntu 16.04.1 LTS (GNU/Linux 4.4.0-34-generic x86_64)
...
<vmusername>@<vmhostname>:~$
```

Now it is time to install and set up additional packages, utilities, development tools, etc. in the guest OS. These steps are not necessary to describe, so that they will be omitted.
