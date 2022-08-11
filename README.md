# OpenBMP with Juniper cRPD
BGP Monitoring Protocol (BMP) test using containerized Juniper cRPD, OpenBMP and Containerlab


## Pre-Requisite

+ Install [Docker](https://www.docker.com/) and [Docker Compose](https://docs.docker.com/compose/install/)
+ Download, Load and Install [Juniper cRPD](https://www.juniper.net/gb/en/dm/crpd-free-trial.html)
  + Install test license, else BGP won't work, although this lab already comes with a test license 
  + I have also used [Juniper cRPD Deployment Guide for Linux Server](https://www.juniper.net/documentation/us/en/software/crpd/crpd-deployment/topics/task/cRPD-Linux-Server-Docker-Routing-Mode.html) as reference
+ Install [wbitt/network-multitool](https://hub.docker.com/r/wbitt/network-multitool) docker image
+ Install [containerlab](https://containerlab.dev/)
+ Install [OpenBMP](https://www.openbmp.org/getting_started.html) docker containers
  + [OpenBMP deployement guide](https://www.openbmp.org/getting_started.html#steps-to-bring-up-openbmp-on-a-single-vm)
  + I have customized the docker-compose.yml settings as I needed to add an Internet proxy for grafana container
  + Also defined a network bridge configuration with a static IPv4 subnet and addresses assigned to each container
  + Kindly customize as per your infrastructure


## My Infrastructure

+ An ubuntu 18.04 LTS virtual machine with 8vCPU and 16GB RAM, running in an eve-ng server, uses an http proxy for Internet
+ I have used the [hmntsharma/clab-crpdmpls](https://github.com/hmntsharma/clab-crpdmpls) lab as base and added the BMP configuration for further testing
  + I request you to take a look at it before you proceed further


## Topology

![image](https://user-images.githubusercontent.com/101124549/164953682-77ca40f2-4faf-4eb3-aded-5df310f1b7e5.png)


## Clone the repository
```
lab@ubuntu1804:~/github$ sudo git clone https://github.com/hmntsharma/openbmp-crpd.git
Cloning into 'openbmp-crpd'...
remote: Enumerating objects: 59, done.
remote: Counting objects: 100% (59/59), done.
remote: Compressing objects: 100% (43/43), done.
remote: Total 59 (delta 12), reused 56 (delta 12), pack-reused 0
Unpacking objects: 100% (59/59), done.
lab@ubuntu1804:~/github$
```


## Deploy the topology
The lab is deployed in 2 steps, first deploy OpenBMP and then the containerlab


### Deploy OpenBMP

Change to the cloned lab directory

```
lab@ubuntu1804:~/github$ cd openbmp-crpd/
lab@ubuntu1804:~/github/openbmp-crpd$ ll
total 28
drwxr-xr-x 5 root root 4096 Apr 24 04:09 clab-openbmp-crpd
-rw-r--r-- 1 root root 8348 Apr 24 04:09 docker-compose.yml
drwxr-xr-x 2 root root 4096 Apr 24 04:09 linux_net_config
-rw-r--r-- 1 root root 1087 Apr 24 04:09 openbmp-crpd.yml
-rw-r--r-- 1 root root  109 Apr 24 04:09 README.md
lab@ubuntu1804:~/github/openbmp-crpd$
```

OpenBMP is deployed as per the steps [here](https://www.openbmp.org/getting_started.html#steps-to-bring-up-openbmp-on-a-single-vm), of which the 6th step is as below

```
lab@ubuntu1804:~/github/openbmp-crpd$ sudo OBMP_DATA_ROOT=/var/openbmp docker-compose -f docker-compose.yml -p obmp up -d
Creating obmp-collector ...
Creating obmp-psql ...
Creating obmp-zookeeper ...
Creating obmp-whois ...
Creating obmp-psql-app ...
Creating obmp-collector
Creating obmp-psql
Creating obmp-grafana ...
Creating obmp-whois
Creating obmp-psql-app
Creating obmp-zookeeper
Creating obmp-zookeeper ... done
Creating obmp-kafka ...
Creating obmp-kafka ... done
lab@ubuntu1804:~/github/openbmp-crpd$
```


#### Verify the containers are up and running

```
lab@ubuntu1804:~/github/openbmp-crpd$ sudo docker ps
CONTAINER ID   IMAGE                             COMMAND                  CREATED          STATUS          PORTS                                       NAMES
9666f03d3b1e   confluentinc/cp-kafka:7.0.1       "/etc/confluent/dock…"   21 seconds ago   Up 19 seconds   0.0.0.0:9092->9092/tcp, :::9092->9092/tcp   obmp-kafka
06148a4cfb86   grafana/grafana:8.3.4             "/run.sh"                27 seconds ago   Up 21 seconds   0.0.0.0:3000->3000/tcp, :::3000->3000/tcp   obmp-grafana
4b8d69af4c7a   confluentinc/cp-zookeeper:7.0.1   "/etc/confluent/dock…"   27 seconds ago   Up 20 seconds   2181/tcp, 2888/tcp, 3888/tcp                obmp-zookeeper
6b6d43abc8bb   openbmp/whois:2.1.0               "/bin/sh -c '/usr/lo…"   27 seconds ago   Up 22 seconds   0.0.0.0:4300->43/tcp, :::4300->43/tcp       obmp-whois
401fe9e65ecc   openbmp/psql-app:2.1.1            "/usr/sbin/run"          27 seconds ago   Up 10 seconds   0.0.0.0:9005->9005/tcp, :::9005->9005/tcp   obmp-psql-app
d2449e61081b   openbmp/postgres:2.1.1            "docker-entrypoint.s…"   27 seconds ago   Up 25 seconds   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   obmp-psql
207b5b9a7b94   openbmp/collector:2.1.1           "/usr/sbin/run"          27 seconds ago   Up 26 seconds   0.0.0.0:5000->5000/tcp, :::5000->5000/tcp   obmp-collector
lab@ubuntu1804:~/github/openbmp-crpd$
```


#### Verify the custom network bridge was also deployed 

```
lab@ubuntu1804:~/github/openbmp-crpd$ sudo docker network ls | grep obmp_openbmp
38f423aba261   obmp_openbmp   bridge    local
lab@ubuntu1804:~/github/openbmp-crpd$



lab@ubuntu1804:~/github/openbmp-crpd$ sudo brctl show br-openbmp
bridge name     bridge id               STP enabled     interfaces
br-openbmp              8000.0242ce2d03fa       no              veth09e97d3
                                                        veth15e0060
                                                        veth27f08b1
                                                        veth4138d88
                                                        veth41c1999
                                                        veth533c3b6
                                                        vethf16387a
lab@ubuntu1804:~/github/openbmp-crpd$



lab@ubuntu1804:~/github/openbmp-crpd$ sudo docker network inspect obmp_openbmp | grep -i -E "name|ipv4"
        "Name": "obmp_openbmp",
                "Name": "obmp-grafana",
                "IPv4Address": "172.18.18.4/24",
                "Name": "obmp-collector",
                "IPv4Address": "172.18.18.6/24",
                "Name": "obmp-psql-app",
                "IPv4Address": "172.18.18.7/24",
                "Name": "obmp-zookeeper",
                "IPv4Address": "172.18.18.2/24",
                "Name": "obmp-whois",
                "IPv4Address": "172.18.18.8/24",
                "Name": "obmp-kafka",
                "IPv4Address": "172.18.18.3/24",
                "Name": "obmp-psql",
                "IPv4Address": "172.18.18.5/24",
            "com.docker.network.bridge.name": "br-openbmp"
lab@ubuntu1804:~/github/openbmp-crpd$



lab@ubuntu1804:~/github/openbmp-crpd$ ip addr show br-openbmp
1879: br-openbmp: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ce:2d:03:fa brd ff:ff:ff:ff:ff:ff
    inet 172.18.18.1/24 brd 172.18.18.255 scope global br-openbmp
       valid_lft forever preferred_lft forever
    inet6 fe80::42:ceff:fe2d:3fa/64 scope link
       valid_lft forever preferred_lft forever
lab@ubuntu1804:~/github/openbmp-crpd$



lab@ubuntu1804:~/github/openbmp-crpd$ ip link show | grep br-openbmp
1879: br-openbmp: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default
1918: veth41c1999@if1917: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-openbmp state UP mode DEFAULT group default
1920: veth533c3b6@if1919: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-openbmp state UP mode DEFAULT group default
1922: veth15e0060@if1921: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-openbmp state UP mode DEFAULT group default
1926: vethf16387a@if1925: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-openbmp state UP mode DEFAULT group default
1928: veth4138d88@if1927: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-openbmp state UP mode DEFAULT group default
1930: veth09e97d3@if1929: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-openbmp state UP mode DEFAULT group default
1934: veth27f08b1@if1933: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-openbmp state UP mode DEFAULT group default
lab@ubuntu1804:~/github/openbmp-crpd$

```


### Deploy Containerlab

The above deployment creates the bridge, which is then referenced in the containerlab topology definition yml file

```
lab@ubuntu1804:~/github/openbmp-crpd$ sudo clab deploy -t openbmp-crpd.yml
INFO[0000] Containerlab v0.25.1 started
INFO[0000] Parsing & checking topology file: openbmp-crpd.yml
INFO[0000] Creating lab directory: /home/lab/github/openbmp-crpd/clab-openbmp-crpd
INFO[0000] Creating docker network: Name="clab", IPv4Subnet="172.20.20.0/24", IPv6Subnet="2001:172:20:20::/64", MTU="1500"
INFO[0000] config file '/home/lab/github/openbmp-crpd/clab-openbmp-crpd/PE3/config/juniper.conf' for node 'PE3' already exists and will not be generated/reset
INFO[0000] Creating container: "HOST3"
INFO[0000] Creating container: "PE3"
INFO[0000] config file '/home/lab/github/openbmp-crpd/clab-openbmp-crpd/PE1/config/juniper.conf' for node 'PE1' already exists and will not be generated/reset
INFO[0000] config file '/home/lab/github/openbmp-crpd/clab-openbmp-crpd/CR2/config/juniper.conf' for node 'CR2' already exists and will not be generated/reset
INFO[0000] Creating container: "HOST1"
INFO[0000] Creating container: "PE1"
INFO[0000] Creating container: "CR2"
INFO[0001] Creating virtual wire: PE1:eth8 <--> br-openbmp:eth8
INFO[0003] Creating virtual wire: PE1:eth1 <--> CR2:eth1
INFO[0004] Creating virtual wire: PE3:eth3 <--> HOST3:eth3
INFO[0004] Creating virtual wire: CR2:eth2 <--> PE3:eth2
INFO[0005] Creating virtual wire: PE1:eth3 <--> HOST1:eth3
INFO[0005] Adding containerlab host entries to /etc/hosts file
+---+-------+--------------+-------------------------+-------+---------+----------------+----------------------+
| # | Name  | Container ID |          Image          | Kind  |  State  |  IPv4 Address  |     IPv6 Address     |
+---+-------+--------------+-------------------------+-------+---------+----------------+----------------------+
| 1 | CR2   | 4fd403e68be9 | crpd:21.4R1.12          | crpd  | running | 172.20.20.3/24 | 2001:172:20:20::3/64 |
| 2 | HOST1 | a3aed457dcca | wbitt/network-multitool | linux | running | 172.20.20.5/24 | 2001:172:20:20::5/64 |
| 3 | HOST3 | 909e22b6bd73 | wbitt/network-multitool | linux | running | 172.20.20.4/24 | 2001:172:20:20::4/64 |
| 4 | PE1   | 04f194959d9c | crpd:21.4R1.12          | crpd  | running | 172.20.20.2/24 | 2001:172:20:20::2/64 |
| 5 | PE3   | 2f2d57a38264 | crpd:21.4R1.12          | crpd  | running | 172.20.20.6/24 | 2001:172:20:20::6/64 |
+---+-------+--------------+-------------------------+-------+---------+----------------+----------------------+
lab@ubuntu1804:~/github/openbmp-crpd$
```


#### Verify HOST3 prefix in vrf CRPD of PE1

```
lab@ubuntu1804:~/github/openbmp-crpd$ sudo docker exec PE1 cli show route table CRPD.inet.0 protocol bgp

CRPD.inet.0: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

192.168.200.0/24   *[BGP/170] 00:00:39, localpref 100, from 3.3.3.3
                      AS path: I, validation-state: unverified
                    >  to 10.1.2.2 via eth1, Push 16, Push 17(top)
lab@ubuntu1804:~/github/openbmp-crpd$
```

#### Verify BMP Session with OpenBMP Collector

The PE1 router is a BMP client of obmp-collector docker container, which is listening on port 5000

```
lab@ubuntu1804:~/github/openbmp-crpd$ sudo docker exec PE1 cli show bgp bmp
Station name: BMP_AIO
  Local address/port: 172.18.18.100/-, Station address/port: 172.18.18.6/5000, active
  State: established Local: 172.18.18.100+38443 Remote: 172.18.18.6+5000
  Last state change: 1:43
  Monitor BGP Peers: enabled
  Route-monitoring: pre-policy
  Hold-down: 600, flaps 3, period 300
  Initiation message: Development/LAB
  Priority: low
  Statistics timeout: 300
  Version: 3
  Routing Instance: default
  Trace options:  all
  Trace file: /var/log//bmp.log size 10485760 files 10
lab@ubuntu1804:~/github/openbmp-crpd$
```

## Grafana Dashboard 

Grafana dashboard is running on https port 3000. which can be seen in the output of ```sudo docker ps``` above
I have used ssh tunneling to access the https dashboard at 127.0.0.1:3000

![image](https://user-images.githubusercontent.com/101124549/164953919-2e54defa-86e5-40d3-828f-b5db5925711d.png)

Our test focuses on the following dashboards

![image](https://user-images.githubusercontent.com/101124549/164953941-ece5115d-421e-4a96-a51c-733ac01c3c3a.png)

![image](https://user-images.githubusercontent.com/101124549/164953983-60f5a68a-2332-4c37-ace2-35d64fd7e48f.png)


## BMP Testing

Activate and deactivate eth3 interface on PE3, to trigger BGP advertisement and withdrawal, which would also reflect in Grafana dashboard


```
lab@ubuntu1804:~/github/openbmp-crpd$ sudo docker exec -it PE3 cli
root@PE3> edit
Entering configuration mode

[edit]
root@PE3# deactivate routing-instances CRPD interface eth3

[edit]
root@PE3# commit
commit complete

[edit]
root@PE3# activate routing-instances CRPD interface eth3

[edit]
root@PE3# commit
commit complete

[edit]
root@PE3# exit
Exiting configuration mode

root@PE3> exit

lab@ubuntu1804:~/github/openbmp-crpd$
```

Withdrawal and Advertisement auto updated in the dashboard below

![image](https://user-images.githubusercontent.com/101124549/164954064-1f829fb6-a728-4929-8464-db28f0237cac.png)


I have also setup traceoptions in PE1 to monitor bmp logs, which can be checked as below
```
root@PE1> monitor start bmp.log

root@PE1>
*** bmp.log ***
Apr 24 02:45:24.806033 task_timer_reset: reset BMPW_a.172.18.18.6+5000_Read
Apr 24 02:45:24.806069 task_timer_set_oneshot_latest: timer BMPW_a.172.18.18.6+5000_Read interval set to 14.234762
Apr 24 02:45:25.613076 station BMP_AIO peer 3.3.3.3 (Internal AS 65001) pw 0x5637631c1780 set_flag 0x10
Apr 24 02:45:25.613128 creating write_job for station BMP_AIO, priority 7
Apr 24 02:45:25.613134 task_job_create_background: create prio 7 job write_job for task BMPa
Apr 24 02:45:25.613881 background dispatch running job write_job for task BMPa
Apr 24 02:45:25.613903 running write_job for station BMP_AIO
Apr 24 02:45:25.613907 start work &s->bmps_work_head=0x5637648f1450 work_head=0x5637631c17a0
Apr 24 02:45:25.613911 bmp_do_peer_work: do work for peer 3.3.3.3 (Internal AS 65001), station BMP_AIO bmppw_flags 0x258
Apr 24 02:45:25.613915 did work for pw 0x5637631c1780: write_res: 0   flags: 58 d 0
Apr 24 02:45:25.613918 requeue peer for more work, pw 0x5637631c1780, station BMP_AIO
Apr 24 02:45:25.613921 bmp_do_peer_work: do work for peer 3.3.3.3 (Internal AS 65001), station BMP_AIO bmppw_flags 0x58
Apr 24 02:45:25.613925 bmp_send_rm_tuple called for pre-policy, peer 3.3.3.3 (Internal AS 65001), station BMP_AIO
Apr 24 02:45:25.613933 bmp_send_rm_tuple called, found pre-policy prefix 65001:100:192.168.200.0/88, peer 3.3.3.3 (Internal AS 65001), station BMP_AIO
Apr 24 02:45:25.613955 generating pre-policy add for prefix 65001:100:192.168.200.0/88, peer 3.3.3.3 (Internal AS 65001), station BMP_AIO
Apr 24 02:45:25.613970 BMPW: creating write_job for station BMP_AIO, priority 7
Apr 24 02:45:25.613979 task_job_create_background: create prio 7 job station_writer_job for task BMPW_a.172.18.18.6+5000
Apr 24 02:45:25.613983 bmp_send_rm_tuple called, no next pre-policy prefix, peer 3.3.3.3 (Internal AS 65001), station BMP_AIO 0xd 0x258
Apr 24 02:45:25.613986 did work for pw 0x5637631c1780: write_res: 0   flags: 248 d 0
Apr 24 02:45:25.614011 send per station adj-rib-out feed, station BMP_AIO
Apr 24 02:45:25.614015 bgp_bmp_do_station_ribout_rm_work: All adj ribout work done for station BMP_AIO
Apr 24 02:45:25.614018 check if station done: work empty=1 loc rib work empty=1 adj rib work empty=1 write_res: 0
Apr 24 02:45:25.614021 write_job work complete, station BMP_AIO
Apr 24 02:45:25.614024 task_job_delete: delete background job write_job for task BMPa
Apr 24 02:45:25.614033 background dispatch completed job write_job for task BMPa
Apr 24 02:45:25.614187 background dispatch running job station_writer_job for task BMPW_a.172.18.18.6+5000
Apr 24 02:45:25.614202 BMPW: running write job for station BMP_AIO
Apr 24 02:45:25.614228 BMPW: type 0 (RM), len 132, ver 3, pre-policy, for Peer 3.3.3.3, station BMP_AIO
Apr 24 02:45:25.614238  Peer AS: 65001 Peer BGP Id: 3.3.3.3 Time: 1650768325:60656 (Apr 24 02:45:25)
Apr 24 02:45:25.614242   Update: message type 2 (Update) length 84
Apr 24 02:45:25.614246   Update: Update PDU length 84
Apr 24 02:45:25.614255   Update: flags 0x40 code Origin(1): IGP
Apr 24 02:45:25.614265   Update: flags 0x40 code ASPath(2) length 0: <null>
Apr 24 02:45:25.614268   Update: flags 0x40 code LocalPref(5): 100
Apr 24 02:45:25.614274   Update: flags 0xc0 code Extended Communities(16): 2:65001:100
Apr 24 02:45:25.614279   Update: flags 0x90 code MP_reach(14): AFI/SAFI 1/128
Apr 24 02:45:25.614286   Update:        nhop 3.3.3.3 len 12
Apr 24 02:45:25.614303   Update:        65001:100:192.168.200.0/24 (label 16)
Apr 24 02:45:25.614309 BMPW: check if station done bmp_msg_start=(nil), pre policy rib-in = 1, post policy rib-in = 1, loc rib = 1, pre policy rib-out = 1, post policy rib-out = 1|1, ressult: Write in buffer
Apr 24 02:45:25.614312 BMPW: requesting write for 132 bytes, to station BMP_AIO
Apr 24 02:45:25.614434 BMPW: all 132 bytes written, station BMP_AIO
Apr 24 02:45:25.614446 BMPW: write_job work complete, station BMP_AIO
Apr 24 02:45:25.614449 task_job_delete: delete background job station_writer_job for task BMPW_a.172.18.18.6+5000
Apr 24 02:45:25.614454 background dispatch completed job station_writer_job for task BMPW_a.172.18.18.6+5000
Apr 24 02:45:39.041315 task_timer_reset: reset BMPW_a.172.18.18.6+5000_Read
Apr 24 02:45:39.041426 task_timer_set_oneshot_latest: timer BMPW_a.172.18.18.6+5000_Read interval set to 14.023228


root@PE1>
```


## Save the lab

After each config commit, it is best practice to save the lab, else all the configuration would be lost

```
lab@ubuntu1804:~/github/openbmp-crpd$ sudo clab save -t openbmp-crpd.yml
INFO[0000] Parsing & checking topology file: openbmp-crpd.yml
INFO[0000] saved cRPD configuration from CR2 node to /home/lab/github/openbmp-crpd/clab-openbmp-crpd/CR2/config/juniper.conf
INFO[0000] saved cRPD configuration from PE1 node to /home/lab/github/openbmp-crpd/clab-openbmp-crpd/PE1/config/juniper.conf
INFO[0000] saved cRPD configuration from PE3 node to /home/lab/github/openbmp-crpd/clab-openbmp-crpd/PE3/config/juniper.conf
lab@ubuntu1804:~/github/openbmp-crpd$
```

## Shutdown the containerlab and OpenBMP deployment

```
lab@ubuntu1804:~/github/openbmp-crpd$ sudo clab destroy -t openbmp-crpd.yml
INFO[0000] Parsing & checking topology file: openbmp-crpd.yml
INFO[0000] Destroying lab: openbmp-crpd
INFO[0000] Removed container: CR2
INFO[0000] Removed container: HOST3
INFO[0000] Removed container: PE3
INFO[0000] Removed container: HOST1
INFO[0000] Removed container: PE1
INFO[0000] Removing containerlab host entries from /etc/hosts file
lab@ubuntu1804:~/github/openbmp-crpd$
```

```
lab@ubuntu1804:~/github/openbmp-crpd$ sudo OBMP_DATA_ROOT=/var/openbmp docker-compose -p obmp down
Stopping obmp-kafka     ... done
Stopping obmp-grafana   ... done
Stopping obmp-zookeeper ... done
Stopping obmp-whois     ... done
Stopping obmp-psql-app  ... done
Stopping obmp-psql      ... done
Stopping obmp-collector ... done
Removing obmp-kafka     ... done
Removing obmp-grafana   ... done
Removing obmp-zookeeper ... done
Removing obmp-whois     ... done
Removing obmp-psql-app  ... done
Removing obmp-psql      ... done
Removing obmp-collector ... done
Removing network obmp_openbmp
lab@ubuntu1804:~/github/openbmp-crpd$
```


# Thank You!
