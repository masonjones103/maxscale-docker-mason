# Changing a MariaDB master-slave server to a sharded using Maxscale in Docker-compose

This readme will help you make the necessary edits to convert your server with one master and two slaves into a sharded server with two masters using mariadb maxscale. We will build this using Docker-compose.

# Configuration

## Changes made to your docker-compose.yml file
Open your docker-compose.yml file. 

Change the first service under services from:
```
    master:
        image: mariadb:10.3
        environment:
            MYSQL_ALLOW_EMPTY_PASSWORD: 'Y'
        volumes:
            - ./sql/master:/docker-entrypoint-initdb.d
        command: mysqld --log-bin=mariadb-bin --binlog-format=ROW --server-id=3000
        ports:
            - "4001:3306"
```
to:
```
    server1:
        image: mariadb:10.3
        environment:
            MYSQL_ALLOW_EMPTY_PASSWORD: 'Y'
        volumes:
            - ./sql/server1:/docker-entrypoint-initdb.d
        command: mysqld --log-bin=mariadb-bin --binlog-format=ROW --server-id=3000
        ports:
            - "4001:3306"
```

Change the second service from:
```

    slave1:
        image: mariadb:10.3
        depends_on:
            - master
        environment:
            MYSQL_ALLOW_EMPTY_PASSWORD: 'Y'
        volumes:
            - ./sql/slave:/docker-entrypoint-initdb.d
        command: mysqld --log-bin=mariadb-bin --binlog-format=ROW --server-id=3001 --log-slave-updates
        ports:
            - "4002:3306"
```
to:
```
    server2:
        image: mariadb:10.3
        environment:
            MYSQL_ALLOW_EMPTY_PASSWORD: 'Y'
        volumes:
            - ./sql/server2:/docker-entrypoint-initdb.d
        command: mysqld --log-bin=mariadb-bin --binlog-format=ROW --server-id=3001
        ports:
            - "4002:3306"
```

The third service:
```
    slave2:
        image: mariadb:10.3
        depends_on:
            - master
        environment:
            MYSQL_ALLOW_EMPTY_PASSWORD: 'Y'
        volumes:
            - ./sql/slave:/docker-entrypoint-initdb.d
        command: mysqld --log-bin=mariadb-bin --binlog-format=ROW --server-id=3002 --log-slave-updates
        ports:
            - "4003:3306"
```
can be deleted, you will not need it.

Next change the maxscale service from:
```
    maxscale:
        image: mariadb/maxscale:latest
        depends_on:
            - master
            - slave1
            - slave2
        volumes:
            - ./maxscale.cnf.d:/etc/maxscale.cnf.d
        ports:
            - "4006:4006"  # readwrite port
            - "4008:4008"  # readonly port
            - "8989:8989"  # REST API port
```
to:
```
    maxscale:
        image: mariadb/maxscale:latest
        depends_on:
            - server1
            - server2
        volumes:
            - ./maxscale.cnf.d:/etc/maxscale.cnf.d
        ports:
            - "4000:4000"  # sharded port
            - "4006:4006"  # readwrite port
            - "4008:4008"  # readonly port
            - "8989:8989"  # REST API port
```
This will tell maxscale to start after your two master servers and you can use the port 4000 to access the sharded service.

After all these changes, your docker-compose.yml file should look like this:
```
version: '2'
services:
    server1:
        image: mariadb:10.3
        environment:
            MYSQL_ALLOW_EMPTY_PASSWORD: 'Y'
        volumes:
            - ./sql/server1:/docker-entrypoint-initdb.d
        command: mysqld --log-bin=mariadb-bin --binlog-format=ROW --server-id=3000
        ports:
            - "4001:3306"

    server2:
        image: mariadb:10.3
        environment:
            MYSQL_ALLOW_EMPTY_PASSWORD: 'Y'
        volumes:
            - ./sql/server2:/docker-entrypoint-initdb.d
        command: mysqld --log-bin=mariadb-bin --binlog-format=ROW --server-id=3001
        ports:
            - "4002:3306"

    maxscale:
        image: mariadb/maxscale:latest
        depends_on:
            - server1
            - server2
        volumes:
            - ./maxscale.cnf.d:/etc/maxscale.cnf.d
        ports:
            - "4000:4000"  # sharded port
            - "4006:4006"  # readwrite port
            - "4008:4008"  # readonly port
            - "8989:8989"  # REST API port
```


