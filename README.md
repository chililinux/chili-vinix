# `VINIX`

## Dependencias
	git
	flex
	bison
	libncurses-dev
	build-essential
	bc
	kmod
	cpio
	liblz4-tool
	lz4
	libncurses-dev
	libelf-dev
	libssl-dev
	syslinux
	dosfstools

## Criando um diretório de trabalho
```
	mkdir ~/vinix
	cd ~/vinix
```

## Fontes do Kernel:
* kernel Linux

		git clone --depth=1 https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git

* Linux-Libre - http://linux-libre.fsfla.org/pub/linux-libre/releases

		wget http://linux-libre.fsfla.org/pub/linux-libre/releases/6.4.3-gnu/linux-libre-6.4.3-gnu.tar.xz
		wget http://linux-libre.fsfla.org/pub/linux-libre/releases/6.4.3-gnu/patch-6.4-gnu-6.4.3-gnu.xz
		tar -xvf linux-libre-6.4.3-gnu.tar.xz

	ln -s linux-6.4.3 linux

## Preparando Kernel
	Gerando um .config mínimo
		cd ~/vinix/linux
		make tinyconfig

## Ajustando o .config do kernel
	make menuconfig
- General setup ---> Initial RAM filesystem and RAM disk (initramfs/initrd) support ---> yes (somente gzip)
- General setup ---> Configure standard kernel features ---> Enable support for printk ---> yes
- 64-bit kernel ---> yes
- Executable file formats / Emulations ---> Kernel support for ELF binaries ---> yes
- Executable file formats / Emulations ---> Kernel support for scripts starting with #! ---> yes
- Device Drivers ---> Generic Driver Options ---> Maintain a devtmpfs filesystem to mount at /dev ---> yes
- Device Drivers ---> Generic Driver Options ---> Automount devtmpfs at /dev, after the kernel mounted the rootfs ---> yes
- Device Drivers ---> Character devices ---> Enable TTY ---> yes
- Device Drivers ---> Character devices ---> Serial drivers ---> 8250/16550 and compatible serial support ---> yes
- Device Drivers ---> Character devices ---> Serial drivers ---> Console on 8250/16550 and compatible serial port ---> yes
- File systems ---> Pseudo filesystems ---> /proc file system support ---> yes
- File systems ---> Pseudo filesystems ---> sysfs file system support ---> yes
- Network Supporting ---> TCP/IP networking
- Device Drivers ---> Network device support --> Ethernet driver support -> Intel 82586..
- Device Drivers ---> Network device support --> Ethernet driver support -> Intel devicesPRO/1000 Gigabit e PRO/1000 PCI-Express
- Device Drivers ---> PCI support ->>
- General setup ---> Configure standard kernel features (expert users) -> Posix Clocks & Timers
- Networking support --> Networking options --> TCP/IP networking -> IP: kernel level autoconfiguration

## Compilando kernel
	make clean
	make
	cp arch/x86_4/boot/bzImage ../


## Fontes do Busybox:
	cd ~/vinix
	wget https://busybox.net/downloads/busybox-1.36.1.tar.bz2
	tar xvf busybox-1.36.1.tar.bz2
	ln -s busybox-1.36.1 busybox

## Busybox - gerando .config minimo
	cd ~/vinix/busybox
	#make allnoconfig
	make defconfig

## Customizando e incluindo ferramentas no config do busybox
	make menuconfig

`Exemplos de ferramentas mínimas a incluir`
* Settings > Support files > 2GB
* Settings > Build static binary (no shared libs)
* Coreutils > cat, du, echo, ls, sleep, uname (change Operating system name to anything you want)
* Console Utilities > clear
* Editors > vi
* Init Utilities > poweroff, reboot, init, Support reading an inittab file
* Linux System Utilities > mount, umount, lspci
* Miscellaneous Utilities > less
* Network Utilities > ip, ping
* Process Utilities > free, ps
* Shells > ash, Optimize for size instead of speed, Alias support, Help builtin

## compilando busybox
	make clean
	make
	make install

## Init
	cd ~/vinix
	mkdir filesystem
	cd filesystem

	mkdir -pv {dev,proc,etc/init.d,sys,tmp}
	mknod dev/console c 5 1
	mknod dev/null c 1 3

```
cat >> welcome << EOF
	Bem vindo ao ViNIX!
	      _ _   _ _
	__   _(_) \ | (_)_  __
	\ \ / / |  \| | \ \/ /
	 \ V /| | |\  | |>  <
	  \_/ |_|_| \_|_/_/\_\
	EOF
```

```
cat >> etc/inittab << EOF
::sysinit:/etc/init.d/rc
::askfirst:/bin/sh
::restart:/sbin/init
::ctrlaltdel:/sbin/reboot
::shutdown:/bin/umount -a -r
EOF
```

```
cat >> etc/init.d/rc << EOF
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys

#configure network
ip link set lo up
ip link set eth0 up
ip addr add 192.168.0.200/24 dev eth0
ip route add default via 192.168.0.1

#service telnet
/usr/sbin/telnetd

clear
cat /welcome

#https://busybox.net/FAQ.html#job_control
setsid sh -c 'exec sh </dev/tty1 >/dev/tty1 2>&1'
EOF
```

## Ajustando permissoes
	chmod +x etc/init.d/rc
	chown -R root:root .

## Copiando Busybox para filesystem
	cd ~/vinix/busybox
	cp -r _install/* ../filesystem/

## Gerando arquivo rootfs
	cd ~/vinix/filesystem
	find . | cpio -H newc -o | gzip -9 > ../rootfs.cpio.gz

## Preparando imagem de disco raw
	cd ~/vinix
	dd if=/dev/zero of=vinix.img bs=1k count=4096
	mkdosfs vinix.img
	syslinux --install vinix.img

	cd ~/vinix

```
cat >> syslinux.cfg << EOF
UI menu.c32
prompt 0
timeout 120
MENU TITLE Main Menu

DEFAULT linux
LABEL vinix
   MENU LABEL GNU/Linux VINIX
	SAY [ BOOTING ViNIX v0.1.0 ]
	LINUX bzImage
	APPEND initrd=rootfs.cpio.gz
	KEYMAP br-abnt2

LABEL reboot
	MENU LABEL Reiniciar
	KERNEL reboot.c32
	APPEND --

LABEL poweroff
	MENU LABEL Desligar
	KERNEL poweroff.c32
	APPEND --
EOF
```

## Ajustando permissoes
	chmod +x syslinux.cfg

## Completando imagem de disco
	sudo mount -o loop vinix.img /mnt
	sudo cp bzImage rootfs.cpio.gz syslinux.cfg *.c32 /mnt/

	ls /mnt/
	bzImage ldlinux.c32 ldlinux.sys rootfs.cpio.gz syslinux.cfg
	sudo umount /mnt

## Testando com qemu
	sudo qemu-system-x86_64 \
		-machine accel=kvm \
		-cpu host \
		-smp "$(nproc)" \
		-m 4G \
		-drive file=vinix.img,format=raw,if=none,id=disk1 \
		-device ide-hd,drive=disk1,bootindex=1 \
		-device virtio-scsi-pci,id=scsi0 \
		-netdev user,id=net0 \
		-device e1000,netdev=net0 \
		-serial stdio
