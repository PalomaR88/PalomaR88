---
title: "Trabajando con iSCSI"
categories:
- Sistemas
excerpt: |
  Vamos a configurar un sistema que exporte algunos targets por iSCSI y los conecte a diversos clientes.
aside: true
---

![iSCSI](https://github.com/MoralG/Trabajando_con_iSCSI/raw/master/image/iSCSI.jpg)

**Vamos a configurar un sistema que exporte algunos targets por iSCSI y los conecte a diversos clientes, explicando con detalle la forma de trabajar.**

Vamos a crear un escenario con Vagrant. Vamos a crear una máquina servidor, conectada por una red interna a dos cliente, un cliente Debian y otro Window. al servidor le añadiremos 3 volúmenes. Añadiremos un interfaz con conexión a internet para poder descargar los paquetes necesarios.

```shell
Vagrant.configure("2") do |config|

  config.vm.define :servidor1 do |servidor1|
    d1 = ".vagrant/disco1.vdi"
    d2 = ".vagrant/disco2.vdi"
    d3 = ".vagrant/disco3.vdi"
    servidor1.vm.box = "debian/buster64"
    servidor1.vm.hostname = "Servidor1"

    servidor1.vm.network :public_network,:bridge=>"enp3s0"
    #servidor.vm.network :public_network,:bridge=>"wlp2s0"
    servidor1.vm.network :private_network, ip: "192.168.100.1", virtualbox__intnet:"mired1"

    servidor1.vm.provider :virtualbox do |v|
	    if not File.exist?(d1)
	    	v.customize ["createhd", "--filename", d1, "--size", 1 *  1024]
	    end
	    v.customize ["storageattach", :id, "--storagectl", "SATA Controller", "--port", 1,"--device", 0, "--type", "hdd", "--medium", d1]

	    if not File.exist?(d2)
	    	v.customize ["createhd", "--filename", d2, "--size", 1 * 1024]
	    end
	    v.customize ["storageattach", :id, "--storagectl", "SATA Controller", "--port", 2,"--device", 0, "--type", "hdd", "--medium", d2]

        if not File.exist?(d3)
            v.customize ["createhd", "--filename", d3, "--size", 1 * 1024]
        end
        v.customize ["storageattach", :id, "--storagectl", "SATA Controller", "--port", 3,"--device", 0, "--type", "hdd", "--medium", d3]
    end
  end

  config.vm.define :cliente1 do |cliente1|
    cliente1.vm.box = "debian/buster64"
    cliente1.vm.hostname = "Cliente1"
    cliente1.vm.network :public_network,:bridge=>"enp3s0"
    #cliente1.vm.network :public_network,:bridge=>"wlp2s0"
    cliente1.vm.network :private_network, ip: "192.168.100.2", virtualbox__intnet:"mired1"
  end

  config.vm.define :cliente1 do |cliente2|
    cliente2.vm.box = "vdelarosa/windows-10"
    cliente2.vm.hostname = "Cliente2"
    cliente2.vm.network :public_network,:bridge=>"enp3s0"
    #cliente2.vm.network :public_network,:bridge=>"wlp2s0"
    cliente2.vm.network :private_network, ip: "192.168.100.3", virtualbox__intnet:"mired1"
  end

end
```

## Tarea 1

Crearemos un target con una LUN y lo conectaremos a un cliente GNU/Linux. ¿Cómo escaneamos desde el cliente buscando los targets disponibles y utilizando la unidad lógica proporcionada?, formateándola si es necesario y montándola.

Conceptos:

> **iSCSI**: Es un estándar de red de almacenamiento que vincula instalaciones de almacenamiento de datos. Facilita la transferencia de datos a través de redes LAN, WAN o Internet. Todo esto quiere decir que el espacio en el servidor de almacenamiento será considerado como discos locales por el sistema operativo del cliente pero todos los datos tranferidos al disco se transfiren realmente a través de la red al servidor de almacenamiento.
>
>**LUN** (Logical Unit Number): Un LUN se representa como un dispositivo SCSI direccionable individualmente que forma parte de un dispositivo SCDI físico, es decir, una dirección que identifica un disco completo o un trocito de un dispositivo de bloque del Servidor.
>
>**Target**: Es el encargado de asignar virtualmente los LUN creados al sistema operativo del cliente.

### Configuración del servidor

En el servidor hemos asignado tres discos de 1GB, los cuales vamos a agrupar en un LVM para tener un espacio de almacenamiento de 3GB y poder gestionarlo como si fuese un solo dispositivo de bloque.

###### Comprobamos los disco asignados al Servidor
~~~
lsblk -l
    NAME MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    sda    8:0    0 19.8G  0 disk 
    sda1   8:1    0 18.8G  0 part /
    sda2   8:2    0    1K  0 part 
    sda5   8:5    0 1021M  0 part [SWAP]
    sdb    8:16   0    1G  0 disk 
    sdc    8:32   0    1G  0 disk 
    sdd    8:48   0    1G  0 disk 
~~~

Vamos a descargar el paquete `lvm2` que instalará los paquetes del kernel, el daemon del espacio de usuario y todo lo demás que necesitamos para trabajar con LVM.

###### Descargamos e instalamos `lvm2`
~~~
sudo apt install lvm2
~~~

Ahora vamos a definir los volúmenes físicos donde indicaremos el disco `sdb`.

###### Definiendo el volumen físico
~~~
sudo pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
~~~

Después de definir el volumen físico tenemos que crear el grupo de volúmenes. a este grupo de volúmenes lo llamaremo `vgCliente1` con el volumen físico `sdb`.

###### Creando el grupo de volúmenes
~~~
sudo vgcreate vgCliente1 /dev/sdb
  Volume group "vgCliente1" successfully created
~~~

Ahora vamos a crear un volumen lógico, el cual va a ser el espacio que contendrá el sistema de ficheros. Este lo podemos crear del tamaño que queramos dependiendo del tamaño todal del grupo de volúmenes, en nuestro caso va a ser de 500Mb y lo llamaremos `lv1` perteneciente al grupo de volumenes `vgCliente1`.

###### Creando el volumen lógico
~~~
sudo lvcreate -L 500M -n lv1 vgCliente1
  Logical volume "lv1" created.
~~~

Ya tenemos nuestro volumen lógico creado, ahora podemos ya trabajar con el para crear el LUN y asignarlo mediante un target al cliente.

###### Comprobamos que se ha creado de forma correcta
~~~
sudo lvs
  LV   VG         Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv1  vgCliente1 -wi-a----- 500.00m

sudo lvdisplay 
  --- Logical volume ---
  LV Path                /dev/vgCliente1/lv1
  LV Name                lv1
  VG Name                vgCliente1
  LV UUID                xR20wc-5YFN-af3g-UQGD-LH6Y-MoUL-3F7H2G
  LV Write Access        read/write
  LV Creation host, time Servidor1, 2020-02-01 12:30:02 +0000
  LV Status              available
  # open                 0
  LV Size                500.00 MiB
  Current LE             125
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           254:0
~~~

Ya tenemos todo lo necesario para trabajar con iSCSI pero tenemos que descargar el paquete tgt, que va a instalar los paquetes necesario para crear target.

###### Descargamos e instalamos el paquete `tgt` 
~~~
sudo apt install tgt
~~~

Ahora tenemmos que crear el target, que se configura el fichero `/etc/tgt/targets.conf` con la siguiente configuración:

------------------------------------------------------
~~~
<target <Estructura _del_target>>
    backing-store <Nombre_del_volumen_lógico>
</target>
~~~
* Estructura de los target iSCSI: `iqn.<AÑO>-<MES>.<DOMINO_INVERTIDO>:<NOMBRE_DE_LA_MÁQUINA>.<TARGET><NÚMERO>`

* Nombre del volumen logico: 
~~~
sudo lvdisplay | egrep "LV Path" | awk '{ print $3 }'
  /dev/vgCliente1/lv1
~~~
------------------------------------------------------

En nuestro caso quedaría de la siguiente forma:
~~~
<target iqn.2020-01.com:tg1>
    backing-store /dev/vgCliente1/lv1
</target>
~~~

Tenemos que reiniciar el servicio de `tgt` para que reconzca la modificación del fichero `/etc/tgt/targets.conf`

###### Reiniciamos el servicio
~~~
sudo systemctl restart tgt.service 
~~~

###### Comprobamos que se ha creado correctamente
~~~
sudo tgtadm --mode target --op show
Target 1: iqn.2020-01.com:tg1
    System information:
        Driver: iscsi
        State: ready
    I_T nexus information:
    LUN information:
        LUN: 0
            Type: controller
            SCSI ID: IET     00010000
            SCSI SN: beaf10
            Size: 0 MB, Block size: 1
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            SWP: No
            Thin-provisioning: No
            Backing store type: null
            Backing store path: None
            Backing store flags: 
        LUN: 1
            Type: disk
            SCSI ID: IET     00010001
            SCSI SN: beaf11
            Size: 734 MB, Block size: 512
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            SWP: No
            Thin-provisioning: No
            Backing store type: rdwr
            Backing store path: /dev/vgCliente1/vol1
            Backing store flags: 
    Account information:
    ACL information:
        ALL
~~~


### Configuración en el cliente

Para iniciar iSCSI en el cliente hay que instalar el paquete `open-iscsi`:

###### Descargamos e instalamos `open-iscsi`
~~~
sudo apt install open-iscsi
~~~

A continuación se edita el fichero `/etc/iscsi/iscsid.conf` para indicar que de forma automática escanee los target accesibles al cliente, para eso, tenemos que configurrar el parámetro `iscsid.startup` en `automatic`.

###### Modificamos la línea
~~~
iscsid.startup = automatic
~~~

Para que se actualicen los cambios tenemos que reiniciar el servicio de `open-iscsi`.

###### Reiniciamos el servicio
~~~
sudo systemctl restart open-iscsi.service 
~~~

Ahora vamos a buscar los target disponibles y accesibles del Servidor. Para listar los target tenemos que indicarle la dirección del Servidor.

###### Listamos los target
~~~
sudo iscsiadm -m node
  192.168.100.1:3260,1 iqn.2020-01.com:tg1
~~~

Tenemos el listado de los target disponibles del servidor, solo nos queda conectarnos al target que queramos.

###### Nos conectamos al target `iqn.2020-01.com:tg1`
~~~
sudo iscsiadm -m node -T iqn.2020-01.com:tg1 --portal "192.168.100.1" --login
  Logging in to [iface: default, target: iqn.2020-01.com:tg1, portal: 192.168.100.1,3260] (multiple)
  Login to [iface: default, target: iqn.2020-01.com:tg1, portal: 192.168.100.1,3260] successful.
~~~

Podemos comprobar que se nos ha asignado un nuevo disco de 500MB, el cual es el LUM1 que tenemos en el Servidor.
###### Comprobamos que tenemos el nuevo disco
~~~
lsblk -l
  NAME MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
  sda    8:0    0 19.8G  0 disk 
  sda1   8:1    0 18.8G  0 part /
  sda2   8:2    0    1K  0 part 
  sda5   8:5    0 1021M  0 part [SWAP]
  sdb    8:16   0  500M  0 disk 
~~~

Ahora vamos a particionarlo, formatearlo y montarlo como un disco normal.

Lo particionamos con `fdisk` y le indicamos que sea una particion completa y comprobamos que se ha realizado correctamente con `fdisk -l` al ser de tipo **VIRTUAL_DISK**
###### Particionamos
~~~
sudo fdisk /dev/sdb

  Welcome to fdisk (util-linux 2.33.1).
  Changes will remain in memory only, until you decide to write them.
  Be careful before using the write command.

  Device does not contain a recognized partition table.
  Created a new DOS disklabel with disk identifier 0x0f7ae951.

  Command (m for help): n
  Partition type
     p   primary (0 primary, 0 extended, 4 free)
     e   extended (container for logical partitions)
  Select (default p): 

  Using default response p.
  Partition number (1-4, default 1): 
  First sector (2048-1023999, default 2048): 
  Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-1023999, default 1023999): 

  Created a new partition 1 of type 'Linux' and of size 499 MiB.

  Command (m for help): w
  The partition table has been altered.
  Calling ioctl() to re-read partition table.
  Syncing disks.

