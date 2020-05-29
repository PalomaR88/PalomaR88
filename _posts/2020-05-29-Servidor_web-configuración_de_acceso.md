---
title: Introducción a los servidores Web III. Configuración de acceso
author: Paloma R. García Campón
date: 2020-05-29 00:00:00 +0800
categories: [Servicios de red e internet]
tags: [HTTP, servidor web, Apache2]
---

## Configuración de los puertos
Fichero: /etc/apache2/ports.conf
Directiva: Listen

> Ejemplo: Virtual Host basado en IP

~~~
<VirtualHost 172.2.30.40:80>
	ServerAdmin webmaster@example.com
	DocumentRoot "/www/vhost/www2"
	ServerName www2.example.org
	ErrorLog "/www/logs/www2/error_log"
	CustomLog "/www/logs/www2/access_log" combined
</VirtualHost>
~~~

> Ejemplo: Servir el mismo contenido en varias IP

~~~
<VirtualHost 172.2.30.40:80 192.168.1.1>
	DocumentRoot "/www/server1"
	ServerName server.example.com
	ServerAlias server
</VirtualHost>
~~~

> Ejemplo: sirviendo distintos sitios en distintos puertos

Hay que configurar /etc/apache2/ports.conf

~~~
<VirtualHost *:80>
	DocumentRoot "/www/domain-80"
	ServerName www.example.com
</VirtualHost>
~~~
	
~~~
<VirtualHost *:8080>
	DocumentRoot "/www/domain-8080"
	ServerName www.example.com
</VirtualHost>
~~~
	

Nosotros vamos a utilizaar siempre los basados por nombre.


