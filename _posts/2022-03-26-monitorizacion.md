---
tags: ["escritorios", "docker"]
title: "Monitorización de contenedores Docker con Telegraf, InfluxDB y Grafana"
layout: splash
permalink: /monitorización-de-contenedores-docker-Telegraf-InfluxDB-Grafana/
header:
  overlay_color: "#000"
  overlay_filter: "0.5"
  overlay_image: https://cdn.pixabay.com/photo/2017/06/10/11/40/speedometer-2389746_960_720.jpg
excerpt: "Montaremos un panel web con grafana para monitorizar un ambiente de contenedores Docker. Telegraf se encargará de extraer las métricas de Docker, que serán guardadas en la base de datos InfluxDB y luego el panel web Grafana mostrará esos datos a tiempo real en paneles."
---

## Monitorización de un ambiente Docker.
La monitorización en informática es fundamental. Nos permite controlar en todo momento lo que esta pasando y, en caso de fallo, podremos reaccionar rápidamente al problema evitando que este se agrave o se prolongue demasiado. A lo mejor en un entorno doméstico no le vemos sentido a monitorizar servicios pero en un ambiente de producción es completamente necesario tener cada servicio monitorizado al detalle.

En este caso vamos a hablar de monitorizar un ambiente Docker. Existen multitud de herramientas en el mercado que nos permiten chequear nuestros servidores y dichas herramientas se pueden adaptar para la vigilancia de contenedores, aunque en este caso vamos a utilizar herramientas pensadas exclusivamente para monitorizar Docker.

## ¿Qué es importante monitorizar?
Esta es la pregunta del millón cuando nos replanteamos vigilar nuestros servidores. Si empezamos a buscar encontramos herramientas que monitorizan absoulutamente todo, no se les escapa ni un detalle. 

Esto es una arma de doble filo, ya que si monitorizamos cualquier evento que ocurra acabaremos por sobrecargar el servidor nosotros mismos haciendo que el tráfico para los usuarios que acceden se ralentice o, en el peor de los casos, tumbemos nosotros solitos el servidor sin la ayuda de ningún ciberdelicuente encapuchado.

Lo que quiero decir es que tenemos que tener claro lo que queremos controlar para no sobrecargar nuestro servidor, ya que para monitorizar un servicio debe haber una aplicación que le haga continuas peticiones a dicho servicio. Es como si cada 5 segundos pincharamos sobre recargar página para comprobar que la web está funcionando.

Dicho todo esto, los aspectos más importantes a monitorizar son:
* Espacio ocupado en el disco: es muy importante que controlemos el espacio en disco de nuestros servicios, ya que podemos llegar a ocuparlo entero sin darnos cuenta y esto puede desencadenar en un fallo.
* Memoria RAM consumida: al igual que el anterior apartado es muy importante vigilar la RAM, ya que un elevado consumo puede ser síntoma de algún fallo en el servicio.
* Red: la red se puede monitorizar controlando el número de paquetes que se intercambian o midiendo la velocidad con la estos se intercambian. Por ejemplo, una red muy saturada puede indicar un ataque DoS o DDoS o simplemente nuestro servicio tiene tantas peticiones que el servdor no es capaz de atenderlas todas, por lo que tendríamos que pensar en escalar dichos servicios.
* Número de contenedores activos: Esto puede parecer de lo más sencillo pero a la vez es lo más útil ya que, si nosotros hemos levantado 20 contenedores pero sólo aparecen 18 corriendo sabemos que hay 2 de ellos que por cualquier motivo nos están funcionando.
* CPU: Esto viene bien para saber si algún servicio hace cuello de botella, por ejemplo.

## Servicios
Para lograr un buen entorno de monitorización con Docker debemos usar los siguientes servicios.
* ### Telegraf
Este servicio se encarga de obtener las métricas de los servicios que queramos monitorizar y las guarda en una base de datos para poder consultarlas y, en nuestro caso, mostrarlas en un panel web.

Para monitorizar Docker usando Telegraf usaremos el plugin específico que tiene para realizar dicha tarea. Este plugin se comunica directamente con el motor de Docker a través de socket. Los ficheros de socket sirven para intercambiar datos entre programas, explicado a muy groso modo. 

