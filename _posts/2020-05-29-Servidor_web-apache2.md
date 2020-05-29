---
title: Introducción a los servidores Web II. Apache2
author: Paloma R. García Campón
date: 2020-05-29 00:00:00 +0800
categories: [Servicios de red e internet]
tags: [HTTP, servidor web, Apache2]
---

![apache2](/assets/img/sample/apache.jpeg)

El servidor web sabe la página web que te tiene que dar por la dirección que se pide. En la cabecera aparece esta dirección, y se llama host.

Por defecto, cuando instalamos Apache2, tiene un sitio por defecto que se llama **default**. con un **Document Root** configurado, que será **/var/www/html**. En principio solo tiene un nombre, luego el **Name Server** no existe, en principio, cuando tenga más de uno se configura el nombre del servidor para identificar cada sitio. Los distintos ficheros de configuración de los distintos sitios virtuales se encuentran en **/etc/apache2/sites-availables**. El primer fichero que tiene Apache es **000-default.conf** que es el fichero de configuración del sitio por defecto y se encuentra en sites-availables. 

> En realidad, 000-default.conf no es el único fichero que viene por defecto. También encontramos **default-ssl.conf** que es el fichero de configuración de https.

Necesitamos que Apache le de una configuración al Document Root para funcione. Configuración de 000-default.conf:
~~~
<VirtualHost *:80>
	# The ServerName directive sets the request scheme, hostname and port that
	# the server uses to identify itself. This is used when creating
	# redirection URLs. In the context of virtual hosts, the ServerName
	# specifies what hostname must appear in the request's Host: header to
	# match this virtual host. For the default virtual host (this file) this
	# value is not decisive as it is used as a last resort host regardless.
	# However, you must set it for any further virtual host explicitly.
	#ServerName www.example.com

	ServerAdmin webmaster@localhost
	DocumentRoot /var/www/html
...
~~~

En **apache2.conf**, que es el fichero de configuración general, encontramos las direcciones de los directorios y subdirectorios. Cuando un directorio está configurado, sus directorios están configurados automáticamente. 

~~~
...
<Directory />
	Options FollowSymLinks
	AllowOverride None
	Require all denied 
# Todos loos directorios de la raíz están configurados, pero en este punto se deniega el acceso a ellos, luego con esto solo no funcionaría Apache. 
</Directory>

<Directory /usr/share>
	AllowOverride None
	Require all granted
# A diferencia del anterior, en este caso aparece Require all granted que significa que sí permite el acceso a este direcotrio. 
</Directory>

<Directory /var/www/>
	Options Indexes FollowSymLinks
	AllowOverride None
	Require all granted
</Directory>
#<Directory /srv/>
#	Options Indexes FollowSymLinks
#	AllowOverride None
#	Require all granted
#</Directory>

...
~~~

De esta forma, en vez de tener una configuración por cada sitio, se configura todo en el mismo fichero. Y, además, se puede alterar las difrecciones las direcciones y, que por ejemplo, configurar un sitio web en /home/usuario/pagina.

Vamos a configurar nuestro servidor web para que nos sirva **www.pag1.com** y  **www.pag2.com**:
- Los ficheros de pag1 los vamos a guardar en /var/www/pag1 y el server name será www.pag1.com. Esta configuración se va a guardar en: /etc/apache2/sites-availables/web-pa1.conf (en principio, lo más fácil es copiar el fichero del sitio por defecto y cambiarle los parámetros).

- El sitio pag2 se encuentra en /var/www/pag2, el name server www.pag2.com y el fichero de configuración web-pag2.conf.

Con esta configuración realizada sigue sin estar disponibles los sitios pag1 y pag2 porque también hay que configurar el direcotrio **/etc/apache2/sites-enabled**. Los ficheros web-pag1.conf y web-pag2.conf deben estar en el direcotrio anterior, bien copiando los ficheros, pero lo más habitual es crear un enlace simbólico (RECOMENDADO).

Esto podemos comprobarlo en el fichero de configuración general, al final del documento:

~~~
# Include the virtual host configurations:
IncludeOptional sites-enabled/*.conf
~~~

Para crear los enlaces simbólicos se puede usar el comando a2ensite y para quitarlo a2dissite (también podemos realizarlo con el comando ln -s).

~~~
$ a2ensite web-pag1.conf
~~~

> Importante: el propietario de los document root debe ser www-data (el usuario apache). 

~~~
chown -R www-data:www-data /var/www/pag1
~~~









