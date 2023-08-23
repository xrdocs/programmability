---
published: true
date: '2023-08-23 17:09 -0400'
title: Dial-in Model-Driven Telemetry with TIG Stack
position: hidden
author: Rahul Sharma
---
{% include toc icon="table" title="ON THIS PAGE" %}

# Background

This article focuses on  establishing Model-Driven Telemetry (MDT) using the Telegraf, InfluxDB, and Grafana (TIG) stack.

If you are well-verse with MDT, YANG models and sensor-group, destination group and subscription, please follow the steps to establish MDT. Otherwise you might want to go through this post first. 

MDT consists of two main components. Firstly, there's a router equipped with pre-installed YANG models and a running grpc server. Through MDT, this router continuously streams mtetrics at specified time intervals. Secondly, the TIG stack, which consumes, stores, and presents this metrics visually.

Let's delve into the individual elements of the TIG stack:

# TIG Stack

1. **Telegraf** :Collects router metrics and stores them in a Time Series Database (TSDB).

2. **InfluxDB** :TSDB that stores metrics from Telegraf. Forwards data to Grafana for visualization.

3. **Grafana** :Receives metrics from InfluxDB and displays them visually (graphs, charts) for analysis.


# gRPC Dial-in

The objective is to create a Dial-in MDT, wherein a collector initiates a grpc channel with a router.

As this is a Dial-in MDT, the router must be operational and prepared to receive a request for gRPC channel by a collector.In short, gRPC server on the router must be up and running.

The initial focus will be on configuring the router aspect.

## Router Configuration

In order for router to receive a grpc channel creation request, the gRPC server should be up and running. You can configure grpc on the router using following set of commands:

```

```

Step4
​
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
​
Step5: Check if data is being streamed to the collector or not.
​
```
RP/0/RP0/CPU0:ios#show telemetry model-driven subscription
Thu Aug 21 11:27:27.751 UTC
Subscription:  Sub1                     State: ACTIVE
-------------
  Sensor groups:
  Id                Interval(ms)        State
  SGroup1           30000               Resolved
​
  Destination Groups:
  Id                Encoding            Transport   State   Port    IP
  DGroup1           self-describing-gpb grpc        Active  57100   10.30.111.165
```
​
The 'Resolved' state indicates that the router successfully identified and resolved the sensor-path specified within the sensor-group.
​
For further insights into this subscription, utilize the following command:
​
```
RP/0/RP0/CPU0:ios#show telemetry model-driven subscription sub1
```
​
The status of this subscription is observed as 'ACTIVE,' indicating that the router is actively streaming data, and the collector (Telegraf) is effectively collecting the data.
​

## Collector configuration

To establish the TIG configuration, the subsequent topology and steps will be used:

![MDT-TIG.png]({{site.baseurl}}/images/MDT-TIG.png)

Step1: Create a following telegraf.conf file.

```
[[inputs.cisco_telemetry_mdt]]
   transport = "grpc"
   service_address = ":57100

[[outputs.influxdb]]

  urls = ["http://influxdb:8086"]
  database = "mdt-db"
  
```
Here is the breakdown of this configuration file:

**inputs.cisco_telemetry_mdt**: This indicates that you're configuring the input section specifically for the 'cisco_telemetry_mdt' plugin. It signifies that Telegraf will be set up to receive data using this plugin.

- **transport = "grpc"**: This line specifies the transport protocol to be used for data reception. In this case, the "grpc" protocol is selected. gRPC is a modern and efficient protocol for remote procedure calls and communication between systems.

- **service_address = ":57100"**: This line defines the service address where the incoming data will be received. It signifies that Telegraf will listen on port number "57100" for incoming data using the specified transport protocol. The colon (":") before the port number indicates that Telegraf will listen on all available network interfaces.

**outputs.influxdb** indicates that you're defining an output configuration for the InfluxDB plugin within the 'outputs' section. It means that the collected data will be sent to InfluxDB.

- **urls = ["http://influxdb:8086"]'** : This line specifies the URLs of the InfluxDB instances to which the data will be sent. In this case, the data will be sent to an InfluxDB instance located at "http://influxdb" and listening on port "8086". This is the destination where Telegraf will send the data.

- **database = "mdt-db"** : Here, you are specifying the name of the InfluxDB database where the data will be stored. In this case, the data will be stored in a database named "mdt-db".


