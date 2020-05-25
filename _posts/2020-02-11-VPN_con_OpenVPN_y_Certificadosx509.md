---
title: "VPN con OpenVPN y certificados x509"
categories:
- Seguridad
excerpt: |
  Vamos a establecer una VPN de acceso remoto y a establecer una VPN de sitio a sitio, (Autenticación será con TLS utilizando certificados X.509 y la asignación de direcciones dinámica).
aside: true
---

![iSCSI](https://github.com/MoralG/VPN_con_OpenVPN_y_Certificadosx509/blob/master/image/OpenVPN.jpg?raw=true)

**Vamos a establecer una VPN de acceso remoto y a establecer una VPN de sitio a sitio, (Autenticación será con TLS utilizando certificados X.509 y la asignación de direcciones dinámica).**

## VPN de acceso remoto con OpenVPN y certificados x509

#### Configura una conexión VPN de acceso remoto entre dos equipos:

* Uno de los dos equipos (el que actuará como servidor) estará conectado a dos redes.
* Para la autenticación de los extremos se usarán obligatoriamente certificados digitales, que se generarán utilizando openssl y se almacenarán en el directorio /etc/openvpn, junto con los parámetros Diffie-Helman y el certificado de la propia Autoridad de Certificación.
* Se utilizarán direcciones de la red 10.99.99.0/24 para las direcciones virtuales de la VPN. La dirección 10.99.99.1 se asignará al servidor VPN.
* Los ficheros de configuración del servidor y del cliente se crearán en el directorio /etc/openvpn de cada máquina, y se llamarán servidor.conf y cliente.conf respectivamente. La configuración establecida debe cumplir los siguientes aspectos:
    * El demonio openvpn se manejará con systemctl.
    * Se debe configurar para que la comunicación esté comprimida.
    * La asignación de direcciones IP será dinámica.
    * Existirá un fichero de log en el equipo.
    * Se mandarán a los clientes las rutas necesarias.

* Tras el establecimiento de la VPN, la máquina cliente debe ser capaz de acceder a una máquina que esté en la otra red a la que está conectado el servidor.
---------------------------------------------------------------------------------

#### Configuración Vagrant Cliente
~~~
Vagrant.configure("2") do |config|

  config.vm.define :cliente1 do |cliente1|
    cliente1.vm.box = "debian/buster64"
    cliente1.vm.hostname = "Cliente1"
    cliente1.vm.network :public_network,:bridge=>"enp3s0"
  end
end
~~~

#### Configuración Vagrant Servidor
~~~
Vagrant.configure("2") do |config|

  config.vm.define :servidor2 do |servidor2|
    servidor2.vm.box = "debian/buster64"
    servidor2.vm.hostname = "Servidor2"
    servidor2.vm.network :public_network,:bridge=>"enp3s0"
    servidor2.vm.network :private_network, ip: "192.168.100.1", virtualbox__intnet:"mired1"
  end
  config.vm.define :cliente2 do |cliente2|
    cliente2.vm.box = "debian/buster64"
    cliente2.vm.hostname = "Cliente2"
    cliente2.vm.network :private_network, ip: "192.168.100.2", virtualbox__intnet:"mired1"
  end
end
~~~

> **NOTA**: Para esta tarea vamos a necesitar crear una Autoridad Certificadora y certificados firmados, [CLIC AQUÍ](https://github.com/MoralG/Certificados_Digitales_y_HTTPS/blob/master/HTTPS_SSL.md#tarea-1-certificado-autofirmado) para saber más.
> 
### Cliente1

Instalamos el paquete `openvpn` para realizar la conexión a la vpn del servidor.
~~~
sudo apt install openvpn
~~~

###### Creamos los datos iguales que la autoridad certificadora
~~~
Country Name (2 letter code) [AU]:ES
State or Province Name (full name) [Some-State]:Sevilla
Locality Name (eg, city) []:Dos Hermanas
Organization Name (eg, company) [Internet Widgits Pty Ltd]:IES Gonzalo Nazareno
Organizational Unit Name (eg, section) []:Paloma 
Common Name (e.g. server FQDN or YOUR name) []:Alejandro Morales Gracia
Email Address []:ale95mogra@gmail.com	
~~~

Pasamos el fichero `VPN_Alejandro.csr` a la Autoridad Certificadora y esperamos a que nos devuelva el certificado firmado y el certificado de CA.

Ademas vamos a mover los certificados y la clave privada a la ruta `/etc/openvpn/client`

### Servidor2

Instalamos el paquete `openvpn` para poder realizar la tarea.
~~~
sudo apt install openvpn
~~~

Creamos la clave Diffie-Hellman con el parámetro **dhparam**:
~~~
cd /etc/openvpn
sudo openssl dhparam -out dhparams.pem 4096
~~~

Activamos el bit de forward para permitir el tráfico a través de el:
~~~
echo 1 > /proc/sys/net/ipv4/ip_forward
~~~

Se edita el fichero `/etc/openvpn/servidor.conf`:
~~~
dev tun
server 10.99.99.0 255.255.255.0 
push "route 192.168.100.0 255.255.255.0"
tls-server
dh /etc/openvpn/dhparams.pem
ca /etc/openvpn/ca.cert.pem
cert /etc/openvpn/serverVPN.crt 
key /etc/openvpn/serverVPN.key
comp-lzo
keepalive 10 60
verb 3
askpass pass.txt
~~~

Vamos a firmar el certificado del cliente y se lo pasaremos para que pueda configurar su máquina.
~~~
openssl x509 -req -in VPN_Alejandro.csr -CA /root/ca/certs/ca.cert.pem -CAkey /root/ca/private/ca.key.pem -CAcreateserial -out VPN_Alejandro.crt
   Signature ok
   subject=C = ES, ST = Sevilla, L = Dos Hermanas, O = IES Gonzalo Nazareno, CN = Alejandro Morales Gracia
   Getting CA Private Key
   Enter pass phrase for /root/ca/private/ca.key.pem:
~~~

Creamos un fichero llamado pass.txt y metemos una contraseña para que lo reconozca el parametro `askpass`.
~~~
sudo touch pass.txt
echo "prueba" > pass.txt
~~~

Configuramos openvpn, reconozca systemd, los ficheros de configuración. Esto se hace descomentando, la siguiente linea, en el fichero `/etc/default/openvpn`.
~~~
AUTOSTART="all"
~~~

Reiniciamos el servicio `openvpn.service`
~~~
sudo systemctl restart openvpn.service
~~~

### Cliente2

Eliminamos la tabla de enrrutmiento por defecto y le asignamos la dirección del Servidor2.
~~~
sudo ip r del default
sudo ip r add default via 192.168.100.20
~~~

### Cliente1

Una vez que tenemos los dos certificados, tenemos que hacer el fichero de configuración del cliente `/etc/openvpn/VPN.conf`, donde vamos a indicar la interfaz `tun`, la dirección a la que se tiene que conectar y los certificados.
~~~
dev tun
remote 172.22.0.56
ifconfig 10.99.99.0 255.255.255.0
pull
tls-client
ca /etc/openvpn/client/ca.cert.pem
cert /etc/openvpn/client/VPN_Alejandro.crt
key /etc/openvpn/client/VPN_Alejandro.key
comp-lzo
keepalive 10 60
verb 3
askpass pass.txt
~~~

Creamos en un fichero llamado pass.txt y metemos una contraseña.
~~~
sudo touch pass.txt
echo "prueba" > pass.txt
~~~

Configuramos para que reconozca systemd los ficheros de configuración. Esto se hacer descomentando la siguiente linea en el fichero `/etc/default/openvpn`.
~~~
AUTOSTART="all"
~~~

Reiniciamos el servicio `openvpn.service`
~~~
sudo systemctl restart openvpn.service
~~~

###### Comprobamos que se ha añadido el `tun`
~~~
ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host 
           valid_lft forever preferred_lft forever
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:8d:c0:4d brd ff:ff:ff:ff:ff:ff
        inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
           valid_lft 85611sec preferred_lft 85611sec
        inet6 fe80::a00:27ff:fe8d:c04d/64 scope link 
           valid_lft forever preferred_lft forever
    3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:b8:70:44 brd ff:ff:ff:ff:ff:ff
        inet 172.22.0.163/16 brd 172.22.255.255 scope global dynamic eth1
           valid_lft 1029sec preferred_lft 1029sec
        inet6 fe80::a00:27ff:feb8:7044/64 scope link 
           valid_lft forever preferred_lft forever
    4: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN group    default qlen 100
        link/none 
        inet 10.99.99.6 peer 10.99.99.5/32 scope global tun0
           valid_lft forever preferred_lft forever
        inet6 fe80::9e0e:4d6c:ad95:ab79/64 scope link stable-privacy 
           valid_lft forever preferred_lft forever

ip r
    default via 10.0.2.2 dev eth0 
    10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 
    10.99.99.1 via 10.99.99.5 dev tun0 
    10.99.99.5 dev tun0 proto kernel scope link src 10.99.99.6 
    172.22.0.0/16 dev eth1 proto kernel scope link src 172.22.2.117 
    192.168.100.0/24 via 10.99.99.5 dev tun0 
~~~

Realizamos un ping a la dirección `192.168.100.2` que es el cliente que tiene el servidor.
~~~
ping 192.168.100.2
    PING 192.168.100.2 (192.168.100.2) 56(84) bytes of data.
    64 bytes from 192.168.100.2: icmp_seq=1 ttl=64 time=1.28 ms
    64 bytes from 192.168.100.2: icmp_seq=2 ttl=64 time=1.43 ms
    64 bytes from 192.168.100.2: icmp_seq=3 ttl=64 time=1.34 ms
    64 bytes from 192.168.100.2: icmp_seq=4 ttl=64 time=0.970 ms
    64 bytes from 192.168.100.2: icmp_seq=5 ttl=64 time=1.39 ms
    ^C
    --- 192.168.100.2 ping statistics ---
    5 packets transmitted, 5 received, 0% packet loss, time 12ms
    rtt min/avg/max/mdev = 0.970/1.280/1.425/0.166 ms
~~~


## VPN sitio a sitio con OpenVPN y certificados x509

#### Configura una conexión VPN sitio a sitio entre dos equipos del cloud:

* Cada equipo estará conectado a dos redes, una de ellas en común
* Para la autenticación de los extremos se usarán obligatoriamente certificados digitales, que se generarán utilizando openssl y se almacenarán en el directorio /etc/openvpn, junto con con los parámetros Diffie-Helman y el certificado de la propia Autoridad de Certificación.
* Se utilizarán direcciones de la red 10.99.99.0/24 para las direcciones virtuales de la VPN.
* Tras el establecimiento de la VPN, una máquina de cada red detrás de cada servidor VPN debe ser capaz de acceder a una máquina del otro extremo.


#### Configuración Vagrant Servidor/Cliente
~~~
Vagrant.configure("2") do |config|

  config.vm.define :servidor1 do |servidor1|
    servidor1.vm.box = "debian/buster64"
    servidor1.vm.hostname = "Servidor/Cliente1"
    servidor1.vm.network :public_network,:bridge=>"enp3s0"
    servidor1.vm.network :private_network, ip: "192.168.200.1", virtualbox__intnet:"mired1"
  end
  config.vm.define :cliente1 do |cliente1|
    cliente1.vm.box = "debian/buster64"
    cliente1.vm.hostname = "Cliente1"
    cliente1.vm.network :private_network, ip: "192.168.200.2", virtualbox__intnet:"mired1"
  end
end
~~~

#### Configuración Vagrant Servidor
~~~
Vagrant.configure("2") do |config|

  config.vm.define :servidor2 do |servidor2|
    servidor2.vm.box = "debian/buster64"
    servidor2.vm.hostname = "Servidor2"
    servidor2.vm.network :public_network,:bridge=>"enp3s0"
    servidor2.vm.network :private_network, ip: "192.168.100.20", virtualbox__intnet:"mired1"
  end
  config.vm.define :cliente2 do |cliente2|
    cliente2.vm.box = "debian/buster64"
    cliente2.vm.hostname = "Cliente2"
    cliente2.vm.network :private_network, ip: "192.168.100.50", virtualbox__intnet:"mired1"
  end
end
~~~

> **NOTA**: Utilizaremos las mismas claves que en la anterior tarea.

### Servidor2

La configuración para hacer una **VPN Site to Site** es igual que en la anterior tarea, pero el fichero `servidor.conf` cambia un poco.

Modificamos fichero de configuración del servidor `servidor.conf`.
~~~
dev tun
ifconfig 10.99.99.1 10.99.99.2
route 192.168.200.0 255.255.255.0
tls-server
dh /etc/openvpn/dhparams.pem
ca /etc/openvpn/ca.cert.pem
cert /etc/openvpn/serverVPN.crt
key /etc/openvpn/serverVPN.key
comp-lzo
keepalive 10 60
log /var/log/server.log
verb 6
askpass pass.txt
~~~

Comprobamos que el tunel se ha creado correctamente.
~~~
ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host 
           valid_lft forever preferred_lft forever
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:8d:c0:4d brd ff:ff:ff:ff:ff:ff
        inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
           valid_lft 80741sec preferred_lft 80741sec
        inet6 fe80::a00:27ff:fe8d:c04d/64 scope link 
           valid_lft forever preferred_lft forever
    3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:90:f5:5a brd ff:ff:ff:ff:ff:ff
        inet 172.22.9.60/16 brd 172.22.255.255 scope global dynamic eth1
           valid_lft 9168sec preferred_lft 9168sec
        inet6 fe80::a00:27ff:fe90:f55a/64 scope link 
           valid_lft forever preferred_lft forever
    4: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:4c:eb:52 brd ff:ff:ff:ff:ff:ff
        inet 192.168.100.20/24 brd 192.168.100.255 scope global eth2
           valid_lft forever preferred_lft forever
        inet6 fe80::a00:27ff:fe4c:eb52/64 scope link 
           valid_lft forever preferred_lft forever
    31: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN group   default qlen 100
        link/none 
        inet 10.99.99.1 peer 10.99.99.2/32 scope global tun0
           valid_lft forever preferred_lft forever
        inet6 fe80::dd5:e8c6:5fd1:f19c/64 scope link stable-privacy 
           valid_lft forever preferred_lft forever
~~~

### Servidor/Cliente1

Antes de configurar el cliente, el servidor tiene que haberle pasado al cliente los fichero `VPN_Alejandro.key`, `VPN_Alejandro.crt` y `ca.cert.pem` como en la anterior práctica.

Ahora podemos crear el fichero de configuración del cliente `cliente.conf`.

~~~
dev tun
ifconfig 10.99.99.2 10.99.99.1
remote 172.22.0.56
route 192.168.100.0 255.255.255.0
tls-client
ca /etc/openvpn/client/ca.cert.pem
cert /etc/openvpn/client/VPN_Alejandro.crt
key /etc/openvpn/client/VPN_Alejandro.key
comp-lzo
keepalive 10 60
log /var/log/host.log
verb 6
askpass pass.txt
~~~

Se activa el bit de forward para dejar pasar el tráfico al Cliente1:
~~~
root@servidor:/home/vagrant# echo 1 > /proc/sys/net/ipv4/ip_forward
~~~

Reiniciamos el servicio
~~~
sudo systemctl restart openvpn
~~~

Comprobamos que el tunel se ha creado correctamente y tenemos el nuevo registro de enrrutamiento.
~~~
ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host 
           valid_lft forever preferred_lft forever
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:8d:c0:4d brd ff:ff:ff:ff:ff:ff
        inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
           valid_lft 84637sec preferred_lft 84637sec
        inet6 fe80::a00:27ff:fe8d:c04d/64 scope link 
           valid_lft forever preferred_lft forever
    3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:b8:70:44 brd ff:ff:ff:ff:ff:ff
        inet 172.22.2.123/16 brd 172.22.255.255 scope global dynamic eth1
           valid_lft 8996sec preferred_lft 8996sec
        inet6 fe80::a00:27ff:feb8:7044/64 scope link 
           valid_lft forever preferred_lft forever
    4: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:7c:e2:d3 brd ff:ff:ff:ff:ff:ff
        inet 192.168.200.1/24 brd 192.168.200.255 scope global eth2
           valid_lft forever preferred_lft forever
        inet6 fe80::a00:27ff:fe7c:e2d3/64 scope link 
           valid_lft forever preferred_lft forever
    21: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN group default qlen 100
        link/none 
        inet 10.99.99.2 peer 10.99.99.1/32 scope global tun0
           valid_lft forever preferred_lft forever
        inet6 fe80::83d:90fb:c65e:93d6/64 scope link stable-privacy 
           valid_lft forever preferred_lft forever

ip r
    default via 10.0.2.2 dev eth0 
    10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 
    10.99.99.1 dev tun0 proto kernel scope link src 10.99.99.2 
    172.22.0.0/16 dev eth1 proto kernel scope link src 172.22.2.123 
    192.168.200.0/24 dev eth2 proto kernel scope link src 192.168.200.1 
~~~




Realizamos un PING al Servidor2 y al Cliente2.
~~~
vagrant@Servidor/Cliente1:/etc/openvpn$ ping 10.99.99.1
    PING 10.99.99.1 (10.99.99.1) 56(84) bytes of data.
    64 bytes from 10.99.99.1: icmp_seq=1 ttl=64 time=2.85 ms
    64 bytes from 10.99.99.1: icmp_seq=2 ttl=64 time=3.33 ms
    64 bytes from 10.99.99.1: icmp_seq=3 ttl=64 time=4.24 ms
    ^C
    --- 10.99.99.1 ping statistics ---
    3 packets transmitted, 3 received, 0% packet loss, time 5ms
    rtt min/avg/max/mdev = 2.851/3.473/4.240/0.576 ms

vagrant@Servidor/Cliente1:/etc/openvpn$ ping 192.168.100.50
    PING 192.168.100.50 (192.168.100.50) 56(84) bytes of data.
    64 bytes from 192.168.100.50: icmp_seq=1 ttl=63 time=3.37 ms
    64 bytes from 192.168.100.50: icmp_seq=2 ttl=63 time=8.10 ms
    64 bytes from 192.168.100.50: icmp_seq=3 ttl=63 time=20.4 ms
    ^C
    --- 192.168.100.50 ping statistics ---
    3 packets transmitted, 3 received, 0% packet loss, time 4ms
    rtt min/avg/max/mdev = 3.367/10.631/20.428/7.191 ms
~~~

### Cliente 1

Eliminamos la tabla de enrrutmiento por defecto y le asignamos la dirección del Servidor/Cliente1.
~~~
sudo ip r del default
sudo ip r add default via 192.168.200.1
~~~

Realizamos un PING al Cliente2 del Servidor2 
~~~
vagrant@Cliente1:~$ ping 192.168.100.50
    PING 192.168.100.50 (192.168.100.50) 56(84) bytes of data.
    64 bytes from 192.168.100.50: icmp_seq=1 ttl=62 time=6.53 ms
    64 bytes from 192.168.100.50: icmp_seq=2 ttl=62 time=3.73 ms
    64 bytes from 192.168.100.50: icmp_seq=3 ttl=62 time=7.20 ms
    ^C
    --- 192.168.100.50 ping statistics ---
    3 packets transmitted, 3 received, 0% packet loss, time 5ms
    rtt min/avg/max/mdev = 3.726/5.819/7.199/1.506 ms
~~~

### Cliente 2

Eliminamos la tabla de enrrutmiento por defecto y le asignamos la dirección del Servidor2.
~~~
sudo ip r del default
sudo ip r add default via 192.168.100.20
~~~

Realizamos un PING al Cliente1 del Servidor/Cliente1 
~~~
vagrant@Cliente2:~$ ping 192.168.100.2
    PING 192.168.100.2 (192.168.100.2) 56(84) bytes of data.
    64 bytes from 192.168.100.2: icmp_seq=1 ttl=62 time=6.53 ms
    64 bytes from 192.168.100.2: icmp_seq=2 ttl=62 time=3.73 ms
    64 bytes from 192.168.100.2: icmp_seq=3 ttl=62 time=7.20 ms
    ^C
    --- 192.168.100.2 ping statistics ---
    3 packets transmitted, 3 received, 0% packet loss, time 5ms
    rtt min/avg/max/mdev = 3.726/5.819/7.199/1.506 ms
~~~