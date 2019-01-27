# springboot-kafka-debezium-ksql

## Goal

The goal of this project is to play with [`Kafka`](https://kafka.apache.org), [`Debezium`](https://debezium.io/) and
[`KSQL`](https://www.confluent.io/product/ksql/). For this, we have: `research-service` that inserts/updates/deletes
records in [`MySQL`](https://www.mysql.com); `Source Connectors` that monitor inserted/updated/deleted records in MySQL
and push messages related to those changes to Kafka; `Sink Connectors` that read messages from Kafka and insert/update
documents in [`Elasticsearch`](https://www.elastic.co); finally, `ksql` that listens for some topics in Kafka, does some
joins and and pushes new messages to new topics in Kafka.

## Microservices

### research-service

Monolithic spring-boot application that exposes a REST API to manage Institutes, Articles, Researchers and Reviews.
The data is saved in MySQL. Besides, if the service is run with the profile `simulation`, it will _simulate_ an
automatic and random creation of reviews.

## Start Environment

### Docker Compose

1. Open one terminal

2. Inside `/springboot-kafka-debezium-ksql` root folder run

```
docker-compose up -d
```
> During the first run, an image for `mysql` and `kafka-connect`will be built, whose names are
> `springboot-kafka-debezium-ksql_mysql` and `springboot-kafka-debezium-ksql_kafka-connect`, respectively.
> To rebuild those images run
> ```
> docker-compose build
> ```
> ---
> To stop and remove containers, networks and volumes type:
> ```
> docker-compose down -v
> ```

3. Wait a little bit until all containers are `Up (healthy)`. To check the status of the containers run the command
```
docker-compose ps
```

### Create connectors

1. In a terminal, run the following script to create the connectors on `kafka-connect`
```
./create-connectors.sh
```

2. You can check the state of the connectors and their tasks on `Kafka Connect UI` (http://localhost:8086) or
running the following script
```
./check-connectors-state.sh
```

### Run research-service

There are two ways to run `research-service`

1. In a new terminal, run the command below to start the service as a REST API
```
mvn spring-boot:run
```
The swagger link is http://localhost:9080/swagger-ui.html

2. Or you can just run a simulation
```
# Using default values (reviews.total=10 and reviews.delay-interval=0)
mvn spring-boot:run -Dspring-boot.run.profiles=simulation

# Changing values
mvn spring-boot:run \
  -Dspring-boot.run.jvmArguments="-Dspring.profiles.active=simulation -Dreviews.total=100 -Dreviews.delay-interval=0"
```
This mode will create automatically and randomly a certain number of reviews.
- `reviews.total`: total number of reviews you want to be created;
- `reviews.delay-interval`: delay between the creation of reviews.

### Run ksql-cli

1. Run the following command to start `ksql-cli`
```
docker run -it --rm --name ksql-cli \
  --network springboot-kafka-debezium-ksql_default \
  --volume $PWD/docker/ksql.commands:/tmp/ksql.commands \
  confluentinc/cp-ksql-cli:5.1.0 http://ksql-server:8088
```

2. Inside `ksql-cli`, run script to create streams and tables
```
RUN SCRIPT '/tmp/ksql.commands';
```

3. Some commands to check if everything is set
```
SET 'auto.offset.reset' = 'earliest';

DESCRIBE REVIEWS_RESEARCHERS_INSTITUTES_ARTICLES;
SELECT * from REVIEWS_RESEARCHERS_INSTITUTES_ARTICLES;
```

## Useful links & commands

### MySQL
```
docker exec -it mysql bash -c 'mysql -uroot -psecret'
use researchdb
SELECT a.id AS review_id, c.id AS article_id, c.title AS article_title, b.id AS reviewer_id, b.first_name, b.last_name, b.institute_id, a.comment \
  FROM reviews a, researchers b, articles c \
  WHERE a.researcher_id = b.id and a.article_id = c.id;
```

### Elasticsearch
```
curl localhost:9200/_cat/indices?v
curl localhost:9200/mysql.researchdb.institutes/_search?pretty
curl localhost:9200/mysql.researchdb.articles/_search?pretty
```

## TODO

- configure `Elasticsearch Sink Connector` to listen successfully from the topic produced by `ksql`.

## REFERENCES

- https://docs.confluent.io/current/ksql/docs/tutorials/basics-docker.html#ksql-quickstart-docker