Step2: Create a following docker-compose.yml file. 
```
version: '3.6'
services:
  telegraf:
    image: telegraf
    container_name: telegraf
    restart: always
    volumes:
    - ./telegraf.conf:/etc/telegraf/telegraf.conf:ro
    depends_on:
      - influxdb
    links:
      - influxdb
    ports:
    - '57100:57100'
       
  influxdb:
    image: influxdb:1.8-alpine
    container_name: influxdb
    restart: always
    environment:
      - INFLUXDB_DB=
      - INFLUXDB_ADMIN_USER=
      - INFLUXDB_ADMIN_PASSWORD=
    ports:
      - '8086:8086'
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

```

Here is the breakdown for this file:

**version: '3.6'**: Specifies the version of the Docker Compose file being used.

**services**: Defines the individual Docker services to be deployed.

- **telegraf**: Specifies the Telegraf service configuration. Here, path of the telegraf.conf is relative docker-compose.yml. In this case, both these files are in the same folder.

- **image: telegraf**: Specifies the Docker image to be used for the Telegraf container.
- **container_name: telegraf**: Names the container as "telegraf".
- **restart: always**: Ensures the Telegraf container restarts automatically if it crashes.
- **volumes**: Mounts the telegraf.conf configuration file into the container.
- **depends_on**: Indicates that this service depends on the "influxdb" service to be running.
- **links**: Creates a link to the "influxdb" service.
- **ports**: Maps port "57100" on the host to port "57100" on the container.

**influxdb**: Specifies the InfluxDB service configuration.

- Similar to the telegraf service, it defines the InfluxDB container configuration.
- Sets environment variables for the InfluxDB database name, admin username, and password.
- Maps port "8086" on the host to port "8086" on the container.
- Mounts a volume for storing InfluxDB data.

**grafana**: Specifies the Grafana service configuration.

- Similar to the other services, it defines the Grafana container configuration.
- Sets environment variables for the Grafana admin username and password.
- Maps port "3000" on the host to port "3000" on the container.
- Mounts a volume for storing Grafana data.

**volumes**: Defines named volumes for persisting data between container restarts.


Step3: Navigate to the directory containing both files and utilize the provided command to initiate container deployment.

```
docker-compose up
```

At this stage, the collector is set to gather metrics when the network device initiates streaming. The following command can be executed to check the status of containers

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

Next, the configuration process involves enabling the network device to initiate data streaming.

## Router Configuration

Step1: Create a destination-group

```
RP/0/RP0/CPU0:ios(config)# telemetry model-driven
RP/0/RP0/CPU0:ios(config-model-driven)# destination-group DGroup1
RP/0/RP0/CPU0:ios(config-model-driven-dest)#  address family ipv4 10.30.111.165 port 57100  
RP/0/RP0/CPU0:ios(config-model-driven-dest-addr)#   encoding self-describing-gpb  
RP/0/RP0/CPU0:ios(config-model-driven-dest-addr)#   protocol grpc no-tls  
RP/0/RP0/CPU0:ios(config-model-driven-dest-addr)# commit   
```
Remember that the IP address in the destination-group pertains to the server hosting Telegraf, and the port number should align with what's specified in the 'telegraf.conf' file.

Additionally, pay attention to the 'protocol grpc no-tls' configuration, as it reflects the usage of grpc dial-out MDT.

Step2: Create a sensor-group

```
RP/0/RP0/CPU0:ios(config)#telemetry model-driven
RP/0/RP0/CPU0:ios(config-model-driven)#sensor-group SGroup1
RP/0/RP0/CPU0:ios(config-model-driven-snsr-grp)# sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
RP/0/RP0/CPU0:ios(config-model-driven-snsr-grp)# commit
```
Additional sensor-paths can be incorporated into this sensor-group at a later stage, facilitating the reception of corresponding streamed data by the collector.

Interested in discovering the accurate sensor path for a desired metric to stream? Delve into the provided article, which elucidates the process of locating a sensor-path for a CLI.

Step3: Create a subscription

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

The 'Resolved' state indicates that the router successfully identified and resolved the sensor-path specified within the sensor-group.

For further insights into this subscription, utilize the following command:

```
RP/0/RP0/CPU0:ios#show telemetry model-driven subscription sub1
```

