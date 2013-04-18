---
title: Установка 1С под линукс в госте
created_at: 8 февраля 2011
layout: post
tags: ['post']
urls: ['/blog/2011/gentoo-softraid-lvm-kvm-1c.html']
intro: Описан процесс создания сервера KVM на основе Gentoo и виртуальной машины для 1С, опять же на Gentoo. Написана давно, в 2010 году, и с тех пор не проверялась. Также недописана установка собственно 1С.
---
Описан процесс создания сервера KVM на основе Gentoo и виртуальной машины для 1С, опять же на Gentoo. [```недописана установка 1С```].
Написана давно, в 2010 году, и с тех пор не проверялась. Также недописана установка собственно 1С.

Используемая документация

* [Gentoo Linux x86 with Software Raid and LVM2 Quick Install Guide](http://www.gentoo.org/doc/en/gentoo-x86+raid+lvm2-quickinstall.xml)
* [RAID/Software](http://en.gentoo-wiki.com/wiki/RAID/Software)
* [KVM](http://en.gentoo-wiki.com/wiki/KVM)


## Cоздание материнской ОС

Загружаеся с gentoo-cd:
```
gentoo nox
```

Настроиваем сеть
```
livecd ~ # net-setup eth0
```

Настраиваем сеть, задаем пароль для того чтобы подключится с удаленной машины и работать в комфорте
```
ifconfig eth0
passwd
/etc/init.d/sshd start
```

Подключаемся к машине (в моем случае это 148)
```
ssh root@10.96.0.148
```

Подключаем модули
```
livecd ~ # modprobe raid0 
livecd ~ # modprobe raid1
livecd ~ # modprobe dm-mod
```

В моем случае я взял для примера пару старых винтов на 40Gb.
При разметке не забываем сменить тип блока ```(82 - swap, fd - Linux raid autodetect)```
```
livecd ~ # fdisk -l /dev/sda

Disk /dev/sda: 40.0 GB, 40019582464 bytes
255 heads, 63 sectors/track, 4865 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Disk identifier: 0xbcc1b5b0

Device Boot      Start         End      Blocks   Id  System
/dev/sda1               1         132     1060258+  82  Linux swap / Solaris
/dev/sda2   *         133         655     4200997+  fd  Linux raid autodetect
/dev/sda3             656        4865    33816825   fd  Linux raid autodetect
```

Копируем разметку с первого на второй
```
# sfdisk -d /dev/sda | sfdisk /dev/sdb
```

Подключаем модули и создем RAID
```
# cd /dev && MAKEDEV md
```

Корень(md0) делаем mirror
```
livecd dev # mdadm --create --verbose /dev/md1 --level=1 --raid-devices=2 \
/dev/sda2 /dev/sdb2
```

Остальной(md1) - как угодно ( в моем случае тоже mirror, но можно и --level=0 (стрип) )
```
livecd dev # mdadm --create --verbose /dev/md2 --level=1 --raid-devices=2 \
/dev/sda3 /dev/sdb3
```

Проверяем готовность(может быть долго, в зависимости от размера )
```
livecd / # cat /proc/mdstat 
Personalities : [raid0] [raid1] [raid6] [raid5] [raid4] [raid10] 
md1 : active raid1 sdb2[1] sda2[0]
      4200896 blocks [2/2] [UU]
      
md2 : active raid0 sdb3[1] sda3[0]
      67633408 blocks 64k chunks
	      
unused devices: <none>
```

Если долго, то можно медитировать на (обновляется раз в 2 сек)
```
livecd dev # watch cat /proc/mdstat
```

Форматируем корень, создаем и подключаем свап
```
livecd dev # mkfs.ext4 -j /dev/md1
livecd dev # mkswap /dev/sda1 && mkswap /dev/sdb1 
livecd dev # swapon -p 1 /dev/sda1 && swapon -p 1 /dev/sdb1
```

Создаем LVM2 ( 4Gb - на /usr/portage и 10Gb на /vm )
```
livecd dev # pvcreate /dev/md2
livecd dev # vgcreate vg /dev/md2 
livecd dev # lvcreate -L4G -nportage vg
livecd dev # lvcreate -L10G -nvm vg
livecd dev # mdadm -Es > /etc/mdadm.conf
```

Проверяем
```
livecd dev # vgs
  VG   #PV #LV #SN Attr   VSize  VFree 
  vg     1   1   0 wz--n- 64.50G 60.50G
livecd dev # lvs
  LV      VG   Attr   LSize Origin Snap%  Move Log Copy%  Convert
  portage vg   -wi-a- 4.00G
  vm      vg   -wi-a- 10.00G
```

Форматируем
```
livecd dev # mkfs.ext4 -j /dev/vg/portage
livecd dev # mkfs.ext4 -j /dev/vg/vm
```

Подключаем все подряд в /mnt/gentoo
```
livecd dev # mount /dev/md1 /mnt/gentoo/
livecd dev # cd /mnt/gentoo/
livecd gentoo # mkdir dev proc vm usr usr/portage
livecd gentoo # mount -t proc proc proc 
livecd gentoo # mount -o bind /dev dev
livecd gentoo # mount /dev/vg/portage usr/portage/
```

Сливаем и устанавиваем [stage3](http://distfiles.gentoo.org/releases/amd64/autobuilds/current-stage3/hardened/)
```
livecd gentoo # wget [зеркало]
livecd gentoo # tar xjpf stage3*
```

Сливаем и устанавливаем [portage](http://distfiles.gentoo.org/snapshots/portage-latest.tar.bz2)
```
livecd gentoo # wget [зеркало]
livecd gentoo # cd usr/
livecd usr # tar xjf ../portage*
```

Входим в систему, устанавливаем пароль на систему и  часовой пояс
```
livecd usr # cp -L /etc/resolv.conf /mnt/gentoo/etc/
livecd usr # chroot /mnt/gentoo /bin/bash
livecd / # env-update && source /etc/profile
livecd / # passwd
livecd / # cp /usr/share/zoneinfo/Asia/Yekaterinburg /etc/localtime
```

Собираем ядро
```
livecd / # emerge gentoo-sources
livecd / # cd /usr/src/linux
livecd linux # make menuconfig
```

Отмечаем обязательные опции
```
Device Drivers --->
[*] Multiple devices driver support (RAID and LVM)  --->
  <*>   RAID support
  [*]     Autodetect RAID arrays during kernel boot
  < >     Linear (append) mode
  <*>     RAID-0 (striping) mode
  <*>     RAID-1 (mirroring) mode

File Systems --->
< > Second extended fs support
  < > Ext3 journalling file system support
  <*> The Extended 4 (ext4) filesystem
  [*]   Use ext4 for ext2/ext3 file systems (NEW)

[*] Virtualization --->
    --- Virtualization
    <*> Kernel-based Virtual Machine (KVM) support
    <*>   KVM for Intel processors support 
   < >   KVM for AMD processors support

Device Drivers --->
    [*] Network device support --->
            <*> Universal TUN/TAP device driver support

Networking support --->
    Networking options --->
        <*> 802.1d Ethernet Bridging
        <*> 802.1Q VLAN Support
```

Компилируем и устанавливаем ядро
```
livecd linux # make && make modules_install
livecd linux # cp arch/x86_64/boot/bzImage /boot 
livecd linux # cp arch/x86_64/boot/bzImage /boot/bzImage-stable
```

Я не могу без vim'а ))
```
	livecd etc # USE="-gpm" emerge vim && export EDITOR=vim
```

Реактируем таблицу разделов
```
livecd etc # vi /etc/fstab
/dev/md1                /               ext4            noatime         0 1
/dev/sda1               none            swap            sw,pri=1        0 0     
/dev/sdb1               none            swap            sw,pri=1        0 0     
/dev/vg/portage         /usr/portage    ext4            noatime         1 2
/dev/vg/vm              /vm             ext4            noatime         1 2
```

Применяем профиль
```
livecd etc # eselect profile list
Available profile symlink targets:
  [1]   default/linux/amd64/10.0 *
  [2]   default/linux/amd64/10.0/desktop
  [3]   default/linux/amd64/10.0/desktop/gnome
  [4]   default/linux/amd64/10.0/desktop/kde
  [5]   default/linux/amd64/10.0/developer
  [6]   default/linux/amd64/10.0/no-multilib
  [7]   default/linux/amd64/10.0/server
  [8]   hardened/linux/amd64/10.0
  [9]   hardened/linux/amd64/10.0/no-multilib
  [10]  selinux/2007.0/amd64
  [11]  selinux/2007.0/amd64/hardened
  [12]  selinux/v2refpolicy/amd64
  [13]  selinux/v2refpolicy/amd64/desktop
  [14]  selinux/v2refpolicy/amd64/developer
  [15]  selinux/v2refpolicy/amd64/hardened
  [16]  selinux/v2refpolicy/amd64/server
livecd init.d # eselect profile set 8
```

Проверяем make.conf, добавляем нужные флаги
```
# vi /etc/make.conf
CFLAGS="-O2 -pipe"
CXXFLAGS="${CFLAGS}"
CHOST="x86_64-pc-linux-gnu"
QEMU_USER_TARGETS="i386 x86_64"
QEMU_SOFTMMU_TARGETS="i386 x86_64"
USE="-gpm unicode mmx sse sse2"
```
Устанавливаем локаль
```
# echo 'en_US.UTF-8 UTF-8' > /etc/locale.gen
# echo 'ru_RU.UTF-8 UTF-8' >> /etc/locale.gen
# locale-gen
# echo 'LANG=en_US.UTF8' > /etc/env.d/02locale
# source /etc/profile
# mkdir /etc/portage
# echo '=app-emulation/qemu-kvm-0.12.5-r1' > /etc/portage/package.keywords
```

Устанавливаем скопом все нужные пакеты
```
# emerge mdadm lvm2 syslog-ng vixie-cron grub bridge-utils \
usermode-utilities usbutils qemu-kvm iptables
```

Настройки сети - серверу отдаем 10.96.0.3, гости возьмут себе сами (при условии что в сети есть DHCP сервер)
```
livecd etc # groupadd kvm
livecd etc # cd conf.d
livecd conf.d # vi net

bridge_br0="eth0 tap0"
config_eth0=( "null" )
config_tap0=( "null" )

config_br0=( "10.96.0.3/24" )
routes_br0=( "default via 10.96.0.254" )
brctl_br0=( "stp off" )

tuntap_tap0="tap"
tunctl_tap0="-g kvm"

depend_br0(){
	need net.tap0
}
```

Устанавливаем загрузку сети при включении
```
livecd conf.d # cd /etc/init.d/
livecd init.d # ln -s net.lo net.br0
livecd init.d # ln -s net.lo net.tap0
livecd init.d # rc-update add net.br0 default
```

Устанавливаем загрузку лога, крона и ssh
```
livecd init.d # rc-update add syslog-ng default
livecd init.d # rc-update add vixie-cron default
livecd init.d # rc-update add sshd default
```

Настраиваем загрузчик на оба винчестера
```
livecd init.d # vi /boot/grub/grub.conf
default 0
timeout 8
splashimage=(hd0,1)/boot/grub/splash.xpm.gz

title Gentoo 
root (hd0,1)
kernel /boot/bzImage root=/dev/md1 rootfstype=ext4

title Gentoo stable kernel
root (hd0,1)
kernel /boot/bzImage-stable root=/dev/md1 rootfstype=ext4

livecd conf.d # grub
grub> root (hd0,1)
grub> setup (hd0)
grub> root (hd1,1)
grub> setup (hd1)
grub> quit
```

Перезагружаемся
```
livecd conf.d # exit
livecd usr # cd /
livecd / # umount /mnt/gentoo/proc /mnt/gentoo/dev /mnt/gentoo/usr/portage /mnt/gentoo
livecd / # reboot
```

После ребута обновляем все пакеты (долго)
```
livecd / # emerge -uND world
```

Наслаждаемся.

## Если все плохо
Если не смогли загрузиться, то грузимся еще раз с cd

В случае каких-либо проблем, для того чтобы поднять файловую систему с CD
```
cd /dev
MAKEDEV md
mdadm --assemble /dev/md1 /dev/sda2 /dev/sdb2
mdadm --assemble /dev/md2 /dev/sda3 /dev/sdb3
vgchange -a y
```

## Проверка raid
Допустим, один из винчестеров умер.

Выключаем машину, ставим другой винчестер вместо, к примеру, первого. И берем другой размер(в моем случае: старый - 40, новый - 80)

Копируем таблицу раделов с первого на второй
```
# sfdisk -d /dev/sda | sfdisk /dev/sdb
# mkswap /dev/sdb1
# mdadm --manage /dev/md1 --add /dev/sdb2
# mdadm --manage /dev/md2 --add /dev/sdb3
```

Наблюдаем за восстановлением
```
# watch cat /proc/mdstat
# grub
root (hd1,1)
setup (hd1)
```

## Установка гостя (Linux-1C)
1 вариант - в виде файла
```
# cd /vm
# qemu-img create -f qcow2 1c.img 8G
```

2 вариант - выделяем место на LVM
```
# lvcreate -L8G -n1c vg
```

Копируем загрузочный DVD в систему для загрузки с него гостя
```
# scp arrrght@10.96.0.141:/Users/arrrght/Distrib/livedvd-amd64-multilib-10.1.iso ./

# vi /start-1c.sh
#!/bin/sh
/usr/bin/kvm \
	-drive file=/dev/vg/1c,if=virtio,boot=on \
	-m 2096 \
	-smp 2 \
	-cdrom /vm/livedvd-amd64-multilib-10.1.iso \
	-net nic,model=virtio \
	-net tap,ifname=tap0,script=no,downscript=no \
	-vnc :1 \
	-boot d
```

Подключаемся через vnc-клиент к 10.96.0.3:5901 и проводим почти те же действия:
```
# passwd
# net-setup eth0
# /etc/init.d/sshd start
```

Подключаемся, затем разбиваем диск на swap(1G) и остальное, и далее по накатанной.

Проше будет с сетью:
```
# echo 'config_eth0=( "10.96.0.98/24" )' > /etc/conf.d/net
# echo 'routes_eth0=( "default via 10.96.0.254" )' >> /etc/conf.d/net
# rc-update add net.eth0 default
```

Одно дополнение при установке Grub:
```
# echo '(hd0) /dev/vda' > /boot/grub/device.map
# grub --device-map=/boot/grub/device.map
> root (hd0,1)
> setup (hd0)
> quit
```

TODO: Дальше (собственно установка 1С)

***PS: 2013 год. Все делается намного проще. Будет время - напишу***
