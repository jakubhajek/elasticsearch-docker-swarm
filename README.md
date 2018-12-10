# Initial requirements

This is example configuration which launches *Elasticsearch Cluster* on *Docker Swarm Cluster*. The entire setup distinguishes following services::

- service with coordination Elasticsearch node role enabled which basically acts like a load balnacer
- service with Elasticsearch master eligible nodes
- service with Elasticsearch data nodes responsible for CRUD operations
- service visualizer which is presenting how containers are spread acorss the Docker Swarm Cluster

In order to run it I recommend to have at least 3 VM's provisioned. You can try AWS or even virtualbox running on your local machine. The easiest way is to setup servers using docker-machine. In this example I am using my AWS account (you can specify amazonec2-access-key and amazonec2_security_key in command line or use it ~/.aws/credentials and defined default profile) 

# Provisioning virtual machines
Execute the following command to provision servers:

`docker-machine create --driver amazonec2 --amazonec2-region eu-central-1 --amazonec2-instance-type t2.medium --amazonec2-security-group swarm-sg --amazonec2-open-port 9200/tcp --amazonec2-open-port 9300/tcp --amazonec2-open-port 2377/tcp --amazonec2-open-port 7946/tcp --amazonec2-open-port 4789/udp --amazonec2-open-port 7946/udp node-1`

The given command should be executed 3 times at least, make sure you change hostname every time you are executing the command,the last argument in the command.
Please note that created AWS security group is only for demonstration purposes, you never should expose crucial ports to the internet. 

# Tuning virtual machines
Before running a stack, minor changes have to be adopted on each of newaly created Vm's, 
1. Edit /etc/sysctl.conf by adding following `vm.max_map_count=262144` or just simple execute `docker-machine ssh node-1 sudo sysctl -w vm.max_map_count=262144` for each of the node. 
2. Edit file `/etc/systemd/system/docker.service.d/10-machine.conf` and modify default ulimit because Elasticsearch cluster is requesting memory locking (see `bootstrap.memory_lock=true`), thus following parameter to ExecStart `--default-ulimit memlock=-1`. You can simple execute that command on the each of the node: 

`docker-machine ssh node-11 sudo "sed -i '/ExecStart=\/usr\/bin\/dockerd/ s/$/--default-ulimit memlock=-1/' /etc/systemd/system/docker.service.d/10-machine.conf"`

Please note that location of that file might be different depending of your Linux distribution. That one works with Ubuntu 16.04 LTS.
After that execute on each of the node `systemctl daemon-reload` and restart Docker daemon `systemctl restart docker`

# Initiating Docker Swarm cluster
Once vm's are created you can initiate Docker swarm cluster. Export environment variables belonging to the one of the nodes:

`eval $(docker-machine env node-1)`

and initiate a cluster: 

`docker swarm init`

then connect to the other vm's and add them accordingly to the cluster, for test purposes you can just add only workers to the cluster. For production environment pleaae consider the appropriate amount of manager nodes to keep RAFT database secure and provide high availability for your Docker Swarm Cluster. 

Make sure the cluster is working by executine command:

# Listing of the nodes consisting Docker Swarm cluster
`docker node ls`
```
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
iqepxq2w46nprlgm55gomf1ic *   node-1              Ready               Active              Leader              18.09.0
1rruyl7x9s9x43rql0x2jibx0     node-2              Ready               Active                                  18.09.0
yqoycyrs9j0cb1me7cwr77764     node-3              Ready               Active                                  18.09.0
56yflx8vy47oi3upycouzpjpj     node-4              Ready               Active                                  18.09.0
```

# Deploying stack 
Than deploy a stack by executing the following command:

`docker stack deploy -c docker-compose.yml es`

# Validating the status of Elasticsearch cluster
After a few minutes you should be able to get cluster state and list of nodes consisting the Elasticsearch cluster. 