The status of this subscription is observed as 'ACTIVE,' indicating that the router is actively streaming data, and the collector (Telegraf) is effectively collecting the data.

## InfluxDB and Grafana

Given that Telegraf stores data in InfluxDB, we will validate this by querying the database. Execute the provided command on the Ubuntu server where your containers are operational.

```
docker exec -it influxdb bash
```

Executing this command will lead you to the bash shell within the InfluxDB container. 

Upon accessing the container's bash shell, database commands can be employed to identify the sensor-path configured within the sensor-group.

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

The presence of the 'mdt-db' database within InfluxDB is apparent. This aligns with the database mentioned in the 'telegraf.conf' file within the output section.


The following sensor-path as a measurement above shows that InfluxDB successfully stores the streamed data from our NCS 5500 device.

```
Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
```

Next, we'll create graphs utilizing InfluxDB data on Grafana.

To access the Grafana dashboard, enter 'http://10.30.111.165:3000/' into your browser's URL field. Keep in mind that this IP address corresponds to the server running Grafana, followed by the relevant port number. Your configuration might have a different address and port.

![login.png]({{site.baseurl}}/images/login.png)

Upon accessing the Grafana dashboard, a login prompt will appear, requiring a username and password. Use the credentials specified in the 'docker-compose.yml' file under the 'environment' section of the 'Grafana' service. In this case, both the username and password are 'admin'.

![login-creds.png]({{site.baseurl}}/images/login-creds.png)

Utilize the provided username and password. You'll be prompted to update your password; however, you can select 'skip' to proceed to the home page.

![homepage.png]({{site.baseurl}}/images/homepage.png)

Now, let's set up the data source for Grafana. To achieve this, click on the 'Add your first data source' widget. Then, apply the filter to locate 'InfluxDB' and proceed to select it.

![add-source.png]({{site.baseurl}}/images/add-source.png)

Assign a name of your choice to the source. Opt for 'InfluxQL' as the query language. For the HTTP URL, input 'http://10.30.111.165:8086'. Leave out the Auth and Custom HTTP Headers sections. In the 'InfluxDB Details', add the 'mdt-db' database. Finally, click on 'Save & Test'.

![source-detail1.png]({{site.baseurl}}/images/source-detail1.png)

![source-detail2.png]({{site.baseurl}}/images/source-detail2.png)

The message 'Datasource is working. 1 measurement found' confirms that Grafana has successfully established a connection to the 'mdt-db' database within InfluxDB.

Next, a dashboard will be created through the subsequent steps:

![homepage.png]({{site.baseurl}}/images/homepage.png)

Access the 'Home' page and select the 'Dashboards' widget. Click on 'Add visualization', pick your data source (in this case, 'mdt-influx'), and the subsequent page will appear.

![add-visual.png]({{site.baseurl}}/images/add-visual.png)

![select-data-source.png]({{site.baseurl}}/images/select-data-source.png)

Select 'select measurement', and the sensor-path configured using the sensor-group will be visible. Choose that sensor-path

![select-measurement.png]({{site.baseurl}}/images/select-measurement.png)

Then, click on 'field(value)', and opt for 'bytes-received'. This action will generate the graph at the top.

![initial-graph.png]({{site.baseurl}}/images/initial-graph.png)

![final-graph.png]({{site.baseurl}}/images/final-graph.png)

Feel free to experiment with this query, and you'll observe the graph corresponding to your query modifications.


# TCP Dial-out

TCP dial-out closely resembles grpc dial-out, requiring just two configuration adjustments in the previously outlined grpc dial-out setup to enable TCP dial-out.

Adjustment 1: Modify the transport in the 'input' plugin from 'grpc' to 'tcp' as demonstrated below:

```
[[inputs.cisco_telemetry_mdt]]
   transport = "tcp"
   service_address = ":57100

[[outputs.influxdb]]

  urls = ["http://influxdb:8086"]
  database = "mdt-db"
  
```

Adjustment 2: Modify the transport in 'destination-group' from 'grpc no-tls' to 'tcp' as demonstrated below:

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

Upon implementing these modifications and executing the 'show telemetry model-driven subscription' command on the router, the resulting output will be as follows:

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

Consequently, this concludes the article, wherein you gained insights into TCP and gRPC Dial-out MDT by employing the TIG stack as a collector.
