---
title: "Cortafuego Perimetral con DMZ"
categories:
- Seguridad
excerpt: |
  Trabajamos mejorando un cortafuego perimetral tradicional, para que sea mas seguro y añadiendo un equipo que haga de DMZ.
aside: true
---

![iSCSI](https://github.com/MoralG/Cortafuego_Perimetral_con_DMZ/blob/master/image/DMZ.png?raw=true)

**Trabajamos mejorando un cortafuego perimetral tradicional, para que sea mas seguro y añadiendo un equipo que haga de DMZ.**

## Esquema de red
------------------------------------------------------------------------------------------------
Vamos a utilizar tres máquinas en openstack, que vamos a crear con la receta heat: escenario3.yaml. La receta heat ha deshabilitado el cortafuego que nos ofrece openstack (todos los puertos de todos los protocolos están abiertos). Una máquina (que tiene asignada una IP flotante) hará de cortafuegos, otra será una máquina de la red interna 192.168.100.0/24 y la tercera será un servidor en la DMZ donde iremos instalando distintos servicios y estará en la red 192.168.200.0/24.

![Tarea1.1](https://github.com/MoralG/Cortafuego_Perimetral_con_DMZ/blob/master/image/Tarea1.1_DMZ.png?raw=true)

## Cumplimientos
------------------------------------------------------------------------------------------------

* #### Política por defecto DROP para las cadenas INPUT, FORWARD y OUTPUT.

Limpiamos las tablas.

~~~
sudo iptables -F
sudo iptables -t nat -F
sudo iptables -Z
sudo iptables -t nat -Z
~~~

Conexión ssh antes de estableces la politica por defecto en Drop para no perder la coonexión.

~~~
sudo iptables -A INPUT -s 172.22.0.0/16 -p tcp -m tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -d 172.22.0.0/16 -p tcp -m tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT

sudo iptables -A INPUT -s 172.23.0.0/16 -p tcp -m tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -d 172.23.0.0/16 -p tcp -m tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT

sudo iptables -A OUTPUT -p tcp -o eth1 -d 192.168.100.0/24 --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -p tcp -i eth1 -s 192.168.100.0/24 --sport 22 -m state --state ESTABLISHED -j ACCEPT

sudo iptables -A OUTPUT -p tcp -o eth2 -d 192.168.200.0/24 --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -p tcp -i eth2 -s 192.168.200.0/24 --sport 22 -m state --state ESTABLISHED -j ACCEPT
~~~

Establecemos la política.

~~~
sudo iptables -P INPUT DROP
sudo iptables -P OUTPUT DROP
sudo iptables -P FORWARD DROP
~~~

* Se pueden usar las extensiones que queremos adecuadas, pero al menos debe implementarse seguimiento de la conexión.
* Debes indicar pruebas de funcionamiento de todos las reglas.


## Tareas
--------------------------------------------------------------------------------------------------

### Tarea 1. La máquina router-fw tiene un servidor ssh escuchando por el puerto 22, pero al acceder desde el exterior habrá que conectar al puerto 2222.

##### Reglas

Hay que activar el 'ip_forward'
~~~
sudo su
echo 1 > /proc/sys/net/ipv4/ip_forward
exit
~~~

Configuramos la redirecion del puerto 2222 al 22
~~~
sudo iptables -t nat -I PREROUTING -p tcp -s 172.22.0.0/16 --dport 2222 -j REDIRECT --to-ports 22

sudo iptables -t nat -I PREROUTING -p tcp -s 172.23.0.0/16 --dport 2222 -j REDIRECT --to-ports 22
~~~

Configuramos la regla para que se pueda hacer conexión desde el puerto 2222
~~~
sudo iptables -A INPUT -s 172.22.0.0/16 -p tcp -m tcp --dport 2222 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -d 172.22.0.0/16 -p tcp -m tcp --sport 2222 -m state --state ESTABLISHED -j ACCEPT

sudo iptables -A INPUT -s 172.23.0.0/16 -p tcp -m tcp --dport 2222 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -d 172.23.0.0/16 -p tcp -m tcp --sport 2222 -m state --state ESTABLISHED -j ACCEPT
~~~

Bloquemos la conexión desde el puerto 22 redirigiendola al Loopback para que se pierda
~~~
sudo iptables -t nat -I PREROUTING -p tcp -s 172.22.0.0/16 --dport 22 --jump DNAT --to-destination 127.0.0.1

sudo iptables -t nat -I PREROUTING -p tcp -s 172.23.0.0/16 --dport 22 --jump DNAT --to-destination 127.0.0.1
~~~

##### Comprobación
~~~
moralg@padano:~$ ssh -A -p 2222 debian@172.22.200.145
    @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
    @    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
    @@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
    IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
    Someone could be eavesdropping on you right now (man-in-the-middle attack)!
    It is also possible that a host key has just been changed.
    The fingerprint for the ECDSA key sent by the remote host is
    SHA256:fnmA6k3OIDwXzXVMgrL3g+JjSjlmzRTU0Ou2xYwDdaE.
    Please contact your system administrator.
    Add correct host key in /home/moralg/.ssh/known_hosts to get rid of this message.
    Offending ECDSA key in /home/moralg/.ssh/known_hosts:40
      remove with:
      ssh-keygen -f "/home/moralg/.ssh/known_hosts" -R "[172.22.200.145]:2222"
    ECDSA host key for [172.22.200.145]:2222 has changed and you have requested strict  checking.
    Host key verification failed.

moralg@padano:~$   ssh-keygen -f "/home/moralg/.ssh/known_hosts" -R "[172.22.200.145]   :2222"
    # Host [172.22.200.145]:2222 found: line 40
    /home/moralg/.ssh/known_hosts updated.
    Original contents retained as /home/moralg/.ssh/known_hosts.old
    moralg@padano:~$ ssh -A -p 2222 debian@172.22.200.145
    Linux router-fw 4.19.0-6-cloud-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20)   x86_64

    The programs included with the Debian GNU/Linux system are free software;
    the exact distribution terms for each program are described in the
    individual files in /usr/share/doc/*/copyright.

    Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
    permitted by applicable law.
    Last login: Fri Dec 13 10:02:20 2019 from 172.22.1.248

debian@router-fw:~$ exit

moralg@padano:~$ ssh -A -p 22 debian@172.22.200.145
    ssh: connect to host 172.22.200.145 port 22: Connection timed out
~~~

### Tarea 2. Desde la LAN y la DMZ se debe permitir la conexión ssh por el puerto 22 al la máquina router-fw.

##### Reglas

LAN
~~~
sudo iptables -A INPUT -s 192.168.100.10/24 -p tcp -m tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -d 192.168.100.10/24 -p tcp -m tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
~~~

DMZ
~~~
sudo iptables -A INPUT -s 192.168.200.10/16 -p tcp -m tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A OUTPUT -d 192.168.200.10/16 -p tcp -m tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
~~~

##### Comprobación

LAN
~~~
debian@router-fw:~$ ssh -A debian@192.168.100.10
    Linux lan 4.19.0-6-cloud-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20) x86_64

    The programs included with the Debian GNU/Linux system are free software;
    the exact distribution terms for each program are described in the
    individual files in /usr/share/doc/*/copyright.

    Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
    permitted by applicable law.
    Last login: Fri Dec 13 11:53:58 2019 from 192.168.100.2

debian@lan:~$ ssh debian@192.168.100.2
    The authenticity of host '192.168.100.2 (192.168.100.2)' can't be established.
    ECDSA key fingerprint is SHA256:fnmA6k3OIDwXzXVMgrL3g+JjSjlmzRTU0Ou2xYwDdaE.
    Are you sure you want to continue connecting (yes/no)? yes
    Warning: Permanently added '192.168.100.2' (ECDSA) to the list of known hosts.
    Linux router-fw 4.19.0-6-cloud-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20)   x86_64

    The programs included with the Debian GNU/Linux system are free software;
    the exact distribution terms for each program are described in the
    individual files in /usr/share/doc/*/copyright.

    Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
    permitted by applicable law.
    Last login: Fri Dec 13 12:01:34 2019 from 172.22.1.248

debian@router-fw:~$ exit
~~~

DMZ
~~~
debian@router-fw:~$ ssh -A debian@192.168.200.10
    Linux dmz 4.19.0-6-cloud-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20) x86_64

    The programs included with the Debian GNU/Linux system are free software;
    the exact distribution terms for each program are described in the
    individual files in /usr/share/doc/*/copyright.

    Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
    permitted by applicable law.
    Last login: Fri Dec 13 11:23:14 2019 from 192.168.200.2

