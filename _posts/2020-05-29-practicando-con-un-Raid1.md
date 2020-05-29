---
title: Practicando con un RAID 1
author: Paloma R. García Campón
date: 2020-05-29 00:00:00 +0800
categories: [Seguridad]
tags: [RAID, Vagrant]
---

![raid1](/assets/img/sample/practica-raid1.png)

En este post se van a realizar una serie de ejercicios para practicar la gestión del almacenamiento de la información.

> Se crea una máquina virtual con un sistema operativo Linux. Esta máquina va a tener conectada dos discos virtuales de 1 GB. Se debe instalar el software necesario para crear un raid 0 con dichos discos:

#### Definición del fichero Vagrant para definir la máquina con los discos. Y se instala el programa mdadm que permite la construcción del RAID.

~~~
# -- mode: ruby --
 # vi: set ft=ruby :0

 Vagrant.configure("2") do |config|
  disco = ".vagrant/disco1.vdi" 
  disco2 = ".vagrant/disco2.vdi"  
  config.vm.box = "debian/buster64" 
  config.vm.hostname = "Almacenamiento" 
  config.vm.network :public_network,:bridge=>"wlp1s0"   
  config.vm.provider :virtualbox do |v|
    if not File.exist?(disco)
      v.customize ["createhd", "--filename", disco, "--size", 1 *  1024]
    end
      v.customize ["storageattach", :id, "--storagectl", "SATA Controller", "--port", 1,"--device", 0, "--type", "hdd", "--medium", disco]
    if not File.exist?(disco2)
      v.customize ["createhd", "--filename", disco2, "--size", 1 * 1024]
    end
      v.customize ["storageattach", :id, "--storagectl", "SATA Controller", "--port", 2,"--device", 0, "--type", "hdd", "--medium", disco2]
  config.vm.provision "shell", run: "always",
    inline: "sudo apt-get install -y mdadm" 
  end
 end
~~~


#### Se crea una raid llamado md0 con los dos discos conectados a la máquina.

~~~
vagrant@Almacenamiento:~$ sudo mdadm --create /dev/md1 --level=1 --raid-device=2 /dev/sdb /dev/sdc
mdadm: /dev/sdb appears to be part of a raid array:
       level=raid1 devices=2 ctime=Fri Sep 27 18:43:38 2019
mdadm: partition table exists on /dev/sdb but will be lost or
       meaningless after creating array
mdadm: Note: this array has metadata at the start and
    may not be suitable as a boot device.  If you plan to
    store '/boot' on this device please ensure that
    your boot-loader understands md/v1.x metadata, or use
    --metadata=0.90
mdadm: /dev/sdc appears to be part of a raid array:
       level=raid1 devices=2 ctime=Fri Sep 27 18:43:38 2019
Continue creating array? y
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md1 started.
~~~


#### Se comprueba las características y el estado del RAID.

~~~
vagrant@Almacenamiento:~$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda      8:0    0 19.8G  0 disk  
├─sda1   8:1    0 18.8G  0 part  /
├─sda2   8:2    0    1K  0 part  
└─sda5   8:5    0 1021M  0 part  [SWAP]
sdb      8:16   0    1G  0 disk  
└─md1    9:0    0 1022M  0 raid1 
sdc      8:32   0    1G  0 disk  
└─md1    9:0    0 1022M  0 raid1 
vagrant@Almacenamiento:~$ sudo mdadm --detail /dev/md1
/dev/md1:
           Version : 1.2
     Creation Time : Fri Sep 27 18:47:40 2019
        Raid Level : raid1
        Array Size : 1046528 (1022.00 MiB 1071.64 MB)
     Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
      Raid Devices : 2
     Total Devices : 2
       Persistence : Superblock is persistent

       Update Time : Fri Sep 27 18:47:46 2019
             State : clean 
    Active Devices : 2
   Working Devices : 2
    Failed Devices : 0
     Spare Devices : 0

