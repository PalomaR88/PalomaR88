---
title: Practicando con un RAID 5
author: Paloma R. García Campón
date: 2020-05-29 00:00:00 +0800
categories: [Seguridad]
tags: [RAID, Vagrant]
---

![raid1](/assets/img/sample/raid5.png)

A continuación se van a realizar una serie de ejercicios para practicar con un RAID 5.

Para ello, se va a crear un RAID 5 en una máquina de 2 GB, utilizando discos virtuales de 1 GB. 

#### CReqación del RAID, llamado md5, con los discos que se han conectado a la máquina.

Discos:

~~~
vagrant@Almacenamiento:~$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0 19.8G  0 disk 
├─sda1   8:1    0 18.8G  0 part /
├─sda2   8:2    0    1K  0 part 
└─sda5   8:5    0 1021M  0 part [SWAP]
sdb      8:16   0    1G  0 disk 
sdc      8:32   0    1G  0 disk 
sdd      8:48   0    1G  0 disk 
sde      8:64   0    1G  0 disk 
~~~

Creación del raid:

~~~
vagrant@Almacenamiento:~$ sudo mdadm --create /dev/md5 --level=5 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md5 started.
~~~


#### Comprobación de las características:

Para ver los detalles del raid usamos el comando mdadm con la opción --detail.

~~~
vagrant@Almacenamiento:~$ sudo mdadm --detail /dev/md5 
/dev/md5:
           Version : 1.2
     Creation Time : Mon Oct  7 22:16:55 2019
        Raid Level : raid5
        Array Size : 2093056 (2044.00 MiB 2143.29 MB)
     Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
      Raid Devices : 3
     Total Devices : 3
       Persistence : Superblock is persistent

       Update Time : Mon Oct  7 22:17:06 2019
             State : clean 
    Active Devices : 3
   Working Devices : 3
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : Almacenamiento:5  (local to host Almacenamiento)
              UUID : 96f19c3a:a41ee026:203a6acb:294d3d83
            Events : 18

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       3       8       48        2      active sync   /dev/sdd
~~~

El raid que hemos creado tiene 2GB de capacidad.


#### Se crea un volumen lógico de 500Mb en el raid 5.

Para crear el volumen es necesario descargarse el paquete lvm2 y para crear el volumen se usa el siguiente comando:

~~~
vagrant@Almacenamiento:~$ sudo pvcreate /dev/md5
  Physical volume "/dev/md5" successfully created.
vagrant@Almacenamiento:~$ sudo vgcreate raid5vol /dev/md5
  Volume group "raid5vol" successfully created
vagrant@Almacenamiento:~$ sudo lvcreate raid5vol -L 500M -n storage
  Logical volume "storage" created.
~~~

Comprobamos que se ha realizado correctamente:

~~~
vagrant@Almacenamiento:~$ lsblk
NAME                 MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda                    8:0    0 19.8G  0 disk  
├─sda1                 8:1    0 18.8G  0 part  /
├─sda2                 8:2    0    1K  0 part  
└─sda5                 8:5    0 1021M  0 part  [SWAP]
sdb                    8:16   0    1G  0 disk  
└─md5                  9:5    0    2G  0 raid5 
  └─raid5vol-storage 253:0    0  500M  0 lvm   
sdc                    8:32   0    1G  0 disk  
└─md5                  9:5    0    2G  0 raid5 
  └─raid5vol-storage 253:0    0  500M  0 lvm   
sdd                    8:48   0    1G  0 disk  
└─md5                  9:5    0    2G  0 raid5 
  └─raid5vol-storage 253:0    0  500M  0 lvm   
sde                    8:64   0    1G  0 disk  
~~~


#### Se formatea con un sistema de archivo xfs.

Para formatear con el sistema de archivo xfs es necesario el paquete xfsprogs.

