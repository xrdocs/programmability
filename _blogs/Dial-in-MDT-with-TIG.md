---
published: true
date: '2023-08-23 17:09 -0400'
title: Dial-in Model-Driven Telemetry with TIG Stack
position: hidden
author: Rahul Sharma
---
{% include toc icon="table" title="ON THIS PAGE" %}

# Background

<p align="justify">This article focuses on  establishing Model-Driven Telemetry (MDT) using the Telegraf, InfluxDB, and Grafana (TIG) stack.
<br>
<br>  
If you are well-verse with MDT, YANG models and sensor-group, destination group and subscription, please follow the steps to establish MDT. Otherwise you might want to go through this post first. 
<br>
<br>  
MDT consists of two main components. Firstly, there's a router equipped with pre-installed YANG models and a running grpc server. Through MDT, this router continuously streams mtetrics at specified time intervals. Secondly, the TIG stack, which consumes, stores, and presents this metrics visually.
<br>
<br>  
Let's delve into the individual elements of the TIG stack:</p>

# TIG Stack

<p align="justify">1. <b>Telegraf</b> :Collects router metrics and stores them in a Time Series Database (TSDB).
<br>
<br>  
2. <b>InfluxDB</b> :TSDB that stores metrics from Telegraf. Forwards data to Grafana for visualization.
<br>
<br>  
3. <b>Grafana</b> :Receives metrics from InfluxDB and displays them visually (graphs, charts) for analysis.</p>


# gRPC Dial-in

<p align="justify">The objective is to create a Dial-in MDT, wherein a collector initiates a grpc channel with a router.
<br>
<br>  
As this is a Dial-in MDT, the router must be operational and prepared to receive a request for gRPC channel by a collector.In short, gRPC server on the router must be up and running.
<br>
<br>  
The initial focus will be on configuring the router aspect.</p>

## Router Configuration

<p align="justify">In order for router to receive a grpc channel creation request, the gRPC server should be up and running. You can configure grpc on the router using following set of commands:</p>
  
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

## Collector configuration
<p align="justify">To establish the TIG configuration, the subsequent topology and steps will be used:</p>

![MDT-TIG.png]({{site.baseurl}}/images/MDT-TIG.png)

Step1: Create a following telegraf.conf file.

```
[[inputs.gnmi]]
   addresses = ["10.30.111.168:57100"]
   username = "cisco"
   password = "cisco123!"

[[inputs.gnmi.subscription]]
   name = "ifcounters"
   origin = "Cisco-IOS-XR-infra-statsd-oper"
   path = "/infra-statistics/interfaces/interface/latest/generic-counters"
   subscription_mode = "sample"
   sample_interval = "30s"

[[outputs.influxdb]]

   urls = ["http://localhost:8086"]
   database = "mdt-db"
```
Here is the breakdown of this configuration file:

<p align="justify"><b>[[inputs.gnmi]]:</b> This section defines a configuration for gathering data using the gNMI (gRPC Network Management Interface) protocol. gNMI is typically used for network device management and telemetry.
<br>
<br>  
- <b>addresses</b>: Specifies the IP address and port (57100) of the gNMI server on the network device.
  <br>
  <br> 
- <b>username</b>: The username to authenticate with when connecting to the gNMI server.
  <br>
  <br> 
- <b>password</b>: The password associated with the provided username for authentication.
  <br>
  <br> 
<b>[[inputs.gnmi.subscription]]</b>: This section configures a specific subscription for telemetry data from the network device. It defines what data to retrieve, how often to sample it, and the subscription mode.
  <br>
  <br> 
- <b>name</b>: A user-defined name for this subscription configuration, in this case, "ifcounters".
  <br>
- <b>origin</b>: YANG Model name (in this case, "Cisco-IOS-XR-infra-statsd-oper").<br>
- <b>path</b>: Path from root of the YANG to the leaf that collector would request from router.<br>
- <b>subscription_mode</b>: Sets the subscription mode. In this case, it's set to "sample", indicating that data should be sampled periodically.(one of: "target_defined", "sample", "on_change")<br>
- <b>sample_interval</b>: Specifies the interval at which data should be sampled, here set to "30s" (30 seconds).<br>
<b>[[outputs.influxdb]]</b>: This section configures the output destination for the collected data, using the InfluxDB database system. InfluxDB is commonly used for time-series data storage.<br>

