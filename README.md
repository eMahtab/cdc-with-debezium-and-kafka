# Change Data Capture with Debezium and Kafka

This demo is based on [Stream your PostgreSQL changes into Kafka with Debezium](https://www.youtube.com/watch?v=YZRHqRznO-o) , we stream database table changes (postgres in this case) to a Kafka topic using [Debezium](https://debezium.io/).

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

## Step 2 : Create products table and set the replication 

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

## Step 4 : Insert/Update/Delete records in products table

```sql
INSERT INTO products(id, name) VALUES (1, 'HOKA Clifton 9');

INSERT INTO products(id, name) VALUES (2, 'HOKA Rincon 3');

UPDATE products set name='HOKA Rincon 3 Men' where id=2;

DELETE FROM products where id=1;
```

## Step 5 : Verify the changes in Kafka topic

```
docker run --tty --network cdc-with-debezium-and-kafka_default confluentinc/cp-kafkacat kafkacat -b kafka:9092 -C -s key=s -s value=avro -r http://schema-registry:8081 -t postgres.public.products
```

```
{"before": null, "after": {"Value": {"id": 1, "name": {"string": "HOKA Clifton 9"}}}, "source": {"version": "1.4.2.Final", "connector": "postgresql", "name": "postgres", "ts_ms": 1735994564233, "snapshot": {"string": "false"}, "db": "test_db", "schema": "public", "table": "products", "txId": {"long": 491}, "lsn": {"long": 23890392}, "xmin": null}, "op": "c", "ts_ms": {"long": 1735994565173}, "transaction": null}
% Reached end of topic postgres.public.products [0] at offset 1
{"before": null, "after": {"Value": {"id": 2, "name": {"string": "HOKA Rincon 3"}}}, "source": {"version": "1.4.2.Final", "connector": "postgresql", "name": "postgres", "ts_ms": 1735995253734, "snapshot": {"string": "false"}, "db": "test_db", "schema": "public", "table": "products", "txId": {"long": 492}, "lsn": {"long": 23890976}, "xmin": null}, "op": "c", "ts_ms": {"long": 1735995254345}, "transaction": null}
% Reached end of topic postgres.public.products [0] at offset 2
{"before": {"Value": {"id": 2, "name": {"string": "HOKA Rincon 3"}}}, "after": {"Value": {"id": 2, "name": {"string": "HOKA Rincon 3 Men"}}}, "source": {"version": "1.4.2.Final", "connector": "postgresql", "name": "postgres", "ts_ms": 1735995346936, "snapshot": {"string": "false"}, "db": "test_db", "schema": "public", "table": "products", "txId": {"long": 493}, "lsn": {"long": 23891656}, "xmin": null}, "op": "u", "ts_ms": {"long": 1735995347285}, "transaction": null}
% Reached end of topic postgres.public.products [0] at offset 3
{"before": {"Value": {"id": 1, "name": {"string": "HOKA Clifton 9"}}}, "after": null, "source": {"version": "1.4.2.Final", "connector": "postgresql", "name": "postgres", "ts_ms": 1735995463993, "snapshot": {"string": "false"}, "db": "test_db", "schema": "public", "table": "products", "txId": {"long": 494}, "lsn": {"long": 23892064}, "xmin": null}, "op": "d", "ts_ms": {"long": 1735995464049}, "transaction": null}
```
!["Kafka Topic"](images/kafka-topic.png?raw=true)


# References :
https://www.youtube.com/watch?v=YZRHqRznO-o
