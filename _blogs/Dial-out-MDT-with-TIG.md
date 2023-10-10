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

# Introduction

<p align="justify">
This article focuses on establishing Dial-out Model-Driven Telemetry (MDT) using the Telegraf, InfluxDB, and Grafana (TIG) stack.
<br>
<br>  
If well-versed in MDT and YANG models, please proceed with the steps to establish MDT. Otherwise, it is advisable to review the following article first.</p>
> [Introduction to Model-Driven Telemetry](https://xrdocs.io/programmability/blogs/Model-Driven-Telemetry/)  

# MDT Components

MDT consists of two main components:

<p align="justify"><b>1. Router</b> equipped with pre-installed YANG models and a running grpc server. Through MDT, this router continuously streams metrics at specified time intervals.</p>

<p align="justify"><b>2. TIG stack</b> which consumes, stores, and presents this metrics visually.
</p>

Now it's time to delve into the individual elements of the TIG stack.

# TIG Stack

<b>1. Telegraf: </b>Collects router metrics and stores them in a Time Series Database (TSDB).

<p align="justify"><b>2. InfluxDB: </b>TSDB stores metrics from Telegraf and forwards it to Grafana for visualization.</p>

<p align="justify"><b>3. Grafana:</b> Receives metrics from InfluxDB and displays them visually for analysis.</p>

# gRPC Dial-out

<p align="justify">The objective is to create a Dial-out MDT, wherein a router initiates a grpc channel with a collector, specifically Telegraf.
<br>
<br>  
As this is a Dial-out MDT, the collector must be operational and prepared to receive the request when router initiates a grpc channel. Once this channel is created, router will start streaming metrics to the collector. The initial focus will be on configuring the collector.</p>

## Telegraf configuration

To establish the TIG configuration, the subsequent topology and steps will be used:

![MDT-TIG.png]({{site.baseurl}}/images/MDT-TIG.png)

<b>Step1:</b> Create a following telegraf.conf file.

```
# This section enables collector to listen to port 57100 using 'grpc' as a transport.
[[inputs.cisco_telemetry_mdt]]
   transport = "grpc"   # (one of tcp,grpc)
   service_address = ":57100"

# This section defines where will the metric that collector receives will be stored.
[[outputs.influxdb]]

  urls = ["http://influxdb:8086"]
  database = "mdt-db"
  
```

<b>Step2:</b> Create a following docker-compose.yml file. 
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
<p align="justify"><b>Step3:</b> Navigate to the directory containing both files and utilize the provided command to initiate container deployment.</p>

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
<b>Step1:</b> Create a <b>destination-group:</b> This comprises of collector information such as IP address, port number, encoding and protocol. In short, <b>where to send</b> the data.</p>

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
Additionally, pay attention to 'protocol grpc no-tls' configuration. This needs to be mentioned explicitly as TLS is enabled by default.</p>

<b>Step2:</b> Create a <b>sensor-group:</b> This is a group of sensor-path(s) that router streams. In short, <b>what to send</b>.

```
RP/0/RP0/CPU0:ios(config)#telemetry model-driven
RP/0/RP0/CPU0:ios(config-model-driven)#sensor-group SGroup1
RP/0/RP0/CPU0:ios(config-model-driven-snsr-grp)# sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
RP/0/RP0/CPU0:ios(config-model-driven-snsr-grp)# commit
```
<p align="justify">Additional sensor-paths can be incorporated into this sensor-group at a later stage, facilitating the reception of corresponding streamed data by the collector.</p>

<p align="justify">Interested in discovering the accurate sensor path for a desired metric to stream? Delve into the following article, which elucidates the process of locating a sensor-path for a CLI.</p>

> [CLI to sensor-path](https://xrdocs.io/programmability/blogs/CLI-to-sensor-path/)

<b>Step3:</b> Create a <b>subscription:</b> This maps a destination-group to a sensor-group. 

```
RP/0/RP0/CPU0:ios(config)telemetry model-driven  
RP/0/RP0/CPU0:ios(config-model-driven)#subscription Sub1  
RP/0/RP0/CPU0:ios(config-model-driven-subs)#sensor-group-id SGroup1 sample-interval 30000  
RP/0/RP0/CPU0:ios(config-model-driven-subs)#destination-id DGroup1  
RP/0/RP0/CPU0:ios(config-mdt-subscription)# commit  
```

<b>Step4:</b> Confirm the configuration.

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

<b>Step5:</b> Check if data is being streamed to the collector or not.

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

<p align="justify">Since Telegraf stores data in InfluxDB, this can be confirmed by executing the provided command on the Ubuntu server where docker containers are operational.</p>

```
docker exec -it influxdb bash
```

<p align="justify">Execution of this command will result in access to the bash shell within the InfluxDB container.
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
<br>
The following sensor-path as a measurement above shows that InfluxDB is successfully storing the data being streamed from  NCS 5500 device.</p>

```
Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
```

Next, graphs will be generated using InfluxDB data in Grafana.

<p align="justify">To access the Grafana dashboard, input 'http://10.30.111.165:3000/' into the URL field of the browser. It is important to note that the IP address corresponds to the server hosting Grafana, accompanied by the relevant port number. Configuration settings may differ, utilizing alternative addresses and ports.</p>

![login.png]({{site.baseurl}}/images/login.png)

<p align="justify">Upon accessing the Grafana dashboard, a login prompt will appear, requiring a username and password. Use the credentials specified in the 'docker-compose.yml' file under the 'environment' section of the 'Grafana' service. In this case, both the username and password are 'admin'.</p>

![login-creds.png]({{site.baseurl}}/images/login-creds.png)

<p align="justify">Utilize the provided username and password. There will be a prompt to update the password; nevertheless, selecting 'skip' is an option to proceed to the home page.</p>

![homepage.png]({{site.baseurl}}/images/homepage.png)

<p align="justify">Now, setting up the data source for Grafana involves clicking on the 'Add your first data source' widget. Afterward, apply the filter to locate 'InfluxDB' and proceed with the selection.</p>

![add-source.png]({{site.baseurl}}/images/add-source.png)

<p align="justify">Assign a chosen name to the source, select 'InfluxQL' as the query language, and input 'http://10.30.111.165:8086' for the HTTP URL. Exclude the Auth and Custom HTTP Headers sections. In the 'InfluxDB Details,' include the 'mdt-db' database. Lastly, click 'Save & Test.'</p>

![source-detail1.png]({{site.baseurl}}/images/source-detail1.png)

![source-detail2.png]({{site.baseurl}}/images/source-detail2.png)

<p align="justify">The message 'Datasource is working. 1 measurement found' confirms that Grafana has successfully established a connection to the 'mdt-db' database within InfluxDB.</p>

Next, a dashboard will be created through the subsequent steps:

![homepage.png]({{site.baseurl}}/images/homepage.png)

<p align="justify">Access the 'Home' page and select the 'Dashboards' widget. Click on 'Add visualization', select the data source (in this case, 'mdt-influx'), and the subsequent page will appear.</p>

![add-visual.png]({{site.baseurl}}/images/add-visual.png)

![select-data-source.png]({{site.baseurl}}/images/select-data-source.png)

<p align="justify">Select 'select measurement', and the sensor-path configured using the sensor-group will be visible. Choose that sensor-path.</p>

![select-measurement.png]({{site.baseurl}}/images/select-measurement.png)

Finally, click on 'field(value)', and opt for 'bytes-received'. This action will generate the graph at the top.

![initial-graph.png]({{site.baseurl}}/images/initial-graph.png)

![final-graph.png]({{site.baseurl}}/images/final-graph.png)

Congratulations! An MDT has been successfully established using the dial-out method.

# TCP Dial-out

<p align="justify">TCP Dial-out closely resembles grpc Dial-out, requiring just two configuration adjustments in the previously outlined grpc dial-out setup.
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

# Conclusion

<p align="justify"> In this article, you gained insights into TCP and gRPC Dial-out MDT by employing the TIG stack as a collector. Please feel free to experiment with various grafana queries, and observe the graph adjustments corresponding to query modifications.</p>
