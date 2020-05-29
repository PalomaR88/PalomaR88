---
title: Introducción a los servidores Web I
author: Paloma R. García Campón
date: 2020-05-29 00:00:00 +0800
categories: [Servicios de red e internet]
tags: [HTTP, servidor web]
---



El servidor web no guarda ningún recuerdo de quien ha hecho una petición, no tiene memoria.

### Funcionamiento de protocolo
- Línea inicial del mensaje: Petición (GET), respuesta (HTTP)
- Cabecera del mensaje
- Cuerpo del mensaje

Métodos de envío de datos utilizados:
- GET: solicita un documento al servidor.
- HEAD: Similar a GET, pero solo pide las cabeceras HTTP. Pide información de un recurso, pero no el recurso en sí, sino la caberas.
- POST: Manda datos al servidor para su procesado, como GET pero enviando información (formulario).
- PUT: Permite subir un fichero al servidor. Almacena el documento enviado en el cuerpo del mensaje.
- DELETE: Elimina el cocumento referenciado en la URL.

### Código de estados
1xx: Mensaje informativo
2xx: Éxito
- **200 OK** Mensaje satisfactorio.
- **201 Created** Resultado de la creación de uno o más recursos nuevos.
- **202 Accepted** La petición ha sido recibida pero no se ha actuado al respecto.
- **204 No Content** La solicitud ha tenido éxito, pero el cliente no necesita alejarse de su página actual.
3xx Redirección
- **300 Múltiple Choice** Tiene más de una respuesta posible.
- **301 Moved Permanently** El recurso solicitado ha sido movido permanentemente.
- **302 Found** El recurso solicitado ha sido movido temporalmente.
- **304 Not Modified** No hay necesidad de retransmitir los recursos solicitados.
4xx: Error del cliente
- **400 Bad Request** El servidor no puede o no procesará la petición debido a algo que es percibido como un error del cliente.
- **401 Unauthorized** La solicitud no se ha aplicado porque no tenemos autorización.
- **403 Forbidden** No tienes permiso para acceder al fichero o directorio.
- **404 Not Found** No se encuentra respuesta o no está disponible.
5xx: Error en el servidor
- **500 Internal Server Error** Ha habido un error en el servidor pero el servidor no puede ser más específico.
- **501 Not Implemented** El servidor no admite la funcionalidad requerida para complir la solicitud. 
- **502 Bad Gateway** Problema de comunicación entre dos servidores. 
- **503 Service Unavailable** El servidor advierte de que la página no está disponible temporalmente.

![404](/assets/img/sample/404-not-found.jpg)


### Cabeceras
- Host
- User-agent: tiene mucha información como el navegador que está accediendo a nuestra página, etc.
- Server
- Cache-control: información sobre la cache. 
- Content-type
- Content-encoding
- Expires: pide la fecha de expiración al servidor. Esto lo utiliza el proxy que tiene páginas cacheadas y las tiene guardadas para ver si han cambiado. 
- Location: muy importante para las redirecciones. Pide un contenido, pero no está, luego se envía la nueva dirección en el location. 
- Set-cookie.

COOKIES: las cookies son información que el navegagor guarda en memoria o en el disco duro dentro de ficheros de texto, a solicitud del servidor. Lo guarda el cliente, el servidor busca si cada cliente tiene esta información guardada (por ejemplo, el carrito de la compra).

SESION: es lo mismo que las cookies pero la información la guarda en el servidor. HTTP es un protocolo sin manejo de estado. Las sesiones nos permiten definir estados, para ello el servidor almacenará la información necesaria para llevar el seguimiento de la sesión. 

AUTENTIFICACIÓN: características de los servidores web que cada vez se usa menos.Es el servidor web el que pide la autentificación, no la aplicación web. Esto es una de las cosas que antes era muy común de los servodores web y ya no.

CONEXIONES PERSISTENTES: permite que varias peticiones y respuestas sean tranferidas usando la misma conexión TCP. 





