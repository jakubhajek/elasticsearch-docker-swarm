# Initial requirements

This is example configuration which launches *Elasticsearch Cluster* on *Docker Swarm Cluster*. The entire setup distinguish following services:

- service with coordination Elasticsearch node role enabled which basically acts like a load balnacer
- service with Elasticsearch master eligible nodes
- service with Elasticsearch data nodes responsible for CRUD operations
- service visualizer which is presenting how containers are spread acorss the Docker Swarm Cluster

In order to run it I recommend to have at least 4 VM's provisioned. You can try AWS or even virtualbox running on your local machine. The easiest way is to setup servers using docker-machine. In this example I am using my AWS account (you can specify amazonec2-access-key and amazonec2_security_key in command line or use it ~/.aws/credentials and defined default profile) 

## Persitence in SWARM

Elasticsearch is distributed document oriented database, in our scenario it requires to keep a data in safe place. That's why I used `volume` features and use `constraint` to run data node exactly on the origin node where it was initially deployed. 
Use replica crea

## Elasticsearch configuration files

I moved the entire Elasticsearch configuration into sepreate YML files, because the only a few parameters are taken from environment variables. The same has been done with tunning JVM. 

# Provisioning virtual machines
Execute the following command to provision servers:

```shell
docker-machine create --driver amazonec2 --amazonec2-region eu-central-1 --amazonec2-instance-type t2.medium --amazonec2-security-group swarm-sg --amazonec2-open-port 9200/tcp --amazonec2-open-port 9300/tcp --amazonec2-open-port 2377/tcp --amazonec2-open-port 7946/tcp --amazonec2-open-port 4789/udp --amazonec2-open-port 7946/udp node-1
```

The given command should be executed 4 times at least, make sure you change hostname every time you are executing the command,the last argument in the command.
Please note that created AWS security group is only for demonstration purposes, you never should expose crucial ports to the internet. 

# Tuning virtual machines
Before running a stack, minor changes have to be adopted on each of newaly created Vm's, 
1. Edit /etc/sysctl.conf by adding following `vm.max_map_count=262144` or just simple execute 
```shell
docker-machine ssh node-1 sudo sysctl -w vm.max_map_count=262144
``` 
for each of the node. 
Better to add that add it paramentnly:
```shell 
docker-machine ssh swarm-4  sudo "sed -i '$ a vm\.max_map_count=262144' /etc/sysctl.conf"
```
2. Edit file `/etc/systemd/system/docker.service.d/10-machine.conf` and modify default ulimit because Elasticsearch cluster is requesting memory locking (see `bootstrap.memory_lock=true`), thus following parameter to ExecStart `--default-ulimit memlock=-1`. You can simple execute that command on the each of the node: 

```shell
docker-machine ssh node-1 sudo "sed -i '/ExecStart=\/usr\/bin\/dockerd/ s/$/--default-ulimit memlock=-1/' /etc/systemd/system/docker.service.d/10-machine.conf"
```

Please note that location of that file might be different depending of your Linux distribution. That one works with Ubuntu 16.04 LTS.
After that execute on each of the node `systemctl daemon-reload` and restart Docker daemon `systemctl restart docker`

# Initiating Docker Swarm cluster
Once vm's are created you can initiate Docker swarm cluster. Export environment variables belonging to the one of the nodes:

```shell
eval $(docker-machine env node-1)
```

and initiate a cluster: 

```shell
docker swarm init
```

then connect to the other vm's and add them accordingly to the cluster, for test purposes you can just add only workers to the cluster. For production environment pleaae consider the appropriate amount of manager nodes to keep RAFT database secure and provide high availability for your Docker Swarm Cluster. 

Make sure the cluster is working by executine command:

# Listing of the nodes consisting Docker Swarm cluster
```shell
docker node ls
```

```
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
iqepxq2w46nprlgm55gomf1ic *   node-1              Ready               Active              Leader              18.09.0
1rruyl7x9s9x43rql0x2jibx0     node-2              Ready               Active                                  18.09.0
yqoycyrs9j0cb1me7cwr77764     node-3              Ready               Active                                  18.09.0
56yflx8vy47oi3upycouzpjpj     node-4              Ready               Active                                  18.09.0
```
# Create configuration in SWARM 

Swarm provides nice to feature to store configuration files into RAFT, so executing following commands to create appropriate configugration:

```shell 
docker config create es-coordination es-coordination.yml
docker config create es-data es-data.yml
docker config create es-master es-master.yml
```

Create JVM configuration for each instances. I decided to have separate configuration for Coordination, Master and Data nodes to have flexibility in assiging JVM memory to these instances. Coordination and Master nodes usually required less memory than Data nodes.

```shell 
docker config create jvm-options-coordination jvm.options
docker config create jvm-options-data jvm.options
docker config create jvm-options-master jvm.options
```