debian@dmz:~$ ssh debian@192.168.200.2
    The authenticity of host '192.168.200.2 (192.168.200.2)' can't be established.
    ECDSA key fingerprint is SHA256:fnmA6k3OIDwXzXVMgrL3g+JjSjlmzRTU0Ou2xYwDdaE.
    Are you sure you want to continue connecting (yes/no)? yes
    Warning: Permanently added '192.168.200.2' (ECDSA) to the list of known hosts.
    Linux router-fw 4.19.0-6-cloud-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20)   x86_64

    The programs included with the Debian GNU/Linux system are free software;
    the exact distribution terms for each program are described in the
    individual files in /usr/share/doc/*/copyright.

    Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
    permitted by applicable law.
    Last login: Fri Dec 13 12:02:10 2019 from 192.168.100.10

debian@router-fw:~$ exit
~~~

### Tarea 3. La máquina router-fw debe tener permitido el tráfico para la interfaz loopback.

##### Reglas
~~~
sudo iptables -A INPUT -i lo -p icmp -j ACCEPT
sudo iptables -A OUTPUT -o lo -p icmp -j ACCEPT
~~~

##### Comprobación
~~~
debian@router-fw:~$ ping 127.0.0.1
    PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
    ping: sendmsg: Operation not permitted
    ping: sendmsg: Operation not permitted
    ^Lping: sendmsg: Operation not permitted
    ping: sendmsg: Operation not permitted
    ping: sendmsg: Operation not permitted
    ^C
    --- 127.0.0.1 ping statistics ---
    5 packets transmitted, 0 received, 100% packet loss, time 105ms

debian@router-fw:~$ sudo iptables -A INPUT -i lo -p icmp -j ACCEPT
debian@router-fw:~$ sudo iptables -A OUTPUT -o lo -p icmp -j ACCEPT

debian@router-fw:~$ ping 127.0.0.1
    PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
    64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.050 ms
    64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.048 ms
    64 bytes from 127.0.0.1: icmp_seq=3 ttl=64 time=0.074 ms
    64 bytes from 127.0.0.1: icmp_seq=4 ttl=64 time=0.064 ms
    ^C
    --- 127.0.0.1 ping statistics ---
    4 packets transmitted, 4 received, 0% packet loss, time 77ms
    rtt min/avg/max/mdev = 0.048/0.059/0.074/0.010 ms
~~~

### Tarea 4. A la máquina router-fw se le puede hacer ping desde la DMZ, pero desde la LAN se le debe rechazar la conexión (REJECT).

##### Reglas

LAN
~~~
sudo iptables -A INPUT -i eth1 -s 192.168.100.0/24 -p icmp -m icmp --icmp-type echo-request -j REJECT --reject-with icmp-port-unreachable
sudo iptables -A OUTPUT -o eth1 -d 192.168.100.0/24 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
sudo iptables -A OUTPUT -o eth1 -d 192.168.100.0/24 -p icmp -m state --state RELATED -j ACCEPT
~~~

DMZ
~~~
sudo iptables -A INPUT -i eth2 -s 192.168.200.0/24 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
sudo iptables -A OUTPUT -o eth2 -d 192.168.200.0/24 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
~~~

##### Comprobación
LAN
~~~
debian@lan:~$ ping 192.168.100.2
    PING 192.168.100.2 (192.168.100.2) 56(84) bytes of data.
    From 192.168.100.2 icmp_seq=1 Destination Port Unreachable
    From 192.168.100.2 icmp_seq=2 Destination Port Unreachable
    From 192.168.100.2 icmp_seq=3 Destination Port Unreachable
    From 192.168.100.2 icmp_seq=4 Destination Port Unreachable
    From 192.168.100.2 icmp_seq=5 Destination Port Unreachable
    ^C
    --- 192.168.100.2 ping statistics ---
    5 packets transmitted, 0 received, +5 errors, 100% packet loss, time 16ms
~~~

DMZ
~~~
debian@dmz:~$ ping 192.168.200.2
    PING 192.168.200.2 (192.168.200.2) 56(84) bytes of data.
    64 bytes from 192.168.200.2: icmp_seq=1 ttl=64 time=0.896 ms
    64 bytes from 192.168.200.2: icmp_seq=2 ttl=64 time=1.12 ms
    64 bytes from 192.168.200.2: icmp_seq=3 ttl=64 time=1.08 ms
    64 bytes from 192.168.200.2: icmp_seq=4 ttl=64 time=1.15 ms
    64 bytes from 192.168.200.2: icmp_seq=5 ttl=64 time=1.11 ms
    ^C
    --- 192.168.200.2 ping statistics ---
    5 packets transmitted, 5 received, 0% packet loss, time 10ms
    rtt min/avg/max/mdev = 0.896/1.070/1.145/0.098 ms
~~~

### Tarea 5. La máquina router-fw puede hacer ping a la LAN, la DMZ y al exterior.

##### Reglas

LAN
~~~
sudo iptables -A OUTPUT -o eth1 -d 192.168.100.0/24 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
sudo iptables -A INPUT -i eth1 -s 192.168.100.0/24 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
~~~

DMZ
~~~
sudo iptables -A OUTPUT -o eth2 -d 192.168.200.0/24 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
sudo iptables -A INPUT -i eth2 -s 192.168.200.0/24 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
~~~

Exterior
~~~
sudo iptables -A OUTPUT -o eth0 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
sudo iptables -A INPUT -i eth0 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
~~~

##### Comprobación

LAN
~~~
debian@router-fw:~$ ping 192.168.100.10
    PING 192.168.100.10 (192.168.100.10) 56(84) bytes of data.
    64 bytes from 192.168.100.10: icmp_seq=1 ttl=64 time=1.02 ms
    64 bytes from 192.168.100.10: icmp_seq=2 ttl=64 time=0.857 ms
    64 bytes from 192.168.100.10: icmp_seq=3 ttl=64 time=0.802 ms
    64 bytes from 192.168.100.10: icmp_seq=4 ttl=64 time=0.696 ms
    64 bytes from 192.168.100.10: icmp_seq=5 ttl=64 time=0.791 ms
    ^C
    --- 192.168.100.10 ping statistics ---
    5 packets transmitted, 5 received, 0% packet loss, time 10ms
    rtt min/avg/max/mdev = 0.696/0.832/1.015/0.106 ms
~~~

DMZ
~~~
debian@router-fw:~$ ping 192.168.200.10
    PING 192.168.200.10 (192.168.200.10) 56(84) bytes of data.
    64 bytes from 192.168.200.10: icmp_seq=1 ttl=64 time=1.80 ms
    64 bytes from 192.168.200.10: icmp_seq=2 ttl=64 time=0.997 ms
    64 bytes from 192.168.200.10: icmp_seq=3 ttl=64 time=0.790 ms
    64 bytes from 192.168.200.10: icmp_seq=4 ttl=64 time=1.05 ms
    64 bytes from 192.168.200.10: icmp_seq=5 ttl=64 time=1.14 ms
    ^C
    --- 192.168.200.10 ping statistics ---
    5 packets transmitted, 5 received, 0% packet loss, time 10ms
    rtt min/avg/max/mdev = 0.790/1.154/1.801/0.344 ms
~~~

Exterior
~~~
debian@router-fw:~$ ping 1.1.1.1
    PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
    64 bytes from 1.1.1.1: icmp_seq=1 ttl=55 time=41.4 ms
    64 bytes from 1.1.1.1: icmp_seq=2 ttl=55 time=41.0 ms
    64 bytes from 1.1.1.1: icmp_seq=3 ttl=55 time=41.9 ms
    64 bytes from 1.1.1.1: icmp_seq=4 ttl=55 time=42.1 ms
    64 bytes from 1.1.1.1: icmp_seq=5 ttl=55 time=42.10 ms
    ^C
    --- 1.1.1.1 ping statistics ---
    5 packets transmitted, 5 received, 0% packet loss, time 11ms
    rtt min/avg/max/mdev = 41.032/41.874/42.950/0.673 ms
~~~

### Tarea 6. Desde la máquina DMZ se puede hacer ping y conexión ssh a la máquina LAN.

##### Reglas

SSH
~~~
sudo iptables -A FORWARD -i eth2 -o eth1 -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i eth1 -o eth2 -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
~~~

PING
~~~
sudo iptables -A FORWARD -i eth2 -o eth1 -s 192.168.200.0/24 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
sudo iptables -A FORWARD -i eth1 -o eth2 -d 192.168.200.0/24 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
~~~

##### Comprobación
SSH
~~~
debian@dmz:~$ ssh 192.168.100.10
    The authenticity of host '192.168.100.10 (192.168.100.10)' can't be established.
    ECDSA key fingerprint is SHA256:YZMnTboVGppp2MCQN/Jz89AI3MR/Rx9pnQrB/R4jmJk.
    Are you sure you want to continue connecting (yes/no)? yes
    Warning: Permanently added '192.168.100.10' (ECDSA) to the list of known hosts.
    Linux lan 4.19.0-6-cloud-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20) x86_64

    The programs included with the Debian GNU/Linux system are free software;
    the exact distribution terms for each program are described in the
    individual files in /usr/share/doc/*/copyright.

    Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
    permitted by applicable law.
    Last login: Tue Dec 31 16:44:04 2019 from 192.168.100.2

