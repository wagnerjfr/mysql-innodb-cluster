# mysql-innodb-cluster
Setting up [MySQL InnoDB Cluster](https://dev.mysql.com/doc/refman/8.0/en/mysql-innodb-cluster-userguide.html) with [MySQL Shell](https://dev.mysql.com/doc/mysql-shell/8.0/en/) (plus [MySQL Router](https://dev.mysql.com/doc/mysql-router/8.0/en/)) using just Docker containers.

**MySQL InnoDB cluster** provides a complete high availability solution for MySQL. Each MySQL server instance runs [MySQL Group Replication](https://dev.mysql.com/doc/refman/8.0/en/group-replication.html), which provides the mechanism to replicate data within InnoDB clusters, with built-in failover.

P.S. Take a look at this [tutorial](https://github.com/wagnerjfr/mysql-group-replication-docker) with you want to see how is to setup MySQL Group Replication just using Docker containers.

The following tutorial steps will lead us to have a final result like this:

![alt text](https://github.com/wagnerjfr/mysql-innodb-cluster/blob/master/img/figure1.png)

## 1. Launch the MySQL 8 containers and grant user access to them

#### Create a Docker network
`$ docker network create innodbnet`

#### Launch 4 MySQL 8 containers
```
for N in 1 2 3 4
  do docker run -d --name=mysql$N --hostname=mysql$N --net=innodbnet \
      -e MYSQL_ROOT_PASSWORD=root mysql/mysql-server:8.0
done
```
Wait for them to be with `healthy` status

It's possible to be checked running `docker ps -a`
```console
CONTAINER ID        IMAGE                    COMMAND                  CREATED              STATUS                        PORTS                 NAMES
c5629922be3b        mysql/mysql-server:8.0   "/entrypoint.sh mysq…"   About a minute ago   Up About a minute (healthy)   3306/tcp, 33060/tcp   mysql4
f3bb82dbef99        mysql/mysql-server:8.0   "/entrypoint.sh mysq…"   About a minute ago   Up About a minute (healthy)   3306/tcp, 33060/tcp   mysql3
f0217f84abb4        mysql/mysql-server:8.0   "/entrypoint.sh mysq…"   About a minute ago   Up About a minute (healthy)   3306/tcp, 33060/tcp   mysql2
4b35e3b33855        mysql/mysql-server:8.0   "/entrypoint.sh mysq…"   About a minute ago   Up About a minute (healthy)   3306/tcp, 33060/tcp   mysql1
```

#### Grant new user access
Let's create a *user* named **inno** with *password* with the same value **inno** and grant it the required access in each of the MySQL servers.
```
for N in 1 2 3 4
do docker exec -it mysql$N mysql -uroot -proot \
  -e "CREATE USER 'inno'@'%' IDENTIFIED BY 'inno';" \
  -e "GRANT ALL privileges ON *.* TO 'inno'@'%' with grant option;" \
  -e "reset master;"
done
```
Check whether the users were created successfuly:
```
for N in 1 2 3 4
do docker exec -it mysql$N mysql -uinno -pinno \
  -e "SHOW VARIABLES WHERE Variable_name = 'hostname';" \
  -e "SELECT user FROM mysql.user where user = 'inno';"
done
```
This output is expected:
```console
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+--------+
| Variable_name | Value  |
+---------------+--------+
| hostname      | mysql1 |
+---------------+--------+
+------+
| user |
+------+
| inno |
+------+
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+--------+
| Variable_name | Value  |
+---------------+--------+
| hostname      | mysql2 |
+---------------+--------+
+------+
| user |
+------+
| inno |
+------+
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+--------+
| Variable_name | Value  |
+---------------+--------+
| hostname      | mysql3 |
+---------------+--------+
+------+
| user |
+------+
| inno |
+------+
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+--------+
| Variable_name | Value  |
+---------------+--------+
| hostname      | mysql4 |
+---------------+--------+
+------+
| user |
+------+
| inno |
+------+
```

## 2. Configure the MySQL servers to join InnoDB Cluster

Run the command below to access the MySQL Shell of container **mysql1**.
```
$ docker exec -it mysql1 mysqlsh -uroot -proot -S/var/run/mysqld/mysqlx.sock
```
![alt text](https://github.com/wagnerjfr/mysql-innodb-cluster/blob/master/img/figure2.png)

#### Is the MySQL instance ready for InnoDB Cluster?

Let's verify that by running `dba.checkInstanceConfiguration()` which will check the configuration of each instance.

Run the command below, following by typing the password `inno` to check configuration in **mysql1**:
```
dba.checkInstanceConfiguration("inno@mysql1:3306")
```
The expected output is:
```console
 MySQL  localhost+ ssl  JS > dba.checkInstanceConfiguration("inno@mysql1:3306")
Please provide the password for 'inno@mysql1:3306': ****
Validating local MySQL instance listening at port 3306 for use in an InnoDB cluster...

This instance reports its own address as mysql1:3306
Clients and other cluster members will communicate with it through this address by default. If this is not correct, the report_host MySQL system variable should be changed.

Checking whether existing tables comply with Group Replication requirements...
No incompatible tables detected

Checking instance configuration...

NOTE: Some configuration options need to be fixed:
+--------------------------+---------------+----------------+--------------------------------------------------+
| Variable                 | Current Value | Required Value | Note                                             |
+--------------------------+---------------+----------------+--------------------------------------------------+
| binlog_checksum          | CRC32         | NONE           | Update the server variable                       |
| enforce_gtid_consistency | OFF           | ON             | Update read-only variable and restart the server |
| gtid_mode                | OFF           | ON             | Update read-only variable and restart the server |
| server_id                | 1             | <unique ID>    | Update read-only variable and restart the server |
+--------------------------+---------------+----------------+--------------------------------------------------+

Some variables need to be changed, but cannot be done dynamically on the server.
NOTE: Please use the dba.configureInstance() command to repair these issues.

{
    "config_errors": [
        {
            "action": "server_update", 
            "current": "CRC32", 
            "option": "binlog_checksum", 
            "required": "NONE"
        }, 
        {
            "action": "server_update+restart", 
            "current": "OFF", 
            "option": "enforce_gtid_consistency", 
            "required": "ON"
        }, 
        {
            "action": "server_update+restart", 
            "current": "OFF", 
            "option": "gtid_mode", 
            "required": "ON"
        }, 
        {
            "action": "server_update+restart", 
            "current": "1", 
            "option": "server_id", 
            "required": "<unique ID>"
        }
    ], 
    "status": "error"
}
```
You can see that **mysql1** needs to have some of its configuration changed.

It's not necessary to run the commands below, but if you want to try, the same can be done and observed when running:
```
dba.checkInstanceConfiguration("inno@mysql2:3306")
```
```
dba.checkInstanceConfiguration("inno@mysql2:3306")
```
```
dba.checkInstanceConfiguration("inno@mysql4:3306")
```

Similar outputs are expected.

#### Configure MySQL Servers for InnoDB Cluster

We saw in the previous section that the MySQL instances needs to be configured.

That can be done with just one command: `dba.configureInstance()`.

Let's run the command below to configure **mysql1**:
```
dba.configureInstance("inno@mysql1:3306")
```

Type the password `inno` and `y` (yes) for the 2 questions when asked.

Expected output:
```console
 MySQL  localhost+ ssl  JS > dba.configureInstance("inno@mysql1:3306")
Please provide the password for 'inno@mysql1:3306': ****
Configuring local MySQL instance listening at port 3306 for use in an InnoDB cluster...

This instance reports its own address as mysql1:3306
Clients and other cluster members will communicate with it through this address by default. If this is not correct, the report_host MySQL system variable should be changed.

NOTE: Some configuration options need to be fixed:
+--------------------------+---------------+----------------+--------------------------------------------------+
| Variable                 | Current Value | Required Value | Note                                             |
+--------------------------+---------------+----------------+--------------------------------------------------+
| binlog_checksum          | CRC32         | NONE           | Update the server variable                       |
| enforce_gtid_consistency | OFF           | ON             | Update read-only variable and restart the server |
| gtid_mode                | OFF           | ON             | Update read-only variable and restart the server |
| server_id                | 1             | <unique ID>    | Update read-only variable and restart the server |
+--------------------------+---------------+----------------+--------------------------------------------------+

Some variables need to be changed, but cannot be done dynamically on the server.
Do you want to perform the required configuration changes? [y/n]: y
Do you want to restart the instance after configuring it? [y/n]: y
Configuring instance...
The instance 'mysql1:3306' was configured for InnoDB cluster usage.
Restarting MySQL...
ERROR: Remote restart of MySQL server failed: MySQL Error 3707 (HY000): Restart server failed (mysqld is not managed by supervisor process).
Please restart MySQL manually
```

Don't worry about the error message `ERROR: Remote restart of MySQL server failed` for now. We will restart all the containers later.

Let's configure **mysql2**, **mysql3** and **mysql4**, by running the commands below individually:
```
dba.configureInstance("inno@mysql2:3306")
```
```
dba.configureInstance("inno@mysql3:3306")
```
```
dba.configureInstance("inno@mysql4:3306")
```

So far so good.. Let's exit the MySQL Shell terminal. Type `\exit` and press ENTER.

In the OS terminal, restart the four MySQL 8 containers:
```
$ docker restart mysql1 mysql2 mysql3 mysql4
```
Wait for the four MySQL 8 containers to be with status `healthy` before continuing.

## 3. Create the InnoDB Cluster

Run the command below to access the MySQL Shell of **mysql1** container.
```
$ docker exec -it mysql1 mysqlsh -uroot -proot -S/var/run/mysqld/mysqlx.sock
```

Connect to the seed instance and type the password `inno`:
```
\c inno@mysql1:3306
```

Create the cluster:
```
var cluster = dba.createCluster("mycluster")
```

Expected output:
```console
 MySQL  mysql1:3306 ssl  JS > var cluster = dba.createCluster("mycluster")
A new InnoDB cluster will be created on instance 'mysql1:3306'.

Validating instance at mysql1:3306...

This instance reports its own address as mysql1:3306

Instance configuration is suitable.
Creating InnoDB cluster 'mycluster' on 'mysql1:3306'...

Adding Seed Instance...
Cluster successfully created. Use Cluster.addInstance() to add MySQL instances.
At least 3 instances are needed for the cluster to be able to withstand up to
one server failure.
```
Execute the command `cluster.status()` to check cluster status:

Output:
```console
 MySQL  mysql1:3306 ssl  JS > cluster.status()
{
    "clusterName": "mycluster", 
    "defaultReplicaSet": {
        "name": "default", 
        "primary": "mysql1:3306", 
        "ssl": "REQUIRED", 
        "status": "OK_NO_TOLERANCE", 
        "statusText": "Cluster is NOT tolerant to any failures.", 
        "topology": {
            "mysql1:3306": {
                "address": "mysql1:3306", 
                "mode": "R/W", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.17"
            }
        }, 
        "topologyMode": "Single-Primary"
    }, 
    "groupInformationSourceMember": "mysql1:3306"
}
```

See its description, run `cluster.describe()`:

Output:
```console
 MySQL  mysql1:3306 ssl  JS > cluster.describe()
{
    "clusterName": "mycluster", 
    "defaultReplicaSet": {
        "name": "default", 
        "topology": [
            {
                "address": "mysql1:3306", 
                "label": "mysql1:3306", 
                "role": "HA", 
                "version": "8.0.17"
            }
        ], 
        "topologyMode": "Single-Primary"
    }
}
```
Up to now, the cluster has just one node. In the next section we will add more three nodes.

## 4. Add the other MySQL servers to the InnoDB Cluster

MySQL server **mysql2**, **mysql3** and **mysql4** will be added to the cluster.

In MySQL Shell run the the commands below followed by its password `inno`:
```
cluster.addInstance("inno@mysql2:3306")
```
The question `Please select a recovery method [C]lone/[I]ncremental recovery/[A]bort (default Clone):` will be displayed.

Choose the option `[I]ncremental`, so type `I` and the press ENTER.

The expected output is:
```console
 MySQL  mysql1:3306 ssl  JS > cluster.addInstance("inno@mysql2:3306")
Please provide the password for 'inno@mysql2:3306': ****

NOTE: The target instance 'mysql2:3306' has not been pre-provisioned (GTID set is
empty). The Shell is unable to decide whether incremental distributed state
recovery can correctly provision it.
The safest and most convenient way to provision a new instance is through
automatic clone provisioning, which will completely overwrite the state of
'mysql2:3306' with a physical snapshot from an existing cluster member. To use
this method by default, set the 'recoveryMethod' option to 'clone'.

The incremental distributed state recovery may be safely used if you are sure
all updates ever executed in the cluster were done with GTIDs enabled, there
are no purged transactions and the new instance contains the same GTID set as
the cluster or a subset of it. To use this method by default, set the
'recoveryMethod' option to 'incremental'.


Please select a recovery method [C]lone/[I]ncremental recovery/[A]bort (default Clone): I
Validating instance at mysql2:3306...

This instance reports its own address as mysql2:3306

Instance configuration is suitable.
A new instance will be added to the InnoDB cluster. Depending on the amount of
data on the cluster this might take from a few seconds to several hours.

Adding instance to the cluster...

Monitoring recovery process of the new cluster member. Press ^C to stop monitoring and let it continue in background.
Incremental distributed state recovery is now in progress.

* Waiting for distributed recovery to finish...
NOTE: 'mysql2:3306' is being recovered from 'mysql1:3306'
* Distributed recovery has finished

The instance 'mysql2:3306' was successfully added to the cluster.
```

Do the same for **mysql3** and **mysql4**.
```
cluster.addInstance("inno@mysql3:3306")
```
```
cluster.addInstance("inno@mysql4:3306")
```
Let's see the status and description of our cluster now:
```
cluster.status()
```
Output:
```console
 MySQL  mysql1:3306 ssl  JS > cluster.status()
{
    "clusterName": "mycluster", 
    "defaultReplicaSet": {
        "name": "default", 
        "primary": "mysql1:3306", 
        "ssl": "REQUIRED", 
        "status": "OK", 
        "statusText": "Cluster is ONLINE and can tolerate up to ONE failure.", 
        "topology": {
            "mysql1:3306": {
                "address": "mysql1:3306", 
                "mode": "R/W", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.17"
            }, 
            "mysql2:3306": {
                "address": "mysql2:3306", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.17"
            }, 
            "mysql3:3306": {
                "address": "mysql3:3306", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.17"
            }, 
            "mysql4:3306": {
                "address": "mysql4:3306", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.17"
            }
        }, 
        "topologyMode": "Single-Primary"
    }, 
    "groupInformationSourceMember": "mysql1:3306"
}
```
```
cluster.describe()
````
Output:
```console
 MySQL  mysql1:3306 ssl  JS > cluster.describe()
{
    "clusterName": "mycluster", 
    "defaultReplicaSet": {
        "name": "default", 
        "topology": [
            {
                "address": "mysql1:3306", 
                "label": "mysql1:3306", 
                "role": "HA", 
                "version": "8.0.17"
            }, 
            {
                "address": "mysql2:3306", 
                "label": "mysql2:3306", 
                "role": "HA", 
                "version": "8.0.17"
            }, 
            {
                "address": "mysql3:3306", 
                "label": "mysql3:3306", 
                "role": "HA", 
                "version": "8.0.17"
            }, 
            {
                "address": "mysql4:3306", 
                "label": "mysql4:3306", 
                "role": "HA", 
                "version": "8.0.17"
            }
        ], 
        "topologyMode": "Single-Primary"
    }
}
```

Our InnoDB Cluster with four nodes is up and running.

Type `\exit` and press ENTER to leave the Shell and go to the OS terminal.

Next section we will bootstrap a MySQL Router which will help load balance the traffic to the cluster.

## 5. MySQL Router Bootstrap

We are going to launch a new [MySQL Router Docker image](https://hub.docker.com/r/mysql/mysql-router) as a container:
```
$ docker run -d --name mysql-router --net=innodbnet \
   -e MYSQL_HOST=mysql1 \
   -e MYSQL_PORT=3306 \
   -e MYSQL_USER=inno \
   -e MYSQL_PASSWORD=inno \
   -e MYSQL_INNODB_CLUSTER_MEMBERS=4 \
   mysql/mysql-router
```
Check it was correctly launched:
```
$ docker logs mysql-router
```
Expected output:
```console
wfranchi@computer:~/GitHub/InnoDBCluster$ docker logs mysql-router 
Succesfully contacted mysql server at mysql1. Checking for cluster state.
0
12
Successfully contacted cluster with 4 members. Bootstrapping.
Succesfully contacted mysql server at mysql1. Trying to bootstrap.
Please enter MySQL password for inno: 
# Bootstrapping MySQL Router instance at '/tmp/mysqlrouter'...

- Checking for old Router accounts
  - No prior Router accounts found
- Creating mysql account mysql_router1_q53qqa0v8ta6@'%' for cluster management
- Storing account in keyring
- Adjusting permissions of generated files
- Creating configuration /tmp/mysqlrouter/mysqlrouter.conf

# MySQL Router configured for the InnoDB cluster 'mycluster'

After this MySQL Router has been started with the generated configuration

    $ mysqlrouter -c /tmp/mysqlrouter/mysqlrouter.conf

the cluster 'mycluster' can be reached by connecting to:

## MySQL Classic protocol

- Read/Write Connections: localhost:6446
- Read/Only Connections:  localhost:6447

## MySQL X protocol

- Read/Write Connections: localhost:64460
- Read/Only Connections:  localhost:64470

Starting mysql-router.
Loading all plugins.
  plugin 'logger:' loading
  plugin 'metadata_cache:mycluster' loading
  plugin 'routing:mycluster_default_ro' loading
  plugin 'routing:mycluster_default_rw' loading
  plugin 'routing:mycluster_default_x_ro' loading
  plugin 'routing:mycluster_default_x_rw' loading
Initializing all plugins.
  plugin 'logger' initializing
logging facility initialized, switching logging to loggers specified in configuration
2019-07-27 10:20:10 main INFO [7f9696c9e880]   plugin 'metadata_cache' initializing
2019-07-27 10:20:10 main INFO [7f9696c9e880]   plugin 'routing' initializing
2019-07-27 10:20:10 main INFO [7f9696c9e880] Starting all plugins.
2019-07-27 10:20:10 main INFO [7f9692691700]   plugin 'metadata_cache:mycluster' starting
2019-07-27 10:20:10 main INFO [7f9691e90700]   plugin 'routing:mycluster_default_ro' starting
2019-07-27 10:20:10 main INFO [7f969168f700]   plugin 'routing:mycluster_default_rw' starting
2019-07-27 10:20:10 routing INFO [7f9691e90700] [routing:mycluster_default_ro] started: listening on 0.0.0.0:6447, routing strategy = round-robin-with-fallback
2019-07-27 10:20:10 main INFO [7f9690e8e700]   plugin 'routing:mycluster_default_x_ro' starting
2019-07-27 10:20:10 routing INFO [7f969168f700] [routing:mycluster_default_rw] started: listening on 0.0.0.0:6446, routing strategy = first-available
2019-07-27 10:20:10 main INFO [7f9683fff700]   plugin 'routing:mycluster_default_x_rw' starting
2019-07-27 10:20:10 routing INFO [7f9690e8e700] [routing:mycluster_default_x_ro] started: listening on 0.0.0.0:64470, routing strategy = round-robin-with-fallback
2019-07-27 10:20:10 routing INFO [7f9683fff700] [routing:mycluster_default_x_rw] started: listening on 0.0.0.0:64460, routing strategy = first-available
2019-07-27 10:20:10 main INFO [7f9696c9e880] Running.
2019-07-27 10:20:10 metadata_cache INFO [7f9692691700] Starting Metadata Cache
2019-07-27 10:20:10 metadata_cache INFO [7f9692691700] Connections using ssl_mode 'PREFERRED'
2019-07-27 10:20:10 metadata_cache INFO [7f9696c9a700] Starting metadata cache refresh thread
2019-07-27 10:20:10 metadata_cache INFO [7f9696c9a700] Potential changes detected in cluster 'mycluster' after metadata refresh
2019-07-27 10:20:10 metadata_cache INFO [7f9696c9a700] Metadata for cluster 'mycluster' has 1 replicasets:
2019-07-27 10:20:10 metadata_cache INFO [7f9696c9a700] 'default' (4 members, single-master)
2019-07-27 10:20:10 metadata_cache INFO [7f9696c9a700]     mysql1:3306 / 33060 - role=HA mode=RW
2019-07-27 10:20:10 metadata_cache INFO [7f9696c9a700]     mysql2:3306 / 33060 - role=HA mode=RO
2019-07-27 10:20:10 metadata_cache INFO [7f9696c9a700]     mysql3:3306 / 33060 - role=HA mode=RO
2019-07-27 10:20:10 metadata_cache INFO [7f9696c9a700]     mysql4:3306 / 33060 - role=HA mode=RO
2019-07-27 10:20:10 routing INFO [7f9696c9a700] Routing routing:mycluster_default_x_rw listening on 64460 got request to disconnect invalid connections: metadata change
2019-07-27 10:20:10 routing INFO [7f9696c9a700] Routing routing:mycluster_default_x_ro listening on 64470 got request to disconnect invalid connections: metadata change
2019-07-27 10:20:11 routing INFO [7f9696c9a700] Routing routing:mycluster_default_ro listening on 6447 got request to disconnect invalid connections: metadata change
2019-07-27 10:20:11 routing INFO [7f9696c9a700] Routing routing:mycluster_default_rw listening on 6446 got request to disconnect invalid connections: metadata change
```
That's it! Our InnoDB Cluster has now a MySQL Router too.

Pay attention to the information below. It will be used in the next section:
```console
## MySQL Classic protocol

- Read/Write Connections: localhost:6446
- Read/Only Connections:  localhost:6447
```

## 6. Add some data and check the replication

First create a new MySQL 8 container named **mysql-client** which will server as a client:
```
$ docker run -d --name=mysql-client --hostname=mysql-client --net=innodbnet \
   -e MYSQL_ROOT_PASSWORD=root mysql/mysql-server:8.0
```
When the container is `healthy`, run the command below to add some data to the cluster:
```
$ docker exec -it mysql-client mysql -h mysql-router -P 6446 -uinno -pinno \
  -e "create database TEST; use TEST; CREATE TABLE t1 (id INT NOT NULL PRIMARY KEY) ENGINE=InnoDB; show tables;" \
  -e "INSERT INTO TEST.t1 VALUES(1); INSERT INTO TEST.t1 VALUES(2); INSERT INTO TEST.t1 VALUES(3);"
```
* Note 1: ***mysql-client*** container is connecting to MySQL using the ***mysql-router*** container: `-h mysql-router -P 6446`.
* Note 2: We need to use port **6446** because it's the R/W connection port.

Run a query the check whether the data was inserted:
```
$ docker exec -it mysql-client mysql -h mysql-router -P 6447 -uinno -pinno \
  -e "SELECT * FROM TEST.t1;"
```
* Note 3: in the above query the router's R/O (Read/Only) port **6447** was used.

Output:
```console
mysql: [Warning] Using a password on the command line interface can be insecure.
+----+
| id |
+----+
|  1 |
|  2 |
|  3 |
+----+
```
Just to make sure that data is being replicated among the cluster's nodes, you can run the query in MySQL 8 servers containers directly:
```
for N in 1 2 3 4
do docker exec -it mysql$N mysql -uinno -pinno \
  -e "SHOW VARIABLES WHERE Variable_name = 'hostname';" \
  -e "SELECT * FROM TEST.t1;"
done
```
Expected output:
```console
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+--------+
| Variable_name | Value  |
+---------------+--------+
| hostname      | mysql1 |
+---------------+--------+
+----+
| id |
+----+
|  1 |
|  2 |
|  3 |
+----+
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+--------+
| Variable_name | Value  |
+---------------+--------+
| hostname      | mysql2 |
+---------------+--------+
+----+
| id |
+----+
|  1 |
|  2 |
|  3 |
+----+
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+--------+
| Variable_name | Value  |
+---------------+--------+
| hostname      | mysql3 |
+---------------+--------+
+----+
| id |
+----+
|  1 |
|  2 |
|  3 |
+----+
mysql: [Warning] Using a password on the command line interface can be insecure.
+---------------+--------+
| Variable_name | Value  |
+---------------+--------+
| hostname      | mysql4 |
+---------------+--------+
+----+
| id |
+----+
|  1 |
|  2 |
|  3 |
+----+
```
It seems to be working.. :+1:

Let's connect again to the MySQL Shell through our client container using the router's host and port:
```
$ docker exec -it mysql-client mysqlsh -h mysql-router -P 6447 -uinno -pinno
```
Get the cluster we create before:
```
var cluster = dba.getCluster("mycluster")
```
Then, call the describe function:
```
cluster.status()
```
![alt text](https://github.com/wagnerjfr/mysql-innodb-cluster/blob/master/img/figure1.png)

Don not close this terminal. We will try the cluster's fault tolerance in the next section.

## 7. Fault tolerance

We know from the figure of the last section that **mysql1** is the leader of the cluster.

Let's stop the container and check what will happen to the cluster.

Open a separate OS terminal and run:
```
$ docker stop mysql1
```

Go back to your MySQL Shell terminal and re-run:
```
cluster.status()
```

The cluster is still up and running, but we have a new "master", **mysql2**. **mysql1** is now with status `"(MISSING)"` and mode `"n/a"`.

Console output:
```console
 MySQL  mysql-router:6447 ssl  JS > cluster.status();
{
    "clusterName": "mycluster", 
    "defaultReplicaSet": {
        "name": "default", 
        "primary": "mysql2:3306", 
        "ssl": "REQUIRED", 
        "status": "OK_PARTIAL", 
        "statusText": "Cluster is ONLINE and can tolerate up to ONE failure. 1 member is not active", 
        "topology": {
            "mysql1:3306": {
                "address": "mysql1:3306", 
                "mode": "n/a", 
                "readReplicas": {}, 
                "role": "HA", 
                "shellConnectError": "MySQL Error 2005 (HY000): Unknown MySQL server host 'mysql1' (0)", 
                "status": "(MISSING)"
            }, 
            "mysql2:3306": {
                "address": "mysql2:3306", 
                "mode": "R/W", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.17"
            }, 
            "mysql3:3306": {
                "address": "mysql3:3306", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.17"
            }, 
            "mysql4:3306": {
                "address": "mysql4:3306", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.17"
            }
        }, 
        "topologyMode": "Single-Primary"
    }, 
    "groupInformationSourceMember": "mysql2:3306"
}
```

Go back to the OS terminal and start **mysql1**:
```
$ docker start mysql1
```

Finally, back to MySQL Shell terminal, run `cluster.status()`:

If you are fast enough you notice that **mysql1** will start `"RECOVERING"` in the cluster:
```console
 MySQL  mysql-router:6447 ssl  JS > cluster.status();
{
    "clusterName": "mycluster", 
    "defaultReplicaSet": {
        "name": "default", 
        "primary": "mysql2:3306", 
        "ssl": "REQUIRED", 
        "status": "OK", 
        "statusText": "Cluster is ONLINE and can tolerate up to ONE failure.", 
        "topology": {
            "mysql1:3306": {
                "address": "mysql1:3306", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "recoveryStatusText": "Recovery in progress", 
                "role": "HA", 
                "status": "RECOVERING", 
                "version": "8.0.17"
            }, 
            "mysql2:3306": {
                "address": "mysql2:3306", 
                "mode": "R/W", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.17"
            }, 
            "mysql3:3306": {
                "address": "mysql3:3306", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.17"
            }, 
            "mysql4:3306": {
                "address": "mysql4:3306", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.17"
            }
        }, 
        "topologyMode": "Single-Primary"
    }, 
    "groupInformationSourceMember": "mysql2:3306"
}
```
And afer some seconds, it's back `"ONLINE"` again, but as a "slave" in `"R/O"` mode.
```console
 MySQL  mysql-router:6447 ssl  JS > cluster.status();
{
    "clusterName": "mycluster", 
    "defaultReplicaSet": {
        "name": "default", 
        "primary": "mysql2:3306", 
        "ssl": "REQUIRED", 
        "status": "OK", 
        "statusText": "Cluster is ONLINE and can tolerate up to ONE failure.", 
        "topology": {
            "mysql1:3306": {
                "address": "mysql1:3306", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.17"
            }, 
            "mysql2:3306": {
                "address": "mysql2:3306", 
                "mode": "R/W", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.17"
            }, 
            "mysql3:3306": {
                "address": "mysql3:3306", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.17"
            }, 
            "mysql4:3306": {
                "address": "mysql4:3306", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.17"
            }
        }, 
        "topologyMode": "Single-Primary"
    }, 
    "groupInformationSourceMember": "mysql2:3306"
}
```

## 8. Clean up

Stop the running containers:
```
docker stop mysql1 mysql2 mysql3 mysql4 mysql-router mysql-client
```
Remove the stopped containers:
```
docker rm mysql1 mysql2 mysql3 mysql4 mysql-router mysql-client
```
