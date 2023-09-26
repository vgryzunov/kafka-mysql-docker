# MySQL 8.1 Kafka Connect on Docker

This project inspired by the article [MySQL 8 Kafka Connect Tutorial on Docker](https://dev.to/cosmostail/mysql-8-kafka-connect-tutorial-on-docker-479p). The code presented in this article needs updating to work as expected.

Dockerfile and needed jars are included into this project. That's why you may skip
preparation steps from the original article.

## Starting Mysql container

To start MySQL container execute command:

```bash
docker-compose up -d mysql
```
Now we need to create test database, a table in this database and insert some test
data. This can be dome by entering inside created container:
```bash
docker exec -it kafka-mysql-docker-mysql-1 bash
```
After executing this command you should see the prompt of bash inside Mysql container:
```
bash-4.4#
```
Enter `mysql -uroot -ptest` command after this prompt. MySQL interaction screen must be
opened in response:
```
Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```
Copy and paste MySQL code into this screen:
```mysql
CREATE DATABASE IF NOT EXISTS connect_test;
USE connect_test;

CREATE TABLE IF NOT EXISTS test (
id serial NOT NULL PRIMARY KEY,
name varchar(100),
email varchar(200),
department varchar(200),
modified timestamp default CURRENT_TIMESTAMP NOT NULL,
INDEX `modified_index` (`modified`)
);

INSERT INTO test (name, email, department) VALUES ('alice', 'alice@abc.com', 'engineering');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
INSERT INTO test (name, email, department) VALUES ('bob', 'bob@abc.com', 'sales');
exit;
```
Press *Enter* to execute the last SQL command.
To exit from container, enter `exit` after bash prompt.

## Starting zookeeper and kafka
Now zookeeper and kafka containers must be started:
```bash
docker-compose up -d zookeeper kafka
```

```
[+] Running 2/2
 ✔ Container kafka-mysql-docker-zookeeper-1  Started                                                                                                                                                                     0.5s 
 ✔ Container kafka-mysql-docker-kafka-1      Started
```
The state of containers can be checked by executing
```bash
docker ps
```

```
CONTAINER ID   IMAGE                             COMMAND                  CREATED              STATUS              PORTS                                                    NAMES
6bedf2c8c7ec   confluentinc/cp-kafka:7.5.0       "/etc/confluent/dock…"   About a minute ago   Up About a minute   9092/tcp, 0.0.0.0:29092->29092/tcp                       kafka-mysql-docker-kafka-1
315336d22500   confluentinc/cp-zookeeper:7.5.0   "/etc/confluent/dock…"   About a minute ago   Up About a minute   2181/tcp, 2888/tcp, 3888/tcp, 0.0.0.0:32181->32181/tcp   kafka-mysql-docker-zookeeper-1
631ee6abfe35   mysql:8.1                         "docker-entrypoint.s…"   16 minutes ago       Up 16 minutes       0.0.0.0:3306->3306/tcp, 33060/tcp                        kafka-mysql-docker-mysql-1
```
## Kafka connector topics
Connector needs three kafka topics to be created:
```bash
docker-compose run --rm kafka kafka-topics --create --topic quickstart-avro-offsets --partitions 1 --replication-factor 1 --if-not-exists --bootstrap-server kafka:29092 --config cleanup.policy=compact
```
```
[+] Creating 1/0
 ✔ Container kafka-mysql-docker-zookeeper-1  Running                                                                                                                                                                     0.0s 
Created topic quickstart-avro-offsets.
```

```bash
docker-compose run --rm kafka kafka-topics --create --topic quickstart-avro-config  --partitions 1 --replication-factor 1 --if-not-exists --bootstrap-server kafka:29092 --config cleanup.policy=compact
```

```
[+] Creating 1/0
 ✔ Container kafka-mysql-docker-zookeeper-1  Running                                                                                                                                                                     0.0s 
Created topic quickstart-avro-config.
```

```bash
docker-compose run --rm kafka kafka-topics --create --topic quickstart-avro-status  --partitions 1 --replication-factor 1 --if-not-exists --bootstrap-server kafka:29092 --config cleanup.policy=compact
```
```
[+] Creating 1/0
 ✔ Container kafka-mysql-docker-zookeeper-1  Running                                                                                                                                                                     0.0s 
Created topic quickstart-avro-status.
```

## Starting Kafka connect
Now everything is ready to start kafka-connect container:
```bash
docker-compose up -d kafka-connector-mysql
```
``` 
[+] Running 4/4
 ✔ Container kafka-mysql-docker-zookeeper-1              Running                                                                                                                                                         0.0s 
 ✔ Container kafka-mysql-docker-kafka-1                  Running                                                                                                                                                         0.0s 
 ✔ Container kafka-mysql-docker-mysql-1                  Running                                                                                                                                                         0.0s 
 ✔ Container kafka-mysql-docker-kafka-connector-mysql-1  Started 
```
JDBC Source connector is created by calling REST API of the kafka connector
service:
```bash
curl -X POST \
-H "Content-Type: application/json" \
--data '{ "name": "quickstart-jdbc-source", "config": { "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector", "tasks.max": 1, "connection.url": "jdbc:mysql://mysql:3306/connect_test", "connection.user": "root", "connection.password": "test", "mode": "incrementing", "incrementing.column.name": "id", "timestamp.column.name": "modified", "topic.prefix": "quickstart-jdbc-", "poll.interval.ms": 1000, "mode": "incrementing", "incrementing.column.name": "id", "table.whitelist": "test"} }' \
http://localhost:28082/connectors
```
``` 
{"name":"quickstart-jdbc-source","config":{"connector.class":"io.confluent.connect.jdbc.JdbcSourceConnector","tasks.max":"1","connection.url":"jdbc:mysql://mysql:3306/connect_test","connection.user":"root","connection.pass
word":"test","mode":"incrementing","incrementing.column.name":"id","timestamp.column.name":"modified","topic.prefix":"quickstart-jdbc-","poll.interval.ms":"1000","table.whitelist":"test","name":"quickstart-jdbc-source"},"t
asks":[],"type":"source"}
```

## Checking created source connector
Status of created source connector cen be checked my making REST call:
```bash
curl -s -X GET http://localhost:28082/connectors/quickstart-jdbc-source/status
```
``` 
{"name":"quickstart-jdbc-source","connector":{"state":"RUNNING","worker_id":"localhost:28082"},"tasks":[{"id":0,"state":"RUNNING","worker_id":"localhost:28082"}],"type":"source"}
```

Finally, verification of created  events can be done by starting consumer container:    
```bash
docker-compose run --rm kafka kafka-console-consumer --bootstrap-server kafka:29092  --topic quickstart-jdbc-test --from-beginning
```
``` 
[+] Creating 1/0
 ✔ Container kafka-mysql-docker-zookeeper-1  Running                                                                                                                                                                     0.0s 
{"schema":{"type":"struct","fields":[{"type":"int64","optional":false,"field":"id"},{"type":"string","optional":true,"field":"name"},{"type":"string","optional":true,"field":"email"},{"type":"string","optional":true,"field
":"department"},{"type":"int64","optional":false,"name":"org.apache.kafka.connect.data.Timestamp","version":1,"field":"modified"}],"optional":false,"name":"test"},"payload":{"id":1,"name":"alice","email":"alice@abc.com","d
epartment":"engineering","modified":1695723874000}}
{"schema":{"type":"struct","fields":[{"type":"int64","optional":false,"field":"id"},{"type":"string","optional":true,"field":"name"},{"type":"string","optional":true,"field":"email"},{"type":"string","optional":true,"field
":"department"},{"type":"int64","optional":false,"name":"org.apache.kafka.connect.data.Timestamp","version":1,"field":"modified"}],"optional":false,"name":"test"},"payload":{"id":2,"name":"bob","email":"bob@abc.com","depar
tment":"sales","modified":1695723874000}}
{"schema":{"type":"struct","fields":[{"type":"int64","optional":false,"field":"id"},{"type":"string","optional":true,"field":"name"},{"type":"string","optional":true,"field":"email"},{"type":"string","optional":true,"field
":"department"},{"type":"int64","optional":false,"name":"org.apache.kafka.connect.data.Timestamp","version":1,"field":"modified"}],"optional":false,"name":"test"},"payload":{"id":3,"name":"bob","email":"bob@abc.com","depar
tment":"sales","modified":1695723874000}}
{"schema":{"type":"struct","fields":[{"type":"int64","optional":false,"field":"id"},{"type":"string","optional":true,"field":"name"},{"type":"string","optional":true,"field":"email"},{"type":"string","optional":true,"field
{"schema":{"type":"struct","fields":[{"type":"int64","optional":false,"field":"id"},{"type":"string","optional":true,"field":"name"},{"type":"string","optional":true,"field":"email"},{"type":"string","optional":true,"field
":"department"},{"type":"int64","optional":false,"name":"org.apache.kafka.connect.data.Timestamp","version":1,"field":"modified"}],"optional":false,"name":"test"},"payload":{"id":8,"name":"bob","email":"bob@abc.com","depar
tment":"sales","modified":1695723874000}}
{"schema":{"type":"struct","fields":[{"type":"int64","optional":false,"field":"id"},{"type":"string","optional":true,"field":"name"},{"type":"string","optional":true,"field":"email"},{"type":"string","optional":true,"field
":"department"},{"type":"int64","optional":false,"name":"org.apache.kafka.connect.data.Timestamp","version":1,"field":"modified"}],"optional":false,"name":"test"},"payload":{"id":9,"name":"bob","email":"bob@abc.com","depar
tment":"sales","modified":1695723874000}}
{"schema":{"type":"struct","fields":[{"type":"int64","optional":false,"field":"id"},{"type":"string","optional":true,"field":"name"},{"type":"string","optional":true,"field":"email"},{"type":"string","optional":true,"field
":"department"},{"type":"int64","optional":false,"name":"org.apache.kafka.connect.data.Timestamp","version":1,"field":"modified"}],"optional":false,"name":"test"},"payload":{"id":10,"name":"bob","email":"bob@abc.com","depa
rtment":"sales","modified":1695723874000}}
^CProcessed a total of 10 messages
```
To stop the consumer container, press Ctrl-C.

## Stopping containers
To tear down all created containers, simply run:
```bash
docker-compose down
```

If you need to leave MySQL running, stop all containers excepy MySQL:
```bash
docker-compose down kafka-connector-mysql kafka zookeeper
```
```
[+] Running 4/4
 ✔ Container kafka-mysql-docker-kafka-connector-mysql-1  Removed                                                                                                                                                         3.5s 
 ✔ Container kafka-mysql-docker-kafka-1                  Removed                                                                                                                                                         2.3s 
 ✔ Container kafka-mysql-docker-zookeeper-1              Removed                                                                                                                                                         1.0s 
 ! Network kafka-mysql-docker_default                    Resource is still in use 
```
