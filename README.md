# About this repo

collect data from Junos devices using SNMP with telegraf.  
store collected data in influxdb  
query influxdb database with cli and python to extract data   

In this demo we will use SNMPv2 to collect on Junos devices (100.123.1.0, 100.123.1.1, 100.123.1.2) these details: 
- Device `hostname` 
- BGP `bgpPeerState` and `bgpPeerRemoteAddr`. With a filter on some neighbors (192.168.*)
- Interface `ifName`, `ifHCInOctets`, `ifHCOutOctets`. With a filter on some interfaces (ge-0/0/0, ge-0/0/1, ge-0/0/2, ge-0/0/3) 

# About telegraf

telegraf is an open source collector written in GO.  
Telegraf collects data and writes them into a database.  
It is plugin-driven (it has input plugins, output plugins, ...)  

# About influxdb

influxdb is an open source time series database written in GO.  

# Requirements 

## Junos configuration

here's the SNMP configuration on junos devices   
```
jcluser@vMX-addr-0> show configuration snmp
community public;
```
## Docker

### Install Docker on ubuntu

Check if Docker is already installed
```
$ docker --version
```

If it was not already installed, install it:
```
$ sudo apt-get update
```
```
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
```
```
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```
```
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```
```
$ sudo apt-get update
```
```
$ sudo apt-get install docker-ce
```
```
$ sudo docker run hello-world
```
```
$ sudo groupadd docker
```
```
$ sudo usermod -aG docker $USER
```

Exit the ssh session and open an new ssh session and run these commands to verify you installed Docker properly:  
```
$ docker run hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/engine/userguide/
```
```
$ docker --version
Docker version 18.03.1-ce, build 9ee9f40
```


# Influxdb

pull docker images 
```
$ docker pull influxdb
```
Verify
```
$ docker images influxdb
```
Instanciate an influxdb container
```
$ docker run -d --name influxdb -p 8083:8083 -p 8086:8086 influxdb
```
Verify
```
$ docker ps | grep influxdb
```
for troubleshooting purposes you can run this command
```
$ docker logs influxdb
```
start a shell session in the influxdb container
```
$ docker exec -it influxdb bash
```
run this command to read the influxdb configuration file
```
# more /etc/influxdb/influxdb.conf
```
create a user and a database
```
# influx
Connected to http://localhost:8086 version 1.7.0
InfluxDB shell version: 1.7.0
Enter an InfluxQL query
> CREATE DATABASE juniper
> show databases
name: databases
name
----
_internal
juniper
>  CREATE USER "juniper" WITH PASSWORD 'juniper'
> show users
user   admin
----   -----
influx false
> exit
# 
```
exit the influxdb container
```
# exit
```

# Telegraf

get ip address used by containers
```
$ ifconfig docker0
```

pull docker images 
```
$ docker pull telegraf
```
Verify
```
$ docker images telegraf
```
create a telegraf configuration file ([use this file](telegraf.conf))  
it will use SNMP input plugin and influxb output plugin
SNMP will be used to collect interfaces details and BGP details. 
Influxdb is database to store the data collected 