Telegraf obtiene las métricas de Docker a través del fichero socket que tiene el propio Docker ubicado en _/var/run/docker.sock_. Por cierto, cuando hablamos de métricas nos referimos a la medición de un software en base a unos parámetros.

Al estar en un contenedor Telegraf no puede ver el fichero socket de por sí, por lo que será necesario montarlo dentro.

Para habilitar el plugin de Docker en Telegraf es necesario copiar las siguientes líneas al fichero de configuración de Telegraf, ubicado en _/etc/telegraf/telegraf.conf_:
~~~yaml
[[inputs.docker]]
  ## Docker Endpoint
  ##   To use TCP, set endpoint = "tcp://[ip]:[port]"
  ##   To use environment variables (ie, docker-machine), set endpoint = "ENV"
  endpoint = "unix:///var/run/docker.sock"

  ## Set to true to collect Swarm metrics(desired_replicas, running_replicas)
  ## Note: configure this in one of the manager nodes in a Swarm cluster.
  ## configuring in multiple Swarm managers results in duplication of metrics.
  gather_services = false

  ## Only collect metrics for these containers. Values will be appended to
  ## container_name_include.
  ## Deprecated (1.4.0), use container_name_include
  container_names = []

  ## Set the source tag for the metrics to the container ID hostname, eg first 12 chars
  source_tag = false

  ## Containers to include and exclude. Collect all if empty. Globs accepted.
  container_name_include = []
  container_name_exclude = []

  ## Container states to include and exclude. Globs accepted.
  ## When empty only containers in the "running" state will be captured.
  ## example: container_state_include = ["created", "restarting", "running", "removing", "paused", "exited", "dead"]
  ## example: container_state_exclude = ["created", "restarting", "running", "removing", "paused", "exited", "dead"]
  # container_state_include = []
  # container_state_exclude = []

  ## Timeout for docker list, info, and stats commands
  timeout = "5s"

  ## Whether to report for each container per-device blkio (8:0, 8:1...),
  ## network (eth0, eth1, ...) and cpu (cpu0, cpu1, ...) stats or not.
  ## Usage of this setting is discouraged since it will be deprecated in favor of 'perdevice_include'.
  ## Default value is 'true' for backwards compatibility, please set it to 'false' so that 'perdevice_include' setting
  ## is honored.
  perdevice = true

  ## Specifies for which classes a per-device metric should be issued
  ## Possible values are 'cpu' (cpu0, cpu1, ...), 'blkio' (8:0, 8:1, ...) and 'network' (eth0, eth1, ...)
  ## Please note that this setting has no effect if 'perdevice' is set to 'true'
  # perdevice_include = ["cpu"]

  ## Whether to report for each container total blkio and network stats or not.
  ## Usage of this setting is discouraged since it will be deprecated in favor of 'total_include'.
  ## Default value is 'false' for backwards compatibility, please set it to 'true' so that 'total_include' setting
  ## is honored.
  total = false

  ## Specifies for which classes a total metric should be issued. Total is an aggregated of the 'perdevice' values.
  ## Possible values are 'cpu', 'blkio' and 'network'
  ## Total 'cpu' is reported directly by Docker daemon, and 'network' and 'blkio' totals are aggregated by this plugin.
  ## Please note that this setting has no effect if 'total' is set to 'false'
  # total_include = ["cpu", "blkio", "network"]

  ## docker labels to include and exclude as tags.  Globs accepted.
  ## Note that an empty array for both will include all labels as tags
  docker_label_include = []
  docker_label_exclude = []

  ## Which environment variables should we use as a tag
  tag_env = ["JAVA_HOME", "HEAP_SIZE"]

  ## Optional TLS Config
  # tls_ca = "/etc/telegraf/ca.pem"
  # tls_cert = "/etc/telegraf/cert.pem"
  # tls_key = "/etc/telegraf/key.pem"
  ## Use TLS but skip chain & host verification
  # insecure_skip_verify = false
~~~

