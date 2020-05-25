---
title: "Métricas, logs y monitorización con Netdata"
categories:
- Sistemas
excerpt: |
  Instalamos el servicio de Netdata y lo configuramos para recopilar información de nuestros servidores de métricas, logs y monitorización.
aside: true
---
**Instalamos el servicio de Netdata y lo configuramos para recopilar información de nuestros servidores de métricas, logs y monitorización.**

![iSCSI](https://github.com/MoralG/Metricas_logs_y_monitorizacion_Netdata/raw/master/image/Netdataicono.png)


Netdata es una herramienta para visualizar y monitorear métricas en tiempo real. Escrita en C, con un rendimiento muy óptimo y necesita muy pocos recurso durante su ejecucción.

Consiste en un demonio que cuando se ejecuta se encarga de obtener información en tiempo real y presentarla en la web. La recoleción de información utiliza plugins internos y/o externos.


## Instalación Netdata

Instalamos las dependencias de **netdata**.
~~~
sudo apt-get install zlib1g-dev uuid-dev libuv1-dev liblz4-dev libjudy-dev libssl-dev libmnl-dev gcc make git autoconf autoconf-archive autogen automake pkg-config curl python
~~~

Vamos a clonar de GitHub el repositorio donde se aloja la herramienta **netdata**, para despues instalarla.

~~~
git clone https://github.com/netdata/netdata.git
~~~

Nos dirigimos a la ruta del directorio que se ha clonado e instalamos **netdata**.
~~~
cd netdata/
sudo ./netdata-installer.sh

  ^
  |.-.   .-.   .-.   .-.   .  netdata                                        
  |   '-'   '-'   '-'   '-'   real-time performance monitoring, done right!  
  +----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+--->


  You are about to build and install netdata to your system.

  It will be installed at these locations:

   - the daemon     at /usr/sbin/netdata
   - config files   in /etc/netdata
   - web files      in /usr/share/netdata
   - plugins        in /usr/libexec/netdata
   - cache files    in /var/cache/netdata
   - db files       in /var/lib/netdata
   - log files      in /var/log/netdata
   - pid file       at /var/run/netdata.pid
   - logrotate file at /etc/logrotate.d/netdata

  This installer allows you to change the installation path.
  Press Control-C and run the same command with --help for help.


  NOTE:
  Anonymous usage stats will be collected and sent to Google Analytics.
  To opt-out, pass --disable-telemetry option to the installer.

Press ENTER to build and install netdata to your system > 
~~~

Pulsamos ENTER y comenzara la compilación y posteriormente la instalación de netdata en las rutas que se muestran arriba.

Completada la instalación nos aparecerá el siguente mensaje:
~~~
 --- We are done! --- 

  ^
  |.-.   .-.   .-.   .-.   .-.   .  netdata                          .-.   .-
  |   '-'   '-'   '-'   '-'   '-'   is installed and running now!  -'   '-'  
  +----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+-----+--->

  enjoy real-time performance and health monitoring...
~~~

Ya tenemos **netdata** instalado y el demonio esta activado y funcionando.
~~~
sudo systemctl list-unit-files | grep netdata
  netdata.service                        enabled
~~~

Podemos ver el estado, y el último log nos índica que ha empezado la monitorización en tiempo real.
~~~
sudo systemctl status netdata.service 
  ● netdata.service - Real time performance monitoring
     Loaded: loaded (/lib/systemd/system/netdata.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2020-01-24 10:40:10 UTC; 1 weeks 6 days ago
    Process: 8846 ExecStartPre=/bin/mkdir -p /var/cache/netdata (code=exited, status=0/SUCCESS)
    Process: 8847 ExecStartPre=/bin/chown -R netdata:netdata /var/cache/netdata (code=exited, status=0/ SUCC
    Process: 8848 ExecStartPre=/bin/mkdir -p /var/run/netdata (code=exited, status=0/SUCCESS)
    Process: 8849 ExecStartPre=/bin/chown -R netdata:netdata /var/run/netdata (code=exited, status=0/ SUCCES
   Main PID: 8850 (netdata)
      Tasks: 36 (limit: 1168)
     Memory: 186.1M
     CGroup: /system.slice/netdata.service
             ├─  312 bash /usr/libexec/netdata/plugins.d/tc-qos-helper.sh 1
             ├─ 8850 /usr/sbin/netdata -P /var/run/netdata/netdata.pid -D -W set global process   scheduling 
             ├─ 8901 /usr/bin/python /usr/libexec/netdata/plugins.d/python.d.plugin 1
             ├─ 8907 /usr/libexec/netdata/plugins.d/go.d.plugin 1
             └─32540 /usr/libexec/netdata/plugins.d/apps.plugin 1

  Jan 24 10:40:10 serranito systemd[1]: Starting Real time performance monitoring...
  Jan 24 10:40:10 serranito systemd[1]: Started Real time performance monitoring.
~~~

## Configuración del Servidor (Serranito) 
Ya tenemos funcionando **netdata** en el Servidor Serranito, pero no podemos visualizar la monitorización ni las métricas, ya que no tenemos un servidor web para mostrar la herramienta.

Vamos a instalar apache:
~~~
sudo apt-get install apache2-bin
~~~

Activamos los módulos `proxy` y `proxy_http`, necesarios ya que el puerto de **netdata** es el 19999.
~~~
sudo a2enmod proxy
sudo a2enmod proxy_http
~~~

Activamos también el módulo `rewrite`.
~~~
sudo a2enmod rewrite
~~~

Creamos el virtualhost de apache, para la aplicación web de **netdata**.
~~~
nano /etc/apache2/sites-available/netdata.conf
~~~

Fichero `netdata.conf`.
~~~
<VirtualHost *:80>
    RewriteEngine On
    ProxyRequests Off
    ProxyPreserveHost On

    ServerName netdata.amorales.gonzalonazareno.org

    <Proxy *>
        Require all granted
    </Proxy>

    ProxyPass "/" "http://localhost:19999/" connectiontimeout=5 timeout=30 keepalive=on
    ProxyPassReverse "/" "http://localhost:19999/"

    ErrorLog ${APACHE_LOG_DIR}/netdata-error.log
    CustomLog ${APACHE_LOG_DIR}/netdata-access.log combined
</VirtualHost>
~~~

Activamos el virtualhost creando el enlace símbolico.
~~~
a2ensite netdata
~~~

Reiniciamos los servicios de netdata y apache2.
~~~
systemctl restart netdata.service
systemctl restart apache2.service
~~~

Añadimos DNS el registro al DNS del Servidor Croqueta.
~~~
netdata   IN   CNAME   serranito
~~~

Comprobamos que funciona la página.
![netdata](https://github.com/MoralG/Metricas_logs_y_monitorizacion_Netdata/blob/master/image/Netdata.png?raw=true)

## Configuración de los Clientes (Croqueta, Tortilla, Salmorejo)

Ahora vamos a recoger los datos, para la monitorización y las métricas, de los clientes. Para realizar esto, tenemos que configurar a los clientes como esclavos, que se realiza con un fichero de configuración.

Tenemos que generar una UUID para que reconozca el servidor al cliente a través de la `apy key`. Con el paquete `uuid-runtime` podemos generar un uuid.
~~~
sudo apt install uuid-runtime
~~~

Generamos la uuid con el comando `uuidgen`.
~~~
uuidgen
  4c97ef68-ca16-41e9-b5df-d7f017d9da8e
~~~

Creamos el fichero `/etc/netdata/stream.conf` y añadimos la configuración del maestro para que pueda escuchar a los clientes.
~~~
# -----------------------------------------------------------------------------
# 1. EN LA MÁQUINA SLAVE NETDATA - SOLO PARA ENVIAR MÉTRICAS

[stream]
    enabled = no
    destination =
    api key = 
    timeout seconds = 60
    default port = 19999
    buffer size bytes = 1048576
    reconnect delay seconds = 5
    initial clock resync iterations = 60

# -----------------------------------------------------------------------------
# 2-1. EN LA MÁQUINA MASTER NETDATA - SOLO PARA RECIBIR MÉTRICAS CROQUETA

[4c97ef68-ca16-41e9-b5df-d7f017d9da8e]
    enabled = yes
    allow from = *
    default history = 3600
    default memory mode = ram
    health enabled by default = auto
    default postpone alarms on connect seconds = 60

# -----------------------------------------------------------------------------
# 2-2. EN LA MÁQUINA MASTER NETDATA - SOLO PARA RECIBIR MÉTRICAS TORTILLA

[415e58a5-b1de-41e4-8eac-d749bc572df5]
    enabled = yes
    allow from = *
    default history = 3600
    default memory mode = ram
    health enabled by default = auto
    default postpone alarms on connect seconds = 60

# -----------------------------------------------------------------------------
# 2-3. EN LA MÁQUINA MASTER NETDATA - SOLO PARA RECIBIR MÉTRICAS SALMOREJO

[7a4c6dd7-d71a-46ba-b1d5-150333707ff7]
    enabled = yes
    allow from = *
    default history = 3600
    default memory mode = ram
    health enabled by default = auto
    default postpone alarms on connect seconds = 60

# -----------------------------------------------------------------------------
# 3. PER SENDING HOST SETTINGS, ON MASTER NETDATA THIS IS OPTIONAL - YOU DON'T NEED IT

[MACHINE_GUID]
    enabled = no
    allow from = *
    history = 3600
    memory mode = save
    health enabled = yes
    postpone alarms on connect seconds = 60
~~~

Una vez creado el fichero `stream.conf` podemos reiniciar el servicio `netdata.service` del servidor para que empiece a escuchar.

Como podemos ver al mirar el log del maestro, esta escuchando pero nadie responde y cierra la conexión.

~~~
tail -f /var/log/netdata/error.log
  2020-02-07 21:47:42: netdata INFO  : WEB_SERVER[static1] : POLLFD: LISTENER: client slot 5 (fd 82) from localhost port 43156 is idle for more than 60 seconds - closing it. 
~~~

Para que el maestro escuche a los clientes, estos los vamos a configurar como esclavos. Para esto vamos a instalar netdata como hicimos en el servidor. [IR ARRIBA](https://github.com/MoralG/Metricas_logs_y_monitorizacion_Netdata/blob/master/Metricas_logs_monitorizacion.md#instalaci%C3%B3n-netdata)

Creamos el fichero `/etc/netdata/stream.conf` y añadimos la configuración del esclavo en las máquinas Croqueta, Salmorejo y Tortilla.
~~~ 
# -----------------------------------------------------------------------------
# 1. ON SLAVE NETDATA - THE ONE THAT WILL BE SENDING METRICS

[stream]
    enabled = yes
    destination = 10.0.0.17
    api key = 4c97ef68-ca16-41e9-b5df-d7f017d9da8e
    timeout seconds = 60
    default port = 19999
    buffer size bytes = 1048576
    reconnect delay seconds = 5
    initial clock resync iterations = 60

# -----------------------------------------------------------------------------
# 2. ON MASTER NETDATA - THE ONE THAT WILL BE RECEIVING METRICS

[API_KEY]
    enabled = no
    allow from = *
    default history = 3600
    default memory mode = ram
    health enabled by default = auto
    default postpone alarms on connect seconds = 60

# -----------------------------------------------------------------------------
# 3. PER SENDING HOST SETTINGS, ON MASTER NETDATA THIS IS OPTIONAL - YOU DON'T NEED IT

[MACHINE_GUID]
    enabled = no
    allow from = *
    history = 3600
    memory mode = save
    health enabled = yes
    postpone alarms on connect seconds = 60
~~~

Después reiniciamos el servicio `netdata.service` de los esclavos y al mirar el log nos saldrá los siguiente mensajes

~~~
systemctl restart netdata.service
~~~

**Servidor**
~~~
tail -f /var/log/netdata/error.log | egrep STREAM
  2020-02-07 21:54:23: netdata INFO  : STREAM_RECEIVER[croqueta,[10.0.0.6]:54704] : STREAM croqueta   [receive from [10.0.0.6]:54704]: initializing communication...
  2020-02-07 21:54:23: netdata INFO  : STREAM_RECEIVER[croqueta,[10.0.0.6]:54704] : STREAM croqueta   [receive from [10.0.0.6]:54704]: Netdata is using the newest stream protocol.
  2020-02-07 21:54:23: netdata INFO  : STREAM_RECEIVER[croqueta,[10.0.0.6]:54704] : Postponing health   checks for 60 seconds, on host 'croqueta', because it was just connected.
  2020-02-07 21:54:23: netdata INFO  : STREAM_RECEIVER[croqueta,[10.0.0.6]:54704] : STREAM croqueta   [receive from [10.0.0.6]:54704]: receiving metrics...
  2020-02-07 21:54:30: netdata INFO  : STREAM_RECEIVER[croqueta,[10.0.0.6]:54704] : RRDSET: chart name  'net.eth0' on host 'croqueta' already exists.
  2020-02-07 21:54:47: netdata INFO  : STREAM_RECEIVER[croqueta,[10.0.0.6]:54704] : RRDSET: chart name  'groups.threads' on host 'croqueta' already exists.
  2020-02-07 21:54:47: netdata INFO  : STREAM_RECEIVER[croqueta,[10.0.0.6]:54704] : RRDSET: chart name  'groups.processes' on host 'croqueta' already exists.
  2020-02-07 21:54:47: netdata INFO  : STREAM_RECEIVER[croqueta,[10.0.0.6]:54704] : RRDSET: chart name  'groups.uptime' on host 'croqueta' already exists.
  2020-02-07 21:54:47: netdata INFO  : STREAM_RECEIVER[croqueta,[10.0.0.6]:54704] : RRDSET: chart name  'groups.uptime_min' on host 'croqueta' already exists.
  2020-02-07 21:54:47: netdata INFO  : STREAM_RECEIVER[croqueta,[10.0.0.6]:54704] : RRDSET: chart name  'groups.uptime_avg' on host 'croqueta' already exists.
  2020-02-07 21:54:47: netdata INFO  : STREAM_RECEIVER[croqueta,[10.0.0.6]:54704] : RRDSET: chart name  'groups.uptime_max' on host 'croqueta' already exists.
  2020-02-07 21:54:47: netdata INFO  : STREAM_RECEIVER[croqueta,[10.0.0.6]:54704] : RRDSET: chart name  'groups.mem' on host 'croqueta' already exists.
  2020-02-07 21:54:47: netdata INFO  : STREAM_RECEIVER[croqueta,[10.0.0.6]:54704] : RRDSET: chart name  'groups.vmem' on host 'croqueta' already exists.
  2020-02-07 21:54:47: netdata INFO  : STREAM_RECEIVER[croqueta,[10.0.0.6]:54704] : RRDSET: chart name  'groups.swap' on host 'croqueta' already exists.
~~~

**Cliente**


~~~
systemctl restart netdata.service
  2020-02-07 21:55:13: netdata INFO  : PLUGINSD[apps] : RRDSET: chart name 'groups.swap' on host  'croqueta' already exists.
  2020-02-07 21:55:13: netdata INFO  : PLUGINSD[apps] : RRDSET: chart name 'groups.major_faults' on host  'croqueta' already exists.
  2020-02-07 21:55:13: netdata INFO  : PLUGINSD[apps] : RRDSET: chart name 'groups.minor_faults' on host  'croqueta' already exists.
  2020-02-07 21:55:13: netdata INFO  : PLUGINSD[apps] : RRDSET: chart name 'groups.preads' on host  'croqueta' already exists.
  2020-02-07 21:55:13: netdata INFO  : PLUGINSD[apps] : RRDSET: chart name 'groups.pwrites' on host   'croqueta' already exists.
  2020-02-07 21:55:13: netdata INFO  : PLUGINSD[apps] : RRDSET: chart name 'groups.lreads' on host  'croqueta' already exists.
  2020-02-07 21:55:13: netdata INFO  : PLUGINSD[apps] : RRDSET: chart name 'groups.lwrites' on host   'croqueta' already exists.
  2020-02-07 21:55:13: netdata INFO  : PLUGINSD[apps] : RRDSET: chart name 'groups.files' on host   'croqueta' already exists.
  2020-02-07 21:55:13: netdata INFO  : PLUGINSD[apps] : RRDSET: chart name 'groups.sockets' on host   'croqueta' already exists.
  2020-02-07 21:55:13: netdata INFO  : PLUGINSD[apps] : RRDSET: chart name 'groups.pipes' on host   'croqueta' already exists.
~~~

Comprobamos en la aplicación que nos aparece los clientes de Croqueta, Tortilla y Salmorejo.
![Netdata](https://github.com/MoralG/Metricas_logs_y_monitorizacion_Netdata/blob/master/image/Netdata1.png?raw=true)

Ya tenemos todos los datos de los clientes, agrupados en el servidor y nos lo muestra en nuestra servicio web, pero nos muestra una gran cantidad de métricas que son innecesarias para la funcionalidad que nosotros queremos.

Ahora vamos quitar toda esas métricas sobrantes que nosotros no necesitamos y dejar las más importantes para nosotros, para realizar esto vamos a modificar el fichero `/etc/netdata/netdata.conf`. En este fichero tenenmos todos los campos del Dashboard, los cuales podemos deshabilitar con la opción `enable = no`, como se muestra en el ejemplo del apartado de las variables de entropia:
~~~
[system.entropy]
        history = 5
        enabled = no
        # cache directory = /var/cache/netdata/system.entropy
        # chart type = line
        # type = system
        # family = entropy
        # units = entropy
        # context = system.entropy
        # priority = 1000
        # name = system.entropy
        # title = Available Entropy
        # dim entropy name = entropy
        # dim entropy algorithm = absolute
        # dim entropy multiplier = 1
        # dim entropy divisor = 1
~~~

Deshabilitado, ya no se mostrará en la web.

Además de dejar el Dashboard funcional para nosotros, podemos configurar las notificaciones. Estas notificaciones de las que estoy hablando, son las encargada de avisarte con una alerta si algun valor de los que esta midiendo sobrepasa el límite indicado.

Para modificar las notificaciones tenemos que utilizar el script `/etc/netdata/edit-config` e indicarle el fichero donde este la configuración de la notificación que queramos modificar. Por ejemplo en nuestro caso vamos a modificar la notificación de la CPU.

~~~
sudo ./edit-config health.d/cpu.conf
~~~

Vamos a modificar la alerta del WARNING para que salte cuando la CPU llegue al 65%. 

Además, si nos fijamos en el usuario asignado para esta notificación, en el apartado `to` podemos 
asignar el usuario `silent` y no nos avisará de las alertas.

~~~
template: 10min_cpu_usage
      on: system.cpu
      os: linux
   hosts: *
  lookup: average -10m unaligned of user,system,softirq,irq,guest
   units: %
   every: 1m
    warn: $this > (($status >= $WARNING)  ? (65) : (85))
    crit: $this > (($status == $CRITICAL) ? (85) : (95))
   delay: down 15m multiplier 1.5 max 1h
    info: average cpu utilization for the last 10 minutes (excluding iowait, nice and steal)
      to: sysadmin
~~~

> **NOTA**: `sysadmin` puede mandar la alerta a discord o slack, esto lo explicaremos más abajo

Guardamos la configuración y ya nos saldrá configurado en nuestra web.
![Netdata](https://github.com/MoralG/Metricas_logs_y_monitorizacion_Netdata/blob/master/image/Netdata4.png?raw=true)

Para probar lo de las notificaciones vamos a estresar nuestra máquina. Vamos a instalar el paquete de `stress` y esperaremos a que la utilización de la CPU aumente hasta el 60%.

~~~
sudo apt install stress
~~~

Estresamos la máquina 
~~~
stress --cpu 8 --io 4 --vm 2 --vm-bytes 128M --timeout 200s
  stress: info: [685] dispatching hogs: 8 cpu, 4 io, 2 vm, 0 hdd
~~~

Como podemos ver, la CPU esta al 100%:
![Netdata](https://github.com/MoralG/Metricas_logs_y_monitorizacion_Netdata/blob/master/image/Netdata5.png?raw=true)

Y tenemos las notificaciones que nos avisan:
![Netdata](https://github.com/MoralG/Metricas_logs_y_monitorizacion_Netdata/blob/master/image/Netdata6.png?raw=true)

Esto esta muy bien, pero en el caso de no estar atento a la web, tenemos que buscar un metodo para que nos notifiquen, por eso vamos a configurar netdata para que nos envien las alertas a discord y slack.

### Discord

Para configurar las notificaciones en discord tenemos que crear un servidor en discord y avtivar la opción de **Webhook**
![Netdata](https://github.com/MoralG/Metricas_logs_y_monitorizacion_Netdata/blob/master/image/Netdata2.png?raw=true)

Nos dará un enlace, el cual añadiremos a la configuración de netdata en el fichero `health_alarm_notify.conf` con el script `edit-config`.

~~~
sudo ./edit-config health_alarm_notify.conf
  # discord (discordapp.com) global notification options

  # multiple recipients can be given like this:
  #                  "CHANNEL1 CHANNEL2 ..."

  # enable/disable sending discord notifications
  SEND_DISCORD="YES"

  # Create a webhook by following the official documentation -
  # https://support.discordapp.com/hc/en-us/articles/228383668-Intro-to-Webhooks
  DISCORD_WEBHOOK_URL="https://discordapp.com/api/webhooks/676357546080206848/  ePWMIzAJd-YsjyacjHc01xun_SV6E

  # if a role's recipients are not configured, a notification will be send to
  # this discord channel (empty = do not send a notification for unconfigured
  # roles):
  DEFAULT_RECIPIENT_DISCORD="alarms"
~~~

Como podemos ver, en `DISCORD_WEBHOOK_URL` añadiremos la url proporcionada al activar **Webhook** y en `DEFAULT_RECIPIENT_DISCORD` le indicamos `alarms`.


Le realizamos el mismo comando de antes para estresarlo y nos avisará.
![Netdata](https://github.com/MoralG/Metricas_logs_y_monitorizacion_Netdata/blob/master/image/Netdata8.png?raw=true)

### Slack

Vamos a configurar como en el caso de discord, el fichero `health_alarm_notify.conf` con el script `edit-config`. Y añadimos a `SLACK_WEBHOOK_URL` el enlace que nos da **Webhook** al crearlo en nuestro servidor.

![Netdata](https://github.com/MoralG/Metricas_logs_y_monitorizacion_Netdata/blob/master/image/Netdata7.png?raw=true)

Además le indicamos en `DEFAULT_RECIPIENT_SLACK` que sea `#` para que reconozca el canal que le hemos indicado en el **Webhook**.
~~~
sudo ./edit-config health_alarm_notify.conf
  # slack (slack.com) global notification options

  # multiple recipients can be given like this:
  #                  "RECIPIENT1 RECIPIENT2 ..."

  # enable/disable sending slack notifications
  SEND_SLACK="YES"

  # Login to your slack.com workspace and create an incoming webhook, using the "Incoming Webhooks" App:  ht
  # Do not use the instructions in https://api.slack.com/incoming-webhooks#enable_webhooks, as those  webhoo
  # You need only one for all your netdata servers (or you can have one for each of your netdata).
  # Without the app and a webhook, netdata cannot send slack notifications.
  SLACK_WEBHOOK_URL="https://hooks.slack.com/services/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"

  # if a role's recipients are not configured, a notification will be send
  # to:# - A slack channel (syntax: '#channel' or 'channel')
  # - A slack user (syntax: '@user')
  # - The channel or user defined in slack for the webhook (syntax: '#')
  # empty = do not send a notification for unconfigured roles
  DEFAULT_RECIPIENT_SLACK="#"
~~~

Nos conectamos al usuario netdata
~~~
sudo su -s /bin/bash netdata
~~~

Probamos que funciona con un test que se ejecuta con el siguiente script:
~~~
/usr/libexec/netdata/plugins.d/alarm-notify.sh test

  # SENDING TEST WARNING ALARM TO ROLE: sysadmin
  /etc/netdata/health_alarm_notify.conf: line 435: Colors: command not found
  2020-02-10 18:30:02: alarm-notify.sh: INFO: sent slack notification for: serranito test.chart.  test_alarm is WARNING without specifying a channel
  2020-02-10 18:30:02: alarm-notify.sh: INFO: sent discord notification for: serranito test.chart.  test_alarm is WARNING to 'alarms'
  2020-02-10 18:30:03: alarm-notify.sh: INFO: sent email notification for: serranito test.chart.  test_alarm is WARNING to 'root'
  # OK

  # SENDING TEST CRITICAL ALARM TO ROLE: sysadmin
  /etc/netdata/health_alarm_notify.conf: line 435: Colors: command not found
  2020-02-10 18:30:03: alarm-notify.sh: INFO: sent slack notification for: serranito test.chart.  test_alarm is CRITICAL without specifying a channel
  2020-02-10 18:30:03: alarm-notify.sh: INFO: sent discord notification for: serranito test.chart.  test_alarm is CRITICAL to 'alarms'
  2020-02-10 18:30:03: alarm-notify.sh: INFO: sent email notification for: serranito test.chart.  test_alarm is CRITICAL to 'root'
  # OK

  # SENDING TEST CLEAR ALARM TO ROLE: sysadmin
  /etc/netdata/health_alarm_notify.conf: line 435: Colors: command not found
  2020-02-10 18:30:04: alarm-notify.sh: INFO: sent slack notification for: serranito test.chart.  test_alarm is CLEAR without specifying a channel
  2020-02-10 18:30:04: alarm-notify.sh: INFO: sent discord notification for: serranito test.chart.  test_alarm is CLEAR to 'alarms'
  2020-02-10 18:30:04: alarm-notify.sh: INFO: sent email notification for: serranito test.chart.  test_alarm is CLEAR to 'root'
  # OK
~~~

Como podemos ver, nos llega las alertas del test.

![Netdata](https://github.com/MoralG/Metricas_logs_y_monitorizacion_Netdata/blob/master/image/Netdata9.png?raw=true)