sudo fdisk -l /dev/sdb
  Disk /dev/sdb: 500 MiB, 524288000 bytes, 1024000 sectors
  Disk model: VIRTUAL-DISK    
  Units: sectors of 1 * 512 = 512 bytes
  Sector size (logical/physical): 512 bytes / 512 bytes
  I/O size (minimum/optimal): 512 bytes / 512 bytes
  Disklabel type: dos
  Disk identifier: 0x0f7ae951

  Device     Boot Start     End Sectors  Size Id Type
  /dev/sdb1        2048 1023999 1021952  499M 83 Linux
~~~

###### Formateandolo a **ext4**
~~~
sudo mkfs.ext4 /dev/sdb1
  mke2fs 1.44.5 (15-Dec-2018)
  Creating filesystem with 510976 1k blocks and 128016 inodes
  Filesystem UUID: b7296407-b693-45c3-9d26-163e4aba760c
  Superblock backups stored on blocks: 
  	8193, 24577, 40961, 57345, 73729, 204801, 221185, 401409

  Allocating group tables: done
  Writing inode tables: done
  Creating journal (8192 blocks): done
  Writing superblocks and filesystem accounting information: done 
~~~

###### Montandolo en /mnt
~~~
sudo mount /dev/sdb1 /mnt

lsblk -f
  NAME   FSTYPE LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINT
  sda                                                                     
  ├─sda1 ext4         b9ffc3d1-86b2-4a2c-a8be-f2b2f4aa4cb5   16.2G     7% /
  ├─sda2                                                                  
  └─sda5 swap         f8f6d279-1b63-4310-a668-cb468c9091d8                [SWAP]
  sdb                                                                     
  └─sdb1 ext4         b7296407-b693-45c3-9d26-163e4aba760c    444M     0% /mnt
