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
*   kubectl to deploy the 
*	eksctl to manage the aws cluster from your host machine


For example, you can build the following command to install all MobilityDB build dependencies for Debian-based systems:
```bash
apt install build-essential cmake postgresql-server-dev-11 liblwgeom-dev libproj-dev libjson-c-dev
```


 User's Manual
-------------

### Build Citus & MobilityDB 
This image deploy Citus on top of MobilityDB. The Dockerfile contain both Citus and MobilityDB gist that work adequately.
```bash
git clone https://github.com/bouzouidja/scale_mobilitydb.git
cd scale_mobilitydb
docker build -t scalemobilitydb/scalemobilitydb .
```

### Deployment  on aws services

In this manual we will show you two kind of scaling the MobilityDB on aws services.

### Using Citus cluster
Before doing this step you need to connect within your aws EC2 machine considred as master node. We assume that we have already create and configure one aws EC2 host master node and some aws EC2 host worker node.
- You can run the image as standalone using docker run command
```bash
sudo ssh -i YourKeyPairGenerated.pem ubuntu@EC2_Public_IP_Address

docker run --name scaledb_standalone -p 5432:5432 scalemobilitydb/scalemobilitydb \
            -v /path/on/host_mobilitydb_data/:/path/inside/container_mobilitydb_data\ 

``` 
After running the scaledb instance you can add and scale manually your database using the citus query  select * from citus_add_node('new-node', port);
- Running the image as Citus cluster using docker-compose command



### Using AWS EKS's Kubernetes service
Before running this step, we assume that you have created an aws kubernetes cluster using the eks command-line.