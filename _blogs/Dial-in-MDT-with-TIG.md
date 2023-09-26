---
published: true
date: '2023-08-23 17:09 -0400'
title: Dial-in Model-Driven Telemetry with TIG Stack
position: hidden
author: Rahul Sharma
---
{% include toc icon="table" title="ON THIS PAGE" %}

# Introduction

<p align="justify">This article focuses on  establishing dial-in Model-Driven Telemetry (MDT) using TIG (Telegraf, Influxdb and Grafana) stack.
<br>
<br>
If you are well-versed with Streaming Telemtry, MDT and YANG models, please follow the steps to establish MDT. Otherwise you might want to go through the following post.</p>
> **__** https://xrdocs.io/programmability/blogs/Model-Driven-Telemetry/
<br>
<p align="justify">  
MDT consists of two main components. Firstly, there's a router equipped with pre-installed YANG models and a running grpc server. Through MDT, this router continuously streams metrics at specified time intervals. Secondly, the TIG stack, which consumes, stores, and presents this metrics visually.
<br>
<br>  
Let's delve into the individual elements of the TIG stack:</p>

# TIG Stack

<p align="justify"> <b>1. Telegraf:</b> Collects router metrics and stores them in a Time Series Database (TSDB).
<br>
<br>  
<b>2. InfluxDB:</b> TSDB that stores metrics from Telegraf. Forwards data to Grafana for visualization.
<br>
<br>  
<b>3. Grafana:</b> Receives metrics from InfluxDB and displays them visually using graphs and charts.</p>


# gRPC Dial-in

<p align="justify">The objective is to create a Dial-in MDT, wherein a collector initiates a grpc channel with a router.
<br>
<br>  
Since collector is initialing the grpc channel here, the router must be operational and prepared to receive a request for gRPC channel by a collector. In short, gRPC server on the router must be up and running.
<br>
<br>  
The initial focus will be on configuring the router aspect.</p>

## Router Configuration

<p align="justify"> Configure the gRPC server on the router to receive channel creation requests from the collector, utilizing the following steps:</p>
  
Step1: Configure gRPC server on the router. 

```
RP/0/RP0/CPU0:ios(config)#grpc
RP/0/RP0/CPU0:ios(config-grpc)#port 57100
RP/0/RP0/CPU0:ios(config-grpc)#no-tls
RP/0/RP0/CPU0:ios(config-grpc)#commit
```

Step2: Confirm the configuration.

```
RP/0/RP0/CPU0:ios#show run grpc
Wed Aug 23 21:27:37.876 UTC
grpc
 port 57100
 no-tls
!
```

> **_NOTE:_** The grpc port range is 57344 to 57999, with default being 57400.

## Collector configuration
<p align="justify">The subsequent topology and steps will be used to configure TIG stack:</p>

![MDT-TIG.png]({{site.baseurl}}/images/MDT-TIG.png)

Step1: Create a telegraf.conf file.

```
# This section defines the  settings for the router that Telegraf will dial-in or connect to.
[[inputs.gnmi]]                                                           
   addresses = ["10.30.111.168:57100"]     
   username = "cisco"                      
   password = "cisco123!"

# This section defines the metric that Telegraf wants router to stream.
[[inputs.gnmi.subscription]]
   name = "ifcounters"                                                                   
   origin = "Cisco-IOS-XR-infra-statsd-oper"							  
   path = "/infra-statistics/interfaces/interface/latest/generic-counters"
   subscription_mode = "sample"    # (one of: "target_defined", "sample", "on_change")
   sample_interval = "30s"		   # Cadence between two consecutive samples.

# This section defines where will Telegraf to store the metric.
[[outputs.influxdb]]

   urls = ["http://localhost:8086"]
   database = "mdt-db"
```

<p align="justify">
Step2: Create a docker-compose.yml file.</p> 
```
version: '3.6'							# Version of docker-compose file
services:							    # Individual docker services for TIG
  telegraf:							    # telegraf service configuration	
    image: telegraf								
    container_name: telegraf
    restart: always                     # Restarts automatically if it crashes
    volumes:
    - ./telegraf.conf:/etc/telegraf/telegraf.conf:ro  # Mounts telegraf.conf into the container
    depends_on:						    # Waits until influxdb service is running
      - influxdb
    links:								# Connects this service to influxdb service
      - influxdb
    ports:								# Maps port 57100 on host to container.
    - '57100:57100'
       
  influxdb:								# influxdb service configuration
    image: influxdb:1.8-alpine
    container_name: influxdb
    restart: always
    environment:						# To create a database, user and password.
      - INFLUXDB_DB=
      - INFLUXDB_ADMIN_USER=
      - INFLUXDB_ADMIN_PASSWORD=
    ports:
      - '8086:8086'
    volumes:
      - influxdb_data:/var/lib/influxdb

  grafana:								# grafana service configuration
    image: grafana/grafana
    container_name: grafana-server
    restart: always						# Restarts automatically if it crashes
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

volumes:								# To store data persistently on the host
  grafana_data: {}									 
  influxdb_data: {}

```

<p align="justify">Step3: Navigate to the directory containing both files and utilize the provided command to initiate container deployment.</p>

```
docker-compose up
```

<p align="justify">At this stage, the collector is set to gather metrics when the network device initiates streaming. The following command can be executed to check the status of containers.</p>

```
docker ps
```
Subsequently, the displayed output will be as follows:

