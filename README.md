#### Juliana Zúñiga - A00068120
# Parcial 2 - Sistemas Distribuidos

## Requerimientos
- Docker
- Docker-compose

---
## Desarrollo

### **1. Comandos de linux necesarios para el aprovisionamiento de los servicios solicitados**


#### Instalando y corriendo Elasticsearch
Crear un directorio para almacenar todo el Stack de EFK

```bash
mkdir -p /Users/amyth/installs/efk
```
Instalar y correr Elasticsearch
```bash
sudo apt-get update
sudo apt-get install openjdk-7-jre

java -version
```
Debería tener la siguiente configuración de Java
```bash
java version "1.7.0_75"
Java(TM) SE Runtime Environment (build 1.7.0_75-b13)
Java HotSpot(TM) 64-Bit Server VM (build 24.75-b04, mixed mode)
```

```bash
tar -xzvf elasticsearch-2.1.0.tar.gz
mv elasticsearch-2.1.0 ~/installs/efk/

cd ~/installs/efk/elasticsearch-2.1.0
./bin/elasticsearch

or

cd ~/installs/efk/elasticsearch-2.1.0
./bin/elasticsearch -d
```

Después de correr Elasticsearch, confirmar que la instancia está levantada yendo a la dirección localhost:9200 y debería aparecer lo siguiente

```json
{
  "name" : "Cerise",
  "cluster_name" : "elasticsearch",
  "version" : {
    "number" : "2.1.0",
    "build_hash" : "72cd1f1a3eee09505e036106146dc1949dc5dc87",
    "build_timestamp" : "2015-11-18T22:40:03Z",
    "build_snapshot" : false,
    "lucene_version" : "5.3.1"
  },
  "tagline" : "You Know, for Search"
}
```