debian@lan:~$ exit
~~~

PING
~~~
debian@dmz:~$ ping 192.168.100.10
    PING 192.168.100.10 (192.168.100.10) 56(84) bytes of data.
    64 bytes from 192.168.100.10: icmp_seq=1 ttl=63 time=1.62 ms
    64 bytes from 192.168.100.10: icmp_seq=2 ttl=63 time=1.49 ms
    64 bytes from 192.168.100.10: icmp_seq=3 ttl=63 time=1.87 ms
    64 bytes from 192.168.100.10: icmp_seq=4 ttl=63 time=1.59 ms
    64 bytes from 192.168.100.10: icmp_seq=5 ttl=63 time=1.47 ms
    ^C
    --- 192.168.100.10 ping statistics ---
    5 packets transmitted, 5 received, 0% packet loss, time 12ms
    rtt min/avg/max/mdev = 1.465/1.607/1.867/0.144 ms
~~~

### Tarea 7. Desde la máquina LAN no se puede hacer ping, pero si se puede conectar por ssh a la máquina DMZ.

##### Reglas
SSH
~~~
sudo iptables -A FORWARD -i eth1 -o eth2 -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i eth2 -o eth1 -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
~~~

PING
> Por defecto esta en DROP


##### Comprobación
SSH
~~~
debian@lan:~$ ssh 192.168.200.10
    The authenticity of host '192.168.200.10 (192.168.200.10)' can't be established.
    ECDSA key fingerprint is SHA256:HDhQ4dDIMfEt0p916tj987SZjuhmhqd+qEjGNLikwN8.
    Are you sure you want to continue connecting (yes/no)? yes
    Warning: Permanently added '192.168.200.10' (ECDSA) to the list of known hosts.
    Linux dmz 4.19.0-6-cloud-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20) x86_64

    The programs included with the Debian GNU/Linux system are free software;
    the exact distribution terms for each program are described in the
    individual files in /usr/share/doc/*/copyright.

    Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
    permitted by applicable law.
    Last login: Tue Dec 31 16:46:29 2019 from 192.168.200.2

debian@dmz:~$ exit
~~~

PING
~~~
debian@lan:~$ ping 192.168.200.10
    PING 192.168.200.10 (192.168.200.10) 56(84) bytes of data.
    ^C
    --- 192.168.200.10 ping statistics ---
    247 packets transmitted, 0 received, 100% packet loss, time 1150ms
~~~

### Tarea 8. Configura la máquina router-fw para que las máquinas LAN y DMZ puedan acceder al exterior.

Añadimos las reglas de SNAT para que las máquinas internas puedan acceder al exterior
~~~
sudo iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o eth0 -j MASQUERADE
sudo iptables -t nat -A POSTROUTING -s 192.168.200.0/24 -o eth0 -j MASQUERADE
~~~

### Tarea 9. La máquina LAN se le permite hacer ping al exterior.

##### Reglas
~~~
sudo iptables -A FORWARD -i eth1 -o eth0 -p icmp -m icmp --icmp-type echo-request -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o eth1 -p icmp -m icmp --icmp-type echo-reply -j ACCEPT
~~~

##### Comprobación
~~~
debian@lan:~$ ping 1.1.1.1
    PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
    64 bytes from 1.1.1.1: icmp_seq=1 ttl=54 time=42.6 ms
    64 bytes from 1.1.1.1: icmp_seq=2 ttl=54 time=41.10 ms
    64 bytes from 1.1.1.1: icmp_seq=3 ttl=54 time=68.3 ms
    64 bytes from 1.1.1.1: icmp_seq=4 ttl=54 time=66.2 ms
    64 bytes from 1.1.1.1: icmp_seq=5 ttl=54 time=43.0 ms
    ^C
    --- 1.1.1.1 ping statistics ---
    5 packets transmitted, 5 received, 0% packet loss, time 11ms
    rtt min/avg/max/mdev = 41.959/52.429/68.349/12.161 ms
~~~

### Tarea 10. La máquina LAN puede navegar.

##### Reglas
Activamos DNS
~~~
sudo iptables -A FORWARD -i eth1 -o eth0 -p udp --dport 53 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o eth1 -p udp --sport 53 -m state --state ESTABLISHED -j ACCEPT
~~~

Activamos HTTP
~~~
sudo iptables -A FORWARD -i eth1 -o eth0 -p tcp -m multiport --dports 80 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o eth1 -p tcp -m multiport --sports 80 -m state --state ESTABLISHED -j ACCEPT
~~~

Activamos HTTPS
~~~
sudo iptables -A FORWARD -i eth1 -o eth0 -p tcp -m multiport --dports 443 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o eth1 -p tcp -m multiport --sports 443 -m state --state ESTABLISHED -j ACCEPT
~~~

##### Comprobación
Descargamos un paquete
~~~
debian@lan:~$ sudo apt install tree
Reading package lists... Done
Building dependency tree
Reading state information... Done
The following NEW packages will be installed:
  tree
0 upgraded, 1 newly installed, 0 to remove and 37 not upgraded.
Need to get 49.3 kB of archives.
After this operation, 117 kB of additional disk space will be used.
Get:1 http://deb.debian.org/debian buster/main amd64 tree amd64 1.8.0-1 [49.3 kB]
Fetched 49.3 kB in 0s (247 kB/s)
Selecting previously unselected package tree.
(Reading database ... 26978 files and directories currently installed.)
Preparing to unpack .../tree_1.8.0-1_amd64.deb ...
Unpacking tree (1.8.0-1) ...
Setting up tree (1.8.0-1) ...
~~~

### Tarea 11. La máquina DMZ puede navegar. Instala un servidor web, un servidor ftp y un servidor de correos.

##### Reglas
Activamos DNS
~~~
sudo iptables -A FORWARD -i eth2 -o eth0 -p udp --dport 53 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o eth2 -p udp --sport 53 -m state --state ESTABLISHED -j ACCEPT
~~~

Activamos HTTP
~~~
sudo iptables -A FORWARD -i eth2 -o eth0 -p tcp --dport 80 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o eth2 -p tcp --sport 80 -m state --state ESTABLISHED -j ACCEPT
~~~

Activamos HTTPS
~~~
sudo iptables -A FORWARD -i eth2 -o eth0 -p tcp --dport 443 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i eth0 -o eth2 -p tcp --sport 443 -m state --state ESTABLISHED -j ACCEPT
~~~

##### Comprobación
~~~
debian@dmz:~$ sudo apt install apache2 postfix proftpd
    Reading package lists... Done
    Building dependency tree
    Reading state information... Done
    Note, selecting 'proftpd-basic' instead of 'proftpd'
    The following additional packages will be installed:
      apache2-bin apache2-data apache2-utils libapr1 libaprutil1 libaprutil1-dbd-sqlite3    libaprutil1-ldap
      libbrotli1 libcurl4 libgdbm-compat4 libgdbm6 libhiredis0.14 libjansson4   libldap-2.4-2 libldap-common
      liblua5.2-0 libmemcached11 libmemcachedutil2 libnghttp2-14 libperl5.28 librtmp1   libsasl2-2
      libsasl2-modules libsasl2-modules-db libssh2-1 perl perl-modules-5.28 proftpd-doc     ssl-cert
    Suggested packages:
      apache2-doc apache2-suexec-pristine | apache2-suexec-custom www-browser   libsasl2-modules-gssapi-mit
      | libsasl2-modules-gssapi-heimdal libsasl2-modules-ldap libsasl2-modules-otp  libsasl2-modules-sql
      perl-doc libterm-readline-gnu-perl | libterm-readline-perl-perl make libb-debug-perl
      liblocale-codes-perl procmail postfix-mysql postfix-pgsql postfix-ldap postfix-pcre   postfix-lmdb
      postfix-sqlite sasl2-bin | dovecot-common resolvconf postfix-cdb mail-reader ufw  postfix-doc
      openbsd-inetd | inet-superserver proftpd-mod-ldap proftpd-mod-mysql proftpd-mod-odbc
      proftpd-mod-pgsql proftpd-mod-sqlite proftpd-mod-geoip proftpd-mod-snmp   openssl-blacklist
    The following NEW packages will be installed:
      apache2 apache2-bin apache2-data apache2-utils libapr1 libaprutil1    libaprutil1-dbd-sqlite3
      libaprutil1-ldap libbrotli1 libcurl4 libgdbm-compat4 libgdbm6 libhiredis0.14  libjansson4
      libldap-2.4-2 libldap-common liblua5.2-0 libmemcached11 libmemcachedutil2     libnghttp2-14 libperl5.28
      librtmp1 libsasl2-2 libsasl2-modules libsasl2-modules-db libssh2-1 perl   perl-modules-5.28 postfix
      proftpd-basic proftpd-doc ssl-cert
    0 upgraded, 32 newly installed, 0 to remove and 0 not upgraded.
    Need to get 16.9 MB of archives.
    After this operation, 72.6 MB of additional disk space will be used.
Do you want to continue? [Y/n] y
    Get:3 http://deb.debian.org/debian buster/main amd64 perl-modules-5.28 all 5.28.1-6     [2,873 kB]
~~~

### Tarea 12. Configura la máquina router-fw para que los servicios web y ftp sean accesibles desde el exterior.

> ### Próximamente.

### Tarea 13. El servidor web y el servidor ftp deben ser accesible desde la LAN y desde el exterior.

> ### Próximamente.

### Tarea 14. El servidor de correos sólo debe ser accesible desde la LAN.

En la instalación elegimos *internet site*.

~~~
   ┌───────────────────────────────────┤ Postfix Configuration ├────────────────────────────────────┐
   │ Escoja el tipo de configuración del servidor de correo que se ajusta mejor a sus necesidades.  │
   │                                                                                                │
   │  Sin configuración:                                                                            │
   │   Mantiene la configuración actual intacta.                                                    │
   │  Sitio de Internet:                                                                            │
   │   El correo se envía y recibe directamente utilizando SMTP.                                    │
   │  Internet con «smarthost»:                                                                     │
   │   El correo se recibe directamente utilizando SMTP o ejecutando una                            │
   │   herramienta como «fetchmail». El correo de salida se envía utilizando                        │
   │   un «smarthost».                                                                              │
   │  Sólo correo local:                                                                            │
   │   El único correo que se entrega es para los usuarios locales. No                              │
   │   hay red.                                                                                     │
   │                                                                                                │
   │ Tipo genérico de configuración de correo:                                                      │
   │                                                                                                │
   │                                   Sin configuración                                            │
   │                            >>>>   Sitio de Internet                                            │
   │                                   Internet con «smarthost»                                     │
   │                                   Sistema satélite                                             │
   │                                   Sólo correo local                                            │
   │                                                                                                │
   │                                                                                                │
   │                           <Aceptar>                          <Cancelar>                        │
   │                                                                                                │
   └────────────────────────────────────────────────────────────────────────────────────────────────┘
~~~

Ahora vamos a especificar las redes permitidas por el servidor de correos en el fichero ```/etc/postfix/main.cf```, modificando las siguiente linea:

~~~
mynetworks = 127.0.0.0/8 192.168.100.0/24
~~~

##### Reglas
~~~
sudo iptables -A FORWARD -i eth1 -o eth2 -p tcp --dport 25 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i eth2 -o eth1 -p tcp --sport 25 -m state --state ESTABLISHED -j ACCEPT
~~~

##### Comprobación
~~~
debian@lan:~$ telnet 192.168.200.10 25
    Trying 192.168.200.10...
    Connected to 192.168.200.10.
    Escape character is '^]'.
    220 dmz.novalocal ESMTP Postfix (Debian/GNU)
~~~

### Tarea 15. En la máquina LAN instala un servidor mysql. A este servidor sólo se puede acceder desde la DMZ.

Instalamos en la lan el servidor y en la dmz el cliente de mariadb.


En la la LAN
~~~
sudo apt install mariadb-server
~~~

En la DMZ

~~~
sudo apt install mariadb-client
~~~

Creamos en el servidor la una base de datos y un usuario, además le asignamos los permisos para que podamos acceder desde la DMZ

~~~
MariaDB [(none)]> CREATE DATABASE prueba;
    Query OK, 1 row affected (0.001 sec)

MariaDB [(none)]> CREATE USER user_prueba@"192.168.200.10" IDENTIFIED BY "user_prueba";
    Query OK, 0 rows affected (0.045 sec)

MariaDB [(none)]> GRANT ALL PRIVILEGES ON prueba.* to user_prueba IDENTIFIED BY "user_prueba";
    Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> FLUSH PRIVILEGES;
    Query OK, 0 rows affected (0.001 sec)
~~~

Editamos el fichero ```/etc/mysql/mariadb.conf.d/50-server.cnf``` y añadimos la siguiente linea:

~~~
bind-address            = 0.0.0.0
~~~

Reiniciamos el sericio:

~~~
sudo systemctl restart mariadb.service 
~~~

##### Reglas
~~~
sudo iptables -A FORWARD -i eth2 -o eth1 -p tcp --dport 3306 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i eth1 -o eth2 -p tcp --sport 3306 -m state --state ESTABLISHED -j ACCEPT
~~~

##### Comprobación
~~~
debian@dmz:~$ sudo mysql -u user_prueba -p prueba -h 192.168.100.10
Enter password: 
    Welcome to the MariaDB monitor.  Commands end with ; or \g.
    Your MariaDB connection id is 36
    Server version: 10.3.18-MariaDB-0+deb10u1 Debian 10

    Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [prueba]> use prueba
    Database changed
MariaDB [prueba]>
~~~

## Mejoras
------------------------------------------------------------------------------------------------

#### MEJORA 1: Implementar que el cortafuego funcione después de un reinicio de la máquina.

Con ```root@router-fw:``` creamos el fichero *iptableSave.v4* donde guardaremos todas la reglas de iptables que esten activas en el momento de ejecitar dicho comando.
~~~
iptables-save > /etc/iproute2/iptableSave.v4
~~~

~~~
root@router-fw:~# cat /etc/iproute2/iptableSave.v4 
    # Generated by xtables-save v1.8.2 on Fri Jan  3 18:03:13 2020
    *filter
    :INPUT DROP [29:4732]
    :FORWARD DROP [0:0]
    :OUTPUT DROP [636:48678]
    -A INPUT -s 172.22.0.0/16 -p tcp -m tcp --dport 22 -m state --state NEW,ESTABLISHED     -j ACCEPT
    -A INPUT -s 172.23.0.0/16 -p tcp -m tcp --dport 22 -m state --state NEW,ESTABLISHED     -j ACCEPT
    -A INPUT -s 192.168.100.0/24 -i eth1 -p tcp -m tcp --sport 22 -m state --state  ESTABLISHED -j ACCEPT
    -A INPUT -s 192.168.200.0/24 -i eth2 -p tcp -m tcp --sport 22 -m state --state  ESTABLISHED -j ACCEPT
    -A INPUT -s 172.22.0.0/16 -p tcp -m tcp --dport 2222 -m state --state NEW,ESTABLISHED   -j ACCEPT
    -A INPUT -s 172.23.0.0/16 -p tcp -m tcp --dport 2222 -m state --state NEW,ESTABLISHED   -j ACCEPT
    -A INPUT -s 192.168.100.0/24 -p tcp -m tcp --dport 22 -m state --state NEW, ESTABLISHED -j ACCEPT
    -A INPUT -s 192.168.0.0/16 -p tcp -m tcp --dport 22 -m state --state NEW,ESTABLISHED    -j ACCEPT
    -A INPUT -i lo -p icmp -j ACCEPT
    -A INPUT -s 192.168.100.0/24 -i eth1 -p icmp -m icmp --icmp-type 8 -j REJECT    --reject-with icmp-port-unreachable
    -A INPUT -s 192.168.200.0/24 -i eth2 -p icmp -m icmp --icmp-type 8 -j ACCEPT
    -A INPUT -s 192.168.100.0/24 -i eth1 -p icmp -m icmp --icmp-type 0 -j ACCEPT
    -A INPUT -s 192.168.200.0/24 -i eth2 -p icmp -m icmp --icmp-type 0 -j ACCEPT
    -A INPUT -i eth0 -p icmp -m icmp --icmp-type 0 -j ACCEPT
    -A FORWARD -i eth2 -o eth1 -p tcp -m tcp --dport 22 -m state --state NEW,ESTABLISHED    -j ACCEPT
    -A FORWARD -i eth1 -o eth2 -p tcp -m tcp --sport 22 -m state --state ESTABLISHED -j     ACCEPT
    -A FORWARD -s 192.168.200.0/24 -i eth2 -o eth1 -p icmp -m icmp --icmp-type 8 -j ACCEPT
    -A FORWARD -d 192.168.200.0/24 -i eth1 -o eth2 -p icmp -m icmp --icmp-type 0 -j ACCEPT
    -A FORWARD -i eth1 -o eth2 -p tcp -m tcp --dport 22 -m state --state NEW,ESTABLISHED    -j ACCEPT
    -A FORWARD -i eth2 -o eth1 -p tcp -m tcp --sport 22 -m state --state ESTABLISHED -j     ACCEPT
    -A FORWARD -i eth1 -o eth0 -p icmp -m icmp --icmp-type 8 -j ACCEPT
    -A FORWARD -i eth0 -o eth1 -p icmp -m icmp --icmp-type 0 -j ACCEPT
    -A FORWARD -i eth1 -o eth0 -p udp -m udp --dport 53 -m state --state NEW,ESTABLISHED    -j ACCEPT
    -A FORWARD -i eth0 -o eth1 -p udp -m udp --sport 53 -m state --state ESTABLISHED -j     ACCEPT
    -A FORWARD -i eth1 -o eth0 -p tcp -m multiport --dports 80 -m state --state NEW,    ESTABLISHED -j ACCEPT
    -A FORWARD -i eth0 -o eth1 -p tcp -m multiport --sports 80 -m state --state     ESTABLISHED -j ACCEPT
    -A FORWARD -i eth1 -o eth0 -p tcp -m multiport --dports 443 -m state --state NEW,   ESTABLISHED -j ACCEPT
    -A FORWARD -i eth0 -o eth1 -p tcp -m multiport --sports 443 -m state --state    ESTABLISHED -j ACCEPT
    -A FORWARD -i eth2 -o eth0 -p udp -m udp --dport 53 -m state --state NEW,ESTABLISHED    -j ACCEPT
    -A FORWARD -i eth0 -o eth2 -p udp -m udp --sport 53 -m state --state ESTABLISHED -j     ACCEPT
    -A FORWARD -i eth2 -o eth0 -p tcp -m tcp --dport 80 -m state --state NEW,ESTABLISHED    -j ACCEPT
    -A FORWARD -i eth0 -o eth2 -p tcp -m tcp --sport 80 -m state --state ESTABLISHED -j     ACCEPT
    -A FORWARD -i eth2 -o eth0 -p tcp -m tcp --dport 443 -m state --state NEW,ESTABLISHED   -j ACCEPT
    -A FORWARD -i eth0 -o eth2 -p tcp -m tcp --sport 443 -m state --state ESTABLISHED -j    ACCEPT
    -A FORWARD -i eth1 -o eth2 -p tcp -m tcp --dport 25 -m state --state NEW,ESTABLISHED    -j ACCEPT
    -A FORWARD -i eth2 -o eth1 -p tcp -m tcp --sport 25 -m state --state ESTABLISHED -j     ACCEPT
    -A FORWARD -i eth2 -o eth1 -p tcp -m tcp --dport 3306 -m state --state NEW, ESTABLISHED -j ACCEPT
    -A FORWARD -i eth1 -o eth2 -p tcp -m tcp --sport 3306 -m state --state ESTABLISHED -j   ACCEPT
    -A OUTPUT -d 172.22.0.0/16 -p tcp -m tcp --sport 22 -m state --state ESTABLISHED -j     ACCEPT
    -A OUTPUT -d 172.23.0.0/16 -p tcp -m tcp --sport 22 -m state --state ESTABLISHED -j     ACCEPT
    -A OUTPUT -d 192.168.100.0/24 -o eth1 -p tcp -m tcp --dport 22 -m state --state NEW,    ESTABLISHED -j ACCEPT
    -A OUTPUT -d 192.168.200.0/24 -o eth2 -p tcp -m tcp --dport 22 -m state --state NEW,    ESTABLISHED -j ACCEPT
    -A OUTPUT -d 172.22.0.0/16 -p tcp -m tcp --sport 2222 -m state --state ESTABLISHED -j   ACCEPT
    -A OUTPUT -d 172.23.0.0/16 -p tcp -m tcp --sport 2222 -m state --state ESTABLISHED -j   ACCEPT
    -A OUTPUT -d 192.168.100.0/24 -p tcp -m tcp --sport 22 -m state --state ESTABLISHED     -j ACCEPT
    -A OUTPUT -d 192.168.0.0/16 -p tcp -m tcp --sport 22 -m state --state ESTABLISHED -j    ACCEPT
    -A OUTPUT -o lo -p icmp -j ACCEPT
    -A OUTPUT -d 192.168.100.0/24 -o eth1 -p icmp -m icmp --icmp-type 0 -j ACCEPT
    -A OUTPUT -d 192.168.100.0/24 -o eth1 -p icmp -m state --state RELATED -j ACCEPT
    -A OUTPUT -d 192.168.200.0/24 -o eth2 -p icmp -m icmp --icmp-type 0 -j ACCEPT
    -A OUTPUT -d 192.168.100.0/24 -o eth1 -p icmp -m icmp --icmp-type 8 -j ACCEPT
    -A OUTPUT -d 192.168.200.0/24 -o eth2 -p icmp -m icmp --icmp-type 8 -j ACCEPT
    -A OUTPUT -o eth0 -p icmp -m icmp --icmp-type 8 -j ACCEPT
    COMMIT
    # Completed on Fri Jan  3 18:03:13 2020
    # Generated by xtables-save v1.8.2 on Fri Jan  3 18:03:13 2020
    *nat
    :PREROUTING ACCEPT [13:875]
    :INPUT ACCEPT [3:180]
    :POSTROUTING ACCEPT [6:580]
    :OUTPUT ACCEPT [625:47026]
    -A PREROUTING -s 172.23.0.0/16 -p tcp -m tcp --dport 22 -j DNAT --to-destination    127.0.0.1
    -A PREROUTING -s 172.22.0.0/16 -p tcp -m tcp --dport 22 -j DNAT --to-destination    127.0.0.1
    -A PREROUTING -s 172.23.0.0/16 -p tcp -m tcp --dport 2222 -j REDIRECT --to-ports 22
    -A PREROUTING -s 172.22.0.0/16 -p tcp -m tcp --dport 2222 -j REDIRECT --to-ports 22
    -A POSTROUTING -s 192.168.100.0/24 -o eth0 -j MASQUERADE
    -A POSTROUTING -s 192.168.200.0/24 -o eth0 -j MASQUERADE
    COMMIT
    # Completed on Fri Jan  3 18:03:13 2020
~~~

Creamos la unidad de systemd, para esto, tenemos que crear el fichero ```/etc/systemd/system/restart-iptables.service``` y añadir las siguientes lineas:

~~~
[Unit]
Description=restaurar las iptables
After=networking.service

[Service]
ExecStart=/usr/local/bin/restart-iptables.sh

[Install]
WantedBy=multi-user.target
~~~

Creamos el script en la ruta indicada en el fichero de systemd ```/usr/local/bin/restart-iptables.sh```

~~~
#!/bin/bash
iptables-restore < /etc/iproute2/iptableSave.v4
echo "Reglas de iptables restauradas"
~~~

Cambiamos los permisos del script:

~~~
chmod 744 /usr/local/bin/restart-iptables.sh
~~~

Activamos el servicio para que siempre se active al inicio del sistema:

~~~
systemctl enable restart-iptables.service
    Created symlink /etc/systemd/system/multi-user.target.wants/restart-iptables.service → /etc/systemd/system/restart-iptables.service.
~~~

Iniciamos el servicio:

~~~
systemctl start restart-iptables.service
~~~

##### Comprobación:

~~~
root@router-fw:~# reboot
root@router-fw:~# Connection to 172.22.200.145 closed by remote host.
    Connection to 172.22.200.145 closed.

moralg@padano:~$ ssh -A -p 2222 debian@172.22.200.145
    Linux router-fw 4.19.0-6-cloud-amd64 #1 SMP Debian 4.19.67-2+deb10u1 (2019-09-20)   x86_64

    The programs included with the Debian GNU/Linux system are free software;
    the exact distribution terms for each program are described in the
    individual files in /usr/share/doc/*/copyright.

    Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
    permitted by applicable law.
    Last login: Fri Jan  3 17:03:31 2020 from 172.23.0.54