~~~

Como podemos ver, el disco nuevo que se ha asignado al cliente, lo podemos utilizar como cualquier otro de manear normal. Ahora el único problema que tenemos que si reiniciamos la máquina el disco se desmontaría y desaparecería de nuestra máquina, para que esto no ocurra vamos a configurar un automontaje en la Tarea 2.

## Tarea 2

Vamos a utilizar systemd mount para que el target se monte automáticamente al arrancar el cliente.

Para realizar el montaje automáticamente tenemos que crear una unidad de systemd para que se monte el disco de manera automática al inciar el equipo del cliente pero para que esto funcione, tenemos que configurar primero iSCSI para realice el **login** de manera automática al iniciar el cliente.

Para que realice la sesión con el Servidor tenemos que modificar el parámetro `node.startup = automatic` en el fichero `/etc/iscsi/iscsid.conf` o podemos ejecutar un comando para modificar dicho parámetro.

###### Ejecutamos el comando para poner el login en automático
~~~
sudo iscsiadm -m node -T iqn.2020-01.com:tg1 -o update -n node.startup -v automatic
~~~

Ahora que tenemos el autologin activado podemos crear la unidad de systemd y que monte el disco de manera automática. Vamos a crear un fichero en `/etc/systemd/system`, y el fichero se tiene que llamar como el punto de montaje seguido de `.mount`.

