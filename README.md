# Работа с LVM

На имеющемся образе centos/7 - 1804.02:

1) Уменьшить том под / до 8G
2) Выделить том под /home
3) Выделить том под /var - сделаем в mirror
4) /home - сделать том для снапшотов
5) Прописать монтирование в fstab

Работа со снапшотами:

- сгенерить файлы в /home/
- снять снапшот
- удалить часть файлов
- восстановится со снапшота


## Уменьшение тома под / до 8G

- Посмотреть на диски

```bash
[vagrant@lvm ~]$ sudo -i
[root@lvm ~]# lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk 
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk 
```

- Подготовить временный том для / раздела

```
root@lvm ~]# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
```
- Создать VG  vg_root
```
[root@lvm ~]# vgcreate vg_root /dev/sdb
  Volume group "vg_root" successfully created
```
- Создать логический том  lv_root
```
[root@lvm ~]# lvcreate -n lv_root -l +100%FREE /dev/vg_root
WARNING: ext4 signature detected on /dev/vg_root/lv_root at offset 1080. Wipe it? [y/n]: y
  Wiping ext4 signature on /dev/vg_root/lv_root.
  Logical volume "lv_root" created.
```

- Создать на нем файловую систему и смонтировать его, чтобы перенести туда данные

```
[root@lvm ~]# mkfs.xfs /dev/vg_root/lv_root
meta-data=/dev/vg_root/lv_root   isize=512    agcount=4, agsize=655104 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2620416, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```

```
[root@lvm ~]# mount /dev/vg_root/lv_root /mnt
```

- Копируем все данные с / раздела в /mnt

```
root@lvm ~]# xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt

xfsrestore: using file dump (drive_simple) strategy
xfsrestore: version 3.1.7 (dump format 3.0)
xfsdump: using file dump (drive_simple) strategy
xfsdump: version 3.1.7 (dump format 3.0)
xfsdump: level 0 dump of lvm:/
xfsdump: dump date: Sun Dec  3 05:35:58 2023
xfsdump: session id: 16e31a06-3da3-42e6-b816-ab6c6a9b4a63
xfsdump: session label: ""
xfsdump: ino map phase 1: constructing initial dump list
xfsrestore: searching media for dump
xfsdump: ino map phase 2: skipping (no pruning necessary)
xfsdump: ino map phase 3: skipping (only one dump stream)
xfsdump: ino map construction complete
xfsdump: estimated dump size: 1129366144 bytes
xfsdump: creating dump session media file 0 (media 0, file 0)
xfsdump: dumping ino map
xfsdump: dumping directories
xfsrestore: examining media file 0
xfsrestore: dump description: 
xfsrestore: hostname: lvm
xfsrestore: mount point: /
xfsrestore: volume: /dev/mapper/VolGroup00-LogVol00
xfsrestore: session time: Sun Dec  3 05:35:58 2023
xfsrestore: level: 0
xfsrestore: session label: ""
xfsrestore: media label: ""
xfsrestore: file system id: b60e9498-0baa-4d9f-90aa-069048217fee
xfsrestore: session id: 16e31a06-3da3-42e6-b816-ab6c6a9b4a63
xfsrestore: media id: 1b1857a5-33fd-4520-9e14-6c41117949d3
xfsrestore: searching media for directory dump
xfsrestore: reading directories
xfsdump: dumping non-directory files
xfsrestore: 2732 directories and 23682 entries processed
xfsrestore: directory post-processing
xfsrestore: restoring non-directory files
xfsdump: ending media file
xfsdump: media file size 1106454920 bytes
xfsdump: dump size (non-dir files) : 1093235960 bytes
xfsdump: dump complete: 7 seconds elapsed
xfsdump: Dump Status: SUCCESS
xfsrestore: restore complete: 7 seconds elapsed
xfsrestore: Restore Status: SUCCESS
```
- Проверить 
```
[root@lvm ~]# ls /mnt
bin   data       dev  home  lib64  mnt  proc  run   srv  tmp  vagrant
boot  data-snap  etc  lib   media  opt  root  sbin  sys  usr  var
```
- Переконфигурировать grub для того, чтобы при старте перейти в новый /

