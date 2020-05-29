---
title: Primeros pasos en Vagrant
author: Paloma R. García Campón
date: 2020-05-29 00:00:00 +0800
categories: [Servicios de red e internet]
tags: [entornos de desarollo, virtualización, Vagrant]
---

El cómic web "Hark! A Vagrant" de la artista canadiense Kate Beaton trata con humor eventos y personajes históricos. Pero de ésto no trata este post. A continuación, lo explico. 

![vagrant](/assets/img/sample/Hark_A_Vagrant.png)

## Instalación de Vagrant (desarrollo de escenarios de forma fácil)

~~~
vagrant box add debian/buster64
~~~

### Por qué usar Vagrant:
Por que es fácil, para automatizarlo. 

### Qué se puede hacer:
- Crear máquinas virtuales y contenedores.
- Configurar redes.
- Configurar la máquina
- Aprovisionamiento de la máquina

### Imágenes de los sistemas operativos:
Las imágenes en Vagrant se llaman BOX y lo que hace la aplicación es copiarlas. Existen repositorios de imágenes y es recomendable usar las imágenes oficiales.

**Descargar imagen desde el repositorio oficial:**
~~~
vagrant box add debian/buster64
~~~

**Listar box:**
~~~
vagrant box list
~~~

**Ubicación de los box:**
~~~
$ ls .vagrant.d/boxes/debian-VAGRANTSLASH-buster64/10.0.0/virtualbox/
box.ovf  buster.vmdk  metadata.json  Vagrantfile
~~~


## Creación de una máquina virtual:
1. Crear un directorio y un fichero Vagrantfile:
~~~
vagrant init
~~~

2. Modificar el fichero Vagrantfile:
~~~
Vagrant.configure("2") do |config|
	config.vm.box = "debian/buster64"
	config.vm.hostname="escenario1"
end
~~~

3. Levantar la máquina:
~~~
vagrant up
~~~

4. Acceder a la instancia:
~~~
vagrant ssh
~~~


## Características de las máquinas:
- Va a tener dos usuarios: root y vagrant.
- Tiene una red NAT por defecto eth0 compartida con virtualbox.
- **vagrant ssh** es lo mismo que hacer **ssh -i privad -p2222 vagrant@localhost**. Esto funciona por una regla de NAT que proporciona virtualbox. En la configuración de las redes de Virtualbox se pueden cambiar estas reglas. Por ejemplo, si le indicamos que al acceder a la "ip pública" accedamos al servicio web por TCP, es decir, por el puerto 8080 se llega al 80.
- Es recomendable instalar **rsync** (se instala automáticamente) para sincronizar con el directorio de la máquina Vagrantfile/ con el correspondiente directorio de la máquina anfitrión. 


## Suspender, apagar o destruir:
~~~
vagrant suspend
vagrant halt
vagrant destroy
~~~


## Práctica con Vagrant

**Escenario con dos nodos, una red interna entre ellos y una puente al exterior en uno de los nodos**

Cambiar puerto 80 del nodo1 para que aparezca apache2 del nodo2.

~~~
Vagrant.configure("2") do |config|

  config.vm.define :nodo1 do |nodo1|
    nodo1.vm.box = "debian/buster64"
    nodo1.vm.hostname = "nodo1"
    nodo1.vm.network :public_network,:bridge=>"enp2s0"
    nodo1.vm.network :private_network, ip: "10.1.1.101"
~~~
### Configuración para que el nodo 1 haga funciones de router
~~~
nodo1.vm.provision "shell",run:"always" ,inline: <<-SHELL
~~~	
### Cambiar las rutas por defecto
~~~
sudo ip r del default
sudo ip r add default via 172.22.0.1
~~~
### Cambiar las salidas de la red interna
~~~
sudo iptables -t nat -A POSTROUTING -o eth1 -s 10.1.1.101/24 -j MASQUERADE
~~~
### Cambiar el fichero sysctl
~~~
sudo sysctl -w net.ipv4.ip_forward=1
~~~
### Redirigir puerto 80 al apache del nodo 2
~~~
sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -i eth1 -j DNAT --to-destination 10.1.1.102
~~~
### Reiniciar iptable
~~~
sudo systemctl -p restart
SHELL

end
~~~
~~~
config.vm.define :nodo2 do |nodo2|
  nodo2.vm.box = "debian/buster64"
  nodo2.vm.hostname = "nodo2"
  nodo2.vm.network :private_network, ip: "10.1.1.102"
~~~ 
### Configuración para que el nodo 2 utilice el nodo 1 como router
~~~ 
nodo2.vm.provision "shell",run:"always" ,inline: <<-SHELL
sudo ip r del default
sudo ip r add default via 10.1.1.101
sudo apt-get -y install apache2
SHELL
~~~


