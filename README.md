# gobgp-mrt-injection-lab
This lab demonstrates how to read routes from [MRT files](https://datatracker.ietf.org/doc/html/rfc6396), import those routes in GoBGP, then advertise them to a peer. There is a lot of information online about creating MRT dumps, but information about injecting MRT data back into a router is almost non-existent. There are a few tools and scripts for doing so. The GoBGP support for importing data from MRT files seems to be the most robust option so far.


## Lab Components
The following components were used for this lab:
- Host machine running [Fedora 39](https://fedoraproject.org/server/download/)
- [Containerlab 0.59.0](https://containerlab.dev/)
- [Debian 12](https://www.debian.org/index.html) containers from [Docker Hub](https://hub.docker.com/_/debian) running in Containerlab
- [GoBGP](https://osrg.github.io/gobgp/) version 3.10.0 which was available through the Debian package manager
- [BIRD](https://bird.network.cz/) version 2.0.12 which was available through the Debian package manager
- Custom Docker containers for GoBGP and BIRD
- MRT dumps from RIPE or RouteViews


## Lab Setup
The lab topology according to the Containerlab graph looks like this: 
![Lab Topology](topology.png)  

Some more info about the setup and nodes used:  
| Node | Type | AS | Router ID / lo0 IP | eth1 IP | eth2 IP |
| --- | --- | --- | --- | --- | --- |
| r1 | GoBGP | 65001 | 10.255.255.1 | 10.255.254.1/30 | N/A |
| r2 | BIRD | 65002 | 10.255.255.2 | 10.255.254.2/30 | 10.255.254.5/30 |
| r3 | BIRD | 65003 | 10.255.255.3 | 10.255.254.6/30 | N/A |

The idea is that r1 builds an EBGP adjacency with r2, and r3 builds an EBGP adjacency with r3.  Then, r1 gets some routes injected from an MRT file and exports them to r2 with next-hop-self set. The MRT routes should propagate to r3 as well.  

Note that the BIRD routers (r2 & r3) are configured to export their loopback IP, and reachability between loopback works between these two routers. For r1 this feature is broken; GoBGP doesn't feature built-in support for manipulating FIB / kernel routes, but instead relies on Zebra (a [FRRouting](https://frrouting.org/) component). As FIB integration on node r1 isn't a strict requirement for the purpose of this lab, Zebra integration was skipped.

The containerlab topology is described in the file gobgp-mrt-injection-lab.clab.yml. There are some additional instructions, to set some interfaces and addresses, and to mount configuration files, an MRT file, and starting BIRD. GoBGP needs to be started manually as per the instructions in the next section. Besides the topology file, configuration files, and MRT file, there are two Dockerfiles used to create the required container images.  


## Bootstrapping The Lab
Use the following steps to get the lab up and running:  
**Step 1:** Clone this repository to a convenient location, in this case the home directory of the root user. The rest of the steps must be performed from inside the lab directory  
**Step 2:** Make the required custom Docker images available by using the supplied Dockerfiles and building them:
```
docker build -f ./Dockerfile-bird -t debird .
```
```
docker build -f ./Dockerfile-gobgp -t degobgp .
```
**Step 3:** Your locally available images can be listed and verified like this:
```
docker image ls
```
**Step 4:** Download and decompress an MRT file. In this example, a file from RouteViews from an AMS-IX location was used. Note that software engineering best practices have been applied, so this exact MRT filename is hardcoded in the Containerlab topology file. Give the bzip2 command a bit of time to complete:
```
wget https://archive.routeviews.org/amsix.ams/bgpdata/2024.11/RIBS/rib.20241101.0000.bz2
bzip2 -ckd rib.20241101.0000.bz2 > route-views-ams-ix-1-20241101.mrt
```
**Step 5:** All parts are now in place and the lab can be started:
```
clab deploy
```
**Step 6:** The lab status can be inspected by both Containerlab and Docker:
```
clab inspect
```
```
docker ps -a
```
**Step 7:** With the lab up and running, it's time to connect to r1 and start gobgpd:
```
docker exec -it clab-gobgp-mrt-injection-lab-r1 /bin/bash
```
```
gobgpd -t yaml -f /etc/gobgpd.conf
```
**Step 8:** The gobgpd process has to keep running in the terminal, so open another one (tmux is a good tool for terminal session multiplexing)  


## Scouting The Territory
Use the following steps to check out the lab environment a bit:  
**Step 9:** Connect to r1 again:
```
docker exec -it clab-gobgp-mrt-injection-lab-r1 /bin/bash
```
**Step 10:** GoBGP should have established a neighborship to the BIRD process on r2. Use the following commands to inspect the list of neighbors, and to display some capabilities and statistics for a specific neighbor:
```
root@r1:/# gobgp neighbor
Peer            AS  Up/Down State       |#Received  Accepted
10.255.254.2 65002 00:04:50 Establ      |        2         2

root@r1:/# gobgp neighbor 10.255.254.2
BGP neighbor is 10.255.254.2, remote AS 65002
  BGP version 4, remote router ID 10.255.255.2
  BGP state = ESTABLISHED, up for 00:04:52
  BGP OutQ = 0, Flops = 0
  Hold time is 90, keepalive interval is 30 seconds
  Configured hold time is 90, keepalive interval is 30 seconds

  Neighbor capabilities:
    multiprotocol:
        ipv4-unicast:   advertised and received
    route-refresh:      advertised and received
    extended-nexthop:   advertised
        Local:  nlri: ipv4-unicast, nexthop: ipv6
    graceful-restart:   received
    4-octet-as: advertised and received
    enhanced-route-refresh:     received
    long-lived-graceful-restart:        received
    fqdn:       advertised
      Local:
         name: r1, domain:
  Message statistics:
                         Sent       Rcvd
    Opens:                  1          1
    Notifications:          0          0
    Updates:                0          3
    Keepalives:            10         11
    Route Refresh:          0          0
    Discarded:              0          0
    Total:                 11         15
  Route statistics:
    Advertised:             0
    Received:               2
    Accepted:               2

```
**Step 11:** In a similar way, you can inspect things with BIRD on r2 and r3. Let's connect to r3 and check some things out. The output below shows opening the BIRD CLI tool (birdc), showing neighbors, neighbor details, and the BIRD RIB. Then exiting, showing the FIB, and verifying reachability of a neighbor loopback:
```
root@r3:/# birdc
BIRD 2.0.12 ready.
bird> show protocols
Name       Proto      Table      State  Since         Info
device1    Device     ---        up     17:55:21.897
direct1    Direct     ---        up     17:55:21.897
kernel1    Kernel     master4    up     17:55:21.897
r2         BGP        ---        up     17:55:26.375  Established

bird> show protocols all r2
Name       Proto      Table      State  Since         Info
r2         BGP        ---        up     17:55:26.375  Established
  BGP state:          Established
    Neighbor address: 10.255.254.5
    Neighbor AS:      65002
    Local AS:         65003
    Neighbor ID:      10.255.255.2
    Local capabilities
      Multiprotocol
        AF announced: ipv4
      Route refresh
      Graceful restart
      4-octet AS numbers
      Enhanced refresh
      Long-lived graceful restart
    Neighbor capabilities
      Multiprotocol
        AF announced: ipv4
      Route refresh
      Graceful restart
      4-octet AS numbers
      Enhanced refresh
      Long-lived graceful restart
    Session:          external AS4
    Source address:   10.255.254.6
    Hold timer:       165.479/240
    Keepalive timer:  30.114/80
  Channel ipv4
    State:          UP
    Table:          master4
    Preference:     100
    Input filter:   ACCEPT
    Output filter:  ACCEPT
    Routes:         1 imported, 1 exported, 1 preferred
    Route change stats:     received   rejected   filtered    ignored   accepted
      Import updates:              1          0          0          0          1
      Import withdraws:            0          0        ---          0          0
      Export updates:              2          1          0        ---          1
      Export withdraws:            0        ---        ---        ---          0
    BGP Next hop:   10.255.254.6

bird> show route
Table master4:
10.255.255.3/32      unicast [direct1 17:55:21.898] * (240)
        dev lo0
10.255.255.2/32      unicast [r2 17:55:26.754] * (100) [AS65002i]
        via 10.255.254.5 on eth1

bird> exit
root@r3:/# ip route
default via 172.20.20.1 dev eth0
10.255.254.4/30 dev eth1 proto kernel scope link src 10.255.254.6
10.255.255.2 via 10.255.254.5 dev eth1 proto bird metric 32
172.20.20.0/24 dev eth0 proto kernel scope link src 172.20.20.3

root@r3:/# ping 10.255.255.2
PING 10.255.255.2 (10.255.255.2) 56(84) bytes of data.
64 bytes from 10.255.255.2: icmp_seq=1 ttl=64 time=0.058 ms
```


## Injecting The MRT Data
Use the following steps to inject the MRT data into the lab:  
**Step 12:** Use the next command to connect to r1 again and inject some routes from the MRT file ingo GoBGP. Documentation on the MRT inject feature is sparse. In this case, the trailing number is a *count* value that makes sure only 5 prefixes get injected instead of a few million. There is also an optional flag `--only-best` that you can use to only inject the best known route for a given prefix, instead of all known routes. The inject command with the count value can be used multiple times, GoBGP is smart enough to inject new prefixes from the same file:
```
docker exec -it clab-gobgp-mrt-injection-lab-r1 /bin/bash
```
```
gobgp mrt inject global route-views-ams-ix-1-20241101.mrt 5
```
**Step 13:** Inspect the GoBGP RIB and look for the new routes. You might have to run the previous inject command twice to get more than just a "default" prefix with a few next-hops:
```
root@r1:/# gobgp global rib
   Network              Next Hop             AS_PATH                                      Age        Attrs
*> 0.0.0.0/0            80.249.214.200       6204                                         00:00:01   [{Origin: ?}]
*> 1.0.0.0/24           80.249.210.48        271253 1299 13335                            00:00:01   [{Origin: i} {Aggregate: {AS: 13335, Address: 10.34.6.80}}]
*  1.0.0.0/24           80.249.209.211       3320 6762 13335                              00:00:01   [{Origin: i} {Aggregate: {AS: 13335, Address: 10.34.6.180}} {Communities: 3320:1528, 3320:2010, 3320:9020, 6762:1, 6762:92, 6762:14400, 6762:20007, 6762:20008, 6762:20022, 6762:20023, 6762:20093, 6762:20094, 6762:20096, 6762:20097, 6762:20098, 6762:20099}]
*  1.0.0.0/24           80.249.210.38        34177 13335                                  00:00:01   [{Origin: i} {Aggregate: {AS: 13335, Address: 10.34.5.236}}]
*  1.0.0.0/24           80.249.211.162       213241 13335                                 00:00:01   [{Origin: i} {Aggregate: {AS: 13335, Address: 10.34.6.80}}]
*  1.0.0.0/24           80.249.214.200       6204 13335                                   00:00:01   [{Origin: i} {Aggregate: {AS: 13335, Address: 10.34.6.80}} {Communities: 6204:2000, 6204:2002}]
*  1.0.0.0/24           80.249.213.7         293 13335                                    00:00:01   [{Origin: i} {Med: 1} {Aggregate: {AS: 13335, Address: 10.34.6.80}}]
*  1.0.0.0/24           80.249.210.28        39120 13335                                  00:00:01   [{Origin: i} {Aggregate: {AS: 13335, Address: 172.68.196.1}} {Communities: 13335:10126, 13335:19020, 13335:20050, 13335:20500, 13335:20530} {LargeCommunity: [ 39120:24796:1]}]
*  1.0.0.0/24           80.249.215.28        20253 1299 13335                             00:00:01   [{Origin: i} {Aggregate: {AS: 13335, Address: 10.34.6.80}}]
*  1.0.0.0/24           80.249.213.88        328832 13335                                 00:00:01   [{Origin: i} {Aggregate: {AS: 13335, Address: 172.68.184.1}} {Communities: 13335:10113, 13335:19030, 13335:20050, 13335:20500, 13335:20530, 37497:4102}]
*  1.0.0.0/24           80.249.210.118       37271 13335                                  00:00:01   [{Origin: i} {Med: 0} {Aggregate: {AS: 13335, Address: 10.34.6.80}} {Communities: 13335:10020, 13335:19020, 13335:20050, 13335:20500, 13335:20530, 37271:1005, 37271:2150, 37271:2155, 37271:3528, 37271:4006, 37271:5002, 37271:5200, 37271:5212}]
*  1.0.0.0/24           80.249.211.140       1140 13335                                   00:00:01   [{Origin: i} {Aggregate: {AS: 13335, Address: 10.34.6.80}} {Communities: no-export}]
*  1.0.0.0/24           80.249.208.162       49544 13335                                  00:00:01   [{Origin: i} {Med: 0} {Aggregate: {AS: 13335, Address: 10.34.6.80}} {Communities: 13335:10020, 13335:19020, 13335:20050, 13335:20500, 13335:20530, 49544:23000, 49544:23031}]
*  1.0.0.0/24           80.249.211.234       56987 13335                                  00:00:01   [{Origin: i} {Aggregate: {AS: 13335, Address: 10.34.5.236}}]
*  1.0.0.0/24           80.249.208.34        1103 13335                                   00:00:01   [{Origin: i} {Aggregate: {AS: 13335, Address: 10.34.6.80}} {Communities: 13335:10020, 13335:19020, 13335:20050, 13335:20500, 13335:20530}]
*  1.0.0.0/24           80.249.212.41        398465 13335                                 00:00:01   [{Origin: i} {Aggregate: {AS: 13335, Address: 10.34.6.80}} {Communities: 13335:10020, 13335:19020, 13335:20050, 13335:20500, 13335:20530}]
*> 1.0.4.0/22           80.249.210.48        271253 1299 7545 2764 38803 38803 38803      00:00:01   [{Origin: i}]
*  1.0.4.0/22           80.249.209.211       3320 1299 7545 2764 38803 38803 38803        00:00:01   [{Origin: i} {Communities: 3320:1528, 3320:2010, 3320:9020}]
```
**Step 14:** All good so far, and it's pretty cool to see some real actual internet routes inside your own lab. Connect to r2 and do some more inspection. You can see that BIRD learned a number of routes in the stats, see the routes in the BIRD RIB, and see them in the FIB too:
```
root@r2:/# birdc
BIRD 2.0.12 ready.

bird> show protocols all r1
Name       Proto      Table      State  Since         Info
r1         BGP        ---        up     19:31:11.952  Established
  BGP state:          Established
    Neighbor address: 10.255.254.1
    Neighbor AS:      65001
    Local AS:         65002
    Neighbor ID:      10.255.255.1
    Local capabilities
      Multiprotocol
        AF announced: ipv4
      Route refresh
      Graceful restart
      4-octet AS numbers
      Enhanced refresh
      Long-lived graceful restart
    Neighbor capabilities
      Multiprotocol
        AF announced: ipv4
      Route refresh
      Extended next hop
        IPv6 nexthop: ipv4
      4-octet AS numbers
      Hostname: r1
    Session:          external AS4
    Source address:   10.255.254.2
    Hold timer:       70.379/90
    Keepalive timer:  24.438/30
  Channel ipv4
    State:          UP
    Table:          master4
    Preference:     100
    Input filter:   ACCEPT
    Output filter:  ACCEPT
    Routes:         3 imported, 2 exported, 3 preferred
    Route change stats:     received   rejected   filtered    ignored   accepted
      Import updates:             31          0          0          0         31
      Import withdraws:            0          0        ---          0          0
      Export updates:             33         31          0        ---          2
      Export withdraws:            0        ---        ---        ---          0
    BGP Next hop:   10.255.254.2

bird> show route
Table master4:
0.0.0.0/0            unicast [r1 19:31:49.702] * (100) [AS6204?]
        via 10.255.254.1 on eth1
1.0.4.0/22           unicast [r1 19:31:49.705] * (100) [AS38803i]
        via 10.255.254.1 on eth1
1.0.0.0/24           unicast [r1 19:31:49.703] * (100) [AS13335i]
        via 10.255.254.1 on eth1
10.255.255.3/32      unicast [r3 19:31:03.964] * (100) [AS65003i]
        via 10.255.254.6 on eth2
10.255.255.2/32      unicast [direct1 19:30:59.389] * (240)
        dev lo0

bird> exit

root@r2:/# ip route
default via 172.20.20.1 dev eth0
default via 10.255.254.1 dev eth1 proto bird metric 32
1.0.0.0/24 via 10.255.254.1 dev eth1 proto bird metric 32
1.0.4.0/22 via 10.255.254.1 dev eth1 proto bird metric 32
10.255.254.0/30 dev eth1 proto kernel scope link src 10.255.254.2
10.255.254.4/30 dev eth2 proto kernel scope link src 10.255.254.5
10.255.255.3 via 10.255.254.6 dev eth2 proto bird metric 32
172.20.20.0/24 dev eth0 proto kernel scope link src 172.20.20.4
```
**Step 15:** Finally, you can see that r3 learned the about the routes too:
```
root@r3:/# ip route
default via 172.20.20.1 dev eth0
default via 10.255.254.5 dev eth1 proto bird metric 32
1.0.0.0/24 via 10.255.254.5 dev eth1 proto bird metric 32
1.0.4.0/22 via 10.255.254.5 dev eth1 proto bird metric 32
10.255.254.4/30 dev eth1 proto kernel scope link src 10.255.254.6
10.255.255.2 via 10.255.254.5 dev eth1 proto bird metric 32
172.20.20.0/24 dev eth0 proto kernel scope link src 172.20.20.2
```


## Wrapping Up
You can bring down the lab and remove the containers with `clab destroy`. Containerlab is a perfect platform to play around and try things like this, but probably not the best choice for working with really large routing tables. The next step is to take the idea demonstrated here and use it to inject routes into real hardware based platforms. Maybe even doing things with multiple links, failovers and testing convergence times.


## Information Sources
The following resources were referenced for building this lab:
- [GoBGP documentation](https://github.com/osrg/gobgp/blob/master/README.md)
- [This excellent JPNAP GoBGP Tutorial](https://blog.netravnen.com/wp-content/uploads/2019/08/ixbrforum10day3gobgptutorial-161205210258.pdf) with YAML examples! Such a relieve after the unreadable TOML stuff in the official docs
- [BIRD 2 User's Guide](https://bird.network.cz/?get_doc&f=bird.html&v=20)
- [This excellent BIRD introduction](https://blog.kintone.io/entry/bird) on the Kintone Engineering Blog
- [Another BIRD example](https://deploy.equinix.com/developers/guides/configuring-bgp-with-bird/) by Equinix
- MRT files from RouteViews: https://archive.routeviews.org/
- MRT files from RIPE. Their docs are tricky, you need to know the right directory name to find them: https://data.ris.ripe.net/rrc01/

