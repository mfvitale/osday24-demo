# Debezium Demo

This setup is going to demonstrate how to receive events from PostgreSQL database and stream them down to Elasticsearch using the [Debezium Event Flattening SMT](https://debezium.io/docs/configuration/event-flattening/).

## Table of Contents

* [Elasticsearch Sink](#elasticsearch-sink)
  * [Topology](#topology)
  * [Usage](#usage)
    * [New record](#new-record)
    * [Record update](#record-update)
    * [Record delete](#record-delete)


## Elasticsearch Sink

### Topology

```
                   +-------------+
                   |             |
                   | PostgreSQL  |
                   |             |
                   +------+------+
                          |
                          |
                          |
          +---------------v------------------+
          |                                  |
          |           Kafka Connect          |
          |    (Debezium, ES connectors)     |
          |                                  |
          +---------------+------------------+
                          |
                          |
                          |
                          |
                  +-------v--------+
                  |                |
                  |  Elasticsearch |
                  |                |
                  +----------------+


```
We are using Docker Compose to deploy the following components:

* PostgreSQL
* Kafka
  * ZooKeeper
  * Kafka Broker
  * Kafka Connect with [Debezium](https://debezium.io/) and  [Elasticsearch](https://github.com/confluentinc/kafka-connect-elasticsearch) Connectors
* Elasticsearch

### Usage

How to run:

```shell
# Start the application

docker compose -f docker-compose.yaml up --build -d
```

Check contents of the PostgreSQL database:

```shell
docker compose -f docker-compose.yaml exec postgres bash -c 'PGPASSWORD=$POSTGRES_PASSWORD psql -U $POSTGRES_USER -d inventory -c "select * from inventory.products"'
```
```asciidoc
id  |        name        |                       description                       | weight 
-----+--------------------+---------------------------------------------------------+--------
 101 | scooter            | Small 2-wheel scooter                                   |   3.14
 102 | car battery        | 12V car battery                                         |    8.1
 103 | 12-pack drill bits | 12-pack of drill bits with sizes ranging from #40 to #3 |    0.8
 104 | hammer             | 12oz carpenter's hammer                                 |   0.75
 105 | hammer             | 14oz carpenter's hammer                                 |  0.875
 106 | hammer             | 16oz carpenter's hammer                                 |      1
 107 | rocks              | box of assorted rocks                                   |    5.3
 108 | jacket             | water resistent black wind breaker                      |    0.1
 109 | spare tire         | 24 inch spare tire                                      |   22.2
(9 rows)
```

```shell
# Start Elasticsearch connector

curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @es-sink.json
```

```shell
# Start PostgreSQL connector

curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @source.json
```

Verify that Elasticsearch has the same content:

```shell
curl 'http://localhost:9200/products/_search?pretty'
```
```json
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 9,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "101",
        "_score" : 1.0,
        "_source" : {
          "id" : 101,
          "name" : "scooter",
          "description" : "Small 2-wheel scooter",
          "weight" : 3.14
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "103",
        "_score" : 1.0,
        "_source" : {
          "id" : 103,
          "name" : "12-pack drill bits",
          "description" : "12-pack of drill bits with sizes ranging from #40 to #3",
          "weight" : 0.8
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "104",
        "_score" : 1.0,
        "_source" : {
          "id" : 104,
          "name" : "hammer",
          "description" : "12oz carpenter's hammer",
          "weight" : 0.75
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "105",
        "_score" : 1.0,
        "_source" : {
          "id" : 105,
          "name" : "hammer",
          "description" : "14oz carpenter's hammer",
          "weight" : 0.875
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "106",
        "_score" : 1.0,
        "_source" : {
          "id" : 106,
          "name" : "hammer",
          "description" : "16oz carpenter's hammer",
          "weight" : 1.0
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "107",
        "_score" : 1.0,
        "_source" : {
          "id" : 107,
          "name" : "rocks",
          "description" : "box of assorted rocks",
          "weight" : 5.3
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "108",
        "_score" : 1.0,
        "_source" : {
          "id" : 108,
          "name" : "jacket",
          "description" : "water resistent black wind breaker",
          "weight" : 0.1
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "109",
        "_score" : 1.0,
        "_source" : {
          "id" : 109,
          "name" : "spare tire",
          "description" : "24 inch spare tire",
          "weight" : 22.2
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "102",
        "_score" : 1.0,
        "_source" : {
          "id" : 102,
          "name" : "car battery",
          "description" : "12V car battery",
          "weight" : 8.1
        }
      }
    ]
  }
}
```
#### New record

Insert a new record into PostgreSQL:

```shell
docker compose -f docker-compose.yaml exec postgres bash -c 'PGPASSWORD=$POSTGRES_PASSWORD psql -U $POSTGRES_USER -d inventory'
insert into inventory.products values(default, 'Rubber duck', 'Your debug friend', 10);
```

Check that Elasticsearch contains the new record:

```shell
curl 'http://localhost:9200/products/_search?pretty'
```
```json
     {
  "took" : 902,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 10,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "101",
        "_score" : 1.0,
        "_source" : {
          "id" : 101,
          "name" : "scooter",
          "description" : "Small 2-wheel scooter",
          "weight" : 3.14
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "103",
        "_score" : 1.0,
        "_source" : {
          "id" : 103,
          "name" : "12-pack drill bits",
          "description" : "12-pack of drill bits with sizes ranging from #40 to #3",
          "weight" : 0.8
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "104",
        "_score" : 1.0,
        "_source" : {
          "id" : 104,
          "name" : "hammer",
          "description" : "12oz carpenter's hammer",
          "weight" : 0.75
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "105",
        "_score" : 1.0,
        "_source" : {
          "id" : 105,
          "name" : "hammer",
          "description" : "14oz carpenter's hammer",
          "weight" : 0.875
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "106",
        "_score" : 1.0,
        "_source" : {
          "id" : 106,
          "name" : "hammer",
          "description" : "16oz carpenter's hammer",
          "weight" : 1.0
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "107",
        "_score" : 1.0,
        "_source" : {
          "id" : 107,
          "name" : "rocks",
          "description" : "box of assorted rocks",
          "weight" : 5.3
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "108",
        "_score" : 1.0,
        "_source" : {
          "id" : 108,
          "name" : "jacket",
          "description" : "water resistent black wind breaker",
          "weight" : 0.1
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "109",
        "_score" : 1.0,
        "_source" : {
          "id" : 109,
          "name" : "spare tire",
          "description" : "24 inch spare tire",
          "weight" : 22.2
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "102",
        "_score" : 1.0,
        "_source" : {
          "id" : 102,
          "name" : "car battery",
          "description" : "12V car battery",
          "weight" : 8.1
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "110",
        "_score" : 1.0,
        "_source" : {
          "id" : 110,
          "name" : "Rubber duck",
          "description" : "Your debug friend",
          "weight" : 10.0
        }
      }
    ]
  }
}
```

#### Record update

Update a record in PostgreSQL:

```shell
docker compose -f docker-compose.yaml exec postgres bash -c 'PGPASSWORD=$POSTGRES_PASSWORD psql -U $POSTGRES_USER -d inventory'
update inventory.products set description='Your debug friend! Quack' where name='Rubber duck';
```

Verify that record in Elasticsearch is updated:

```shell
curl 'http://localhost:9200/products/_search?pretty'
```
```json
{
  "took" : 999,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 10,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "101",
        "_score" : 1.0,
        "_source" : {
          "id" : 101,
          "name" : "scooter",
          "description" : "Small 2-wheel scooter",
          "weight" : 3.14
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "103",
        "_score" : 1.0,
        "_source" : {
          "id" : 103,
          "name" : "12-pack drill bits",
          "description" : "12-pack of drill bits with sizes ranging from #40 to #3",
          "weight" : 0.8
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "104",
        "_score" : 1.0,
        "_source" : {
          "id" : 104,
          "name" : "hammer",
          "description" : "12oz carpenter's hammer",
          "weight" : 0.75
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "105",
        "_score" : 1.0,
        "_source" : {
          "id" : 105,
          "name" : "hammer",
          "description" : "14oz carpenter's hammer",
          "weight" : 0.875
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "106",
        "_score" : 1.0,
        "_source" : {
          "id" : 106,
          "name" : "hammer",
          "description" : "16oz carpenter's hammer",
          "weight" : 1.0
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "107",
        "_score" : 1.0,
        "_source" : {
          "id" : 107,
          "name" : "rocks",
          "description" : "box of assorted rocks",
          "weight" : 5.3
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "108",
        "_score" : 1.0,
        "_source" : {
          "id" : 108,
          "name" : "jacket",
          "description" : "water resistent black wind breaker",
          "weight" : 0.1
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "109",
        "_score" : 1.0,
        "_source" : {
          "id" : 109,
          "name" : "spare tire",
          "description" : "24 inch spare tire",
          "weight" : 22.2
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "102",
        "_score" : 1.0,
        "_source" : {
          "id" : 102,
          "name" : "car battery",
          "description" : "12V car battery",
          "weight" : 8.1
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "110",
        "_score" : 1.0,
        "_source" : {
          "id" : 110,
          "name" : "Rubber duck",
          "description" : "Your debug friend! Quack",
          "weight" : 10.0
        }
      }
    ]
  }
}

```


#### Record delete

Delete a record in PostgreSQL:

```shell
docker compose -f docker-compose.yaml exec postgres bash -c 'PGPASSWORD=$POSTGRES_PASSWORD psql -U $POSTGRES_USER -d inventory'
delete from inventory.products where name='Rubber duck';
```

Verify that record in Elasticsearch is deleted:

```shell
curl 'http://localhost:9200/products/_search?pretty'
```
```json
{
  "took" : 943,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 9,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "101",
        "_score" : 1.0,
        "_source" : {
          "id" : 101,
          "name" : "scooter",
          "description" : "Small 2-wheel scooter",
          "weight" : 3.14
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "103",
        "_score" : 1.0,
        "_source" : {
          "id" : 103,
          "name" : "12-pack drill bits",
          "description" : "12-pack of drill bits with sizes ranging from #40 to #3",
          "weight" : 0.8
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "104",
        "_score" : 1.0,
        "_source" : {
          "id" : 104,
          "name" : "hammer",
          "description" : "12oz carpenter's hammer",
          "weight" : 0.75
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "105",
        "_score" : 1.0,
        "_source" : {
          "id" : 105,
          "name" : "hammer",
          "description" : "14oz carpenter's hammer",
          "weight" : 0.875
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "106",
        "_score" : 1.0,
        "_source" : {
          "id" : 106,
          "name" : "hammer",
          "description" : "16oz carpenter's hammer",
          "weight" : 1.0
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "107",
        "_score" : 1.0,
        "_source" : {
          "id" : 107,
          "name" : "rocks",
          "description" : "box of assorted rocks",
          "weight" : 5.3
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "108",
        "_score" : 1.0,
        "_source" : {
          "id" : 108,
          "name" : "jacket",
          "description" : "water resistent black wind breaker",
          "weight" : 0.1
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "109",
        "_score" : 1.0,
        "_source" : {
          "id" : 109,
          "name" : "spare tire",
          "description" : "24 inch spare tire",
          "weight" : 22.2
        }
      },
      {
        "_index" : "products",
        "_type" : "product",
        "_id" : "102",
        "_score" : 1.0,
        "_source" : {
          "id" : 102,
          "name" : "car battery",
          "description" : "12V car battery",
          "weight" : 8.1
        }
      }
    ]
  }
}
```

As you can see there is no longer a 'Rubber duck' as a product.


End the application:

```shell
# Shut down the cluster
docker compose -f docker-compose.yaml down
```