# Change Data Capture with Debezium and Kafka

First make sure docker engine is running on your machine

## Step 1 : docker compose up
Run the **`docker compose up`**

```yml
version: "3.7"
services:
  postgres:
    image: debezium/postgres:13
    ports:
      - 5432:5432
    environment:
      - POSTGRES_USER=test_user
      - POSTGRES_PASSWORD=test_password
      - POSTGRES_DB=test_db

  zookeeper:
    image: confluentinc/cp-zookeeper:5.5.3
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-enterprise-kafka:5.5.3
    depends_on: [zookeeper]
    environment:
      KAFKA_ZOOKEEPER_CONNECT: "zookeeper:2181"
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_BROKER_ID: 1
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_JMX_PORT: 9991
    ports:
      - 9092:9092

  debezium:
    image: debezium/connect:1.4
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: 1
      CONFIG_STORAGE_TOPIC: connect_configs
      OFFSET_STORAGE_TOPIC: connect_offsets
      KEY_CONVERTER: io.confluent.connect.avro.AvroConverter
      VALUE_CONVERTER: io.confluent.connect.avro.AvroConverter
      CONNECT_KEY_CONVERTER_SCHEMA_REGISTRY_URL: http://schema-registry:8081
      CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL: http://schema-registry:8081
    depends_on: [kafka]
    ports:
      - 8083:8083

  schema-registry:
    image: confluentinc/cp-schema-registry:5.5.3
    environment:
      - SCHEMA_REGISTRY_KAFKASTORE_CONNECTION_URL=zookeeper:2181
      - SCHEMA_REGISTRY_HOST_NAME=schema-registry
      - SCHEMA_REGISTRY_LISTENERS=http://schema-registry:8081,http://localhost:8081
    ports:
      - 8081:8081
    depends_on: [zookeeper, kafka]

```

!["Docker containers"](images/docker-compose-up.png?raw=true)

## Step 2 :

Create the products table and set the replication 

```sql
psql -U test_user -d test_db

CREATE TABLE products (id integer primary key, name varchar);

ALTER TABLE public.products REPLICA IDENTITY FULL;
```
!["Docker containers"](images/create-table-and-replication.png?raw=true)


## Step 3 : Register the Debezium Connector for products table
```
 curl -X POST http://localhost:8083/connectors -H 'Content-Type: application/json' -H 'Accept: application/json' -d @debezium.json
```


