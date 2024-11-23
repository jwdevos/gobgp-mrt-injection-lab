# gobgp-mrt-injection-lab
This lab shows how to read routes from [MRT files](https://datatracker.ietf.org/doc/html/rfc6396), import those routes in GoBGP, then advertise them to a peer. There is a lot of information online about how to create MRT dumps, but information about how to get MRT data back into a routing daemon is almost non-existent. There are a few tools and scripts for doing so. The GoBGP support for importing data from MRT files seems to be the most robust option so far.


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
The lab topology according to the containerlab graph looks like this: 
![Lab Topology](topology.png)  

Some more info about the setup and nodes used:  
| Node | Type | AS | Router ID / lo0 IP | eth1 IP | eth2 IP |
| --- | --- | --- | --- | --- | --- |
| r1 | GoBGP | 65001 | 10.255.255.1 | 10.255.254.1/30 | N/A |
| r2 | BIRD | 65002 | 10.255.255.2 | 10.255.254.2/30 | 10.255.254.5/30 |
| r3 | BIRD | 65003 | 10.255.255.3 | 10.255.254.6/30 | N/A |

The idea is that r1 builds an EBGP adjacency with r2, and r3 builds an EBGP adjacency with r3.  Then, r1 gets some routes injected from an MRT file and exports them to r2 with next-hop-self set. The MRT routes should propagate to r3 as well.  


## Bootstrapping The Lab
**Step 1:** Clone this repository to a convenient location. In this case the home directory of the root user was used. **The rest of the steps must be performed from inside the lab directory**  
**Step 2:** Make the required custom Docker containers available by using the supplied Dockerfiles and building them:
```
docker build -f ./Dockerfile-bird -t debird .
docker build -f ./Dockerfile-gobgp -t degobgp .
```
**Step 3:** Bla

bla FIB GoBGP Zebra FRR bla


## Information Sources
The following resources were referenced for building this lab:
- [GoBGP documentation](https://github.com/osrg/gobgp/blob/master/README.md)
- [This excellent JPNAP GoBGP Tutorial](https://blog.netravnen.com/wp-content/uploads/2019/08/ixbrforum10day3gobgptutorial-161205210258.pdf) with YAML examples! Such a relieve after the unreadable TOML stuff in the official docs
- [BIRD 2 User's Guide](https://bird.network.cz/?get_doc&f=bird.html&v=20)
- [This excellent BIRD introduction](https://blog.kintone.io/entry/bird) on the Kintone Engineering Blog
- [Another BIRD example](https://deploy.equinix.com/developers/guides/configuring-bgp-with-bird/) by Equinix