`curl ${IP_ADDRESS}:9200/_cluster/__health?pretty`
```json {
  "cluster_name" : "docker-cluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 7,
  "number_of_data_nodes" : 1,
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

`curl ${IP_ADDRESS}:9200/__cat/node`

```
10.0.0.22 45 56 1 0.03 0.29 0.19 m - fC8ghBT
10.0.0.11 42 55 2 0.46 0.71 0.43 m * xiw-R34
10.0.0.2  39 33 1 0.05 0.53 0.35 - - PPjPo-g
10.0.0.3  49 55 2 0.12 0.42 0.29 - - upCdP2m
10.0.0.5  39 55 2 0.46 0.71 0.43 - - DPCQgwz
10.0.0.12 36 55 2 0.12 0.42 0.29 d - ybt0Jgt
10.0.0.4  38 56 1 0.03 0.29 0.19 - - lmmnRLo
```

Due to service mesh feature built-in Docker Swarm you choose any of the IP addresses of your nodes. Docker Swarm will route the request to the appropriate coordination node running on virtual machines being a member of Docker Swarm cluster. 

# Listing of running stacks
`docker stack ls`
NAME                SERVICES            ORCHESTRATOR
elastic             4                   Swarm

# Listing of running services and appropriate numbers of replicas

`docker service ls`
```
ID                  NAME                   MODE                REPLICAS            IMAGE                             PORTS
yxnunajngch0        elastic_coordination   global              4/4                 elasticsearch:6.5.2
ilh21fuh116z        elastic_data           replicated          1/1                 elasticsearch:6.5.2
x7zzznodcql9        elastic_master         replicated          2/2                 elasticsearch:6.5.2
lbttfnzojei4        elastic_visualizer     replicated          1/1                 dockersamples/visualizer:latest   *:8080->8080/tcp
```
Please note that service coordination has to replicate mode set to `global`, it means that we are requesting to have Elasticsearch coordination instance running on each of the node consisting Docker Swarm cluster.  Other services have defined replicas as an integer. 

# Listing of specific task running on a particular node
`docker service ps elastic_coordination`
```
ID                  NAME                                             IMAGE                 NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
y6lfnbnavy7z        elastic_coordination.yqoycyrs9j0cb1me7cwr77764   elasticsearch:6.5.2   node-3              Running             Running 2 minutes ago                       *:9200->9200/tcp
1f1xk71zug9z        elastic_coordination.iqepxq2w46nprlgm55gomf1ic   elasticsearch:6.5.2   node-1              Running             Running 2 minutes ago                       *:9200->9200/tcp
fpu2bdmnnfl2        elastic_coordination.56yflx8vy47oi3upycouzpjpj   elasticsearch:6.5.2   node-4              Running             Running 2 minutes ago                       *:9200->9200/tcp
l8lozi001l2l        elastic_coordination.1rruyl7x9s9x43rql0x2jibx0   elasticsearch:6.5.2   node-2              Running             Running 2 minutes ago                       *:9200->9200/tcp
```
# Scaling up Elasticsearch cluster by adding extra data nodes

Scaling up Elasticsearch cluster can be achieved by executing the following command:

`docker service scale elastic_data=3`

which means that we are requesting to scaling up to 3 data nodes. 

`docker service ls --filter name=elastic_data`
```
ID                  NAME                MODE                REPLICAS            IMAGE                 PORTS
ilh21fuh116z        elastic_data        replicated          3/3                 elasticsearch:6.5.2
```

`docker service ps elastic_data`
```
ID                  NAME                IMAGE                 NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
pd0d7xqf3bee        elastic_data.1      elasticsearch:6.5.2   node-3              Running             Running 45 minutes ago
vsp6k75b0132        elastic_data.2      elasticsearch:6.5.2   node-4              Running             Running 8 minutes ago
vmxgv4axarsc        elastic_data.3      elasticsearch:6.5.2   node-2              Running             Running 8 minutes ago
```