Nosotros vamos a montar el disco en `/Disco1` por lo tanto el fichero se deberá de llamar **Disco1.mount**

###### Creamos el directorio y el fichero
~~~
sudo mkdir /Disco1
sudo nano /etc/systemd/system/Disco1.mount
~~~

Dentro del fichero vamos a definir la unidad, el punto de montaje y el tipo.

###### Añadimos las siguiente lineas al fichero `/etc/systemd/system/Disco1.mount`
~~~
[Unit]
Description=Montaje de Disco1

[Mount]
What=/dev/sdb1
Where=/Disco1
Type=ext4
Options=_netdev

[Install]
WantedBy=multi-user.target
~~~

> **Description**: Descripción de la utilidad de la unidad.
>
> **What**: La partición que queremos montar `/dev/sdb1`.
>
> **Where**: La ruta donde queremos montar la partición, en nuestro caso `/Disco1`.
>
> **Type**: Le indicamos que el sistema de fichero es ext4 como la partición `/dev/sdb1`.
>
> **Options**: Le índicamos que inicie la unidad de montaje después de que la red este disponible con `_netdev`..
>
> **WantedBy**: Define el estado del sistema de nivel 2.

Ahora solo nos queda reiniciar el daemon de systemd para que detecte los nuevos cambios y detecte la nueva unidad de `.mount`.