```
$ vi telegraf.conf
```
instanciate a telegraf container
```
$ docker run --name telegraf -d -v $PWD/telegraf.conf:/etc/telegraf/telegraf.conf:ro telegraf
```
verify
```
$ docker ps | grep telegraf
```
for troubleshooting purposes you can run this command
```
$ docker logs telegraf
```
start a shell session in the telegraf container
```
$ docker exec -it telegraf bash
```
verify the telegraf configuration file
```
# more /etc/telegraf/telegraf.conf
```
exit the telegraf container
```
# exit
```
# query the influxdb database with cli to verify
start a shell session in the influxdb container
```
$ docker exec -it influxdb bash
```
query the database
```
# influx
Connected to http://localhost:8086 version 1.7.0
InfluxDB shell version: 1.7.0
Enter an InfluxQL query
> show databases
name: databases
name
----
_internal
juniper
> use juniper
Using database juniper
> show measurements
name: measurements
name
----
vMX_BGP
vMX_interfaces
>
```
```
> select * from "vMX_interfaces" order by desc limit 12
name: vMX_interfaces
time                agent_host  host         hostname   ifHCInOctets ifHCOutOctets ifName
----                ----------  ----         --------   ------------ ------------- ------
1544801695000000000 100.123.1.2 803d485462fa vMX-addr-2 653842       899652        ge-0/0/0
1544801695000000000 100.123.1.2 803d485462fa vMX-addr-2 654685       900070        ge-0/0/3
1544801695000000000 100.123.1.2 803d485462fa vMX-addr-2 654482       899569        ge-0/0/2
1544801695000000000 100.123.1.2 803d485462fa vMX-addr-2 654763       898987        ge-0/0/1
1544801695000000000 100.123.1.1 803d485462fa vMX-addr-1 760903       1006250       ge-0/0/0
1544801695000000000 100.123.1.1 803d485462fa vMX-addr-1 654603       899057        ge-0/0/3
1544801695000000000 100.123.1.1 803d485462fa vMX-addr-1 761474       1006808       ge-0/0/2
1544801695000000000 100.123.1.1 803d485462fa vMX-addr-1 761948       1006178       ge-0/0/1
1544801695000000000 100.123.1.0 803d485462fa vMX-addr-0 761686       1007986       ge-0/0/3
1544801695000000000 100.123.1.0 803d485462fa vMX-addr-0 654786       900235        ge-0/0/2
1544801695000000000 100.123.1.0 803d485462fa vMX-addr-0 760969       1007085       ge-0/0/1
1544801695000000000 100.123.1.0 803d485462fa vMX-addr-0 761056       1006102       ge-0/0/0
>
```
```
> select * from "vMX_BGP" order by desc limit 12
name: vMX_BGP
time                agent_host  bgpPeerRemoteAddr bgpPeerState host         hostname
----                ----------  ----------------- ------------ ----         --------
1544801015000000000 100.123.1.2 192.168.3.7       6            db58b9842e56 vMX-addr-2
1544801015000000000 100.123.1.2 192.168.3.5       6            db58b9842e56 vMX-addr-2
1544801015000000000 100.123.1.2 192.168.3.3       6            db58b9842e56 vMX-addr-2
1544801015000000000 100.123.1.2 192.168.3.1       6            db58b9842e56 vMX-addr-2
1544801015000000000 100.123.1.1 192.168.2.7       6            db58b9842e56 vMX-addr-1
1544801015000000000 100.123.1.1 192.168.2.5       6            db58b9842e56 vMX-addr-1
1544801015000000000 100.123.1.1 192.168.2.3       6            db58b9842e56 vMX-addr-1
1544801015000000000 100.123.1.1 192.168.2.1       6            db58b9842e56 vMX-addr-1
1544801015000000000 100.123.1.0 192.168.1.7       6            db58b9842e56 vMX-addr-0
1544801015000000000 100.123.1.0 192.168.1.5       6            db58b9842e56 vMX-addr-0
1544801015000000000 100.123.1.0 192.168.1.3       6            db58b9842e56 vMX-addr-0
1544801015000000000 100.123.1.0 192.168.1.1       6            db58b9842e56 vMX-addr-0
>

```
exit 
```
> exit
#
```
exit the influxdb container
```
# exit 
```
# query the influxdb database with python to verify
install the influxdb python library.
This python library is a client for interacting with InfluxDB.
```
$ pip install influxdb
```
Verify
```
$ pip list | grep influx
```
you can now interact with InfluxDB using Python

run this command to start a python interactive session

```
$ python
```
connect to InfluxDB using Python
```
>>> from influxdb import InfluxDBClient
>>> influx_client = InfluxDBClient('localhost',8086)
```
list the databases
```
>>> influx_client.query('show databases')
ResultSet({'(u'databases', None)': [{u'name': u'_internal'}, {u'name': u'juniper'}]})
```
list measurements for a database
```
>>> influx_client.query('show measurements', database='juniper')
ResultSet({'(u'measurements', None)': [{u'name': u'vMX_BGP'}, {u'name': u'vMX_interfaces'}]})
```
query data from a particular measurement and database
```
>>> gp = influx_client.query('select * from "vMX_interfaces" order by desc limit 3', database='juniper').get_points()
>>> for item in gp:
...     print item['/interfaces/interface/@name']
...
ge-0/0/7
ge-0/0/6
>>> gp = influx_client.query('select * from "/interfaces/"  order by desc limit 2 ', database='juniper').get_points()
>>> for item in gp:
...     print item['/interfaces/interface/subinterfaces/subinterface/state/counters/in-octets']
...
36
26277618
>>>