debian@router-fw:~$ sudo iptables -n -L
    Chain INPUT (policy DROP)
    target     prot opt source               destination         
    ACCEPT     tcp  --  172.22.0.0/16        0.0.0.0/0            tcp dpt:22 state NEW, ESTABLISHED
    ACCEPT     tcp  --  172.23.0.0/16        0.0.0.0/0            tcp dpt:22 state NEW, ESTABLISHED
    ACCEPT     tcp  --  192.168.100.0/24     0.0.0.0/0            tcp spt:22 state  ESTABLISHED
    ACCEPT     tcp  --  192.168.200.0/24     0.0.0.0/0            tcp spt:22 state  ESTABLISHED
    ACCEPT     tcp  --  172.22.0.0/16        0.0.0.0/0            tcp dpt:2222 state NEW,   ESTABLISHED
    ACCEPT     tcp  --  172.23.0.0/16        0.0.0.0/0            tcp dpt:2222 state NEW,   ESTABLISHED
    ACCEPT     tcp  --  192.168.100.0/24     0.0.0.0/0            tcp dpt:22 state NEW, ESTABLISHED
    ACCEPT     tcp  --  192.168.0.0/16       0.0.0.0/0            tcp dpt:22 state NEW, ESTABLISHED
    ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0           
    REJECT     icmp --  192.168.100.0/24     0.0.0.0/0            icmptype 8 reject-with    icmp-port-unreachable
    ACCEPT     icmp --  192.168.200.0/24     0.0.0.0/0            icmptype 8
    ACCEPT     icmp --  192.168.100.0/24     0.0.0.0/0            icmptype 0
    ACCEPT     icmp --  192.168.200.0/24     0.0.0.0/0            icmptype 0
    ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 0

    Chain FORWARD (policy DROP)
    target     prot opt source               destination         
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:22 state NEW, ESTABLISHED
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp spt:22 state  ESTABLISHED
    ACCEPT     icmp --  192.168.200.0/24     0.0.0.0/0            icmptype 8
    ACCEPT     icmp --  0.0.0.0/0            192.168.200.0/24     icmptype 0
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:22 state NEW, ESTABLISHED
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp spt:22 state  ESTABLISHED
    ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 8
    ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 0
    ACCEPT     udp  --  0.0.0.0/0            0.0.0.0/0            udp dpt:53 state NEW, ESTABLISHED
    ACCEPT     udp  --  0.0.0.0/0            0.0.0.0/0            udp spt:53 state  ESTABLISHED
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            multiport dports 80   state NEW,ESTABLISHED
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            multiport sports 80   state ESTABLISHED
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            multiport dports 443  state NEW,ESTABLISHED
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            multiport sports 443  state ESTABLISHED
    ACCEPT     udp  --  0.0.0.0/0            0.0.0.0/0            udp dpt:53 state NEW, ESTABLISHED
    ACCEPT     udp  --  0.0.0.0/0            0.0.0.0/0            udp spt:53 state  ESTABLISHED
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:80 state NEW, ESTABLISHED
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp spt:80 state  ESTABLISHED
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:443 state NEW,    ESTABLISHED
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp spt:443 state     ESTABLISHED
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:25 state NEW, ESTABLISHED
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp spt:25 state  ESTABLISHED
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:3306 state NEW,   ESTABLISHED
    ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp spt:3306 state    ESTABLISHED

    Chain OUTPUT (policy DROP)
    target     prot opt source               destination         
    ACCEPT     tcp  --  0.0.0.0/0            172.22.0.0/16        tcp spt:22 state  ESTABLISHED
    ACCEPT     tcp  --  0.0.0.0/0            172.23.0.0/16        tcp spt:22 state  ESTABLISHED
    ACCEPT     tcp  --  0.0.0.0/0            192.168.100.0/24     tcp dpt:22 state NEW, ESTABLISHED
    ACCEPT     tcp  --  0.0.0.0/0            192.168.200.0/24     tcp dpt:22 state NEW, ESTABLISHED
    ACCEPT     tcp  --  0.0.0.0/0            172.22.0.0/16        tcp spt:2222 state    ESTABLISHED
    ACCEPT     tcp  --  0.0.0.0/0            172.23.0.0/16        tcp spt:2222 state    ESTABLISHED
    ACCEPT     tcp  --  0.0.0.0/0            192.168.100.0/24     tcp spt:22 state  ESTABLISHED
    ACCEPT     tcp  --  0.0.0.0/0            192.168.0.0/16       tcp spt:22 state  ESTABLISHED
    ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0           
    ACCEPT     icmp --  0.0.0.0/0            192.168.100.0/24     icmptype 0
    ACCEPT     icmp --  0.0.0.0/0            192.168.100.0/24     state RELATED
    ACCEPT     icmp --  0.0.0.0/0            192.168.200.0/24     icmptype 0
    ACCEPT     icmp --  0.0.0.0/0            192.168.100.0/24     icmptype 8
    ACCEPT     icmp --  0.0.0.0/0            192.168.200.0/24     icmptype 8
    ACCEPT     icmp --  0.0.0.0/0            0.0.0.0/0            icmptype 8
