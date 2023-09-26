---
published: true
date: '2023-08-21 14:13 -0400'
title: Dial-out Model-Driven Telemetry with TIG Stack
position: hidden
excerpt: >-
  Master the configuration of TCP and gRPC Dial-out MDT using the TIG stack as a
  collector for streamlined data collection and visualization."
author: Rahul Sharma
tags:
  - iosxr
  - cisco
  - TIG Stack
  - Telegraf
  - InfluxDB
  - Grafana
  - Model-Driven Telemetry
  - Dial-out
  - TCP
  - gRPC
---
{% include toc icon="table" title="ON THIS PAGE" %}

# Background

<p align="justify">
This article focuses on  establishing Model-Driven Telemetry (MDT) using the Telegraf, InfluxDB, and Grafana (TIG) stack.
<br>
<br>  
If you are well-verse with MDT, YANG models and sensor-group, destination group and subscription, please follow the steps to establish MDT. Otherwise you might want to go through this post first. 
<br>
<br>  
MDT consists of two main components. Firstly, there's a router equipped with pre-installed YANG models and a running grpc server. Through MDT, this router continuously streams mtetrics at specified time intervals. Secondly, the TIG stack, which consumes, stores, and presents this metrics visually.
</p>

Let's delve into the individual elements of the TIG stack:

# TIG Stack

<b>1. Telegraf: </b>Collects router metrics and stores them in a Time Series Database (TSDB).

<b>2. InfluxDB: </b>TSDB that stores metrics from Telegraf. Forwards data to Grafana for visualization.

<b>3. Grafana:</b> Receives metrics from InfluxDB and displays them visually (graphs, charts) for analysis.


# gRPC Dial-out

<p align="justify">The objective is to create a Dial-out MDT, wherein a router initiates a grpc channel with a collector, specifically Telegraf.
<br>
<br>  
As this is a Dial-out MDT, the collector must be operational and prepared to receive data when router initiates a grpc channel. The initial focus will be on configuring the collector aspect.</p>

## Collector configuration

To establish the TIG configuration, the subsequent topology and steps will be used:

![MDT-TIG.png]({{site.baseurl}}/images/MDT-TIG.png)

Step1: Create a following telegraf.conf file.

```
# This section enables collector to listen to port 57100 using 'grpc' as a transport.
[[inputs.cisco_telemetry_mdt]]
   transport = "grpc"   (one of tcp,grpc)
   service_address = ":57100

# This section defines where will the metric that collector receives will be stored.
[[outputs.influxdb]]

  urls = ["http://influxdb:8086"]
  database = "mdt-db"
  
```

Step2: Create a following docker-compose.yml file. 
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

<p align="justify">Next, the configuration process involves enabling the network device to initiate data streaming.</p>

## Router Configuration
<p align="justify">
Step1: Create a destination-group: This comprises of collector information such as IP address, port number, encoding and protocol. In short, <b>where to send</b> the data.</p>

```
RP/0/RP0/CPU0:ios(config)# telemetry model-driven
RP/0/RP0/CPU0:ios(config-model-driven)# destination-group DGroup1
RP/0/RP0/CPU0:ios(config-model-driven-dest)#  address family ipv4 10.30.111.165 port 57100  
RP/0/RP0/CPU0:ios(config-model-driven-dest-addr)#   encoding self-describing-gpb  
RP/0/RP0/CPU0:ios(config-model-driven-dest-addr)#   protocol grpc no-tls  
RP/0/RP0/CPU0:ios(config-model-driven-dest-addr)# commit   
```
<p align="justify">Remember that the IP address in the destination-group pertains to the server hosting Telegraf, and the port number should align with what's specified in the 'telegraf.conf' file.
<br>
<br>  
Additionally, pay attention to the 'protocol grpc no-tls' configuration, as it reflects the usage of grpc dial-out MDT.</p>

Step2: Create a sensor-group: This is a group of sensor-path that router streams. In short, <b>what to send</b>.

```
RP/0/RP0/CPU0:ios(config)#telemetry model-driven
RP/0/RP0/CPU0:ios(config-model-driven)#sensor-group SGroup1
RP/0/RP0/CPU0:ios(config-model-driven-snsr-grp)# sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
RP/0/RP0/CPU0:ios(config-model-driven-snsr-grp)# commit
```
<p align="justify">Additional sensor-paths can be incorporated into this sensor-group at a later stage, facilitating the reception of corresponding streamed data by the collector.
<br>
<br>  
Interested in discovering the accurate sensor path for a desired metric to stream? Delve into the provided article, which elucidates the process of locating a sensor-path for a CLI.</p>

Step3: Create a subscription: This maps a destionation group to a sensor-group. 

```
RP/0/RP0/CPU0:ios(config)telemetry model-driven  
RP/0/RP0/CPU0:ios(config-model-driven)#subscription Sub1  
RP/0/RP0/CPU0:ios(config-model-driven-subs)#sensor-group-id SGroup1 sample-interval 30000  
RP/0/RP0/CPU0:ios(config-model-driven-subs)#destination-id DGroup1  
RP/0/RP0/CPU0:ios(config-mdt-subscription)# commit  
```

Step4: Confirm the configuration

