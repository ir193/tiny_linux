
This is my note of doing the project for advanced operating systems. I'll try to log tools needed, the configure I used and the pitfalls during the this project.

We’ll be building:

 1. A tiny custom [Linux kernel][1]. And try to make its size as small as possible by disable some kernel feature/function, and also by patch the kernel source code.

 2. A ramdisk served as main filesystem, with some configure to setup network working.

The host and target will both be x86.

Some useful tutorial:

* http://mgalgs.github.io/2015/05/16/how-to-build-a-custom-linux-kernel-for-qemu-2015-edition.html

* http://www.linuxfromscratch.org/

Some useful example:

* http://distro.ibiblio.org/tinycorelinux/


#Preparation#

Setup neccessary toolchains and libraries, at least:

* gcc (for building linux kernel and busybox)
* curl (for download)
* ncurses (for linux menuconfig)

Now, let's get started

First, setup a working directory and some environment variable for convenience

    mkdir $HOME/tiny_linux
    TOP=$HOME/tiny_linux
    cd $TOP

Download source code of [linux kernel][1]:

    curl https://www.kernel.org/pub/linux/kernel/v4.x/linux-4.0.4.tar.xz | tar xJf -

Downlaod source code of [busybox][2]

    curl http://busybox.net/downloads/busybox-1.23.2.tar.bz2 | tar xjf -


#Build Linux Kernel#

Now build a linux kernel with default config. Before configure&make, create a directory for obj files to keep kernel source clean and some variables for convenience

    cd $TOP
    mkdir obj
    mkdir obj/linux_defconfig
    LINUX_SRC=$TOP/linux-4.0.4
    cd $LINUX_SRC
    make O=../obj/linux_defconfig i386_defconfig

Now the config has been write into $TOP/obj/linux_defconfig. Go into it and compile

    cd $TOP/obj/linux_defconfig
    make

If no error message reported, you should see 