~~~

#### MEJORA 2: Utiliza nuevas cadenas para clasificar el tráfico.

> ### Próximamente.

#### MEJORA 3: Consruye el cortafuego utilizando nftables.

Sustituyendo en cada regla ```iptables``` por ```iptables-translate``` nos saldrá por pantalla la regla traduccida a nftables, pero con algunos errores.

~~~
debian@router-fw:~$ sudo iptables-translate -A FORWARD -i eth1 -o eth0 -p tcp -m multiport --dports 443 -m state --state NEW,ESTABLISHED -j ACCEPT
    nft add rule ip filter FORWARD iifname "eth1" oifname "eth0" ip protocol tcp tcp dport 443 ct state new,established  counter accept
~~~

Como vemos en la regla de nftables que nos devuelve, encontramos los siguientes fallos:

* ip: En nftables sería inet
* FORWARD: En nftables tendría que ser forward en minúsculas

Para solucionar estos fallos creamos el siguiente [SCRIPT](https://github.com/MoralG/Cortafuego_Perimetral_con_DMZ/blob/master/Translate-iptables.sh):

``` sh
#!/bin/sh

sudo rm /home/debian/fichero1 2> /dev/null
sudo rm /home/debian/fichero2 2> /dev/null
sudo rm /home/debian/fichero3 2> /dev/null

sudo iptables -S > /home/debian/fichero1
sudo iptables -t nat -S >> /home/debian/fichero1

while read linea 
do
    sudo iptables-translate $linea >> /home/debian/fichero2
done < fichero1

sed 's/ ip /inet/g' /home/debian/fichero2 > /home/debian/fichero3
tr A-Z a-z < /home/debian/fichero3 > /home/debian/nftables.txt 

sudo rm /home/debian/fichero1 2> /dev/null
sudo rm /home/debian/fichero2 2> /dev/null
sudo rm /home/debian/fichero3 2> /dev/null
```

Ejecutamos el script:
~~~
bash Translate-iptables.sh 
~~~

Nos crea un fichero con la reglas de nftables corregidas.
~~~
cat nftables.txt 
    nft add rule inet filter input inet saddr 172.22.0.0/16 tcp dport 22 ct state new,established  counter accept
    nft add rule inet filter input inet saddr 172.23.0.0/16 tcp dport 22 ct state new,established  counter accept
    nft add rule inet filter input iifname "eth1" inet saddr 192.168.100.0/24 tcp sport 22 ct state established  counter accept
    nft add rule inet filter input iifname "eth2" inet saddr 192.168.200.0/24 tcp sport 22 ct state established  counter accept
    nft add rule inet filter input inet saddr 172.22.0.0/16 tcp dport 2222 ct state new,established  counter accept
    nft add rule inet filter input inet saddr 172.23.0.0/16 tcp dport 2222 ct state new,established  counter accept
    nft add rule inet filter input inet saddr 192.168.100.0/24 tcp dport 22 ct state new,established  counter accept
    nft add rule inet filter input inet saddr 192.168.0.0/16 tcp dport 22 ct state new,established  counter accept
    nft add rule inet filter input iifname "lo" inet protocol icmp counter accept
    nft add rule inet filter input iifname "eth1" inet saddr 192.168.100.0/24 icmp type echo-request counter reject
    nft add rule inet filter input iifname "eth2" inet saddr 192.168.200.0/24 icmp type echo-request counter accept
    nft add rule inet filter input iifname "eth1" inet saddr 192.168.100.0/24 icmp type echo-reply counter accept
    nft add rule inet filter input iifname "eth2" inet saddr 192.168.200.0/24 icmp type echo-reply counter accept
    nft add rule inet filter input iifname "eth0" icmp type echo-reply counter accept
    nft add rule inet filter forward iifname "eth2" oifname "eth1" tcp dport 22 ct state new,established  counter accept
    nft add rule inet filter forward iifname "eth1" oifname "eth2" tcp sport 22 ct state established  counter accept
    nft add rule inet filter forward iifname "eth2" oifname "eth1" inet saddr 192.168.200.0/24 icmp type echo-request counter accept
    nft add rule inet filter forward iifname "eth1" oifname "eth2" inet daddr 192.168.200.0/24 icmp type echo-reply counter accept
    nft add rule inet filter forward iifname "eth1" oifname "eth2" tcp dport 22 ct state new,established  counter accept
    nft add rule inet filter forward iifname "eth2" oifname "eth1" tcp sport 22 ct state established  counter accept
    nft add rule inet filter forward iifname "eth1" oifname "eth0" icmp type echo-request counter accept
    nft add rule inet filter forward iifname "eth0" oifname "eth1" icmp type echo-reply counter accept
    nft add rule inet filter forward iifname "eth1" oifname "eth0" udp dport 53 ct state new,established  counter accept
    nft add rule inet filter forward iifname "eth0" oifname "eth1" udp sport 53 ct state established  counter accept
    nft add rule inet filter forward iifname "eth1" oifname "eth0" inet protocol tcp tcp dport 80 ct state new,established  counter accept
    nft add rule inet filter forward iifname "eth0" oifname "eth1" inet protocol tcp tcp sport 80 ct state established  counter accept
    nft add rule inet filter forward iifname "eth1" oifname "eth0" inet protocol tcp tcp dport 443 ct state new,established  counter accept
    nft add rule inet filter forward iifname "eth0" oifname "eth1" inet protocol tcp tcp sport 443 ct state established  counter accept
    nft add rule inet filter forward iifname "eth2" oifname "eth0" udp dport 53 ct state new,established  counter accept
    nft add rule inet filter forward iifname "eth0" oifname "eth2" udp sport 53 ct state established  counter accept
    nft add rule inet filter forward iifname "eth2" oifname "eth0" tcp dport 80 ct state new,established  counter accept
    nft add rule inet filter forward iifname "eth0" oifname "eth2" tcp sport 80 ct state established  counter accept
    nft add rule inet filter forward iifname "eth2" oifname "eth0" tcp dport 443 ct state new,established  counter accept
    nft add rule inet filter forward iifname "eth0" oifname "eth2" tcp sport 443 ct state established  counter accept
    nft add rule inet filter forward iifname "eth1" oifname "eth2" tcp dport 25 ct state new,established  counter accept
    nft add rule inet filter forward iifname "eth2" oifname "eth1" tcp sport 25 ct state established  counter accept
    nft add rule inet filter forward iifname "eth2" oifname "eth1" tcp dport 3306 ct state new,established  counter accept
    nft add rule inet filter forward iifname "eth1" oifname "eth2" tcp sport 3306 ct state established  counter accept
    nft add rule inet filter output inet daddr 172.22.0.0/16 tcp sport 22 ct state established  counter accept
    nft add rule inet filter output inet daddr 172.23.0.0/16 tcp sport 22 ct state established  counter accept
    nft add rule inet filter output oifname "eth1" inet daddr 192.168.100.0/24 tcp dport 22 ct state new,established  counter accept
    nft add rule inet filter output oifname "eth2" inet daddr 192.168.200.0/24 tcp dport 22 ct state new,established  counter accept
    nft add rule inet filter output inet daddr 172.22.0.0/16 tcp sport 2222 ct state established  counter accept
    nft add rule inet filter output inet daddr 172.23.0.0/16 tcp sport 2222 ct state established  counter accept
    nft add rule inet filter output inet daddr 192.168.100.0/24 tcp sport 22 ct state established  counter accept
    nft add rule inet filter output inet daddr 192.168.0.0/16 tcp sport 22 ct state established  counter accept
    nft add rule inet filter output oifname "lo" inet protocol icmp counter accept
    nft add rule inet filter output oifname "eth1" inet daddr 192.168.100.0/24 icmp type echo-reply counter accept
    nft add rule inet filter output oifname "eth1" inet protocol icmp inet daddr 192.168.100.0/24 ct state related  counter accept
    nft add rule inet filter output oifname "eth2" inet daddr 192.168.200.0/24 icmp type echo-reply counter accept
    nft add rule inet filter output oifname "eth1" inet daddr 192.168.100.0/24 icmp type echo-request counter accept
    nft add rule inet filter output oifname "eth2" inet daddr 192.168.200.0/24 icmp type echo-request counter accept
    nft add rule inet filter output oifname "eth0" icmp type echo-request counter accept
    nft add rule inet filter prerouting inet saddr 172.23.0.0/16 tcp dport 22 counter dnat to 127.0.0.1
    nft add rule inet filter prerouting inet saddr 172.22.0.0/16 tcp dport 22 counter dnat to 127.0.0.1
    nft add rule inet filter prerouting inet saddr 172.23.0.0/16 tcp dport 2222 counter redirect to :22
    nft add rule inet filter prerouting inet saddr 172.22.0.0/16 tcp dport 2222 counter redirect to :22
    nft add rule inet filter postrouting oifname "eth0" inet saddr 192.168.100.0/24 counter masquerade 
    nft add rule inet filter postrouting oifname "eth0" inet saddr 192.168.200.0/24 counter masquerade 
~~~