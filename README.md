# scalable MobilityDB on AWS could services

This repository provides a configuration in order to deploy the MobilityDB on AWS cloud using EKS service and Citus cluster.
We prepared an image that combine the mobilityDB environement and citus environement built on top of it.
In addition you can find in the doc folder a different way to scale the MobilityDB on AWS cloud and we described the architecture of Elastic Kubernetes Service. 


[MobilityDB](https://github.com/ULB-CoDE-WIT/MobilityDB) is an open source software program that adds support for temporal and spatio-temporal objects to the [PostgreSQL](https://www.postgresql.org/) database and its spatial extension [PostGIS](http://postgis.net/).


Citus is a PostgreSQL-based distributed RDBMS. For more information, see the [Citus Data website][citus data].




Requirements
------------

*   aws account
*   docker latest version
*   kubectl to deploy application/images from your host machine 
*	eksctl to manage the aws cluster from your host machine






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
You can use the default region as the nearest one from you. In my case i used the eu-west-3 region (Paris).
At this stage we can manage our AWS services remotely from our machine through the credentials stored on the file located in: 
```bash
~/.aws/credentials
```

4. Creating an Amazon EKS cluster (control plane)


```bash
eksctl create cluster \
 --name mobilitydb-cluster \
 --version 1.20 \
 --region eu-west-3 \
 --nodegroup-name linux-nodes \
 --node-type m5.large \
 --ssh-access \
 --nodes 3 
 --nodes-min=2 --nodes-max=5 
```

In the region option you can use the nearest region from your location.
In the node-type option we define the type of the ressource for the created node. AWS provide a lot of ressource type. In my case i defined a m5.large type, which is 2 CPUs, 8G of RAM, 10G of storage. You can find the entire list of node type [here](https://eu-west-3.console.aws.amazon.com/ec2/v2/home?region=eu-west-3#LaunchInstanceWizard:). The nodes-min and node-max options used for the auto scaling node group. 

The creation process take about a 20 minutes of time.

If you want to delete the cluster with all the ressources created just use:
```bash
eksctl delete cluster --name mobilitydb-cluster
```` 


Autoscaler add EC2 node to make the cluster growing according to the metrics

5. View the cluster's ressources

```bash
kubectl get nodes
# You should see your EC2 node as this:
# NAME                                           STATUS   ROLES    AGE     VERSION
# ip-192-168-47-163.eu-west-3.compute.internal   Ready    <none>   8m56s   v1.20.4-eks-6b7464
# ip-192-168-9-100.eu-west-3.compute.internal    Ready    <none>   8m48s   v1.20.4-eks-6b7464
# ip-192-168-95-188.eu-west-3.compute.internal   Ready    <none>   8m52s   v1.20.4-eks-6b7464

```



6. Deploy our scaleMobilityDB image using the kubectl

We have prepared a manifest yaml file that define the environment of our workload mobilityDB. It contain the basics information and configuration in order to configure our Kubernetes cluster.
The deployment instance used to specify the scalemobilitydb docker image and the and to mount volume path. Finnaly the number  of replications to our deployment in order to increase the availability.

configMap instance defined the environement information (postgres user, password, database name).

The most important instances is the PersistentVolume and PersistentVolumeClaim. The PersistentVolume parameter allows to define the class of storage, device and file system allow that store our mobilitydb data, it simply a workers nodes that store data. AWS provides different classes of storages, for more information see [this](https://docs.aws.amazon.com/eks/latest/userguide/storage.html). The PersistentVolumeClaim parameter defines the type of request, access to use in order to interogate our PersistentVolume. A PersistentVolumeClaim has a access type policy – ReadWriteOnce, ReadOnlyMany, or ReadWriteMany. It simply a pod that manage the accesses to storage.
When you create a EKS cluster, by default the PersistentVolume is set to gp2 (General Purpose SSD driver). It's an Amazon EBS (Elastic Block Store) class.
Use this command to see the default storage class.
```bash
kubectl get storageclass
# NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
# gp2 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  15d
```
If you want to create your own storage class and set it as default, follow [this guides](https://docs.aws.amazon.com/eks/latest/userguide/storage-classes.html)

Finnaly the service instance used to expose our mobilitydb workload. All thoses configuration can be updated according to your workload needs.

Putting it all together in mobilitydb-workload.yaml file. Run this command to initialize all the instances. 
```bash
kubectl apply -f mobilitydb-workload.yaml

# deployment.apps/scale-mobilitydb created
# persistentvolume/postgres-pv-volume unchanged
# persistentvolumeclaim/postgres-pv-claim unchanged
# configmap/postgres-config unchanged
# service/scale-mobilitydb created

```
Now you should see all instances running.
```bash
kubectl get all

# NAME                                    READY   STATUS    RESTARTS   AGE
# pod/scale-mobilitydb-7d745544dd-pm2hm   1/1     Running   0          69m

# NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
# service/kubernetes         ClusterIP   10.100.0.1      <none>        443/TCP          15d
# service/scale-mobilitydb   NodePort    10.100.38.140   <none>        5432:30200/TCP   69m

# NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
# deployment.apps/scale-mobilitydb   1/1     1            1           69m

# NAME                                          DESIRED   CURRENT   READY   AGE
# replicaset.apps/scale-mobilitydb-7d745544dd   1         1         1       69m






````

At this stage you can run you psql client to confirm that the scalemobilitydb is deployed.
To run the psql, we need to know on which node the mobilitydb pod is running. the following command show details informations including the ip address that host the scalemobilitydb.  
```bash
kubectl get pod -owide

# NAME                                READY   STATUS    RESTARTS   AGE     IP               NODE                                          NOMINATED NODE   READINESS GATES
# postgres-98c7c5945-rrfwc            1/1     Running   1          4d23h   192.168.56.190   ip-192-168-46-36.eu-west-3.compute.internal   <none>           <none>
# scale-mobilitydb-7d745544dd-gqrf5   1/1     Running   0          19h     192.168.60.214   ip-192-168-46-36.eu-west-3.compute.internal   <none>           <none>


```
In my case the postgres have pod name as postgres-98c7c5945-rrfwc and is running in the node 192.168.56.190.

As we have the host ip and the name of pod that run our mobilitydb instance we can use this command to connect, the password for postgres user is root. We can run our psql client within the pod scale-mobilitydb to confirm that citus and mobilitydb extension it's well created.

```bash

kubectl exec -it  postgres-98c7c5945-rrfwc -- psql -h 192.168.56.190 -U postgres -p 5432 postgres

# Password for user postgres: 
# psql (13.3 (Debian 13.3-1.pgdg100+1))
# SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
# Type "help" for help.

# postgres=# \dx
#                                       List of installed extensions
#     Name    | Version |   Schema   |                             Description                             
# ------------+---------+------------+-------------------------------------------------#--------------------
# citus      | 10.1-1  | pg_catalog | Citus distributed database
#  mobilitydb | 1.0     | public     | Temporal datatypes and functions
# plpgsql    | 1.0     | pg_catalog | PL/pgSQL procedural language
# postgis    | 2.5.5   | public     | PostGIS geometry, geography, and raster spatial types and functions
#(4 rows)

#postgres=# 


```
7. Run mobilitydb queries
.....
In order to make the MobilityDb queries more powerfull, we have used the single node citus that create shards for distributed table,



As we have a complex MobilityDB queries, we may use the Vertical Autoscaler and the Horizontal Autoscaler to optimize the cost according to the query needs.

8. Vertical Pod scaling using the Autoscaler 
The vertical scaling that provide aws it's a mechanism allows us to adjust automatically the pods ressources. This adjustment decrease the cluster cost and can free up cpu and memory to other pods that may need it. The vertical autoscaler analyze the pods demand in order to see if the CPU and memory requirements are appropriate. If adjustments are needed, the vpa-updater relaunches the pods with updated values. 

To deploy the vertical autoscaler, as following is the steps:
```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler/
``` 
deploy the autoscaler pods to your cluster
```bash

./hack/vpa-up.sh
```

Check the vertical autoscaler pods 

```bash
kubectl get pods -n kube-system

# NAME                                       READY   STATUS    RESTARTS   AGE
# aws-node-rx54z                             1/1     Running   0          17d
# aws-node-ttf68                             1/1     Running   0          17d
# coredns-544bb4df6b-8ccvm                   1/1     Running   0          17d
# coredns-544bb4df6b-sbqhz                   1/1     Running   0          17d
# kube-proxy-krz8w                           1/1     Running   0          17d
# kube-proxy-lzm4g                           1/1     Running   0          17d
# metrics-server-9f459d97b-vtd6n             1/1     Running   0          3d11h
# vpa-admission-controller-6cd546c4f-g94vr   1/1     Running   0          38h
# vpa-recommender-6855ff754-f4blx            1/1     Running   0          38h
# vpa-updater-9fd7bfbd5-n9hpn                1/1     Running   0          38h

``` 

9. Horizontal Pod scaling using the Autoscaler


The horizontal Autoscaler that provides AWS allows to increase the number of pods within the cluster, it's a replication controller. This can help your applications scale out to meet increased demand or scale in when resources are not needed, the Horizontal Pod Autoscaler scales your application in or out to try to meet that target.

Before deploying the Horizontal autoscaler, we need the Kubernetes Metric server.
The metric server is an API that collect the ressources statistics from the cluster and expose them for the use of the autoscaler. For more information about the metric server see [here](https://github.com/kubernetes-sigs/metrics-server).


Deploy the metric server
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

``` 
  















Deployment using Citus cluster using AWS EC2 instances
------------

### Build Citus on top of MobilityDB 
This image deploy Citus on top of MobilityDB. The Dockerfile contain both Citus and MobilityDB gist that work adequately. This step need to be executed in all your cluster nodes.
```bash
git clone https://github.com/bouzouidja/scale_mobilitydb.git
cd scale_mobilitydb
docker build -t scalemobilitydb/scalemobilitydb .
```


### Deploy scale-mobilitydb as standalone
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