```
ID                          NAME                       CREATED             UPDATED
nyh52lw5lgdixdlsi1fol67wv   es-coordination            6 minutes ago       6 minutes ago
owfgm5s14igyxkpn9yr619w6y   es-data                    5 minutes ago       5 minutes ago
jmhz9yac68wa73kmfhw1cu254   es-master                  5 minutes ago       5 minutes ago
7v76ed3ahe119t2li3wxumjj4   jvm-options-coordination   3 minutes ago       3 minutes ago
qhcytwr88axtklm4b7as56eur   jvm-options-data           3 minutes ago       3 minutes ago
mzj666cjkjkugfuly9z905026   jvm-options-master         3 minutes ago       3 minutes ago
```

# Deploying stack 
Than deploy a stack by executing the following command:

```shell 
docker stack deploy -c stack-elastic.yml es
```

# Validating the status of Elasticsearch cluster
After a few minutes you should be able to get cluster state and list of nodes consisting the Elasticsearch cluster. 

```shell
curl ${IP_ADDRESS}:9200/_cluster/health?pretty
```
```json 
{
  "cluster_name" : "docker-swarm-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 11,
  "number_of_data_nodes" : 4,
  "active_primary_shards" : 0,
  "active_shards" : 0,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```

# Listing of nodes consisting Elasticsearch Cluster

```shell
curl ${IP_ADDRESS}:9200/_cat/node
```
```
10.0.1.2  27 97 45 2.90 1.39 0.52 - - jeZY0C4
10.0.1.11 44 78 33 1.57 0.82 0.31 - - TtYZYAH
10.0.1.14 28 57 24 0.84 0.43 0.16 d - cCKt4Dd
10.0.1.6  30 97 44 2.90 1.39 0.52 m - g_QL8du
10.0.1.4  40 97 43 2.90 1.39 0.52 d - 5j_gCcq
10.0.1.8  27 97 43 2.90 1.39 0.52 m - vOmxG79
10.0.1.17 43 57 23 0.88 0.39 0.14 - - Af3r11-
10.0.1.15 28 57 24 0.84 0.43 0.16 - - 1DGS4Xd
10.0.1.9  26 78 33 1.57 0.82 0.31 d - QEWrVND
10.0.1.10 35 78 33 1.57 0.82 0.31 m * XugNq4c
10.0.1.19 45 57 23 0.88 0.39 0.14 d - vSI0shW
```

Due to service mesh feature built-in Docker Swarm you choose any of the IP addresses of your nodes. Docker Swarm will route the request to the appropriate coordination node running on virtual machines being a member of Docker Swarm cluster. 

# Listing of running stacks
```shell
docker stack ls
```
```
NAME                SERVICES            ORCHESTRATOR
elastic             7                   Swarm
```
# Listing of running services and appropriate numbers of replicas

```shell
docker service ls
```

```
ID                  NAME                   MODE                REPLICAS            IMAGE                             PORTS
4ygj2afsednm        elastic_coordination     global              4/4                 elasticsearch:6.5.3
c3xgunhn6c7d        elastic_data1            replicated          1/1                 elasticsearch:6.5.3
nz4k8f4zvkv6        elastic_data2            replicated          1/1                 elasticsearch:6.5.3
7d3wjyi5s083        elastic_data3            replicated          1/1                 elasticsearch:6.5.3
oqq9q0vy6t9q        elastic_data4            replicated          1/1                 elasticsearch:6.5.3
uszn9mxmeubh        elastic_master           replicated          3/3                 elasticsearch:6.5.3
5o3usc3ncbb5        elastic_visualizer       replicated          1/1                 dockersamples/visualizer:latest   *:8080->8080/tcp
```
Please note that service coordination has to replicate mode set to `global`, it means that we are requesting to have Elasticsearch coordination instance running on each of the node consisting Docker Swarm cluster.  Other services have defined replicas as an integer. 

# Listing of specific task running on a particular node
```shell
docker service ps elastic_coordination
```

```
ID                  NAME                                             IMAGE                 NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
y6lfnbnavy7z        elastic_coordination.yqoycyrs9j0cb1me7cwr77764   elasticsearch:6.5.3   node-3              Running             Running 2 minutes ago                       *:9200->9200/tcp
1f1xk71zug9z        elastic_coordination.iqepxq2w46nprlgm55gomf1ic   elasticsearch:6.5.3   node-1              Running             Running 2 minutes ago                       *:9200->9200/tcp
fpu2bdmnnfl2        elastic_coordination.56yflx8vy47oi3upycouzpjpj   elasticsearch:6.5.3   node-4              Running             Running 2 minutes ago                       *:9200->9200/tcp
l8lozi001l2l        elastic_coordination.1rruyl7x9s9x43rql0x2jibx0   elasticsearch:6.5.3   node-2              Running             Running 2 minutes ago                       *:9200->9200/tcp
```

