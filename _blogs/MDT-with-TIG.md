---
published: false
date: '2023-08-21 14:13 -0400'
title: MDT-with-TIG
position: hidden
---
# Background

This article focuses on  establishing Model-Driven Telemetry (MDT) using the Telegraf, InfluxDB, and Grafana (TIG) stack.

If you are well-verse with MDT, YANG models and sensor-group, destination group and subscription, please follow the steps to establish MDT. Otherwise you might want to go through this post first. 

MDT consists of two main components. Firstly, there's a network device (such as a router) equipped with pre-installed YANG models and a running grpc server. Through MDT, this network device continuously streams data at specified time intervals. Secondly, the TIG stack comes into play, which consumes, stores, and presents this data visually.

Let's delve into the individual elements of the TIG stack:

1. **Telegraf** : This tool gathers data from the network device and stores it in the InfluxDB time series database.

2. **InfluxDB** : Responsible for storing the data received from Telegraf, InfluxDB then forwards it to Grafana. Grafana utilizes this data to create visual representations.

3. **Grafana** : By retrieving data from InfluxDB, Grafana generates various types of visualizations, including graphs, pie charts, and statistics.

We will now establish a **Dial-out** MDT where in network-device initiates a grpc channel with collector (here, telegraf).

Since this is Dial-out MDT, the collecor should be up and running so that it is ready to collect the data when network-device initiates a grpc channel. So we'll first configure the collector side.

In order to configure TIG, we will use the following steps:

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

**transport = "grpc"**: This line specifies the transport protocol to be used for data reception. In this case, the "grpc" protocol is selected. gRPC is a modern and efficient protocol for remote procedure calls and communication between systems.

**service_address = ":57100"**: This line defines the service address where the incoming data will be received. It signifies that Telegraf will listen on port number "57100" for incoming data using the specified transport protocol. The colon (":") before the port number indicates that Telegraf will listen on all available network interfaces.

**outputs.influxdb** indicates that you're defining an output configuration for the InfluxDB plugin within the 'outputs' section. It means that the collected data will be sent to InfluxDB.

**urls = ["http://influxdb:8086"]'** : This line specifies the URLs of the InfluxDB instances to which the data will be sent. In this case, the data will be sent to an InfluxDB instance located at "http://influxdb" and listening on port "8086". This is the destination where Telegraf will send the data.

**database = "mdt-db"** : Here, you are specifying the name of the InfluxDB database where the data will be stored. In this case, the data will be stored in a database named "mdt-db".


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
      - INFLUXDB_DB=telemetry
      - INFLUXDB_ADMIN_USER=admin
      - INFLUXDB_ADMIN_PASSWORD=admin
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


Step3: Go to the folder where you have both these files and use the following command to spin up containers.

```
docker-compose up
```

At this point, your collector is ready to collect data once network-device starts streaming it.

Now we will configure the network-device to start streaming the data.

Step1: Create a destination-group


```
RP/0/RP0/CPU0:ios(config)# telemetry model-driven
RP/0/RP0/CPU0:ios(config-model-driven)# destination-group DGroup1
RP/0/RP0/CPU0:ios(config-model-driven-dest)#  address family ipv4 10.30.111.165 port 57100  
RP/0/RP0/CPU0:ios(config-model-driven-dest-addr)#   encoding self-describing-gpb  
RP/0/RP0/CPU0:ios(config-model-driven-dest-addr)#   protocol grpc no-tls  
RP/0/RP0/CPU0:ios(config-model-driven-dest-addr)# commit   
```

Step2: Create a sensor-group

```
RP/0/RP0/CPU0:ios(config)#telemetry model-driven
RP/0/RP0/CPU0:ios(config-model-driven)#sensor-group SGroup1
RP/0/RP0/CPU0:ios(config-model-driven-snsr-grp)# sensor-path Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
RP/0/RP0/CPU0:ios(config-model-driven-snsr-grp)# commit
```

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




