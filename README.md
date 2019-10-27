# Ingest JSON from Kafka
This example shows how to ingest JSON records from Kafka to multiple tables in the DataStax database using the DataStax Apache Kafka Connector.

## Project Layout
- [Dockerfile-connector](Dockerfile-connector): Dockerfile to build an image of Kafka Connect with the DataStax Kafka Connector installed.
- [Dockerfile-producer](Dockerfile-producer): Dockerfile to build an image for the producer contained in this repository.
- [docker-compose.yml](docker-compose.yml): Uses Confluent and DataStax docker images to set up Zookeeper, Kafka Brokers, Kafka Connect, DDAC, and the producer container.
- [connector-config.json](connector-config.json): Configuration file for the DataStax Kafka Connector to be used with the distributed Kafka Connect Worker.
- [producer](producer/): Contains the Kafka Java Producer to write records to Kafka. Uses the StringSerializer for the Kafka record key and the JsonSerializer for the Kafka record value.

## Before you begin
#### Prerequisites
- Docker: https://docs.docker.com/v17.09/engine/installation/
- Docker Compose: https://docs.docker.com/compose/install/

## Setup
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
docker build . -t datastax-connect -f Dockerfile-connector
```

Build the JSON Java Producer image
```
docker build . -t kafka-producer -f Dockerfile-producer
```

Start Zookeeper, Kafka Brokers, Kafka Connect, DDAC, and the producer containers
```
docker-compose up -d
```

## Running
Now that everything is up and running, it's time to set up the flow of data from Kafka to the DataStax database.

Create the Kafka Topic named `json-stream` that the connector will read from.
```
docker exec -it kafka-broker bash
```
```
kafka-topics --create --zookeeper zookeeper:2181 --replication-factor 1 --partitions 10 --topic json-stream --config retention.ms=-1
```

Create the DataStax tables that the connector will write to. Note that a single instance of the connector can write Kafka records to multiple tables.
```
docker exec -it datastax-db cqlsh
```
```
create keyspace if not exists kafka_examples with replication = {'class': 'SimpleStrategy', 'replication_factor': 1};
create table if not exists kafka_examples.stocks_table_by_symbol (symbol text, datetime timestamp, exchange text, industry text, name text, value double, PRIMARY KEY (symbol, datetime));
create table if not exists kafka_examples.stocks_table_by_exchange (symbol text, datetime timestamp, exchange text, industry text, name text, value double, PRIMARY KEY (exchange, datetime));
create table if not exists kafka_examples.stocks_table_by_industry (symbol text, datetime timestamp, exchange text, industry text, name text, value double, PRIMARY KEY (industry, datetime));
```

Write 1000 records ( 10 stocks, 100 records per stock ) to Kafka using the JSON Java Producer
```
docker exec -it kafka-producer bash
```
```
mvn clean compile exec:java -Dexec.mainClass=json.JsonProducer -Dexec.args="json-stream 10 100 broker:29092"
```

Start the DataStax Kafka Connector using the Kafka Connect REST API
```
curl -X POST -H "Content-Type: application/json" -d @connector-config.json "http://localhost:8083/connectors"
```

Confirm that the rows were written in the DataStax database
```
docker exec -it datastax-db cqlsh
```
```
select * from kafka_examples.stocks_table_by_symbol limit 10;
select * from kafka_examples.stocks_table_by_exchange limit 10;
select * from kafka_examples.stocks_table_by_industry limit 10;
```