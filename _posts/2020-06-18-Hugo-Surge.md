---
title: Creando una página con Surge y Hugo
author: Paloma R. García Campón
date: 2020-06-18 00:00:00 +0800
categories: [Implantación de aplicaciones web]
tags: [Surge, Hugo]
---

![surge](/assets/img/sample/surge-hugo/surge.png)

![hugo](/assets/img/sample/surge-hugo/hugo.png)

En esta entrada se va a explicar como utilizar Surge y Hugo. 

Otrocambio
**1. Selecciona una combinación entre generador de páginas estáticas y servicio donde desplegar la página web. Escribe tu propuesta en redmine, cada propuesta debe ser original.**
~~~
Surge - Hugo
~~~

**2. Comenta la instalación del generador de página estática. Recuerda que el generador tienes que instalarlo en tu entorno de desarrollo. Indica el lenguaje en el que está desarrollado y el sistema de plantillas que utiliza.**

HUGO: Lenguaje: Go. Sistema de plantillas: Go.

Instalación de Hugo:
~~~
sudo apt-get install hugo
~~~

~~~
$ hugo new site quickstart
Congratulations! Your new Hugo site is created in /home/paloma/DISCO2/CICLO II/IMPLANTACIÓN DE APLICACIONES WEB/1.Generador_pag_estatica/quickstart.

Just a few more steps and you're ready to go:

1. Download a theme into the same-named folder.
   Choose a theme from https://themes.gohugo.io/, or
   create your own with the "hugo new theme <THEMENAME>" command.
2. Perhaps you want to add some content. You can add single files
   with "hugo new <SECTIONNAME>/<FILENAME>.<FORMAT>".
3. Start the built-in live server via "hugo server".

Visit https://gohugo.io/ for quickstart guide and full documentation.
~~~

~~~
$ cd quickstart
$ git init
Inicializado repositorio Git vacío en /home/paloma/DISCO2/CICLO II/IMPLANTACIÓN DE APLICACIONES WEB/1.Generador_pag_estatica/quickstart/.git/
paloma@coatlicue:~/DISCO2/CICLO II/IMPLANTACIÓN DE APLICACIONES WEB/1.Gene
rador_pag_estatica/quickstart$ git submodule add https://github.com/budparr/gohugo-theme-ananke.git themes/ananke
Clonando en '/home/paloma/DISCO2/CICLO II/IMPLANTACIÓN DE APLICACIONES WEB/1.Generador_pag_estatica/quickstart/themes/ananke'...
remote: Enumerating objects: 5, done.
remote: Counting objects: 100% (5/5), done.
remote: Compressing objects: 100% (5/5), done.
remote: Total 1416 (delta 0), reused 1 (delta 0), pack-reused 1411
Recibiendo objetos: 100% (1416/1416), 4.15 MiB | 3.55 MiB/s, listo.
Resolviendo deltas: 100% (764/764), listo.
$ echo 'theme = "ananke"' >> config.toml
~~~




**3. Configura el generador para cambiar el nombre de tu página, el tema o estilo de la página,… Indica cualquier otro cambio de configuración que hayas realizado.**

Modifico el fichero quickstart/content/posts/my-first-post.md para añadir nuevos post. 

Para cambiar el nombre de la página ha que modificar el fichero config.toml.
~~~
$ hugo

                   | EN  
+------------------+----+
  Pages            | 38  
  Paginator pages  |  6  
  Non-page files   |  0  
  Static files     |  3  
  Processed images |  0  
  Aliases          |  6  
  Sitemaps         |  1  
  Cleaned          |  0  

Total in 82 ms
~~~

Para modificar el tema de la página estática nos hemos descargado un repositorio de gitHub con un tema cuyo formato es agradable. Reemplazamos los ficheros que contiene quickstart por los del repositorio y modificamos el fichero config.toml con nuestros parámetros:
~~~
baseURL = "/"
title = "Un pez llamado Wanda"
author = "Paloma R. García Campón"
googleAnalytics = ""
defaultContentLanguage = "en"
language = "en-US"
paginate = 3

theme = "hugo-terrassa-theme"
...
~~~

Los fichero .md que queremos añadir a la página los guardamos en el fichero /quistart/content. En nuestro caso los añadimos en el directorio /post.


**4. Genera un sitio web estático con al menos 3 páginas. Deben estar escritas en Markdown y deben tener los siguientes elementos HTML: títulos, listas, párrafos, enlaces e imágenes. El código que estas desarrollando, configuración del generado, páginas en markdown,… debe estar en un repositorio Git (no es necesario que el código generado se guarde en el repositorio, evitalo usando el fichero .gitignore).**

[El_pez_llamado_Wanda(GitHub)]( https://github.com/PalomaR88/El_pez_llamado_Wanda "El pez llamado Wanda(GitHub)")



[El_pez_llamado_Wanda]( http://el_pez_llamado_wanda.surge.sh/ "El pez llamado Wanda")

**5. Explica el proceso de despliegue utilizado por el servicio de hosting que vas a utilizar.**

En primer lugar hay que instalar surge:
~~~
$ sudo npm install --global surge
npm WARN npm npm does not support Node.js v10.15.2
npm WARN npm You should probably upgrade to a newer version of node as we
npm WARN npm can't make any promises that npm will work with this version.
npm WARN npm Supported releases of Node.js are the latest release of 4, 6, 7, 8, 9.
npm WARN npm You can find the latest version at https://nodejs.org/
/usr/local/bin/surge -> /usr/local/lib/node_modules/surge/lib/cli.js
+ surge@0.21.3
added 137 packages from 113 contributors in 22.737s
~~~


**6. Piensa algún método (script, scp, rsync, git,…) que te permita automatizar la generación de la página (integración continua) y el despliegue automático de la página en el entorno de producción, después de realizar un cambio de la página en el entorno de desarrollo. Muestra al profesor un ejemplo de como al modificar la página se realiza la puesta en producción de forma automática.**
~~~
#! bin/sh

commit=$(git commit -am 'nuevo' | egrep 'content/')
touch ../nuevo.txt
echo $commit > ../nuevo.txt

git add $(cat ../nuevo.txt)
git commit -m 'añado'
git push

rm ../nuevo.txt

hugo
surge public/ el_pez_llamado_wanda.surge.sh
~~~