```
RP/0/RP0/CPU0:ios# show run telemetry model-driven
telemetry model-driven  
 destination-group DGroup1  
   address family ipv4 10.30.111.165 port 57100  
   encoding self-describing-gpb  
   protocol grpc no-tls
  !
 !
 sensor-group SGroup1
  sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
 !  
 subscription Sub1  
  sensor-group-id SGroup1 sample-interval 30000  
  destination-id DGroup1   
```

Step5: Check if data is being streamed to the collector or not.

```
RP/0/RP0/CPU0:ios#show telemetry model-driven subscription
Thu Aug 21 11:27:27.751 UTC
Subscription:  Sub1                     State: ACTIVE
-------------
  Sensor groups:
  Id                Interval(ms)        State
  SGroup1           30000               Resolved

  Destination Groups:
  Id                Encoding            Transport   State   Port    IP
  DGroup1           self-describing-gpb grpc        Active  57100   10.30.111.165
```
<p align="justify">The 'Resolved' state indicates that the router successfully identified and resolved the sensor-path specified within the sensor-group.
<br>
<br>  
For further insights into this subscription, utilize the following command:</p>

```
RP/0/RP0/CPU0:ios#show telemetry model-driven subscription sub1
```
<p align="justify">The status of this subscription is observed as 'ACTIVE,' indicating that the router is actively streaming data, and the collector (Telegraf) is effectively collecting the data.</p>

## InfluxDB and Grafana

<p align="justify">Given that Telegraf stores data in InfluxDB, we will validate this by querying the database. Execute the provided command on the Ubuntu server where your containers are operational.</p>

```
docker exec -it influxdb bash
```

<p align="justify">Executing this command will lead you to the bash shell within the InfluxDB container. 
<br>
<br>  
Upon accessing the container's bash shell, database commands can be employed to identify the sensor-path configured within the sensor-group.</p>

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
<p align="justify">The presence of the 'mdt-db' database within InfluxDB is apparent. This aligns with the database mentioned in the 'telegraf.conf' file within the output section.
<br>
<>br
The following sensor-path as a measurement above shows that InfluxDB successfully stores the streamed data from our NCS 5500 device.</p>

```
Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
```

Next, we'll create graphs utilizing InfluxDB data on Grafana.

<p align="justify">To access the Grafana dashboard, enter 'http://10.30.111.165:3000/' into your browser's URL field. Keep in mind that this IP address corresponds to the server running Grafana, followed by the relevant port number. Your configuration might have a different address and port.</p>

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

Next, a dashboard will be created through the subsequent steps:

![homepage.png]({{site.baseurl}}/images/homepage.png)

<p align="justify">Access the 'Home' page and select the 'Dashboards' widget. Click on 'Add visualization', pick your data source (in this case, 'mdt-influx'), and the subsequent page will appear.</p>

![add-visual.png]({{site.baseurl}}/images/add-visual.png)

![select-data-source.png]({{site.baseurl}}/images/select-data-source.png)

<p align="justify">Select 'select measurement', and the sensor-path configured using the sensor-group will be visible. Choose that sensor-path.</p>

![select-measurement.png]({{site.baseurl}}/images/select-measurement.png)

Finally, click on 'field(value)', and opt for 'bytes-received'. This action will generate the graph at the top.

![initial-graph.png]({{site.baseurl}}/images/initial-graph.png)

![final-graph.png]({{site.baseurl}}/images/final-graph.png)

Congrats!!, you have successfully established an MDT using dial-out method.

Feel free to experiment with this query, and you'll observe the graph corresponding to your query modifications.

# TCP Dial-out

<p align="justify">TCP dial-out closely resembles grpc dial-out, requiring just two configuration adjustments in the previously outlined grpc dial-out setup to enable TCP dial-out.
<br>
<br>  
Adjustment 1: Modify the transport in the 'input' plugin from 'grpc' to 'tcp' as demonstrated below:</p>

```
[[inputs.cisco_telemetry_mdt]]
   transport = "tcp"
   service_address = ":57100

[[outputs.influxdb]]

  urls = ["http://influxdb:8086"]
  database = "mdt-db"
  
```

<p align="justify">Adjustment 2: Modify the transport in 'destination-group' from 'grpc no-tls' to 'tcp' as demonstrated below:</p>

```
RP/0/RP0/CPU0:ios# show run telemetry model-driven
telemetry model-driven  
 destination-group DGroup1  
   address family ipv4 10.30.111.165 port 57100  
   encoding self-describing-gpb  
   protocol tcp
  !
 !
 sensor-group SGroup1
  sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
 !  
 subscription Sub1  
  sensor-group-id SGroup1 sample-interval 30000  
  destination-id DGroup1   
```

<p align="justify">Upon implementing these modifications and executing the 'show telemetry model-driven subscription' command on the router, the resulting output will be as follows:</p>

```
RP/0/RP0/CPU0:ios#show telemetry model-driven subscription
Thu Aug 21 11:27:27.751 UTC
Subscription:  Sub1                     State: ACTIVE
-------------
  Sensor groups:
  Id                Interval(ms)        State
  SGroup1           30000               Resolved

  Destination Groups:
  Id                Encoding            Transport   State   Port    IP
  DGroup1           self-describing-gpb Tcp         Active  57100   10.30.111.165
```

<p align="justify">Consequently, this concludes the article, wherein you gained insights into TCP and gRPC Dial-out MDT by employing the TIG stack as a collector.</p>
