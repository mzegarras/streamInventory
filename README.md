# streamInventory

## 1 Clean
```bash
cd /Users/manuelzegarra/Proyectos/Galaxy/Talleres/streamInventory
docker-compose down
docker volume rm $(docker volume ls -qf dangling=true)
```

## 2 Startup

```bash
docker-compose up -d
```

## 3 Registrar schemas avros

```bash
jq '. | {schema: tojson}' ./data-gen/schemas/OrderEvent.avsc  | \
    curl -X POST http://localhost:8081/subjects/t.orders-value/versions \
         -H "Content-Type:application/json" \
         -d @-

curl -s http://localhost:8081/subjects | jq "."
curl -s http://localhost:8081/subjects/t.orders-value/versions | jq "."
curl -s http://localhost:8081/subjects/t.orders-value/versions/1 | jq "."

```


## 4 Registrar topics

```bash
docker exec kafka01 kafka-topics --bootstrap-server kafka01:9092 \
    --create \
    --topic t.orders \
    --partitions 1 \
    --replication-factor 3
```


## 5 Crear data dummy
```bash
docker exec -it connect-1 \
  curl -X POST http://localhost:8083/connectors \
    -H "Content-Type: application/json" \
    --data @/tmp/data-gen/config/orders_datagen.config

docker exec -it connect-1 \
  curl -X GET http://localhost:8083/connectors/DatagenUsers01

docker exec -it connect-1 \
  curl -X DELETE http://localhost:8083/connectors/DatagenUsers01
  
```

## 6 Connect ksqlDB
```bash
docker exec -it ksqldb-cli ksql http://ksqldb-server:8088
SET 'auto.offset.reset' = 'earliest';
show topics;

print "t.orders" limit 10;
print "t.orders" from beginning;
```

## 6 Crear streams inventory

```bash
CREATE STREAM INVENTORY_ST WITH (
    kafka_topic='t.orders', 
    value_format='avro',
    timestamp = 'created'
  );

SELECT * FROM inventory_st EMIT CHANGES;
describe INVENTORY_ST extended;
```

## 7 Stock por producto
```bash
CREATE OR REPLACE TABLE stock_tb AS
select productId,SUM(amount) AS TOTAL,COUNT(1) AS COUNT 
FROM inventory_st  
group by productId 
emit changes;

SELECT * FROM stock_tb EMIT CHANGES;
SELECT * FROM stock_tb where productId='ALEX'  EMIT CHANGES;

DESCRIBE inventory_st EXTENDED;
```