- <b>urls</b>: Specifies the URLs of the InfluxDB instances to which the data will be sent. In this case, it's a single URL - "http://localhost:8086".
<br>
- <b>database</b>: Specifies the name of the InfluxDB database where the collected data will be stored, here named "mdt-db".
<br>
Step2: Create a following docker-compose.yml file.</p> 
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
<p align="justify"><b>version: '3.6'</b>: Specifies the version of the Docker Compose file being used.
<br>
<br>  
<b>services</b>: Defines the individual Docker services to be deployed.
<br>
<br>  
- <b>telegraf</b>: Specifies the Telegraf service configuration. Here, path of the telegraf.conf is relative docker-compose.yml. In this case, both these files are in the same folder.
<br>
<br>
- <b>image: telegraf</b>: Specifies the Docker image to be used for the Telegraf container.<br>
- <b>container_name: telegraf</b>: Names the container as "telegraf".<br>
- <b>restart: always</b>: Ensures the Telegraf container restarts automatically if it crashes.<br>
- <b>volumes</b>: Mounts the telegraf.conf configuration file into the container.<br>
- <b>depends_on</b>: Indicates that this service depends on the "influxdb" service to be running.<br>
- <b>links</b>: Creates a link to the "influxdb" service.<br>
- <b>ports</b>: Maps port "57100" on the host to port "57100" on the container.<br>
<br>
<br>  
<b>influxdb</b>: Specifies the InfluxDB service configuration.
<br>
<br>  
- Similar to the telegraf service, it defines the InfluxDB container configuration.
- Sets environment variables for the InfluxDB database name, admin username, and password.
- Maps port "8086" on the host to port "8086" on the container.
- Mounts a volume for storing InfluxDB data.
<br>
<br>
<b>grafana</b>: Specifies the Grafana service configuration.
<br>
<br>
- Similar to the other services, it defines the Grafana container configuration.
- Sets environment variables for the Grafana admin username and password.
- Maps port "3000" on the host to port "3000" on the container.
- Mounts a volume for storing Grafana data.
<br>
<br>
<b>volumes</b>: Defines named volumes for persisting data between container restarts.
<br>
<br>  
Step3: Navigate to the directory containing both files and utilize the provided command to initiate container deployment.</p>

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

<p align="justify">Next, the configuration process involves enabling the network device to initiate data streaming.
<br>
<br>  
In order to check if MDT is working or not, execute the following command on the router.</p>

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


<p align="justify">The following sensor-path as a measurement above shows that InfluxDB successfully stores the streamed data from our NCS 5500 device.</p></p>

```
Cisco-IOS-XR-infra-statsd-oper:infra-statistics/interfaces/interface/latest/generic-counters
```

<p align="justify">Next, we'll create graphs utilizing InfluxDB data on Grafana.</p>

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

<p align="justify">Next, a dashboard will be created through the subsequent steps:</p>

![homepage.png]({{site.baseurl}}/images/homepage.png)

<p align="justify">Access the 'Home' page and select the 'Dashboards' widget. Click on 'Add visualization', pick your data source (in this case, 'mdt-influx'), and the subsequent page will appear.</p>

![add-visual.png]({{site.baseurl}}/images/add-visual.png)

![select-data-source.png]({{site.baseurl}}/images/select-data-source.png)

<p align="justify">Select 'select measurement', and the sensor-path configured using the sensor-group will be visible. Choose that sensor-path.</p>

![select-measurement.png]({{site.baseurl}}/images/select-measurement.png)

<p align="justify">Then, click on 'field(value)', and opt for 'bytes-received'. This action will generate the graph at the top.</p>

![initial-graph.png]({{site.baseurl}}/images/initial-graph.png)

![final-graph.png]({{site.baseurl}}/images/final-graph.png)

<p align="justify">Feel free to experiment with this query, and you'll observe the graph corresponding to your query modifications.</p>
