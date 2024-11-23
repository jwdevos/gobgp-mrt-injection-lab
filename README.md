# gobgp-mrt-injection-lab
This lab shows how to read routes from MRT files, inject them into GoBGP, then advertise them to a peer.

The following components were used for this test:
- Host machine running [Fedora 39](https://fedoraproject.org/server/download/)
- [Containerlab 0.59.0](https://containerlab.dev/)
- [Debian 12](https://www.debian.org/index.html) containers from [Docker Hub](https://hub.docker.com/_/debian) running in Containerlab