Сымитировать текущий **root** -> сделать в него [chroot](https://wiki.archlinux.org/title/Chroot_(%D0%A0%D1%83%D1%81%D1%81%D0%BA%D0%B8%D0%B9)) и обновим **grub**:

```
[root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
[root@lvm ~]# chroot /mnt/
[root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done
```

- Обновить образ **initrd**

Загрузочный RAM диск (Initrd) - это образ корневой файловой системы (фс),
который монтируется до того как настоящая корневая фс будет доступна.
Initrd связан с ядром и загружается как часть ядра в процессе загрузки
системы. Ядро монтирует образ initrd в котором находятся необходимые
модули для монтирования корневой фс и уже дальнейшего перехода в этот
корень как основной.

В initrd содержится минимальный набор директорий и исполняемых файлов
для загрузки модулей, например insmod для загрузки модулей ядра.

В случае настольной системы или сервера, initrd временная файловая
система. Её время жизни коротко и служит лишь связующим звеном к
корневой фс.

```
[root@lvm /]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;
> s/.img//g"` --force; done

Executing: /sbin/dracut -v initramfs-3.10.0-862.2.3.el7.x86_64.img 3.10.0-862.2.3.el7.x86_64 --force
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
*** Including module: bash ***
*** Including module: nss-softokn ***
*** Including module: i18n ***
*** Including module: drm ***
*** Including module: plymouth ***
*** Including module: dm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 60-persistent-storage-dm.rules
Skipping udev rule: 55-dm.rules
*** Including module: kernel-modules ***
Omitting driver floppy
*** Including module: lvm ***
Skipping udev rule: 64-device-mapper.rules
Skipping udev rule: 56-lvm.rules
Skipping udev rule: 60-persistent-storage-lvm.rules
*** Including module: qemu ***
*** Including module: resume ***
*** Including module: rootfs-block ***
*** Including module: terminfo ***
*** Including module: udev-rules ***
Skipping udev rule: 40-redhat-cpu-hotplug.rules
Skipping udev rule: 91-permissions.rules
*** Including module: biosdevname ***
*** Including module: systemd ***
*** Including module: usrmount ***
*** Including module: base ***
*** Including module: fs-lib ***
*** Including module: shutdown ***
*** Including modules done ***
*** Installing kernel module dependencies and firmware ***
*** Installing kernel module dependencies and firmware done ***
*** Resolving executable dependencies ***
*** Resolving executable dependencies done***
*** Hardlinking files ***
*** Hardlinking files done ***
*** Stripping files ***
*** Stripping files done ***
*** Generating early-microcode cpio image contents ***
*** Constructing AuthenticAMD.bin ****
*** No early-microcode cpio image needed ***
*** Store current command line parameters ***
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
```

- Для монтирования при загрузке  нужного root внести изменения в файле /boot/grub2/grub.cfg заменитþ rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_roo

```
sed -i 's/rd\.lvm\.lv=VolGroup00\/LogVol00/rd\.lvm\.lv=vg_root\/lv_root/g' /boot/grub2/grub.cfg
```

- Перезагрузится и проверить

```
shutdown -r now
```
```
[vagrant@lvm ~]$ lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00 253:1    0 37.5G  0 lvm  
  └─VolGroup00-LogVol01 253:2    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk 
└─vg_root-lv_root       253:0    0   10G  0 lvm  /
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk
```

- Теперь нам нужно изменить размер старой VG и вернуть на него рут. Для этого удалить старый LV размеров в 40G и создать новый на 8G

```[vagrant@lvm ~]$ sudo -i
[root@lvm ~]# lvremove /dev/VolGroup00/LogVol00
Do you really want to remove active logical volume VolGroup00/LogVol00? [y/n]: y
  Logical volume "LogVol00" successfully removed
[root@lvm ~]# lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00
WARNING: xfs signature detected on /dev/VolGroup00/LogVol00 at offset 0. Wipe it? [y/n]: y
  Wiping xfs signature on /dev/VolGroup00/LogVol00.
  Logical volume "LogVol00" created.
```

- Выполнить команды по созданию на нем файловой системы, монтировнию и копированию всех данных с / раздела в /mnt:

```
[root@lvm ~]# mkfs.xfs /dev/VolGroup00/LogVol00
meta-data=/dev/VolGroup00/LogVol00 isize=512    agcount=4, agsize=524288 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2097152, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@lvm ~]# mount /dev/VolGroup00/LogVol00 /mnt
[root@lvm ~]# xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt
xfsrestore: using file dump (drive_simple) strategy
xfsrestore: version 3.1.7 (dump format 3.0)
xfsdump: using file dump (drive_simple) strategy
xfsdump: version 3.1.7 (dump format 3.0)
xfsdump: level 0 dump of lvm:/
xfsdump: dump date: Sun Dec  3 06:41:00 2023
xfsdump: session id: 03320368-fbd6-4348-a712-78b3927767b7
xfsdump: session label: ""
xfsrestore: searching media for dump
xfsdump: ino map phase 1: constructing initial dump list
xfsdump: ino map phase 2: skipping (no pruning necessary)
xfsdump: ino map phase 3: skipping (only one dump stream)
xfsdump: ino map construction complete
xfsdump: estimated dump size: 1131560128 bytes
xfsdump: creating dump session media file 0 (media 0, file 0)
xfsdump: dumping ino map
xfsdump: dumping directories
xfsrestore: examining media file 0
xfsrestore: dump description: 
xfsrestore: hostname: lvm
xfsrestore: mount point: /
xfsrestore: volume: /dev/mapper/vg_root-lv_root
xfsrestore: session time: Sun Dec  3 06:41:00 2023
xfsrestore: level: 0
xfsrestore: session label: ""
xfsrestore: media label: ""
xfsrestore: file system id: b7b73516-1a9b-4b04-87fc-627bf9d2c1e0
xfsrestore: session id: 03320368-fbd6-4348-a712-78b3927767b7
xfsrestore: media id: 72a74280-f997-47a9-b920-6974bb04c8f9
xfsrestore: searching media for directory dump
xfsrestore: reading directories
xfsdump: dumping non-directory files
xfsrestore: 2775 directories and 23831 entries processed
xfsrestore: directory post-processing
xfsrestore: restoring non-directory files
xfsdump: ending media file
xfsdump: media file size 1108546192 bytes
xfsdump: dump size (non-dir files) : 1095229936 bytes
xfsdump: dump complete: 6 seconds elapsed
xfsdump: Dump Status: SUCCESS
xfsrestore: restore complete: 6 seconds elapsed
xfsrestore: Restore Status: SUCCESS
```

- Переконфигурировать grub, за исключением правки /etc/grub2/grub.cfg

```
[root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
[root@lvm ~]# chroot /mnt/
[root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done
```

```
[root@lvm /]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;
> s/.img//g"` --force; done
Executing: /sbin/dracut -v initramfs-3.10.0-862.2.3.el7.x86_64.img 3.10.0-862.2.3.el7.x86_64 --force

......

*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
```

> Чтобы выполнить сразу перенос /var, пока не перезагружаться и не выходить из под chroot.

## Выделение тома под /var в зеркало

- На свободных дисках создать зеркало

```
[root@lvm boot]# pvcreate /dev/sdc /dev/sdd
  Physical volume "/dev/sdc" successfully created.
  Physical volume "/dev/sdd" successfully created.
```

```[root@lvm boot]# vgcreate vg_var /dev/sdc /dev/sdd
  Volume group "vg_var" successfully created
```

```
[root@lvm boot]# lvcreate -L 950M -m1 -n lv_var vg_var
  Rounding up size to full physical extent 952.00 MiB
  Logical volume "lv_var" created.
```

- Создать на нем ФС и переместить туда /var

```
[root@lvm boot]# mkfs.ext4 /dev/vg_var/lv_var
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
60928 inodes, 243712 blocks
12185 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=249561088
8 block groups
32768 blocks per group, 32768 fragments per group
7616 inodes per group
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done
```

```
[root@lvm boot]# mount /dev/vg_var/lv_var /mnt

[root@lvm boot]# cp -aR /var/* /mnt/ # rsync -avHPSAX /var/ /mnt/
```

- Сохранить содержимое старого var ( на всякий случай) или просто удалить

```
[root@lvm boot]# mkdir /tmp/oldvar && mv /var/* /tmp/oldvar
```

- Смонтировать новый var в каталог /var

```
[root@lvm boot]# umount /mnt
[root@lvm boot]# mount /dev/vg_var/lv_var /var
```

- Изменить fstab для автоматического монтирования /var

```
[root@lvm boot]# echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab

```

```
[root@lvm boot]# cat /etc/fstab 

#
# /etc/fstab
# Created by anaconda on Sat May 12 18:50:26 2018
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/VolGroup00-LogVol00 /                       xfs     defaults        0 0
UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot                   xfs     defaults        0 0
/dev/mapper/VolGroup00-LogVol01 swap                    swap    defaults        0 0
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
#VAGRANT-END
UUID="3e0662f0-4652-432b-b49c-f77085d63f13" /var ext4 defaults 0 0
```

- После чего можно успешно перезагружаться в новый (уменьшенный root) и удалять временную Volume Group
```
NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                        8:0    0   40G  0 disk 
├─sda1                     8:1    0    1M  0 part 
├─sda2                     8:2    0    1G  0 part /boot
└─sda3                     8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00  253:0    0    8G  0 lvm  /
  └─VolGroup00-LogVol01  253:1    0  1.5G  0 lvm  [SWAP]
sdb                        8:16   0   10G  0 disk 
└─vg_root-lv_root        253:7    0   10G  0 lvm  
sdc                        8:32   0    2G  0 disk 
├─vg_var-lv_var_rmeta_0  253:2    0    4M  0 lvm  
│ └─vg_var-lv_var        253:6    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_0 253:3    0  952M  0 lvm  
  └─vg_var-lv_var        253:6    0  952M  0 lvm  /var
sdd                        8:48   0    1G  0 disk 
├─vg_var-lv_var_rmeta_1  253:4    0    4M  0 lvm  
│ └─vg_var-lv_var        253:6    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_1 253:5    0  952M  0 lvm  
  └─vg_var-lv_var        253:6    0  952M  0 lvm  /var
sde                        8:64   0    1G  0 disk 
```
```
[root@lvm ~]# lvremove /dev/vg_root/lv_root
Do you really want to remove active logical volume vg_root/lv_root? [y/n]: y
  Logical volume "lv_root" successfully removed
[root@lvm ~]# vgremove /dev/vg_root
  Volume group "vg_root" successfully removed
[root@lvm ~]# pvremove /dev/sdb
  Labels on physical volume "/dev/sdb" successfully wiped.
```
```
[root@lvm ~]# lsblk
NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                        8:0    0   40G  0 disk 
├─sda1                     8:1    0    1M  0 part 
├─sda2                     8:2    0    1G  0 part /boot
└─sda3                     8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00  253:0    0    8G  0 lvm  /
  └─VolGroup00-LogVol01  253:1    0  1.5G  0 lvm  [SWAP]
sdb                        8:16   0   10G  0 disk 
sdc                        8:32   0    2G  0 disk 
├─vg_var-lv_var_rmeta_0  253:2    0    4M  0 lvm  
│ └─vg_var-lv_var        253:6    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_0 253:3    0  952M  0 lvm  
  └─vg_var-lv_var        253:6    0  952M  0 lvm  /var
sdd                        8:48   0    1G  0 disk 
├─vg_var-lv_var_rmeta_1  253:4    0    4M  0 lvm  
│ └─vg_var-lv_var        253:6    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_1 253:5    0  952M  0 lvm  
  └─vg_var-lv_var        253:6    0  952M  0 lvm  /var
sde                        8:64   0    1G  0 disk 
```

## Выделение тома под /home

- Выделить том под /home по тому же принципу что делали для /var

```
[root@lvm ~]# lvcreate -n LogVol_Home -L 2G /dev/VolGroup00
  Logical volume "LogVol_Home" created.
[root@lvm ~]# mkfs.xfs /dev/VolGroup00/LogVol_Home
meta-data=/dev/VolGroup00/LogVol_Home isize=512    agcount=4, agsize=131072 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=524288, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@lvm ~]# mount /dev/VolGroup00/LogVol_Home /mnt/
[root@lvm ~]# cp -aR /home/* /mnt/
[root@lvm ~]# rm -rf /home/*
[root@lvm ~]# umount /mnt
[root@lvm ~]# mount /dev/VolGroup00/LogVol_Home /home/
```

- Изменить fstab для автоматического монтирования /home

```
[root@lvm ~]# echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab
```
```
[root@lvm ~]# cat /etc/fstab 

#
# /etc/fstab
# Created by anaconda on Sat May 12 18:50:26 2018
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/VolGroup00-LogVol00 /                       xfs     defaults        0 0
UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot                   xfs     defaults        0 0
/dev/mapper/VolGroup00-LogVol01 swap                    swap    defaults        0 0
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
#VAGRANT-END
UUID="3e0662f0-4652-432b-b49c-f77085d63f13" /var ext4 defaults 0 0
UUID="a65399dd-44d7-4992-b1b7-a36d054e7938" /home xfs defaults 0 0
```

## /home - сделаем том для снапшотов

- Сгенерировать файлы в /home/

```
[root@lvm ~]# touch /home/file{1..20}
```

- Снять снапшот

```
[root@lvm ~]# lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home
  Rounding up size to full physical extent 128.00 MiB
  Logical volume "home_snap" created.
```

- Удалить часть файлов

```
[root@lvm ~]# rm -f /home/file{11..20}

```

- Восстановить со снапшота

```
root@lvm ~]# umount /home
[root@lvm ~]# lvconvert --merge /dev/VolGroup00/home_snap
  Merging of volume VolGroup00/home_snap started.
  VolGroup00/LogVol_Home: Merged: 100.00%
[root@lvm ~]# mount /home

[root@lvm ~]# ls /home/file* | column -t
/home/file1
/home/file10
/home/file11
/home/file12
/home/file13
/home/file14
/home/file15
/home/file16
/home/file17
/home/file18
/home/file19
/home/file2
/home/file20
/home/file3
/home/file4
/home/file5
/home/file6
/home/file7
/home/file8
/home/file9
```