Next you will need to rename the folders containing your sql files.

These are located at: ```/maxscale/sql``` and are named ```master``` and ```slave``` by default.

You will want to rename these to ```server1``` and ```server2```.

## Changes made to your example.cnf file
Navigate to your example.cnf file located at ```/maxscale/maxscale.cnf.d/example.cnf```.

Change the address under server1 to ```address=server1``` and the address under server2 to ```address=server2```.

Remove server3, it will not be needed.

Change MariaDB-Monitor from:
```
[MariaDB-Monitor]
type=monitor
module=mariadbmon
servers=server1,server2,server3
user=maxuser
password=maxpwd
auto_failover=true
auto_rejoin=true
enforce_read_only_slaves=1
```
to:
```
[MariaDB-Monitor]
type=monitor
module=mariadbmon
servers=server1,server2
user=maxuser
password=maxpwd
auto_failover=true
auto_rejoin=true
```

Under MariaDB-Monitor, add:
```
[Sharded-Service]
type=service
router=schemarouter
servers=server1,server2
user=maxuser
password=maxpwd
```

Change the Read-Only-Service from:
```
[Read-Only-Service]
type=service
router=readconnroute
servers=server1,server2,server3
user=maxuser
password=maxpwd
router_options=slave
```
to:
```
[Read-Only-Service]
type=service
router=readconnroute
servers=server1,server2
user=maxuser
password=maxpwd
```

Change the Read-Write-Service from:
```
[Read-Write-Service]
type=service
router=readwritesplit
servers=server1,server2,server3
user=maxuser
password=maxpwd
master_failure_mode=fail_on_write
```
to:
```
[Read-Write-Service]
type=service
router=readwritesplit
servers=server1,server2
user=maxuser
password=maxpwd
master_failure_mode=fail_on_write
```

Before the Read-Only-Listener and Read-Write-Listener, add:
```
[Sharded-Service_Listener]
type=listener
service=Sharded-Service
protocol=MariaDBClient
port=4000
```

After all these changes, your example.cnf should look like this:
```
[server1]
type=server
address=server1
port=3306
protocol=MariaDBBackend

[server2]
type=server
address=server2
port=3306
protocol=MariaDBBackend

# Monitor for the servers
# This will keep MaxScale aware of the state of the servers.
# MySQL Monitor documentation:
# https://github.com/mariadb-corporation/MaxScale/blob/2.3/Documentation/Monitors/MariaDB-Monitor.md

[MariaDB-Monitor]
type=monitor
module=mariadbmon
servers=server1,server2
user=maxuser
password=maxpwd
auto_failover=true
auto_rejoin=true

# Service definitions
# Service Definition for a read-only service and a read/write splitting service.

# service for a sharded server
[Sharded-Service]
type=service
router=schemarouter
servers=server1,server2
user=maxuser
password=maxpwd

# ReadConnRoute documentation:
# https://github.com/mariadb-corporation/MaxScale/blob/2.3/Documentation/Routers/ReadConnRoute.md

[Read-Only-Service]
type=service
router=readconnroute
servers=server1,server2
user=maxuser
password=maxpwd

# ReadWriteSplit documentation:
# https://github.com/mariadb-corporation/MaxScale/blob/2.3/Documentation/Routers/ReadWriteSplit.md

[Read-Write-Service]
type=service
router=readwritesplit
servers=server1,server2
user=maxuser
password=maxpwd
master_failure_mode=fail_on_write

# Listener definitions for the services
# Listeners represent the ports the services will listen on.

# sharded listener

[Sharded-Service_Listener]
type=listener
service=Sharded-Service
protocol=MariaDBClient
port=4000

[Read-Only-Listener]
type=listener
service=Read-Only-Service
protocol=MariaDBClient
port=4008

[Read-Write-Listener]
type=listener
service=Read-Write-Service
protocol=MariaDBClient
port=4006
```

