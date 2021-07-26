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




Deployment using Citus cluster
------------

### Build Citus on top of MobilityDB 
This image deploy Citus on top of MobilityDB. The Dockerfile contain both Citus and MobilityDB gist that work adequately. This step need to be executed in all your cluster nodes.
```bash
git clone https://github.com/bouzouidja/scale_mobilitydb.git
cd scale_mobilitydb
docker build -t scalemobilitydb/scalemobilitydb .
```


### Deploy scalemobilitydb as standalone
Before doing this step you need to connect within your aws EC2 machine known as master node. We assume that we have already create and configure one aws EC2 host master node and some aws EC2 host worker node.
- You can run the image as standalone using docker run command, Execute this on all cluster's nodes.
```bash
sudo ssh -i YourKeyPairGenerated.pem ubuntu@EC2_Public_IP_Address

docker run --name scaledb_standalone -p 5432:5432 -e POSTGRES_PASSWORD=postgres scalemobilitydb/scalemobilitydb 

``` 
You can specify the mount volume option in order to fill the mobilityDB dataset from your host machine by adding -v /path/on/host_mobilitydb_data/:/path/inside/container_mobilitydb_data

After running the scalemobilitydb instance, you can add and scale manually your database using the citus query.

```sql
select * from citus_add_node('new-node', port);
```
Check wether if the new-node is added correctely in the cluster.
```sql
select master_get_active_worker_nodes();
-  master_get_active_worker_nodes
-- --------------------------------
--  (new-node,5432)
-- (1 row)
```
Let create MobilityDB table and distribute it on column_dist in order to create shards by hashing the column_dist values. If no nodes added on the cluster than the distribution is seen as single node citus otherwise is multi nodes citus.

```sql
CREATE TABLE mobilitydb_table(
column_dist integer,
T timestamp,
Latitude float,
Longitude float,
Geom geometry(Point, 4326)
);

SELECT create_distributed_table('mobilitydb_table', 'column_dist');
```
fill free to fill the table mobilitydb_table before or after the distribution. At this stage you can run MobilityDB queries on the citus cluster.



### Deploy scalemobilitydb using citus manager
This deployment is similar to the last one except that there is a manager node. It simply listens for new containers tagged with the worker role, then adds them to the config file in a volume shared with the master node.
In the same repository scale_mobilitydb run the command 


- Running the image as Citus cluster using docker-compose command





Deployment using AWS EKS's Kubernetes service
------------

### Install requirements
Before running this step, we assume that you have created an aws kubernetes cluster using the eks command-line.

1. Install kubectl
```bash
curl -o kubectl https://amazon-eks.s3-us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/kubectl
# Check the SHA-256 sum for your downloaded binary.
openssl sha1 -sha256 kubectl

# Apply execute permissions to the binary.
chmod +x ./kubectl

# Copy the binary to a folder in your PATH. If you have already installed a version of kubectl, then we recommend creating a $HOME/bin/kubectl and ensuring that $HOME/bin comes first in your $PATH.

mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin

# (Optional) Add the $HOME/bin path to your shell initialization file so that it is configured when you open a shell. 
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc

# After you install kubectl , you can verify its version with the following command: 
kubectl version --short --client
````

2. Install eksctl
```bash
# Download and extract the latest release of eksctl with the following command. 
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
# Move the extracted binary to /usr/local/bin. 
sudo mv /tmp/eksctl /usr/local/bin
# Test that your installation was successful with the following command.
eksctl version
```

3. Install and configure the aws CLI (Command Line Interface) environment

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
# aws-cli/2.1.29 Python/3.7.4 Linux/4.14.133-113.105.amzn2.x86_64 botocore/2.0.0
```
AWS requires that all incoming requests are cryptographically signed. Let configure some mandatory information in order to use the aws services.

[Access Key ID](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds)


[Secret access Key](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds)


[AWS Region](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-region)


[Output Format](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-format)

Navigate to https://console.aws.amazon.com/iam/home#/home
- In the navigation pane, choose Users.

- Choose the name of the user whose access keys you want to create, and then choose the Security credentials tab.

- In the Access keys section, choose Create access key.

- To view the new access key pair, choose Show. You will not have access to the secret access key again after this dialog box closes. Save the access key id and the secret access key somewhere.
Now run aws configure and copy past them in their corresponding parameter

```bash
aws configure
# AWS Access Key ID [****************FZQ2]: 
# AWS Secret Access Key [****************RVKZ]: 
# Default region name [eu-west-3]: 
# Default output format [None]: 
```
In my case i used the eu-west-3 region (Paris)


4. Required IAM permissions


create first a aws account and create a user 

assign a list of permission for this user in order to allow it to use the aws elastics kubernetes services

4. Create a cluster control plane (master node)

5. create worker node and connect them to cluster. The worker node will be the EC2 instances with certain ressources (CPUs, RAM, storages)
We will create the worker nodes as Node Group (group of nodes), We can scale our node group according to the needs. The auto scaling can be configuring (define maximum and minimun of worker nodes)

6. Deploy the MobilityDB image using the kubectl