```
CONTAINER ID        IMAGE                                                   COMMAND                  CREATED             STATUS                          PORTS                                                                                                                      NAMES
ea8d09f16378        grafana/grafana                                         "/run.sh"                32 seconds ago      Up 30 seconds                   0.0.0.0:3000->3000/tcp                                                                                                     grafana-server
302ca634ace8        telegraf                                                "/entrypoint.sh tele…"   32 seconds ago      Up 31 seconds                   8092/udp, 0.0.0.0:8125->8125/tcp, 8125/udp, 8094/tcp, 0.0.0.0:57100->57100/tcp                                             telegraf
9ef752784c05        influxdb:1.8-alpine                                     "/entrypoint.sh infl…"   34 seconds ago      Up 32 seconds                   0.0.0.0:8086->8086/tcp                                                                                                     influxdb
```

<p align="justify">In order to check if MDT is working or not, execute the following command on the router.</p>

```
RP/0/RP0/CPU0:ios#show telemetry model-driven subscription
Wed Aug 23 21:46:19.300 UTC
Subscription:  GNMI__13821465244912532582  State: ACTIVE
-------------
  Sensor groups:
  Id                               Interval(ms)        State
  GNMI__13821465244912532582_0     30000               Resolved

  Destination Groups:
  Id                 Encoding            Transport   State   Port    Vrf                               IP
  GNMI_1003          gnmi-proto          dialin      Active  24474                                     192.168.122.1
    TLS :             False
```


## InfluxDB and Grafana

<p align="justify">Given that Telegraf stores data in InfluxDB, we will validate this by querying the database. Execute the provided command on the Ubuntu server where your containers are operational.</p>

```
docker exec -it influxdb bash
```

<p align="justify">Executing this command will lead you to the bash shell within the InfluxDB container.</p> 

<p align="justify">Upon accessing the container's bash shell, database commands can be employed to identify the sensor-path configured within the sensor-group.</p>

```
9ef752784c05:/# influx
Connected to http://localhost:8086 version 1.8.10
InfluxDB shell version: 1.8.10
> show databases
name: databases
name
----
mdt-db
_internal
> use mdt-db
Using database mdt-db
> show measurements
name: measurements
name
----
Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
```

<p align="justify">The presence of the 'mdt-db' database within InfluxDB is apparent. This aligns with the database mentioned in the 'telegraf.conf' file within the output section.</p>


<p align="justify">The following sensor-path as a measurement above shows that InfluxDB successfully stores the streamed data from our NCS 5500 device.</p>

```
Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
```

<p align="justify">Next, we'll create graphs utilizing InfluxDB data on Grafana.</p>

<p align="justify">To access the Grafana dashboard, enter 'http://10.30.111.165:3000/' into your browser's URL field. Keep in mind that this IP address corresponds to the server running Grafana, followed by the relevant port number. Your setup might have a different address and port.</p>

![login.png]({{site.baseurl}}/images/login.png)

<p align="justify">Upon accessing the Grafana dashboard, a login prompt will appear, requiring a username and password. Use the credentials specified in the 'docker-compose.yml' file under the 'environment' section of the 'Grafana' service. In this case, both the username and password are 'admin'.</p>

![login-creds.png]({{site.baseurl}}/images/login-creds.png)

<p align="justify">Utilize the provided username and password. You'll be prompted to update your password; however, you can select 'skip' to proceed to the home page.</p>

![homepage.png]({{site.baseurl}}/images/homepage.png)

<p align="justify">Now, let's set up the data source for Grafana. To achieve this, click on the 'Add your first data source' widget. Then, apply the filter to locate 'InfluxDB' and proceed to select it.</p>

![add-source.png]({{site.baseurl}}/images/add-source.png)

<p align="justify">Assign a name of your choice to the source. Opt for 'InfluxQL' as the query language. For the HTTP URL, input 'http://10.30.111.165:8086'. Leave out the Auth and Custom HTTP Headers sections. In the 'InfluxDB Details', add the 'mdt-db' database. Finally, click on 'Save & Test'.</p>

![source-detail1.png]({{site.baseurl}}/images/source-detail1.png)

![source-detail2.png]({{site.baseurl}}/images/source-detail2.png)

<p align="justify">The message 'Datasource is working. 1 measurement found' confirms that Grafana has successfully established a connection to the 'mdt-db' database within InfluxDB.</p>

<p align="justify">Next, a dashboard will be created through the subsequent steps:</p>

![homepage.png]({{site.baseurl}}/images/homepage.png)

<p align="justify">Access the 'Home' page and select the 'Dashboards' widget. Click on 'Add visualization', pick your data source (in this case, 'mdt-influx'), and the subsequent page will appear.</p>

![add-visual.png]({{site.baseurl}}/images/add-visual.png)

![select-data-source.png]({{site.baseurl}}/images/select-data-source.png)

<p align="justify">Click on 'select measurement', and the sensor-path configured using the sensor-group will be visible. Choose that sensor-path.</p>

![select-measurement.png]({{site.baseurl}}/images/select-measurement.png)

<p align="justify">Then, click on 'field(value)', and opt for 'bytes-received'. This action will generate the graph at the top.</p>

![initial-graph.png]({{site.baseurl}}/images/initial-graph.png)

![final-graph.png]({{site.baseurl}}/images/final-graph.png)

<p align="justify">Feel free to experiment with this query, and you'll observe the graph corresponding to your query modifications.</p>
