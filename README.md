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




# Action steps

* start docker container
* start service
* start kafka consumer
* start JMeter request