~~~
vagrant@Almacenamiento:~$ sudo mkfs.xfs -f /dev/mapper/raid5vol-storage
meta-data=/dev/mapper/raid5vol-storage isize=512    agcount=8, agsize=16000 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=0
data     =                       bsize=4096   blocks=128000, imaxpct=25
         =                       sunit=128    swidth=256 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=896, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
vagrant@Almacenamiento:~$ lsblk -f
NAME FSTYPE LABEL UUID                                   FSAVAIL FSUSE% MOUNTPOINT
sda                                                                     
├─sda1
│    ext4         b9ffc3d1-86b2-4a2c-a8be-f2b2f4aa4cb5     16.4G     6% /
├─sda2
│                                                                       
└─sda5
     swap         f8f6d279-1b63-4310-a668-cb468c9091d8                  [SWAP]
sdb  linux_ Almacenamiento:5
│                 96f19c3a-a41e-e026-203a-6acb294d3d83                  
└─md5
     LVM2_m       iAP8kO-hRfR-6IiS-mo8G-z6N8-Wf7V-uDKYcy                
  └─raid5vol-storage
     xfs          086739e0-490e-4dc4-954e-ec96ed48ff66                  
sdc  linux_ Almacenamiento:5
│                 96f19c3a-a41e-e026-203a-6acb294d3d83                  
└─md5
     LVM2_m       iAP8kO-hRfR-6IiS-mo8G-z6N8-Wf7V-uDKYcy                
  └─raid5vol-storage
     xfs          086739e0-490e-4dc4-954e-ec96ed48ff66                  
sdd  linux_ Almacenamiento:5
│                 96f19c3a-a41e-e026-203a-6acb294d3d83                  
└─md5
     LVM2_m       iAP8kO-hRfR-6IiS-mo8G-z6N8-Wf7V-uDKYcy                
  └─raid5vol-storage
     xfs          086739e0-490e-4dc4-954e-ec96ed48ff66                  
sde    
~~~


#### Se monta el volumen en el directorio /mnt/raid5 y se crea un fichero.

~~~
vagrant@Almacenamiento:~$ sudo mkdir /mnt/raid5
vagrant@Almacenamiento:~$ sudo mount /dev/mapper/raid5vol-storage /mnt/raid5/
vagrant@Almacenamiento:~$ sudo touch /mnt/raid5/prueba.md
vagrant@Almacenamiento:~$ ls /mnt/raid5/
prueba.md
~~~

Para que el punto de montaje sea permanente se debe configurar el fichero /etc/fstab y añadir la siguiente línea:

~~~
UUID=iAP8kO-hRfR-6IiS-mo8G-z6N8-Wf7V-uDKYcy /mnt/raid5  xfs     defautls        0       0
~~~


#### Se marca un disco como estrupeado para hacer la comprobación del buen funcionamiento del RAID.

Para estropear el disco usamos la opción -f:

~~~
vagrant@Almacenamiento:~$ sudo mdadm /dev/md5 -f /dev/sdb
mdadm: set /dev/sdb faulty in /dev/md5
vagrant@Almacenamiento:~$ sudo mdadm --detail /dev/md5
/dev/md5:
           Version : 1.2
     Creation Time : Mon Oct  7 22:16:55 2019
        Raid Level : raid5
        Array Size : 2093056 (2044.00 MiB 2143.29 MB)
     Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
      Raid Devices : 3
     Total Devices : 3
       Persistence : Superblock is persistent

       Update Time : Mon Oct  7 22:28:55 2019
             State : clean, degraded 
    Active Devices : 2
   Working Devices : 2
    Failed Devices : 1
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : Almacenamiento:5  (local to host Almacenamiento)
              UUID : 96f19c3a:a41ee026:203a6acb:294d3d83
            Events : 20

    Number   Major   Minor   RaidDevice State
       -       0        0        0      removed
       1       8       32        1      active sync   /dev/sdc
       3       8       48        2      active sync   /dev/sdd

       0       8       16        -      faulty   /dev/sdb
~~~

Comprobamos que podemos seguir accediendo al fichero:
~~~
vagrant@Almacenamiento:~$ ls /mnt/raid5/
prueba.md
~~~