###### Reiniciamos el demonio
~~~
sudo systemctl daemon-reload 
~~~

Iniciamos la unidad creada de Disco1.mount y la habilitamos para que inicie automáticamente al inicio del sistema.

###### Iniciamos y habilitar la unidad `Disco1.mount`
~~~
sudo systemctl start Disco1.mount
sudo systemctl enable Disco1.mount
  Created symlink /etc/systemd/system/multi-user.target.wants/Disco1.mount → /etc/systemd/system/Disco1.mount.
~~~

Ahora comprobamos que al reinciar el cliente se nos monta de manera automática en `Disco1`.

###### Comprobamación
~~~
vagrant@Cliente1:~$ sudo reboot
  Connection to 127.0.0.1 closed by remote host.
  Connection to 127.0.0.1 closed.

moralg@padano:~/Vagrant/iSCSI$ vagrant ssh cliente1 
  Linux Cliente1 4.19.0-6-amd64 #1 SMP Debian 4.19.67-2+deb10u2 (2019-11-11) x86_64

  The programs included with the Debian GNU/Linux system are free software;
  the exact distribution terms for each program are described in the
  individual files in /usr/share/doc/*/copyright.

  Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
  permitted by applicable law.
  Last login: Sun Feb  2 18:52:38 2020 from 10.0.2.2

vagrant@Cliente1:~$ lsblk -l
  NAME MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
  sda    8:0    0 19.8G  0 disk 
  sda1   8:1    0 18.8G  0 part /
  sda2   8:2    0    1K  0 part 
  sda5   8:5    0 1021M  0 part [SWAP]
  sdb    8:16   0  500M  0 disk 
  sdb1   8:17   0  499M  0 part /Disco1
~~~

## Tarea 3

Crearemos un target con 2 LUN y autenticación por CHAP y la conectaremos a un cliente windows. ¿Cómo se escanea la red en windows y cómo se utilizan las unidades nuevas (formateándolas con NTFS)?


Ahora vamos a definir los volúmenes físicos `sdc` y `sdd`, crearemos el grupo de volúmenes `vgCliente2` y por último crearemos el volumen lógico de 1GB y otro de 500MV para crear los 2 LUN.

###### Definiendo el volumen físico
~~~
sudo pvcreate /dev/sdc /dev/sdd
  Physical volume "/dev/sdc" successfully created.
  Physical volume "/dev/sdd" successfully created.
~~~

###### Creando el grupo de volúmenes
~~~
sudo vgcreate vgCliente2 /dev/sdc /dev/sdd
  Volume group "vgCliente2" successfully created
~~~

###### Creando los volúmenes lógico
~~~
sudo lvcreate -L 500M -n lv2 vgCliente2
  Logical volume "lv2" created.

sudo lvcreate -L 1G -n lv3 vgCliente2
  Logical volume "lv3" created.
~~~

###### Comprobamos que se ha creado correctamente
~~~
sudo lvs
  LV   VG         Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lv1  vgCliente1 -wi-ao---- 500.00m                                                    
  lv2  vgCliente2 -wi-a----- 500.00m                                                    
  lv3  vgCliente2 -wi-a-----   1.00g                                                    

sudo lvdisplay
  --- Logical volume ---
  LV Path                /dev/vgCliente2/lv2
  LV Name                lv2
  VG Name                vgCliente2
  LV UUID                pIM3VO-oUwc-AOVO-uwS6-FYlW-tX5T-g7FGPa
  LV Write Access        read/write
  LV Creation host, time Servidor1, 2020-02-02 20:00:35 +0000
  LV Status              available
  # open                 0
  LV Size                500.00 MiB
  Current LE             125
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           254:1
   
  --- Logical volume ---
  LV Path                /dev/vgCliente2/lv3
  LV Name                lv3
  VG Name                vgCliente2
  LV UUID                d59SdU-rhC5-D9mS-6Os4-UVwi-zThl-CwHqxZ
  LV Write Access        read/write
  LV Creation host, time Servidor1, 2020-02-02 20:00:49 +0000
  LV Status              available
  # open                 0
  LV Size                1.00 GiB
  Current LE             256
  Segments               2
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           254:2
   
  --- Logical volume ---
  LV Path                /dev/vgCliente1/lv1
  LV Name                lv1
  VG Name                vgCliente1
  LV UUID                xR20wc-5YFN-af3g-UQGD-LH6Y-MoUL-3F7H2G
  LV Write Access        read/write
  LV Creation host, time Servidor1, 2020-02-01 12:30:02 +0000
  LV Status              available
  # open                 1
  LV Size                500.00 MiB
  Current LE             125
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           254:0
~~~

Ahora tenemmos que crear el target con usuario y contraseña en `incominguser`, que se configura el fichero `/etc/tgt/targets.conf` con la siguiente configuración:

###### Creamos el target con los dos LUM
~~~
<target iqn.2020-01.com:tg2>
    backing-store /dev/vgCliente2/lv2
    backing-store /dev/vgCliente2/lv3
    incominguser usuario1 UsuarioPassSec
</target>
~~~

> NOTA: La contraseña debe de tener entre 12 y 16 caracteres para que no se error.

Tenemos que reiniciar el servicio de `tgt` para que reconzca la modificación del fichero `/etc/tgt/targets.conf`

###### Reiniciamos el servicio
~~~
sudo systemctl restart tgt.service 
~~~

###### Comprobamos que se ha creado correctamente
~~~
sudo tgtadm --mode target --op show | egrep -A 50 "iqn.2020-01.com:tg2"
Target 2: iqn.2020-01.com:tg2
    System information:
        Driver: iscsi
        State: ready
    I_T nexus information:
    LUN information:
        LUN: 0
            Type: controller
            SCSI ID: IET     00020000
            SCSI SN: beaf20
            Size: 0 MB, Block size: 1
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            SWP: No
            Thin-provisioning: No
            Backing store type: null
            Backing store path: None
            Backing store flags: 
        LUN: 1
            Type: disk
            SCSI ID: IET     00020001
            SCSI SN: beaf21
            Size: 524 MB, Block size: 512
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            SWP: No
            Thin-provisioning: No
            Backing store type: rdwr
            Backing store path: /dev/vgCliente2/lv2
            Backing store flags: 
        LUN: 2
            Type: disk
            SCSI ID: IET     00020002
            SCSI SN: beaf22
            Size: 1074 MB, Block size: 512
            Online: Yes
            Removable media: No
            Prevent removal: No
            Readonly: No
            SWP: No
            Thin-provisioning: No
            Backing store type: rdwr
            Backing store path: /dev/vgCliente2/lv3
            Backing store flags: 
    Account information:
        usuario1
    ACL information:
~~~

Ahora vamos a iniciar el segundo target creado en una máquina Window, para realizar esto vamos abrir el **Iniciador iSCSI**.

###### Abrimos el Iniciador iSCSI
![Tarea1.1](https://github.com/MoralG/Trabajando_con_iSCSI/blob/master/image/Tarea1.1_iSCSI.png?raw=true)

En la pestaña **Detección** vamos a seleccionar en **Detectar portal..** e indicamos la dirección de nuestro servidor y el puerto. 

![Tarea1.2](https://github.com/MoralG/Trabajando_con_iSCSI/blob/master/image/Tarea1.2_iSCSI.png?raw=true)

###### Detectamos el Servidor
![Tarea1.3](https://github.com/MoralG/Trabajando_con_iSCSI/blob/master/image/Tarea1.3_iSCSI.png?raw=true)


Si no ha salido ningun mensaje, es que ha dectectado el Servidor y si vamos a la pestaña **Destino** nos saldrá los target disponibles.

###### Vemos los target disponibles 
![Tarea1.4](https://github.com/MoralG/Trabajando_con_iSCSI/blob/master/image/Tarea1.4_iSCSI.png?raw=true)

Nos sale que el target `iqn.2020-01.com:tg2` esta incativo, para activarlo tenemos que darle a **Conectar**.

Nos saltará una ventana donde antes de darle a **Aceptar** vamos a Activar el inicio de sesión CHAP dandole a **Opciones Avanzadas...**.

###### Seleccionamos opciones avazadas
![Tarea1.5](https://github.com/MoralG/Trabajando_con_iSCSI/blob/master/image/Tarea1.5_iSCSI.png?raw=true)

Activamos el inicio de sesion CHAP y ponemos el usuario y contraseña, los mismo que asignamos en la creación del target.

###### Habilitamos el inicio de CHAP y introducciomos el usuario y contraseña
![Tarea1.6](https://github.com/MoralG/Trabajando_con_iSCSI/blob/master/image/Tarea1.6_iSCSI.png?raw=true)

Ahora al darle a **Aceptar** nos saldrá en estado **Conectado**.

###### Comprobamos que esta conectado
![Tarea1.7](https://github.com/MoralG/Trabajando_con_iSCSI/blob/master/image/Tarea1.7_iSCSI.png?raw=true)

Ahora ya tenemos los discos asignado y disponibles para inicializarlos y formatearlos a NTFS. Para hacer esto vamos a abrir **Administración de discos**

###### Abrimos el administración de discos
![Tarea1.8](https://github.com/MoralG/Trabajando_con_iSCSI/blob/master/image/Tarea1.8_iSCSI.png?raw=true)

###### Inicializamos los discos
![Tarea1.9](https://github.com/MoralG/Trabajando_con_iSCSI/blob/master/image/Tarea1.9_iSCSI.png?raw=true)

###### Formateamos los discos a NTFS
![Tarea1.10](https://github.com/MoralG/Trabajando_con_iSCSI/blob/master/image/Tarea10_iSCSI.png?raw=true)
![Tarea1.11](https://github.com/MoralG/Trabajando_con_iSCSI/blob/master/image/Tarea1.11_iSCSI.png?raw=true)

Comprobamos que nos salen montados y podemos utilizarlos de manera normal.

###### Comprobamos que aparecen montados
![Tarea1.12](https://github.com/MoralG/Trabajando_con_iSCSI/blob/master/image/Tarea1.12_iSCSI.png?raw=true)

###### Creamos un fichero
![Tarea1.13](https://github.com/MoralG/Trabajando_con_iSCSI/blob/master/image/Tarea1.13_iSCSI.png?raw=true)