## Opciones del direcorio
Directiva: options
- All
- FollowSymLinks. Los ficheros que están en otro direcotrio que no esté configurado no se pueden ver. Pero, si en uno de los ficheros tengo un link simbólico que me lleve a ese lugar, con esta opción te permite ver ese fichero. 
- Indexes. Si el cliente solicita un directorio en el que no exista ninguno de los ficheros especificados en **DirectoryIndex**, el servidor ofrecerá un listado de los archivos del directorio.
- MultiViews (este ya no se utiliza) Hay determinadas opciones de la cabecera que según lo que aparezca muestra un fichero u otro. Ejemplo: Muestra la página en inglés si tu navegador está en inglés y en español si está en español. 
- SymLinksIfOwnerMatch. Esto es como FollowSymLinks pero más restrictivo, puesto que para que se vean los ficheros ambos ficheros (el que está dentro y el que está fuera, deben ser del mismo propietario.
- ExecCGI (este ya no se utiliza) Para el uso de programas CGI, pero esto está en desuso. 

En /etc/apache2/apache2.conf

~~~
<Directory /var/www/>
	Options Indexes FollowSymLinks
	AllowOverride None
	Require all granted
</Directory>
~~~

O en /etc/apache2/sites-available/000-default.conf, donde especifica que el fichero va a tener las mismas opciones que el padre, más determinadas opciones o - determinadas opciones.

~~~
<Directory /var/www/default>
	Options +Multiviews
	AllowOverride None
	Require all granted
</Directory>
~~~

~~~
<Directory /var/www/default>
	Options -Indexes
	AllowOverride None
	Require all granted
</Directory>
~~~

Si tenemos una página web en un hosting y, en él, un servidor web Apache y queremos cambiar las opciones, pero no eres el administrador, puedes configurarlo. Esto es una opción única de Apache. Se configura en **.htaccess**, que está ubicado en el documentRoot donde puedes poner las mismas opciones que en los casos anteriores. Pero tiene que estar permitida con **AllowOverride** All/None.

~~~
<Directory /var/www/default>
	Options -Indexes
	AllowOverride All
	Require all granted
</Directory>
~~~

> Lo lógico es que en el fichero de configuración general /etc/apache2/apache2.conf no tenga esta opción activada y que se active en los direcotrios específicos donde se quieran activar. 


## Mapeo de URL
	**URL**				**Fichero**
http://172.2.22.1	 	  /var/www/html/index.html
http://172.2.22.1/entrada	  /var/www/html/entrada/index.html
http://172.2.22.1/entrada/ima.jpg /var/www/html/entrada/index.html/img.jpg

El ejemplo anterior es lo normal pero en algunos casos se utilizan los alias. Los alias nos permite que el servidor sirva ficheros desde cualquier ubicación del sistema de archivo aunque esté fuera del direcotrio indicado en el DocumentRoot. Cuando creamos un alias, a continuación debemos crear un directory para darle los permisos oportunos.

~~~
Alias "/image" "/ftp/pub/image"
<Directory "/ftp/pub/image">
	Require all granted
</Directory>
~~~

**AliasMatch** es para usar expresiones irregulares.

~~~
AliasMatch "^/image/(.*)$" "/ftp/pub/image/$1"
~~~


## Redirecciones
La directiva redirect es usada para pedir a cliente que haga otra petición a una URL diferente. Normalmente la usamos cuando el recurso al que queremos acceder ha cambiado de localización.

En las redirecciones el usuario pide una dirección y el servidor le responde que eso no está ahí, sino en otra dirección. 

- Temporales (302)
~~~
Redirect "/service" "http://www.pagina.com/service"
Redirect "/one" "/two"
~~~

-Permanentes (301)
~~~
Redirect permanent "/one" "http://www.pagina2.com/two"
Redirect 301 "/otro" /otra"
~~~


## Páginas de errores personalizadas
Podemos cambiar las páginas de error:
1. Desplegar un texto diferente, en lugar de los mensajes que aparecen por defecto.
2. Redireccionar la petición a una URL local.
3. Redireccionar la petición a una IRL externa.

- Con la directiva ErrorDocument
- Con el fichero: /etcapache2/conf-available/localizad-error-pages.conf

**Cambiando el idio de las páginas de error personalizadas**
En el directorio **/usr/share/apache2/error** nos encontramos fichero tipo mapa donde se encuentra definidas las páginas de error personalizadas para distintos idiomas.


## Control de acceso
¿Quién tiene permiso para entrar en los recursos?

- Require
- Require not

~~~
Require all granted - todos tienen permisos
Require all denied - ninguno
Require user userid [userid] - determinados usuarios tienen permiso
Require group [group-name] - determinados grupos
Require valid-user - determinados usuarios validados
Require ip 10 x.x.x.x - determinados usuarios que entran por una red
Require host dominio
Require local
~~~

Esto es algo que ha cambio de Apache 2.2 a 2.4. Hay que tener cuidado con lo que se compia y pega de internet. 

Existen bloques de instrucciones para el control de acceso.
- RequireAll
- RequireAny
- RequireNone


## Autentificación básica
Mi virtual host o parte de el necesita de un usuario y una contraseña para acceder a él. Para ello se necesita un directory dodne se va a configurar la autentificación.

~~~
<Directory "/var/www/pagina1/privado">
	AuthUserFile "Palabra de paso"
	AuthType Basic
	Require valid-user
</Directory>
~~~

Pasos:
1. Crear el fichero de usuario (la opción -c solo se utiliza utiliza la primera vez):
~~~
htpasswd [-c] /etc/apache2/claves/passwd.txt usuario1
~~~

2. Crear el fichero de grupos.
~~~
NombreGrupo: usuario1 usuario2 usuario3
~~~

Esta opción casi no se utiliza, porque la contraseña se envía en claro. 


## Autentificación digest
Esta opción es más segura que la anterior. Se guarda el nombre del usuario, un nom.bre de dominio y la contraseña. 
~~~
<Directory "/var/www/pagina1/provado">
	AuthUserFile "/etc/apache/claves/digest.txt
	AuthName "dominio"
	AuthType Digest
	Require valid-user
</Directory>
~~~

Para crear el fichero de usuarios
~~~
htdigest [-c] /etc/apache2/claves/digest.txt dominio usuario1
~~~


## Políticas de autentificación y control de acceso

Apache 2.2
Como se debía comportar el servidor cuando tenemos varias instrucciones de control de acceso (allow, deny, require). De esta manera:
- Satisfy All: se tenían que cumplir todas las condiciones para obtener el acceso. 
- Satisfy Any: Bastaba con que se cumpliera una de las conficiones. 

Apache 2.4
Las políticas de acceso la podemos indicar usando las directivas:
- RequireAll: todas las condiciones dentro del bloque se deben cumplir para obtener el acceso.
- RequireAny: al menos una de las condicionies en el bloque se debe cumplir.
- RequireNone: ninguna de las condiciones se deben cumplir para perminir el acceso. 


## .htaccess
Si permites aquí y en otro sitio se contradice, este archivo reescribe todo. 
Directiva: **AllowOverride**
- All
- None
- AuthConfig
- FileInfo
- Indexes
- Limit








