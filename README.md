# scalable MobilityDB

This image provides the combination of the mobilityDB extension and citus extension on top of it.

Citus is a PostgreSQL-based distributed RDBMS. For more information, see the [Citus Data website][citus data].


[MobilityDB](https://github.com/ULB-CoDE-WIT/MobilityDB) is an open source software program that adds support for temporal and spatio-temporal objects to the [PostgreSQL](https://www.postgresql.org/) database and its spatial extension [PostGIS](http://postgis.net/).

This repository contains code and the documentation for running the [BerlinMOD](http://dna.fernuni-hagen.de/secondo/BerlinMOD/BerlinMOD.html) benchmark on MobilityDB.



[image size]: https://microbadger.com/images/citusdata/citus
[release]: https://github.com/citusdata/docker/releases/latest
[license]: LICENSE
[citus data]: https://www.citusdata.com
[docker-postgres]: https://hub.docker.com/_/postgres/
[compose-config]: docker-compose.yml
[workerlist-gen]: https://github.com/citusdata/workerlist-gen



Requirements
------------

*   aws account
*   docker latest version
*   kubectl
*	eksctl


For example, you can build the following command to install all MobilityDB build dependencies for Debian-based systems:
```bash
apt install build-essential cmake postgresql-server-dev-11 liblwgeom-dev libproj-dev libjson-c-dev
```


 User's Manual
-------------

### Build Citus & MobilityDB 
This image deploy Citus on top of MobilityDB.
```bash
git clone https://github.com/bouzouidja/scale_mobilitydb.git
cd scale_mobilitydb
docker build -t scalemobilitydb/scalemobilitydb .
```

### Deployment  on aws services

