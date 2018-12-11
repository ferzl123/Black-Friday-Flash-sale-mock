# Introductoin

Simulated a situation when web service receiving large-volume of requests in a short time.
1. Applied JMeter to simulate 2000 concurrent HTTP requests by users in 5 seconds.
2. Created web service by Node.js and Express, and prepared 4 Docker containers for ZooKeeper, Kafka, Redis and MySQL.
3. Implemented Redis to store the counter representing the number of currently remaining stock.
4. Configured Kafka to monitor success messages of flash sale from web service.

![Alt text](https://github.com/ferzl123/Black-Friday-Flash-sale-mock/blob/master/1.png "Optional title")

# Environment Configuration

## JMeter

Create a thread group with 2000 numbers of threads(users) in 5 seconds
* New a Http Requests sampler with 
* Host name: localhost 
* Port number: 3000
* HTTP Method: Post
* request Path: /mock/mock

## Redis, Kafka, ZooKeeper, MySQL in docker environment

Install dokcer engine, docker machine, docker comose

### dokcer configuration
create the congiguration file docker-compose.yml

```
version: '2'
services:
  zookeeper:
    image: wurstmeister/zookeeper
    ports:
      - "2181:2181"
  kafka:
    image: wurstmeister/kafka
    links:
      - zookeeper:zk
    ports:
      - "9092:9092"
    depends_on:
      - zookeeper
    environment:
      KAFKA_ADVERTISED_HOST_NAME: 10.18.129.58
      KAFKA_ADVERTISED_PORT: "9092"
      KAFKA_ZOOKEEPER_CONNECT: zk:2181
  redis:
    image: redis
    ports:
      - "6379:6379"
  mysql:
    image: mysql
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: 123
```
4 services corresponding to 4 docker containers: ZooKeeper, Kafka, Redis, MySQL

* KAFKA_ADVERTISED_HOST_NAME: need to be changed to local IP from ```ipconfig```
* MYSQL_ROOT_PASSWORD: from personal pc

start the docker, and compose them up automatically by Docker
```
cd ./path
docker-compose up
```

* Create the topic in Kafka: CAR_NUMBER
* Create Counter in redis(value: 100, intial stock 100)
* Create the table(a self-increment ID fieldt and a timestamp-format data field)

# Code
```javascript

var express = require('express');
var router = express.Router();
var redis = require('redis');
var kafka = require('kafka-node');
var Producer = kafka.Producer;
var kafkaClient = new kafka.Client();
var producer = new Producer(kafkaClient);
var count = 0;

router.post('/seckill', function (req, res) {
    console.log('count=' + count++);
    var fn = function (optionalClient) {
        if (optionalClient == 'undefined' || optionalClient == null) {
            var client = redis.createClient();
        }else{
            var client = optionalClient;
        }
        client.on('error', function (er) {
            console.trace('Here I am');
            console.error(er.stack);
            client.end(true);
        });
        client.watch("counter");
        client.get("counter", function (err, reply) {
            if (parseInt(reply) > 0) {
                var multi = client.multi();
                multi.decr("counter");
                multi.exec(function (err, replies) {
                    if (replies == null) {
                        console.log('should have conflict')
                        fn(client);
                    } else {
                        var payload = [
                            {
                                topic: 'CAR_NUMBER',
                                messages: 'buy 1 car',
                                partition: 0
                            }
                        ];
                        producer.send(payload, function (err, data) {
                            console.log(data);
                        });
                        res.send(replies);
                        client.end(true);
                    }
                });
            } else {
                console.log("sold out!");
                res.send("sold out!");
                client.end(true);
            }
        })
    };
    fn();
});

module.exports = router;
```

1-8，，Node.js MVC framework with Express and Redis and Kafka
10，利用Express提供的方法暴露出一个path为/seckill的POST方法。
第12行，定义了一个方法，在54行会调用。
第13-22行，新建了一个redis client并且监听error事件。
第23行，这行代码非常关键，它的作用是让redis client监视Redis中的counter值，之后会启动一个事务，如果在事务提交的时候发现有其它client修改了counter值的话，就会放弃这个事务。
第24行，通过redis client的异步方法获取counter的值，因为redis的get操作是原子的，所以在这里不用担心有并发读写的问题。
第25-28行，判断返回的库存值是否大于0，如果大于0，通过client.multi()启动一个事务，通过decr()方法将counter值减1，最后通过exec()方法提交事务；如果小于0，则执行第47行，打印卖完了并且关闭redis client。
第29-46行，这里我们看一下multi.exec()中的这个回调方法。在前面我们已经使用watch对counter进行了监视。如果在事务提交过程中有其它client修改了counter值的话，回调方法中的replies参数就会是null，可以看到第29-31行，程序会打印“可能有冲突”并且再次调用fn方法重试。

如果replies的值不为null，就会使用kafka的producer发送一条message到CAR_NUMBER topic。




# Action steps

* start docker container
* start service
* start kafka consumer
* start JMeter request
