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
