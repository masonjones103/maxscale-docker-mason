# Changing a MariaDB master-slave server to a sharded using Maxscale in Docker-compose

## Running
The MaxScale docker-compose setup contains MaxScale
configured with a two masters sharded server setup. To start it, run the
following commands in this directory.

```
docker-compose build
docker-compose up -d
```

After MaxScale and the servers have started (takes a few minutes), you can find
the sharded router on port 4000. The
user `maxuser` with the password `maxpwd` can be used to test the cluster.
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

To stop the containers, execute the following command. Optionally, use the -v
flag to also remove the volumes.

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

The cluster is configured to utilize automatic failover. To illustrate this you can stop the master
container and watch for maxscale to failover to the other master and then show it rejoining
after recovery:
```
$ docker-compose stop master
Stopping maxscaledocker_master_1 ... done
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

## Changes made to convert from master-slave to sharded
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
Can be deleted, you will not need it.

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