#### Instalando y corriendo Kibana
Descarcar kibana de [este enlace](https://www.elastic.co/downloads/kibana), una vez descargado mover el archivo a la carpeta que se creó *~installs/efk* y descomprimirlo

```bash
mv ~/Downloads/kibana-4.3.0-darwin-x64.tar.gz ~/installs/efk
cd ~/installs/efk
tar -xzvf kibana-4.3.0-darwin-x64.tar.gz
```

A continuación, correr kibana de la siguiente forma

```bash
cd kibana-4.3.0-darwin-x64
./bin/kibana
```

Navegar a http://0.0.0.0:5601 y ahí estará el dashboard de Kibana

#### Instalando y corriendo Fluentd
Para la instalación de Fluentd, éste provee un script que automatiza el proceso de instalación, éstos scripts están disponibles para:
- ubuntu: Trusty, Precise and Lucid
- debian:Â Jessie, Wheezy and Squeeze.

Usar los siguientes comandos (dependiendo del sistema operativo) para obtenerlos:
```bash
## Ubuntu Trusty
curl -L https://toolbelt.treasuredata.com/sh/install-ubuntu-trusty-td-agent2.sh | sh
 
## Ubuntu Precise
curl -L https://toolbelt.treasuredata.com/sh/install-ubuntu-precise-td-agent2.sh | sh
 
## Ubuntu Lucid
curl -L https://toolbelt.treasuredata.com/sh/install-ubuntu-lucid-td-agent2.sh | sh
 
## Debian Jessie
curl -L https://toolbelt.treasuredata.com/sh/install-debian-jessie-td-agent2.sh | sh
 
## Debian Wheezy
curl -L https://toolbelt.treasuredata.com/sh/install-debian-wheezy-td-agent2.sh | sh
 
## Debian Squeeze
curl -L https://toolbelt.treasuredata.com/sh/install-debian-squeeze-td-agent2.sh | sh
```

Iniciar el td-agent
```bash
/etc/init.d/td-agent restart

#Asegurarse que el td-agent está corriendo	
/etc/init.d/td-agent status
```

#### Juntar EFK, Elasticsearch, Fluentd y Kibana stack
Obtener los plugins requeridos por Fluentd
```bash
udo apt-get install make libcurl4-gnutls-dev --yes
sudo /opt/td-agent/embedded/bin/fluent-gem install fluent-plugin-elasticsearch
sudo /opt/td-agent/embedded/bin/fluent-gem install fluent-plugin-record-reformer
```

Ahora para enviar la información de Fluentd a Elasticsearch abrir el archivo *etc/td-agent/td-agent.conf* y reemplazar la configuración existente con la siguiente

```html
<source>
    type syslog
    port 5140
    tag  system
</source>
 
<match system.*.*>
    type record_reformer
    tag efkl
    facility ${tag_parts[1]}
    severity ${tag_parts[2]}
</match>
 
<match efkl>
    type copy
    <store>
       type elasticsearch
       logstash_format true
       flush_interval 15s
    </store>
</match>
```
Correr fluentd usando el siguiente comando
```bash
## Ubuntu
sudo service td-agent start
 
## Mac OS X
sudo launchctl load /Library/LaunchDaemons/td-agent.plist
```
También necesitaríamos decirle a syslog / rsyslog que transmita los datos de registro a fluentd. Así que vamos a abrir el archivo de configuración de syslog.

```bash
## Ubuntu
sudo vim /etc/rsyslog.conf
 
## Mac OS X
sudo vim /etc/rsyslog.conf
```

y agregue la siguiente línea a él. Esto le dice a syslog que reenvíe los datos de registro al host 127.0.0.1 que es nuestro host local en el puerto 5140. Como fluentd escucha el puerto 5140 de forma predeterminada.

```bash
*.*                             @127.0.0.1:5140
```

Ahora reiniciar el servicio
```bash
curl -XPUT 'http://localhost:9200/kibana/' -d '{"index.mapper.dynamic": true}'
```

Ahora dirígete a tu dashboard de kibana navegando en tu navegador web a 'http://0.0.0.0:5601' y elige la pestaña de configuración e ingresa **kibana*/** en el campo “index name or pattern”. A continuación, desmarque "Index contains time-based events" y haga clic en el botón Crear.

Ahora ir a la pestaña **Discover** y ahí debería ver los logs del syslog

### **2. Archivos Dockerfile**
```Dockerfile
# fluentd/Dockerfile
FROM fluent/fluentd:v0.12.40-debian
RUN ["gem", "install", "fluent-plugin-elasticsearch", "--no-rdoc", "--no-ri", "--version", "1.9.2"]
```
### **3. Archivos para el despliegue de la infraestructura**
```yml
version: '2'
services:
  web:
    image: httpd
    ports:
      - "80:80"
    links:
      - fluentd
    logging:
      driver: "fluentd"
      options:
        fluentd-address: localhost:24224
        tag: httpd.access

  fluentd:
    build: ./fluentd
    volumes:
      - ./fluentd/conf:/fluentd/etc
    links:
      - "elasticsearch"
    ports:
      - "24224:24224"
      - "24224:24224/udp"

  elasticsearch:
    image: elasticsearch
    expose:
      - 9200
    ports:
      - "9200:9200"

  kibana:
    image: kibana
    links:
      - "elasticsearch"
    ports:
      - "5601:5601"
```

### **4. Estructura de carpetas y archivos necesarios para el aprovisionamiento**

#### Estructura de archivos

```
sd-exam2
    ├── A00068012
    │   ├── docker-compose.yml
    │   ├── fluentd
    │   │   ├── conf
    │   │   │   └── fluent.conf
    │   │   └── Dockerfile
    │   └── README.md
    └── README.md
```

#### Fluent.conf
```c
# fluentd/conf/fluent.conf
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>
<match *.**>
  @type copy
  <store>
    @type elasticsearch
    host elasticsearch
    port 9200
    logstash_format true
    logstash_prefix fluentd
    logstash_dateformat %Y%m%d
    include_tag_key true
    type_name access_log
    tag_key @log_name
    flush_interval 1s
  </store>
  <store>
    @type stdout
  </store>
</match>
```

### **5. Pruebas del funcionamiento**
#### Contenedores corriendo
Los puertos en los que corre cada uno de los servicios son los siguientes:
- Servidor Web: 80
- ElasticSearch: 9200
- Fluentd: 24224
- Kibana: 5601

![][1]

#### Servidor Web
Este servidor sólo se necesitaba para ser al que se le lanzaban las peticiones, éste en envía los logs de las peticiones http que le realizan al contenedor de Fluentd.

![][2]

#### ElasticSearch
Fluentd envía los logs que recibe al contenedor de Elasticsearch, éste funciona como una base de datos almacenándolos, éstos logs pueden ser accedidos por medio de peticiones a ciertos endpoints

![][3]

#### Kibana
En este contenedor se visualizan los logs que están en el contenedor de Elasticsearch.

![][4]

[1]: images/Containers.png
[2]: images/Webserver.png
[3]: images/Elastic.png
[4]: images/kibana.png