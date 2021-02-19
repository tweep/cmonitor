# Monitoring containers with cmon 

To monitor container metrics ( cpu, memory, disk ) via grafana, we need a few things to set up. 

1) modify runWorker.pl 
A modified version is in `docker/shared`. The modified version starts the `cmonitor_collector` in the background
and once the parent (perl) processes are done doing the work, the process is terminated. 

There are many reasons we do it this way : 
- we can't run daemons in docker. Why ? Docker containers only exit if all processes are finished - so if you 
  run a process in the background in docker, the container will stay around, kind of forever and not exit. 

- the ensembl ehive container has a `simple_init` entrypoint. This little routine is neeed if you run processes 
  in the LOCAL queue. The routine waits until all submitted LOCAL child processes are done, then it exists. 
  Without this routine, the beekeper submits jobs and exits, and all child LOCAL processes will be killed. 

- If you try to run the cmonitor in the background, the `simple_init` process above will keep the 
  container running forever. We don't like this. 

- We only want to monitor the runWorker.pl in aWS, not the init / seed steps.



## Requirements 
- EC2 instance with influxDB
  [Installation instructions](https://docs.influxdata.com/influxdb/v1.8/introduction/install/) 
- cmonitor monitoring tool   


# Building cmonitor  
Fire up your ubuntu ehive docker container, build the cmonitor tool in it 
and copy the binaries out of the container. 

Then copy them into your `docker/bin` folder, so you can just start them 
when your contaier starts.

## Build in ubuntu ehive contaienr 

```
  docker run -it -v "/Users/vogelj4/:/tmp" ehive-sentieon-jg:latest  

  git clone https://github.com/f18m/cmonitor.git 
  cd cmonitor 
  make  
  cp src/cmonitor_collector  /tmp  
  exit
```


## Setup cmonitor in dockerfile  
Sample data every 3 seconds and set it to a remote InfluxDB server 

Your dockerfile could look like this: 

``` 
  COPY cmonitor_collector /usr/bin/    

  CMD /usr/bin/cmonitor_collector --remote-ip=10.158.141.220 --remote-port=8086 

  CMD /usr/bin/cmonitor_collector \
      --sampling-interval=3 \
      --output-filename=mycontainer.json \
      --output-directory /perf ; \
    myapplication
```

## Install grafana 


```
sudo yum install grafana
sudo systemctl start influxdb && systemctl enable influxdb
```
 

## Install grafana on centos AWS EC2 
Notes here : https://computingforgeeks.com/how-to-install-grafana-on-centos-7/

```
ssh -i ehive-jhv.pem centos@10.158.141.220 

wget https://dl.grafana.com/oss/release/grafana-7.4.2-1.x86_64.rpm
sudo yum install grafana-7.4.2-1.x86_64.rpm  
sudo yum install fontconfig
sudo yum install freetype*
sudo yum install urw-fonts
``` 

## Start grafana on centos AWS EC2  

``` 
sudo systemctl enable --now grafana-server 
sudo systemctl enable grafana-server.service

systemctl status  grafana-server  
sudo grafana-cli plugins install alexanderzobnin-zabbix-app
sudo systemctl restart grafana-server


``` 

## Modify Firewall of EC2 instance

``` 
sudo yum install firewalld 
sudo systemctl start firewalld
sudo firewall-cmd --zone=public --add-port=3000/tcp --permanent  
sudo firewall-cmd --add-port=3000/tcp --permanent
sudo firewall-cmd --reload

sudo systemctl status firewalld  


``` 

# Check that docker container can push to InfluxDB  

Debug access to influxDB from Macbook / other servers. 
Somehow i cant' connect andi don't see new data arrving from my containers.

```
  nc -zv 10.158.141.220 8086 

  nc: connectx to 10.158.141.220 port 8086 (tcp) failed: Connection refused 

  # Is this the AWS EC2 security groups or the host firewall ? 
  # LEt's stop the firewall on host and find out 

  sudo firewall-cmd --state 
  sudo systemctl stop firewalld
  sudo firewall-cmd --state

  nc -zv 10.158.141.220 8086 

  NOTE: make sure you dont forget the "--permament" flag, otherwise your rule gets lost when re-loading !!!!

  WORKS ! So the shitty firewall is not set up correctly. 


  ssh -i ehive-jhv.pem centos@10.158.141.220 
  sudo firewall-cmd --add-port=8086/tcp --permanent
  sudo firewall-cmd --zone=public --add-port=8086/tcp --permanent    

  sudo firewall-cmd --zone=public --permanent --list-ports

  sudo firewall-cmd --reload
  sudo systemctl start firewalld
  sudo systemctl status firewalld  
```

## Grafana
Also change your security groups in EC2. Grafana should be running here : 

http://10.158.141.220:3000/?orgId=1  

Username/login: admin/admin . 

- Access ui, go to plugins, enable Zabbix plugin



``` 

## Debug - try to connect  
Try from inside the host and from outside (mac) :
``` 
nc -v  10.158.141.220 3000 

```


## Prepare influxdb  

```
influx
show databases 
drop DATABASE cmonitor;  
CREATE DATABASE cmonitor;  
use cmonitor;  
show series;  
show users ; 
SELECT * FROM "stat_cpu0" LIMIT 20' 
CREATE USER gneadmin WITH PASSWORD 'grafana' WITH ALL PRIVILEGES
```


Explore influxDB's cmon schema 


SELECT * FROM "stat_cpu0"
SHOW RETENTION POLICIES ; 
show series ;  ! All series is listed in the first ROW. 

select * from proc_meminfo ;  

Queris  for dashboard; 


select MemFree from proc_meminfo ; 



