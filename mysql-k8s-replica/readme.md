# MySQL Group Replication in Kubernetes

With evolvement of Kubernetes we are seeing emerging different tools, patterns, CI/CD methodologies for containerised services now more than ever. But can we see that fast phased kubernetes containerised movement in data layer for databases? Probably, not and there’s a reason for it.

Inherently kubernetes is designed to deploy stateless services, but databases on other hand are stateful services which needs to maintain high availability for the stateless services to operate optimally. Therefore, its always challenging to deploy and maintain database service in a distributed environment such as kubernetes cluster.

Further, importance of high availability of a database in kubernetes environment is also very crucial. Because most real world services heavily depend on database(s) to maintain ‘states’ for stateless services. Specially in Saga/Database patterns in microservice paradigm where each service solely depend on its datasource as ‘only source of truth’. If this datasource have only one replica there’s a possibility of running into unresolvable data inconsistency state (specially in microservices) if a one datasource fails to handle critical transactions.

There are several options available as distributed mysql solution in k8s. Following are the most popular options to date.

### Galera  
first open source solution developed by company named Codership to provide mysql virtually synchronised replica solution.

### Percona 
this is another open source flavour of Galera implementation. It origins from Galera, but later morphed into its own version.
### MySQL Group Replication  
Built into mysql as a plugin, this was GA released (stable) in 2016.Bit late to distributed solution race (compared to galera) but still a strong contender, as its built-in to mysql and no need of any external resources to manage the replica members. By now it has evolved into more mature, out of the box, production ready distributed solution for mysql.
There are many pros and cons for each of this mysql distributed solutions. I’m yet to discover all of these (may be a topic for later article). But for now i’m leaning towards mysql group replication as its available with trusty mysql out of the box with production ready stamp on it.

Further, one of main motivation to write this blog is, there’s no much materials on how to deploy fully automated robust kubernetes mysql distributed group replication setup from scratch on internet.

#### Part I: Objectives and Approach
Before dive into implementation, let me explain end objectives:

- Setup Mysql 5.7 group replication in k8s with fully automated deployment (no manual configs).
* All mysql nodes must able to accept read/writes (multi master mysql setup).
+ Any mysql pod can go unresponsive at any given time (deployment should able to self heal with no data corruptions).
- Minimum read/write service interruptions for basic failure scenarios.
* No 3rd party tools or solutions (vanilla mysql 5.7, vanilla k8s setup — no sidecars — no magic configurations :) )
 For this demo implementation, i’ll consider two node mysql setup for clarity. With little bit of k8s config knowledge this can be parameterised and scaled upto 9 replicas (mysql group replication limit).


### High Level Architecture
For demonstration two mysql nodes were used to keep the deployment simple. But it’s recommended to use minimum of 3 members in a group to achieve true advantage of group replication (specially to handle split brain scenarios with quorum vote). Once familiar with two member setup, its easy to extend number of members based on the usecase.

#### Part II : Implementation
Skip to “Kubernetes Configurations :” section if you are impatient and need to deploy the cluster quickly.

#### Setup :
- Kubernetes v1.20
+ MySQL v5.7.35 — mysql/mysql-server:5.7.35 (latest official mysql 5.x docker image by the time writing this blog)
Demo deployment is configured to deploy in “default” namespace. This can be changed via hostname configurations in my.cnf as explained later on.

#### MySQL Group Replication Configuration Summary :
This section categorises minimum configurations needed to get the MySQL group replication up and running. This categorisation will be the base for k8s implementation in next section.

Official mysql group replication documentation is comprehensive and provides lots of details on each configuration.

All mysql configurations are applied in “/etc/my.cnf”. These configurations can be separated into following classifications :

1. Group configurations — Common configurations to the given replication group
```
    # General replication settings
    gtid_mode = ON
    enforce_gtid_consistency = ON
    master_info_repository = TABLE
    relay_log_info_repository = TABLE
    binlog_checksum = NONE
    log_slave_updates = ON
    log_bin = binlog
    binlog_format = ROW
    transaction_write_set_extraction = XXHASH64
    loose-group_replication_start_on_boot = ON
    #loose-group_replication_ssl_mode = REQUIRED
    #loose-group_replication_recovery_use_ssl = 1

    # Shared replication group configuration
    loose-group_replication_group_name = "85cbd4a0-7338-46f1-b15e-28c1a26f465e"
    loose-group_replication_ip_whitelist = "mysql-0.mysql.default.svc.cluster.local,mysql-1.mysql.default.svc.cluster.local"
    loose-group_replication_group_seeds = "mysql-0.mysql.default.svc.cluster.local:33061,mysql-1.mysql.default.svc.cluster.local:33061"

    # Single or Multi-primary mode? Uncomment these two lines
    # for multi-primary mode, where any host can accept writes
    loose-group_replication_single_primary_mode = OFF
    loose-group_replication_enforce_update_everywhere_checks = ON

```