## Running
The MaxScale docker-compose setup contains MaxScale configured with a two masters sharded server setup. 

To start it, run the following commands in this directory.

```
docker-compose build
docker-compose up -d
```

After MaxScale and the servers have started (takes a few minutes), you can find the sharded router on port 4000. 

The user `maxuser` with the password `maxpwd` can be used to test the cluster.
Assuming the mariadb client is installed on the host machine:
```
$ mysql -umaxuser -pmaxpwd -h 127.0.0.1 -P 4000
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 5
Server version: 10.2.12 2.2.9-maxscale mariadb.org binary distribution

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]>
```
You can edit the [`maxscale.cnf.d/example.cnf`](./maxscale.cnf.d/example.cnf)
file and recreate the MaxScale container to change the configuration.

To stop the containers, execute the following command. 

```docker-compose down```

Optionally, use the -v flag to also remove the volumes.


To run maxctrl in the container to see the status of the cluster:
```
$ docker-compose exec maxscale maxctrl list servers
┌─────────┬─────────┬──────┬─────────────┬─────────────────┬──────────┬───────────────┐
│ Server  │ Address │ Port │ Connections │ State           │ GTID     │ Monitor       │   
├─────────┼─────────┼──────┼─────────────┼─────────────────┼──────────┤───────────────┤
│ server1 │ server1 │ 3306 │ 0           │ Master, Running │          │MariaDB-Monitor│
├─────────┼─────────┼──────┼─────────────┼─────────────────┼──────────┼───────────────┤
│ server2 │ server2 │ 3306 │ 0           │ Running         │          │MariaDB-Monitor│
└─────────┴─────────┴──────┴─────────────┴─────────────────┴──────────┘───────────────┘



```

The cluster is configured to utilize automatic failover. 

To illustrate this you can stop the master container and watch for maxscale to failover to the other master and then show it rejoining
after recovery:
```
$ docker-compose stop master
Stopping maxscaledocker_server1 ... done
$ docker-compose exec maxscale maxctrl list servers
┌─────────┬─────────┬──────┬─────────────┬─────────────────┬──────────┬───────────────┐
│ Server  │ Address │ Port │ Connections │ State           │ GTID     │ Monitor       │   
├─────────┼─────────┼──────┼─────────────┼─────────────────┼──────────┤───────────────┤
│ server1 │ server1 │ 3306 │ 0           │ Down            │          │MariaDB-Monitor│
├─────────┼─────────┼──────┼─────────────┼─────────────────┼──────────┼───────────────┤
│ server2 │ server2 │ 3306 │ 0           │ Master, Running │          │MariaDB-Monitor│
└─────────┴─────────┴──────┴─────────────┴─────────────────┴──────────┘───────────────┘

$ docker-compose start master
Starting master ... done
$ docker-compose exec maxscale maxctrl list servers
┌─────────┬─────────┬──────┬─────────────┬─────────────────┬──────────┬───────────────┐
│ Server  │ Address │ Port │ Connections │ State           │ GTID     │ Monitor       │   
├─────────┼─────────┼──────┼─────────────┼─────────────────┼──────────┤───────────────┤
│ server1 │ server1 │ 3306 │ 0           │ Running         │          │MariaDB-Monitor│
├─────────┼─────────┼──────┼─────────────┼─────────────────┼──────────┼───────────────┤
│ server2 │ server2 │ 3306 │ 0           │ Master, Running │          │MariaDB-Monitor│
└─────────┴─────────┴──────┴─────────────┴─────────────────┴──────────┘───────────────┘

```

Once complete, to remove the cluster and maxscale containers:

```
docker-compose down -v
```

## Connecting to MariaDB
To connect to MariaDB, use the command ```mariadb -umaxuser -pmaxpwd -h 127.0.0.1 -P 4000```.

To import a database, use the command ```source``` followed by the path to your sql file.

For example, my command was ```source /home/masonjones103/sql-files/shard1/shard1.sql```.

To confirm the data has been imported correctly, you can use the command ```show databases;```.