#### Una vez marcado como estropeado, se retira del RAID.

Para retirarlo del raid se usa la opción --remove del comando mdadm.

~~~
vagrant@Almacenamiento:~$ sudo mdadm /dev/md5 --remove /dev/sdb
mdadm: hot removed /dev/sdb from /dev/md5
You have mail in /var/mail/vagrant
vagrant@Almacenamiento:~$ sudo mdadm --detail /dev/md5
/dev/md5:
           Version : 1.2
     Creation Time : Mon Oct  7 22:16:55 2019
        Raid Level : raid5
        Array Size : 2093056 (2044.00 MiB 2143.29 MB)
     Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
      Raid Devices : 3
     Total Devices : 2
       Persistence : Superblock is persistent

       Update Time : Mon Oct  7 22:31:01 2019
             State : clean, degraded 
    Active Devices : 2
   Working Devices : 2
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : Almacenamiento:5  (local to host Almacenamiento)
              UUID : 96f19c3a:a41ee026:203a6acb:294d3d83
            Events : 21

    Number   Major   Minor   RaidDevice State
       -       0        0        0      removed
       1       8       32        1      active sync   /dev/sdc
       3       8       48        2      active sync   /dev/sdd
~~~
   

#### Se cambia por un nuevo disco nuevo (el dispositivo de bloque se llama igual), see añáde al array y se comprueba como se sincroniza con el anterior.

~~~
vagrant@Almacenamiento:~$ lsblk
NAME                 MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda                    8:0    0 19.8G  0 disk  
├─sda1                 8:1    0 18.8G  0 part  /
├─sda2                 8:2    0    1K  0 part  
└─sda5                 8:5    0 1021M  0 part  [SWAP]
sdb                    8:16   0    1G  0 disk  
sdc                    8:32   0    1G  0 disk  
└─md5                  9:5    0    2G  0 raid5 
  └─raid5vol-storage 253:0    0  500M  0 lvm   /mnt/raid5
sdd                    8:48   0    1G  0 disk  
└─md5                  9:5    0    2G  0 raid5 
  └─raid5vol-storage 253:0    0  500M  0 lvm   /mnt/raid5
sde                    8:64   0    1G  0 disk  
vagrant@Almacenamiento:~$ sudo mdadm /dev/md5 --add /dev/sdb
mdadm: added /dev/sdb
vagrant@Almacenamiento:~$ sudo mdadm --detail /dev/md5
/dev/md5:
           Version : 1.2
     Creation Time : Mon Oct  7 22:16:55 2019
        Raid Level : raid5
        Array Size : 2093056 (2044.00 MiB 2143.29 MB)
     Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
      Raid Devices : 3
     Total Devices : 3
       Persistence : Superblock is persistent

       Update Time : Mon Oct  7 22:32:22 2019
             State : clean 
    Active Devices : 3
   Working Devices : 3
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : Almacenamiento:5  (local to host Almacenamiento)
              UUID : 96f19c3a:a41ee026:203a6acb:294d3d83
            Events : 40

    Number   Major   Minor   RaidDevice State
       4       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       3       8       48        2      active sync   /dev/sdd
~~~

Si usamos el comando mdadm --detail /dev/md5 lo suficientemente rápido se puede observar como el estado del nuevo disco es: spare rebuilding.


#### Se añade otro disco como reserva. Se vuelve a simular el fallo de un disco y se comprueba como automática se realiza la sincronización con el disco de reserva.

Tras añadir un disco como reserva, este aparece en los detalles del raid con el estado spare. 