Ahora nos encontramos con otro inconveniente. No todos los usuarios están autorizados a usar el fichero _docker.sock_, por lo que si de primeras consultamos los logs del contendor Telegraf obtendremos muchos errores de acceso denegado, y las tablas de la base de datos InfluxDB estarán vacías.

Para solucionarlo he visto que mucha gente le cambia los permisos al fichero _docker.sock_, y eso NO SE DEBE HACER BAJO NINGÚN CONCEPTO, ya que si lo hacemos mal podríamos crear un fallo de seguridad y un atacante podría tirar todos nuestros servicios, o incluso denegarnos el permiso a nosotros mismos.

Para arreglar este fallo se me ha ocurrido una chapuza al "estilo compadre". Antes que nada vamos a mirar los permisos del fichero _docker.sock_

~~~
srw-rw---- 1 root docker 0 abr  8 16:10 /var/run/docker.sock
~~~

Vemos que sólo el usuario root y los usuarios que pertenezcan al grupo _docker_ tienen acceso. La chapuza consiste en crear un grupo docker en el fichero _/etc/group_ del contenedor Telegraf y añadir al usuario _telegraf_ a ese grupo. Esto se hace ya que el usuario por defecto del contenedor Telegraf es "telegraf" y este usuario es el que intenta acceder al fichero, pero como no es _root_ ni pertenece al grupo _docker_ no podrá usarlo.

Lo que nosotros hacemos es crear un grupo con el mismo GUID y con el mismo nombre para hacerle creer al fichero de socket que realmente ese usuario pertenece al grupo _docker_ cuando realmente pertenece a un grupo que se llama igual pero que no tiene nada que ver con Docker.

Listo, ahora nuestro usuario _telegraf_ podrá obtener todas las métricas de Docker usando su fichero de socket. Chapuza de cuñado de barra de bar de nivel 4.

Retomando un poco el hilo, habíamos conseguido que Telegraf obtuviera las métricas de Docker, pero ahora necesitaremos un sitio donde almacenarlas. Para ello existen las bases de datos, y en concreto usaremos InfluxDB. Para ello vamos a vovler al fichero de configuración de Telegraf y vamos a añadir la siguiente línea:
~~~yaml
[[outputs.influxdb]]
  urls = ["http://influxdb:8086"]
  database = "influx"
  timeout = "5s"
  username = "admin"
  password = "admin"
~~~
Donde:
* urls: será la IP con el puerto donde está alojado InfluxDB. En nuestro caso podemos usar el nombre del contenedor en vez de la dirección IP porque como hemos levantado todos los contenedores con un fichero docker-compose éste ha creado una red custom donde están todos ellos y, al ser una red custom, dispone de resolución de nombres.
* databse: será la base de datos donde se guardarán las métricas.
* timeout: el tiempo máximo que un usuario estará conectado a la base de datos hasta que sea desconectado.
* username: el nombre de usuario con el accederemos a la base de datos .
* password: la contraseña del usuario especificado anteriormente.

Resumiendo un poco. Ya tenemos un servicio que se encarga de traerse las métricas de Docker y almacenarla en una base de datos. Ahora toca configurar esa bae de datos.

* ### InfluxDB
A lo largo de todo el artículo hemos hablado de almacenar las métricas en una base de datos, pero dicha base de datos es específica para guardar este tipo de contenido. 

InfluxDB es una base de datos de series temporales (TSDB). Un TSDB es un software pensado para tratar información ocurrida en series de tiempo. Normalmente esa información son mediciones, eventos, muestreos, monitorización, etc...

A parte de ser muy eficiente en esta tarea, InfluxDB usa InfluxQL, un lenguaje propio que permite interactuar con los datos usando SQL. Además, esta base de datos es de código abierto.

Tanto Telegraf como InfluxDB forman parte del conocido stack TICK:
* Telegraf: extrae métricas.
* InfluxDB: Almacena esas métricas.
* Chronograf: permite mostrar e interpretar esas métricas.
* Kapacitor: es un framework para procesar datos.

