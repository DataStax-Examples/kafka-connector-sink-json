# Ingest JSON from Kafka to Cassandra
This example shows how to ingest JSON records from [Kafka](https://kafka.apache.org/) to multiple tables in the [Cassandra](https://cassandra.apache.org/) database using the [DataStax Apache Kafka Connector](https://docs.datastax.com/en/kafka/doc/index.html).

Contributor(s): [Chris Splinter](https://github.com/csplinter), [Tomasz Lelek](https://github.com/tomekl007)

Have Questions? We're here to help: https://community.datastax.com/

Want to learn more about the DataStax Kafka Connector? Take a free, [short course on DataStax Academy](https://academy.datastax.com/resources/getting-started-datastax-apache-kafka%E2%84%A2-connector)

## Objectives
- How to ingest JSON records from Kafka to Cassandra databases
- How to use docker and docker-compose to quickly set up an environment with Zookeeper, Kafka Brokers, Kafka Connect and Cassandra

## Project Layout
- [Dockerfile-connector](Dockerfile-connector): Dockerfile to build an image of Kafka Connect with the DataStax Kafka Connector installed.
- [Dockerfile-producer](Dockerfile-producer): Dockerfile to build an image for the producer contained in this repository.
- [docker-compose.yml](docker-compose.yml): Uses [Confluent](https://www.confluent.io/) and Cassandra docker images to set up Zookeeper, Kafka Brokers, Kafka Connect, Apache Cassandra, and the producer container.
- [connector-config.json](connector-config.json): Configuration file for the DataStax Kafka Connector to be used with the distributed Kafka Connect Worker.
- [producer](producer/): Contains the Kafka Java Producer to write records to Kafka. Uses the StringSerializer for the Kafka record key and the JsonSerializer for the Kafka record value.

## How this works
After running the docker and docker-compose commands, there will be 5 docker containers running, all using the same docker network.

After writing records to the Kafka Brokers, the DataStax Kafka Connector will be started which will start the stream of records from Kafka to the Cassandra database, writing a single record to three different tables in the database, showing how to achieve the common Cassandra pattern of denormalization with the connector.

## Setup & Running
### Prerequisites
- Docker: https://docs.docker.com/v17.09/engine/installation/
- Docker Compose: https://docs.docker.com/compose/install/

### Setup
Clone this repository
```
git clone https://github.com/DataStax-Examples/kafka-connector-sink-json.git
```

Go to the directory
```
cd kafka-connector-sink-json
```

Build the DataStax Kafka Connector image
```
docker build --no-cache -t datastax-connect -f Dockerfile-connector .
```

Build the JSON Java Producer image
```
docker build . -t kafka-producer -f Dockerfile-producer
```

Start Zookeeper, Kafka Brokers, Kafka Connect, Cassandra, and the producer containers
```
docker-compose up -d
```

### Running
Now that everything is up and running, it's time to set up the flow of data from Kafka to Cassandra.

#### Create the Kafka topic
Start a bash shell on the Kafka Broker
```
docker exec -it kafka-broker bash
```
Create the topic
```
kafka-topics --create --zookeeper zookeeper:2181 --replication-factor 1 --partitions 10 --topic json-stream --config retention.ms=-1
```

#### Create the Cassandra tables
Start a cqlsh shell on the Cassandra node
```
docker exec -it cassandra cqlsh
```
Create the tables that the connector will write to. Note that a single instance of the connector can write Kafka records to multiple tables.
```
create keyspace if not exists kafka_examples with replication = {'class': 'SimpleStrategy', 'replication_factor': 1};
create table if not exists kafka_examples.stocks_table_by_symbol (symbol text, datetime timestamp, exchange text, industry text, name text, value double, PRIMARY KEY (symbol, datetime));
create table if not exists kafka_examples.stocks_table_by_exchange (symbol text, datetime timestamp, exchange text, industry text, name text, value double, PRIMARY KEY (exchange, datetime));
create table if not exists kafka_examples.stocks_table_by_industry (symbol text, datetime timestamp, exchange text, industry text, name text, value double, PRIMARY KEY (industry, datetime));
```

#### Load data into Kafka
Start a bash shell on the Kafka Producer
```
docker exec -it kafka-producer bash
```
Write 1000 records ( 10 stocks, 100 records per stock ) to Kafka using the JSON Java Producer in this project
```
mvn clean compile exec:java -Dexec.mainClass=json.JsonProducer -Dexec.args="json-stream 10 100 broker:29092"
```
There will be many lines of output in your console as Maven pulls down the dependencies. The following output means that it completed successfully
```
2020-03-09 18:01:34.268 [json.JsonProducer.main()] INFO  - Completed loading 1000/1000 records to Kafka in 1 seconds
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 20.254 s
[INFO] Finished at: 2020-03-09T18:01:34+00:00
[INFO] Final Memory: 31M/215M
[INFO] ------------------------------------------------------------------------
```

#### Start the DataStax Kafka Connector
Execute the following command from the machine where docker is running to start the connector using the Kafka Connect REST API
```
curl -X POST -H "Content-Type: application/json" -d @connector-config.json "http://localhost:8083/connectors"
```

#### Confirm rows written to Cassandra
Start a cqlsh shell on the Cassandra node
```
docker exec -it cassandra cqlsh
```

Confirm rows were written to each of the Cassandra tables
```
select * from kafka_examples.stocks_table_by_symbol limit 10;
```
```
 symbol | datetime                        | exchange | industry | name        | value
--------+---------------------------------+----------+----------+-------------+----------
    XOM | 2020-03-09 18:27:07.289000+0000 |     NYSE |   ENERGY | EXXON MOBIL | 79.53462
    XOM | 2020-03-09 18:27:17.289000+0000 |     NYSE |   ENERGY | EXXON MOBIL | 79.94343
    XOM | 2020-03-09 18:27:27.289000+0000 |     NYSE |   ENERGY | EXXON MOBIL | 79.46183
    XOM | 2020-03-09 18:27:37.289000+0000 |     NYSE |   ENERGY | EXXON MOBIL |  80.1765
    XOM | 2020-03-09 18:27:47.289000+0000 |     NYSE |   ENERGY | EXXON MOBIL | 80.44787
    XOM | 2020-03-09 18:27:57.289000+0000 |     NYSE |   ENERGY | EXXON MOBIL |  79.9512
    XOM | 2020-03-09 18:28:07.289000+0000 |     NYSE |   ENERGY | EXXON MOBIL | 80.08623
    XOM | 2020-03-09 18:28:17.289000+0000 |     NYSE |   ENERGY | EXXON MOBIL | 80.42811
    XOM | 2020-03-09 18:28:27.289000+0000 |     NYSE |   ENERGY | EXXON MOBIL | 80.22866
    XOM | 2020-03-09 18:28:37.289000+0000 |     NYSE |   ENERGY | EXXON MOBIL | 80.00116

(10 rows)
```
```
select * from kafka_examples.stocks_table_by_exchange limit 10;
```
```
 exchange | datetime                        | industry | name  | symbol | value
----------+---------------------------------+----------+-------+--------+-----------
   NASDAQ | 2020-03-09 18:27:06.289000+0000 |     TECH | APPLE |   APPL | 208.25739
   NASDAQ | 2020-03-09 18:27:16.289000+0000 |     TECH | APPLE |   APPL | 208.39239
   NASDAQ | 2020-03-09 18:27:26.289000+0000 |     TECH | APPLE |   APPL | 208.26644
   NASDAQ | 2020-03-09 18:27:36.289000+0000 |     TECH | APPLE |   APPL | 207.48437
   NASDAQ | 2020-03-09 18:27:46.289000+0000 |     TECH | APPLE |   APPL | 207.42801
   NASDAQ | 2020-03-09 18:27:56.289000+0000 |     TECH | APPLE |   APPL | 207.62685
   NASDAQ | 2020-03-09 18:28:06.289000+0000 |     TECH | APPLE |   APPL | 207.62004
   NASDAQ | 2020-03-09 18:28:16.289000+0000 |     TECH | APPLE |   APPL | 206.49582
   NASDAQ | 2020-03-09 18:28:26.289000+0000 |     TECH | APPLE |   APPL | 206.21018
   NASDAQ | 2020-03-09 18:28:36.289000+0000 |     TECH | APPLE |   APPL | 205.53896

(10 rows)
```
```
select * from kafka_examples.stocks_table_by_industry limit 10;
```
```
 industry | datetime                        | exchange | name    | symbol | value
----------+---------------------------------+----------+---------+--------+----------
   RETAIL | 2020-03-09 18:27:04.289000+0000 |     NYSE | WALMART |    WMT | 89.45163
   RETAIL | 2020-03-09 18:27:14.289000+0000 |     NYSE | WALMART |    WMT | 89.36504
   RETAIL | 2020-03-09 18:27:24.289000+0000 |     NYSE | WALMART |    WMT | 89.24324
   RETAIL | 2020-03-09 18:27:34.289000+0000 |     NYSE | WALMART |    WMT | 89.83376
   RETAIL | 2020-03-09 18:27:44.289000+0000 |     NYSE | WALMART |    WMT |  90.1238
   RETAIL | 2020-03-09 18:27:54.289000+0000 |     NYSE | WALMART |    WMT |  89.5875
   RETAIL | 2020-03-09 18:28:04.289000+0000 |     NYSE | WALMART |    WMT | 90.08323
   RETAIL | 2020-03-09 18:28:14.289000+0000 |     NYSE | WALMART |    WMT | 89.49746
   RETAIL | 2020-03-09 18:28:24.289000+0000 |     NYSE | WALMART |    WMT | 89.15786
   RETAIL | 2020-03-09 18:28:34.289000+0000 |     NYSE | WALMART |    WMT | 89.12892

(10 rows)
```
