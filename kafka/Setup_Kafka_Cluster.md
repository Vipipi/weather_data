## Environment

1. Kafka cluster: 3 brokers
2. Producer: 1
3. Consumer: 2(Not in the same Consumer Group)

```
borker1		172.18.0.2	
borker2		172.18.0.3	
borker3		172.18.0.4	
producer	172.18.0.5	
consumer1	172.18.0.6	
consumer2	172.18.0.7
```


## Docker Container

### docker-compose.yml
```
version: '2'
services:
  broker1: 
    image: bolingcavalry/ssh-kafka292081-zk346:0.0.1
    container_name: broker1
    ports:
      - "19011:22"
    restart: always
  broker2: 
    image: bolingcavalry/ssh-kafka292081-zk346:0.0.1
    container_name: broker2
    depends_on:
      - broker1
    ports:
      - "19012:22"  
    restart: always
  broker3: 
    image: bolingcavalry/ssh-kafka292081-zk346:0.0.1
    container_name: broker3
    depends_on:
      - broker2
    ports:
      - "19013:22"
    restart: always
  producer: 
    image: bolingcavalry/ssh-kafka292081-zk346:0.0.1
    container_name: producer
    links: 
      - broker1:hostb1
      - broker2:hostb2
      - broker3:hostb3
    ports:
      - "19014:22"
    restart: always
  consumer1: 
    image: bolingcavalry/ssh-kafka292081-zk346:0.0.1
    container_name: consumer1
    depends_on:
      - producer
    links: 
      - broker1:hostb1
      - broker2:hostb2
      - broker3:hostb3
    ports:
      - "19015:22"
    restart: always
  consumer2: 
    image: bolingcavalry/ssh-kafka292081-zk346:0.0.1
    container_name: consumer2
    depends_on:
      - consumer1
    ports:
      - "19016:22"
    links: 
      - broker1:hostb1
      - broker2:hostb2
      - broker3:hostb3
    restart: always
```

### Bring up all docker containers
`docker-compose up -d`


## Configuration

### Hosts
ssh into broker1, 2, 3 and check their ip addresses.
Add following lines to the end of `/etc/hosts`

```
broker1 172.18.0.2
broker2 172.18.0.3
broker3 172.18.0.4

```

### ZooKeeper

#### config
ssh into broker1, 2, 3
Run following cmd

broker1: `echo 1 > /usr/local/work/zkdata/myid`  
broker2: `echo 2 > /usr/local/work/zkdata/myid`  
broker3: `echo 3 > /usr/local/work/zkdata/myid`  

#### start
broker1, 2, 3: `/usr/local/work/zookeeper-3.4.6/bin/zkServer.sh start`

Should be able to see `Mode: leader` or `Mode: follower`


### Kafka

#### config
ssh into broker1, 2, 3 and producer
For broker1, go to `/usr/local/work/kafka_2.9.2-0.8.1/config/server.properties`, make two changes:
1. Under Server Basics, change `broker.id=0` to `broker.id=1` which is the same as the `myid` file. Make these change for broker2 and broker3
2. Under Zookeeper, change `zookeeper.connect=localhost:2181` to `zookeeper.connect=broker1:2181,broker2:2181,broker3:2181`. Make these change for broker2, broker3 and producer



#### start
`nohup /usr/local/work/kafka_2.9.2-0.8.1/bin/kafka-server-start.sh /usr/local/work/kafka_2.9.2-0.8.1/config/server.properties >/usr/local/work/log/kafka.log 2>1 &`

### Verify
1. From broker1, create a topic named `test001`
`/usr/local/work/kafka_2.9.2-0.8.1/bin/kafka-topics.sh --create --zookeeper broker1:2181,broker2:2181,broker3:2181 --replication-factor 1 --partitions 3 --topic test001`

2. From broker2, check all the topics
`/usr/local/work/kafka_2.9.2-0.8.1/bin/kafka-topics.sh --list --zookeeper broker1:2181,broker2:2181,broker3:2181`

3. Kafka logs are under `/tmp/kafka-logs/`

4. From producer, enter console mode:
`/usr/local/work/kafka_2.9.2-0.8.1/bin/kafka-console-producer.sh --broker-list broker1:9092,broker2:9092,broker3:9092 --topic test001`

5. From both consumer, enter console mode:
`/usr/local/work/kafka_2.9.2-0.8.1/bin/kafka-console-consumer.sh --zookeeper broker1:2181,broker1:2181,broker1:2181 --from-beginning --topic test001`