~~~
vagrant@Almacenamiento:~$ sudo mdadm /dev/md5 --add /dev/sde
mdadm: added /dev/sde
vagrant@Almacenamiento:~$ sudo mdadm --detail /dev/md5
/dev/md5:
           Version : 1.2
     Creation Time : Mon Oct  7 22:16:55 2019
        Raid Level : raid5
        Array Size : 2093056 (2044.00 MiB 2143.29 MB)
     Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
      Raid Devices : 3
     Total Devices : 4
       Persistence : Superblock is persistent

       Update Time : Mon Oct  7 22:34:11 2019
             State : clean 
    Active Devices : 3
   Working Devices : 4
    Failed Devices : 0
     Spare Devices : 1

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : Almacenamiento:5  (local to host Almacenamiento)
              UUID : 96f19c3a:a41ee026:203a6acb:294d3d83
            Events : 41

    Number   Major   Minor   RaidDevice State
       4       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       3       8       48        2      active sync   /dev/sdd

       5       8       64        -      spare   /dev/sde
~~~

Volvemos a estropear uno de los discos para que el disco de reserva entre a formar parte activa del raid. 

~~~
vagrant@Almacenamiento:~$ sudo mdadm /dev/md5 -f /dev/sdb
mdadm: set /dev/sdb faulty in /dev/md5
vagrant@Almacenamiento:~$ sudo mdadm --detail /dev/md5 
/dev/md5:
           Version : 1.2
     Creation Time : Mon Oct  7 22:16:55 2019
        Raid Level : raid5
        Array Size : 2093056 (2044.00 MiB 2143.29 MB)
     Used Dev Size : 1046528 (1022.00 MiB 1071.64 MB)
      Raid Devices : 3
     Total Devices : 4
       Persistence : Superblock is persistent

       Update Time : Mon Oct  7 22:35:06 2019
             State : clean, degraded, recovering 
    Active Devices : 2
   Working Devices : 3
    Failed Devices : 1
     Spare Devices : 1

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

    Rebuild Status : 7% complete

              Name : Almacenamiento:5  (local to host Almacenamiento)
              UUID : 96f19c3a:a41ee026:203a6acb:294d3d83
            Events : 44

    Number   Major   Minor   RaidDevice State
       5       8       64        0      spare rebuilding   /dev/sde
       1       8       32        1      active sync   /dev/sdc
       3       8       48        2      active sync   /dev/sdd

       4       8       16        -      faulty   /dev/sdb
~~~

#### Se redimensiona el volumen y el sistema de archivo de 500Mb al tamaño del RAID.

Con el comando vgs se muestra información sobre los grupos de volúmenes:

~~~
vagrant@Almacenamiento:~$ sudo vgs
  VG       #PV #LV #SN Attr   VSize VFree
  raid5vol   1   1   0 wz--n- 1.99g 1.50g
You have new mail in /var/mail/vagrant
~~~

Para extender el volumen utilizamos el comando lvextend seguido del volumen que queremos extender, la opción -L con el tamaño que queremos y la opción -r para indicar que vamos a redimensionar. 

~~~
vagrant@Almacenamiento:~$ sudo lvextend /dev/mapper/raid5vol-storage -L +1.5G -r
  Size of logical volume raid5vol/storage changed from 500.00 MiB (125 extents) to <1.99 GiB (509 extents).
  Logical volume raid5vol/storage successfully resized.
meta-data=/dev/mapper/raid5vol-storage isize=512    agcount=8, agsize=16000 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=0
data     =                       bsize=4096   blocks=128000, imaxpct=25
         =                       sunit=128    swidth=256 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=896, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 128000 to 521216
~~~

Para comprobar que hemos realizado el redimensionamiento correctamente:
~~~
vagrant@Almacenamiento:~$ df -h
Filesystem                    Size  Used Avail Use% Mounted on
udev                          228M     0  228M   0% /dev
tmpfs                          49M  3.6M   45M   8% /run
/dev/sda1                      19G  1.1G   17G   7% /
tmpfs                         242M     0  242M   0% /dev/shm
tmpfs                         5.0M     0  5.0M   0% /run/lock
tmpfs                         242M     0  242M   0% /sys/fs/cgroup
tmpfs                          49M     0   49M   0% /run/user/1000
/dev/mapper/raid5vol-storage  2.0G   29M  2.0G   2% /mnt/raid5
~~~