* ### Grafana
Llegados a este punto tenemos una base de datos con un montón de información de nuestros contenedores, ahora sólo hace falta un software que extraiga esos datos de Influx y que los interprete a través de cómodos páneles web. El software que buscamos es Grafana.

Grafana se encargará de hacer peticiones a InfluxDB para traerse las métricas e interpretarlas. Para mostrar la información es necesario crear dashboards, formados por paneles y cada panel mostrará una métrica diferente interpretada de una manera diferente.

Hay métricas que son interpretadas más claras a través de una gráfica (consumo de CPU, RAM, disco, etc...) y otras que se pueden representar con un número (nº de contenedores levantados, nº de imágenes descargadas, etc...).

Antes que nada tenemos que configurar Grafana para acceder a la base de datos de InfluxDB. Esto se hace configurando una nueva fuente de datos. Tenemos una lista muy grande de bases de datos y software de métricas a los que puede acceder Grafana, entre los que está InfluxDB.

Cuando seleccionamos InfluxDB como proveedor de datos se nos abrirá un menú para configurar una serie de parámetros necesarios para establecer una conexión, como:
* La ip con el puerto donde se encuentra InfluxDB. Si usamos contenedores docker levantados con un stack de Docker-compose podremos utilizar nombres en vez de IP.
* Nombre de la base de datos.
* Usuario de acceso a esa base de datos.
* Contraseña de acceso para ese usuario.

También existen otros parámetros que podemos establecer, como el lenguaje que se usará para hacer las consultas a la base de datos entre otras posibilidades.

Una vez que Grafana tiene acceso a la base de datos sólo queda crear un Dashboard y añadirle paneles. Cuando creemos un nuevo panel tenemos que seleccionar la fuente de datos de InfluxDB que configuramos anteriormente. Una vez seleccionada tendremos que extraer "a mano" la metrica que queremos mostrar. Para ello hay un menú que nos permite realizar la consulta en un lenguaje parecido al SQL seleccionando las acciones y las columnas que usaremos. Después sólamente tendremos que seleccionar el tipo de panel que más se adecue a nuestras necesidades.

Para realizar tu mismo los dashboard es necesario leer la documentación del plugin de Docker para Telegraf para saber que información se almacena y cómo lo hace, aparte de buscar los paneles. Para facilitar esta tarea la comunidad ha realizado unos dashboard que ya vienen configurados con todos los paneles y con todas las consultas ya hechas, para que nosotros no tengamos que preocuparnos de nada.

En caso que queremos usar uno de estos dashborad simplemente tenemos que copiar el código identificador y pegarlo en el apartado de añadir nuevo panel através de grafana.com.

## Despliegue.

Como seguramente os habéis fijado, no he añadido configuración ni a InfluxDB ni a Grafana. Esto es porque InlfuxDB se configura directamente a través del fichero Docker-compose y Grafana se configura mediante web.

Para desplegarlo copiaremos y pegaremos las siguiente líneas en un fichero **docker-compose.yaml**
~~~yaml
version: '3.6'
services:
  telegraf:
    image: telegraf
    container_name: telegraf
    restart: always
    volumes:
    - ./telegraf.conf:/etc/telegraf/telegraf.conf:ro
    - /var/run/docker.sock:/var/run/docker.sock
    depends_on:
      - influxdb
    links:
      - influxdb
    ports:
    - '8125:8125'

  influxdb:
    image: influxdb:1.8-alpine
    container_name: influxdb
    restart: always
    environment:
      - INFLUXDB_DB=influx
      - INFLUXDB_ADMIN_USER=admin
      - INFLUXDB_ADMIN_PASSWORD=admin
    ports:
      - '8086:8086'
      - '8083:8083'
    volumes:
      - influxdb_data:/var/lib/influxdb

  grafana:
    image: grafana/grafana
    container_name: grafana-server
    restart: always
    depends_on:
      - influxdb
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_INSTALL_PLUGINS=
    links:
      - influxdb
    ports:
      - '3000:3000'
    volumes:
      - grafana_data:/var/lib/grafana

volumes:
  grafana_data: {}
  influxdb_data: {}
~~~