2. Member specific configurations — Specific configurations to the given member.
```
    server_id = 1
    server-id = 1
    bind-address = "mysql-0.mysql.default.svc.cluster.local"
    report_host = "mysql-0.mysql.default.svc.cluster.local"
    loose-group_replication_local_address = "mysql-0.mysql.default.svc.cluster.local:33061"
```

These are unique values for each member node. For each member node need to add these configurations separately.

3. Dynamic configurations — Dynamic configurations are conditional which needs to be applied based on availability of the mysql cluster.

```
## Bootstrap if this member node is the first node joining the group
loose-group_replication_bootstrap_group = ON

```
*my.cnf — dynamic configuration based on group status — if no member available


```
## If members available already do not bootstrap
loose-group_replication_bootstrap_group = OFF
```
*my.cnf — dynamic configuration based on group status — if at-least one group member available



Bootstrap needs to be done only once, when the first node boots up in the group. All subsequent member nodes should start with bootstrap off to join already running group.

Above configuration categorisation allows us to get them deployed in k8s cluster with much clarity. How to deploy these, will be explained in next section.

#### Kubernetes Configuration :
Following are the k8s configurations for two member group replication cluster. It uses several popular k8s features together to to pull it off.


+ configmap.yaml (To maintain mysql member configs and init container configs,logics)

- Statefulset.yaml

* service.yaml


To apply these configs make sure your k8s cluster is running and kubectl is configured correctly.

Afterwards just download above 3 configs to an empty directory and apply via kubectl.

`kubectl apply -f .`

Thats it! two node mysql GR cluster will get deployed to ‘default’ namespace in the cluster.

#### Part III : Testing
This is just a two node mysql cluster to demonstrate the GR cluster implementation. This can be extended upto 9 nodes with small tweaks to the configuration.

Once both mysql nodes are up and running execute below query in both mysql nodes to confirm both nodes are visible to each other.

` SELECT * FROM performance_schema.replication_group_members;`
Use below command to execute mysql queries directly from kubectl. (change pod name from mysql-0 to mysql-1 to access mysql-1)

``` 
kubectl exec --stdin --tty mysql-0 -- /bin/sh -c "mysql -uroot -proot -e 'SELECT * FROM performance_schema.replication_group_members;'" 
```
two mysql pods are up and running

![aaa](https://user-images.githubusercontent.com/24589033/229521259-ec8aa9d3-83df-4a0e-a28a-5a4686850408.jpg)

Replication group member list in both pods

When both replication group members visible in each node with ONLINE status, it confirms MySQL group replication cluster is healthy any up and running.

![bbb](https://user-images.githubusercontent.com/24589033/229521542-467a0ada-dac0-40d2-b1dd-8e509558def3.jpg)


To test if this MySQL cluster work as expected in failure events, i’ve created a small mysql client pod to simulate database activity.

+ test.yaml

This script will increment value continuously(one insert per each 500ms) in a test table data column. if there’s any mismatches from previous value it’ll add NOK to the status column.


I have deleted the mysql pods randomly to verify if it can failover successfully (Please refer above gif to understand the how I did it) and it worked flawlessly. When a pod was deleted mysql writes weren’t interrupted and both pods were able to sync once new MySQL pod was available again.

Note that i haven’t used any persistent volumes (PVs) for the pods in this demo, hence, recovery took few seconds to finish the data sync. This is because newly spawned pod had to sync full database from scratch (as it didn’t have a PV attached). If we had separate PVs attached for each node recovery will be much faster.

This concludes the test and confirms MySQL group replication is able to handle inbound writes as well as read requests during pod failure scenarios without any database service interruptions.

On final note; if you are a business user whom needs a high available business solution deployed in kubernetes, but of cause doesn’t want to handle any of the technical hassle, Connext GO is the solution for you. Connext GO allows you to build and deploy business solutions on top of kubernetes platforms without managing any of Kubernetes cluster internals.


source: https://medium.com/@pumudu88/mysql-group-replication-in-kubernetes-3fb770b0eac4