> Kernel: arch/x86/boot/bzImage is ready  (#1)

Test you bright new kernel with qemu:

    qemu-system-i386 -kernel $TOP/obj/linux_defconfig/arch/x86/boot/bzImage

You should see linux running and finally a kernel panic. To see full log:

    qemu-system-i386 -kernel $TOP/obj/linux_defconfig/arch/x86/boot/bzImage -append "console=ttyS0" -nographic

Note that when use `-nographic`, you can quit by hit `Ctrl-C` then `a`, then type `quit`

The error log should be:

> [    3.711331] VFS: Cannot open root device "(null)" or unknown-block(0,0): error -6
> [    3.730405] Please append a correct "root=" boot option; here are the available partitions:

That means our kernel works fine, but it can't find root device. Next we create the root device, a ramdisk.

Before that, just copy the kernel image into obj directory.

    cp $TOP/obj/linux_defconfig/arch/x86/boot/bzImage $TOP/obj
    KERNEL=$TOP/obj/bzImage

#Build Root Filesystem#

Now we focus on the userland level, we will use [Busybox][2]. BusyBox combines tiny versions of many common UNIX utilities into a single small executable. It provides replacements for most of the utilities you usually find in GNU fileutils, shellutils, etc.

First we build busybox:
    
    cd $TOP
    mkdir obj/busybox
    cd $TOP/busybox-1.23.2
    make O=../obj/busybox/ defconfig

IMPORTANT, config busybox to static linked

    make O=../obj/busybox/ menuconfig

choose:

    Busybox Setting:
        Build Option
            Build Busybox as a static binary

Ok, just build it (in obj directory, to keep source code clean)
               
    cd $TOP/obj/busybox
    make
    make install

Note that here we use a make install, but this will NOT install busybox into your HOST system. Relax.

You can simplely run the busybox to test if it works properly.

    ./busybox ls

Now create another directory to hold our filesystem hierarchy, and copy busybox into it.

    cd $TOP/
    mkdir ramdisk
    cd ramdisk
    cp -av $TOP/obj/busybox/_install/* .

Then create a simple init script for debug. 

Recall what we learn on class, [init][3] process is the first userspace process. It takes care of services, runlevels and so on. All processes will fork from init. In real linux distribution, some typical init process are: systemd, Upstart, SysV

    echo -e '#!/bin/sh \n /bin/sh' > init
    chmod +x init

To ensure kernel can execute script, we need linux kernel compiled with 

    Kernel support for scripts starting with #!

But when finish debug, we will use symbolic link busybox as init, then this option can be turned off.

check again the symbolic link is correct. Then we can pack the filesystem into a cpio file. 

    cd $TOP/ramdisk
    find . -print0 | cpio --null -ov --format=newc | gzip -9 > $TOP/obj/initramfs.cpio.gz
    RAMDISK=$TOP/obj/initramfs.cpio.gz

RUN IT~:

    qemu-system-i386 -kernel $KERNEL -initrd $RAMDISK

If success, a shell will be ready when finish booting.

If you saw:

> /bin/sh: can't access tty; job control turned off

Don't worry, that's normal because our init simply launch a shell without necessary setup. We work on it in next step.


#Userland: init and mount#

We step back and remove the debug script. And create a symbolic link with busybox to subtitude it:

    cd $TOP/ramdisk
    rm init
    ln -s bin/busybox init

So after kernel booting into userland, busybox will be called as init. Then busybox check configurations and do its job.

Before we write init configurations, we create some directory:

    cd $TOP/ramdisk
    mkdir -pv {bin,sbin,etc,proc,sys,usr/{bin,sbin},dev}

That's more like a real linux system.

Then busybox configuration:
    
    cd $TOP/ramdisk
    cd etc
    vim inittab

The content of inittab can be:

    ::sysinit:/etc/init.d/rcS

    ::askfirst:-/bin/sh

    ::restart:/sbin/init

    ::ctrlaltdel:/sbin/reboot

    ::shutdown:/bin/umount -a -r

    ::shutdown:/sbin/swapoff -a


And rcS:

    mkdir init.d
    cd init.d
    vim rcS

rcS will called after system init, we mount filesystem and initialize network here

    #!/bin/sh

    mount proc
    mount -o remount,rw /
    mount -a

    clear                               
    echo "Booting Tiny Linux"

Don't forget attribute:
    
    chmod +x rcS

See the `mount -a`, it will check /etc/fstab. We create it:

    cd $TOP/ramdisk/etc
    vim fstab

The content:
    
    # /etc/fstab
    proc            /proc        proc    defaults          0       0
    sysfs           /sys         sysfs   defaults          0       0
    devtmpfs        /dev         devtmpfs  defaults          0       0

OK for now, that's finish, pack it and run:

    cd $TOP/ramdisk
    find . -print0 | cpio --null -ov --format=newc | gzip -9 > $TOP/obj/initramfs.cpio.gz
    qemu-system-i386 -kernel $KERNEL -initrd $RAMDISK

We should see:

    Booting Tiny Linux
    Please press Enter to activate this console.

Press Enter and get a shell. Check /dev /proc and /sys

    ls /dev
    ls /proc
    ls /sys

Result should not be empty

Finally, check ethernet card driver working:

    ifconfig -a

We should see `eth0` if ethernet card driver success.


#Network#

Keep editing /etc/init.d/rcS, add 

    /sbin/ifconfig lo 127.0.0.1 up
    /sbin/route add 127.0.0.1 lo &


    ifconfig eth0 up
    ip addr add 10.0.2.15/24 dev eth0
    ip route add default via 10.0.2.2

That's all.

You can test it in your tiny system:

    wget http://[you HOST os ip]:port/file


#Reducing the kernel size#

run 

    cd $TOP/linux-4.0.4
    make O=../obj/linux_defconfig menuconfig

turn off unneccessary features! Especially devices drivers

To be continued
see http://elinux.org/Work_on_Tiny_Linux_Kernel


#FAQ#

1. I can't use `make menuconfig`

Maybe you need install ncurse

2. How can I see full log?

    qemu-system-i386 -kernel $TOP/obj/linux_defconfig/arch/x86/boot/bzImage -append "console=ttyS0" -nographic

or inside you GUEST OS

    dmesg | less

3. When I run qemu `error reading initrd: No such file or directory`

Maybe you continue your work in anther shell? Check environment variable $TOP $KERNEL $RAMDISK is correct

4. VFS: Cannot open root device "(null)"

check -initrd parameter and kernel compiled with initramfs support.

5. Kernel panic at boot: not syncing. No init found.

check busybox must be *staticaly* compiled

6. Generate ramdisk is slow and very large

Execute command right under ramdisk root.

7. ifconfig report /proc/net/dev: No such file or directory

/proc is not properly mount. check fstab and rcS.

8. I can't see eth0

check ethernet device driver support. By default, QEMU emulate a Intel E1000 card. Note <M> means loadable kernel module and need mannually load.

9. I can't ping

It's normal. Because QEMU default ethernet driver doesn't support ICMP.

10. A lot of error message

It's normal as long as network can work.

11. I can't get DHCP working

NEITHER CAN I!

12. But I can get DHCP working?

DO tell me!


#Reference#
https://www.kernel.org/

http://busybox.net/

http://en.wikipedia.org/wiki/Inithttp://elinux.org/Work_on_Tiny_Linux_Kernel

http://mgalgs.github.io/2015/05/16/how-to-build-a-custom-linux-kernel-for-qemu-2015-edition.html

http://distro.ibiblio.org/tinycorelinux/



[1]: https://www.kernel.org/
[2]: http://busybox.net/
[3]: http://en.wikipedia.org/wiki/Init

[4]: http://elinux.org/Work_on_Tiny_Linux_Kernel
[5]: http://mgalgs.github.io/2015/05/16/how-to-build-a-custom-linux-kernel-for-qemu-2015-edition.html
[6]: http://distro.ibiblio.org/tinycorelinux/