Consistency Policy : resync

              Name : Almacenamiento:0  (local to host Almacenamiento)
              UUID : 52dc25de:6db58569:a5090dd2:fa90c880
            Events : 17

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
~~~

El raid tiene 1GB de capacidad.


#### Creación de una partición primaria de 500Mb en el raid0.

~~~
vagrant@Almacenamiento:~$ sudo fdisk /dev/md1

Welcome to fdisk (util-linux 2.33.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0x12327ca2.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 
First sector (2048-2093055, default 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-2093055, default 2093055): +500M

Created a new partition 1 of type 'Linux' and of size 500 MiB.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
~~~

#### Se va a formatear esa partición con un sistema de archivo ext3.

~~~
vagrant@Almacenamiento:~$ sudo mkfs.ext3 /dev/md1p1
mke2fs 1.44.5 (15-Dec-2018)
Creating filesystem with 512000 1k blocks and 128016 inodes
Filesystem UUID: 91bfcc07-3d5f-40a1-98be-6b2e97b247d5
Superblock backups stored on blocks: 
    8193, 24577, 40961, 57345, 73729, 204801, 221185, 401409

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done 

vagrant@Almacenamiento:~$ lsblk -f
NAME FSTYPE LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINT
sda                                                                   
├─sda1
│    ext4         b9ffc3d1-86b2-4a2c-a8be-f2b2f4aa4cb5   16.4G     6% /
├─sda2
│                                                                     
└─sda5
     swap         f8f6d279-1b63-4310-a668-cb468c9091d8                [SWAP]
sdb  linux_ Almacenamiento:0
│                 52dc25de-6db5-8569-a509-0dd2fa90c880                
└─md1

  └─md1p1
     ext3         91bfcc07-3d5f-40a1-98be-6b2e97b247d5                
sdc  linux_ Almacenamiento:0
│                 52dc25de-6db5-8569-a509-0dd2fa90c880                
└─md1

  └─md1p1
     ext3         91bfcc07-3d5f-40a1-98be-6b2e97b247d5  
~~~


#### Y se monta la partición en el directorio /mnt/raid0. Además, se crea un fichero.

~~~
vagrant@Almacenamiento:/mnt$ sudo mkdir /mnt/raid0
vagrant@Almacenamiento:/mnt$ sudo mount /dev/md1p1 /mnt/raid0
vagrant@Almacenamiento:/mnt$ lsblk
NAME      MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda         8:0    0 19.8G  0 disk  
├─sda1      8:1    0 18.8G  0 part  /
├─sda2      8:2    0    1K  0 part  
└─sda5      8:5    0 1021M  0 part  [SWAP]
sdb         8:16   0    1G  0 disk  
└─md1       9:0    0 1022M  0 raid1 
  └─md1p1   259:1    0  500M  0 part  /mnt/raid0
sdc         8:32   0    1G  0 disk  
└─md1       9:0    0 1022M  0 raid1 
  └─md1p1   259:1    0  500M  0 part  /mnt/raid0
~~~

Para hacerlo de forma permanente editamos el fichero fstab con el siguiente formato:

~~~
UUID=91bfcc07-3d5f-40a1-98be-6b2e97b247d5       /mnt/raid0     ext3 
~~~


#### Se va a simular que un disco se estropea y comprobar cómo se peude seguir accediendo al fichero.

~~~
:/mnt/raid0$ sudo mdadm /dev/md1 -f /dev/sdb1
mdadm: stat failed for /dev/sdb1: No such file or directory
You have mail in /var/mail/vagrant

vagrant@Almacenamiento:/mnt/raid0$ sudo mdadm --detail /dev/md1
/dev/md1:
           Version : 1.2
     Creation Time : Mon Sep 30 10:34:45 2019
        Raid Level : raid1
        Array Size : 1046528 (1022.00 MiB 1071.64 MB)
     Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
      Raid Devices : 2
     Total Devices : 2
       Persistence : Superblock is persistent

       Update Time : Mon Sep 30 10:38:01 2019
             State : clean 
    Active Devices : 1
   Working Devices : 1
    Failed Devices : 1
     Spare Devices : 0

Consistency Policy : resync

              Name : Almacenamiento:1  (local to host Almacenamiento)
              UUID : 3bf5f319:97fef2ae:8f298378:9a9f3cf7
            Events : 17

    Number   Major   Minor   RaidDevice State
       -       0        0        0      removed

       1       8       32        1      active sync   /dev/sdc

       0       8       16        -      faulty   /dev/sdb
~~~

~~~
vagrant@Almacenamiento:/mnt/raid0$ ls
fic_prueba  lost+found
~~~


#### Se recupera el estado del raid y se comprueba si podemos seguir accediendo al fichero.

~~~
vagrant@Almacenamiento:/mnt/raid0$ sudo mdadm --remove /dev/md1 /dev/sdb
mdadm: hot removed /dev/sdb from /dev/md1

vagrant@Almacenamiento:/mnt/raid0$ lsblk
NAME      MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda         8:0    0 19.8G  0 disk  
├─sda1      8:1    0 18.8G  0 part  /
├─sda2      8:2    0    1K  0 part  
└─sda5      8:5    0 1021M  0 part  [SWAP]
sdb         8:16   0    1G  0 disk  
sdc         8:32   0    1G  0 disk  
└─md1       9:0    0 1022M  0 raid1 
  └─md1p1   259:1    0  500M  0 part  /mnt/raid0
vagrant@Almacenamiento:/mnt/raid0$ sudo mdadm /dev/md1 --add /dev/sdb
mdadm: added /dev/sdb
vagrant@Almacenamiento:/mnt/raid0$ lsblk
NAME      MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda         8:0    0 19.8G  0 disk  
├─sda1      8:1    0 18.8G  0 part  /
├─sda2      8:2    0    1K  0 part  
└─sda5      8:5    0 1021M  0 part  [SWAP]
sdb         8:16   0    1G  0 disk  
└─md1       9:0    0 1022M  0 raid1 
  └─md1p1   259:1    0  500M  0 part  /mnt/raid0
sdc         8:32   0    1G  0 disk  
└─md1       9:0    0 1022M  0 raid1 
  └─md1p1   259:1    0  500M  0 part  /mnt/raid0

vagrant@Almacenamiento:~$ sudo mdadm --detail /dev/md1
/dev/md1:
           Version : 1.2
     Creation Time : Mon Sep 30 10:34:45 2019
        Raid Level : raid1
        Array Size : 1046528 (1022.00 MiB 1071.64 MB)
     Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
      Raid Devices : 2
     Total Devices : 2
       Persistence : Superblock is persistent

       Update Time : Mon Sep 30 10:40:41 2019
             State : clean 
    Active Devices : 2
   Working Devices : 2
    Failed Devices : 0
     Spare Devices : 0

Consistency Policy : resync

              Name : Almacenamiento:1  (local to host Almacenamiento)
              UUID : 3bf5f319:97fef2ae:8f298378:9a9f3cf7
            Events : 17

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sd
~~~


#### Se añade un nuevo disco.
Volvermos a modificar el fichero Vagrantfile para añadir el nuevo disco:

~~~
# -- mode: ruby --
 # vi: set ft=ruby :0

 Vagrant.configure("2") do |config|
  disco = ".vagrant/disco1.vdi" 
  disco2 = ".vagrant/disco2.vdi" 
  disco3=".vagrant/disco3.vdi"  
  config.vm.box = "debian/buster64" 
  config.vm.hostname = "Almacenamiento" 
  config.vm.network :public_network,:bridge=>"wlp1s0"   
  config.vm.provider :virtualbox do |v|
    if not File.exist?(disco)
      v.customize ["createhd", "--filename", disco, "--size", 1 *  1024]
    end
      v.customize ["storageattach", :id, "--storagectl", "SATA Controller", "--port", 1,"--device", 0, "--type", "hdd", "--medium", disco]
    if not File.exist?(disco2)
      v.customize ["createhd", "--filename", disco2, "--size", 1 * 1024]
    end
      v.customize ["storageattach", :id, "--storagectl", "SATA Controller", "--port", 2,"--device", 0, "--type", "hdd", "--medium", disco2] 
    if not File.exist?(disco3)
      v.customize ["createhd", "--filename", disco3, "--size", 1 *  1024]
    end
      v.customize ["storageattach", :id, "--storagectl", "SATA Controller", "--port", 3,"--device", 0, "--type", "hdd", "--medium", disco3]

  config.vm.provision "shell", run: "always",
    inline: "sudo apt-get install -y mdadm" 
  end
 end
~~~

Y añadimos el nuevo disco al RAID.

~~~
vagrant@Almacenamiento:~$ sudo mdadm --grow /dev/md1 --raid-device=3 --add /dev/sdd
mdadm: added /dev/sdd
raid_disks for /dev/md1 set to 3

vagrant@Almacenamiento:~$ sudo mdadm --detail /dev/md1
/dev/md1:
           Version : 1.2
     Creation Time : Mon Sep 30 10:34:45 2019
        Raid Level : raid1
        Array Size : 1046528 (1022.00 MiB 1071.64 MB)
     Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
      Raid Devices : 3
     Total Devices : 3
       Persistence : Superblock is persistent

       Update Time : Wed Oct  2 18:03:48 2019
             State : clean 
    Active Devices : 3
   Working Devices : 3
    Failed Devices : 0
     Spare Devices : 0

Consistency Policy : resync

              Name : Almacenamiento:1  (local to host Almacenamiento)
              UUID : 3bf5f319:97fef2ae:8f298378:9a9f3cf7
            Events : 40

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
~~~


#### Se redimensiona la partición y el sistema de archivo de 500Mb al tamaño del raid.

~~~
vagrant@Almacenamiento:~$ sudo fdisk /dev/md1

Welcome to fdisk (util-linux 2.33.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Command (m for help): d
Selected partition 1
Partition 1 has been deleted.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

vagrant@Almacenamiento:~$ sudo fdisk /dev/md1

Welcome to fdisk (util-linux 2.33.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): 

Using default response p.
Partition number (1-4, default 1): 
First sector (2048-2093055, default 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-2093055, default 2093055): 

Created a new partition 1 of type 'Linux' and of size 1021 MiB.
Partition #1 contains a ext3 signature.

Do you want to remove the signature? [Y]es/[N]o: n

Command (m for help): w

The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

vagrant@Almacenamiento:~$ lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda           8:0    0 19.8G  0 disk  
├─sda1        8:1    0 18.8G  0 part  /
├─sda2        8:2    0    1K  0 part  
└─sda5        8:5    0 1021M  0 part  [SWAP]
sdb           8:16   0    1G  0 disk  
└─md1         9:127  0 1022M  0 raid1 
  └─md1p1     259:1  0 1021M  0 part  
sdc           8:32   0    1G  0 disk  
└─md1         9:127  0 1022M  0 raid1 
  └─md1p1     259:1  0 1021M  0 part  
sdd           8:48   0    1G  0 disk  
└─md1         9:127  0 1022M  0 raid1 
  └─md1p1     259:1  0 1021M  0 part

vagrant@Almacenamiento:~$ sudo resize2fs  /dev/md1p1
resize2fs 1.44.5 (15-Dec-2018)
Filesystem at /dev/md1p1 is mounted on /mnt/raid0; on-line resizing required
old_desc_blocks = 2, new_desc_blocks = 4
The filesystem on /dev/md1p1 is now 1045504 (1k) blocks long.
~~~






