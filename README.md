# Exploring Apache Kafka vendor tooling: KSQL, Kafka connect

## References

- https://www.confluent.io/product/ksql/
- https://kafka.apache.org/documentation/#connect

## Goals

- Use the KSQL DSL to perform streaming queries.
- Use Kafka Connect to export CSV results from KSQL to a file.

## Running Kafka, KSQL, and the Schema Registry

1. Bootstrap a 3-node kafka cluster, the KSQL server, and the Schema Registry with the following command:

```sh
docker-compose up
```

2. In a new terminal, start the Kafka CLI tools, as follows. You will be dropped into a bash shell
from where you can interact with the Kafka brokers:

```sh
docker-compose -f docker-compose.tools.yml run kafka-tools
```

3. Create two new topics "users" and "logins" and populate them with data:

```sh
./bin/kafka-topics.sh --bootstrap-server kafka-1:19092 --create --topic users --partitions 12 --replication-factor 3
./bin/kafka-topics.sh --bootstrap-server kafka-1:19092 --create --topic logins --partitions 12 --replication-factor 3
```

4. Populate the "users" topic by using the kafka-console-producer:

```sh
./bin/kafka-console-producer.sh --broker-list kafka-1:19092 --topic users --property "parse.key=true" --property "key.separator=:"
```

Copy the following data into the terminal, and then press Ctrl+d to exit back to the shell:

```json
1:{"id":1, "name":"Jane Doe"}
2:{"id":2, "name":"John Smith"}
3:{"id":3, "name":"Mr. Meeseeks"}
```

5. Populate the "logins" topic by using the kafka-console-producer:

```sh
./bin/kafka-console-producer.sh --broker-list kafka-1:19092 --topic logins --property "parse.key=true" --property "key.separator=:"
```

Copy the following data into the terminal, and then press Ctrl+d to exit back to the shell:

```json
1:{"time": 1000, "user_id": 1}
2:{"time": 2000, "user_id": 2}
3:{"time": 3000, "user_id": 3}
2:{"time": 4000, "user_id": 2}
2:{"time": 5000, "user_id": 2}
3:{"time": 6000, "user_id": 3}
3:{"time": 7000, "user_id": 3}
3:{"time": 8000, "user_id": 3}
1:{"time": 9000, "user_id": 1}
1:{"time": 10000, "user_id": 1}
1:{"time": 11000, "user_id": 1}
2:{"time": 12000, "user_id": 2}
2:{"time": 13000, "user_id": 2}
1:{"time": 14000, "user_id": 1}
```

6. In a new terminal, start the KSQL CLI tool:

```sh
docker-compose -f docker-compose.ksql.yml run ksql
```

7. At the KSQL command prompt, enter the following command to list the existing topics:

```ksql
list topics;
```

You will see something like the following:

```
 Kafka Topic | Registered | Partitions | Partition Replicas | Consumers | ConsumerGroups
-----------------------------------------------------------------------------------------
 _schemas    | false      | 1          | 3                  | 0         | 0
 logins      | false      | 12         | 3                  | 0         | 0
 users       | false      | 12         | 3                  | 0         | 0
-----------------------------------------------------------------------------------------
```

8. View the messsages on the "users" topic, then hit Ctrl+C to get back to the prompt:

```ksql
print 'users' from beginning;
```

You will see something like the following:

```
Format:JSON
{"ROWTIME":1557363960776,"ROWKEY":"2","id":2,"name":"John Smith"}
{"ROWTIME":1557363960776,"ROWKEY":"3","id":3,"name":"Mr. Meeseeks"}
{"ROWTIME":1557363960744,"ROWKEY":"1","id":1,"name":"Jane Doe"}
```

9. View the messages on the "logins" topic, then hit Ctrl+C to get back to the prompt:

```ksql
print 'logins' from beginning;
```

You will see something like the following:

```
Format:JSON
{"ROWTIME":1557364037528,"ROWKEY":"3","time":3000,"user_id":3}
{"ROWTIME":1557364037533,"ROWKEY":"3","time":6000,"user_id":3}
{"ROWTIME":1557364037533,"ROWKEY":"3","time":7000,"user_id":3}
{"ROWTIME":1557364037534,"ROWKEY":"3","time":8000,"user_id":3}
{"ROWTIME":1557364037527,"ROWKEY":"2","time":2000,"user_id":2}
{"ROWTIME":1557364037533,"ROWKEY":"2","time":4000,"user_id":2}
{"ROWTIME":1557364037533,"ROWKEY":"2","time":5000,"user_id":2}
{"ROWTIME":1557364037535,"ROWKEY":"2","time":12000,"user_id":2}
{"ROWTIME":1557364037535,"ROWKEY":"2","time":13000,"user_id":2}
{"ROWTIME":1557364037510,"ROWKEY":"1","time":1000,"user_id":1}
{"ROWTIME":1557364037534,"ROWKEY":"1","time":9000,"user_id":1}
{"ROWTIME":1557364037534,"ROWKEY":"1","time":10000,"user_id":1}
{"ROWTIME":1557364037534,"ROWKEY":"1","time":11000,"user_id":1}
{"ROWTIME":1557364037535,"ROWKEY":"1","time":14000,"user_id":1}
```

10. Create a KSQL table around the "users" topic:

```ksql
CREATE TABLE users_by_id (id BIGINT, name VARCHAR)
  WITH (KAFKA_TOPIC='users', VALUE_FORMAT='JSON', KEY='id');
```

11. Create a KSQL stream around the "logins" topic":

```ksql
CREATE STREAM logins_by_id (time BIGINT, user_id VARCHAR)
  WITH (KAFKA_TOPIC='logins', VALUE_FORMAT='JSON', KEY='user_id');
```

12. List the existing KSQL tables:

```ksql
show tables;
```

You should see the following:

```
Table Name  | Kafka Topic | Format | Windowed
-----------------------------------------------
USERS_BY_ID | users       | JSON   | false
-----------------------------------------------
```

13. List the existing KSQL streams:

```ksql
show streams;
```

You should see the following:

```
 Stream Name  | Kafka Topic | Format
-------------------------------------
 LOGINS_BY_ID | logins      | JSON
-------------------------------------
```

14. View the details of the "users_by_id" table:

```ksql
describe users_by_id;
```

You should see the following:

```
Name                 : USERS_BY_ID
 Field   | Type
-------------------------------------
 ROWTIME | BIGINT           (system)
 ROWKEY  | VARCHAR(STRING)  (system)
 ID      | BIGINT
 NAME    | VARCHAR(STRING)
-------------------------------------
```

15. View the details of the "logins_by_id" table:

```ksql
describe logins_by_id;
```

You should see the following:

```
Name                 : LOGINS_BY_ID
 Field   | Type
-------------------------------------
 ROWTIME | BIGINT           (system)
 ROWKEY  | VARCHAR(STRING)  (system)
 TIME    | BIGINT
 USER_ID | VARCHAR(STRING)
-------------------------------------
```

16. View the content in the "user_by_id" table, beginning with the earliest entry. The query can take a little time to return the initial results:

```ksql
SET 'auto.offset.reset' = 'earliest';
SELECT * from users_by_id;
```

The "earliest" setting tells KSQL that every query in this KSQL session should begin from the earliest
messages on each topic, table, and stream.

17. View the content in the "logins_by_id" stream, beginning with the earliest entry. The query can take a little time to return the initial results:

```ksql
SELECT * from logins_by_id;
```

18. Now, join the "users_by_id" table with the "logins_by_id" stream, to see which users are logging in over time:

```ksql
SELECT time, user_id, name FROM logins_by_id LEFT JOIN users_by_id ON logins_by_id.user_id = users_by_id.id;
```

You should see something like the following:

```
1000 | 1 | Jane Doe
9000 | 1 | Jane Doe
10000 | 1 | Jane Doe
11000 | 1 | Jane Doe
14000 | 1 | Jane Doe
3000 | 3 | Mr. Meeseeks
6000 | 3 | Mr. Meeseeks
7000 | 3 | Mr. Meeseeks
2000 | 2 | John Smith
4000 | 2 | John Smith
5000 | 2 | John Smith
8000 | 3 | Mr. Meeseeks
12000 | 2 | John Smith
13000 | 2 | John Smith
```

19. See how many times users are logging in within a time window:

```ksql
SELECT CAST(windowStart() AS BIGINT), user_id, name, count(time)
    FROM logins_by_id LEFT JOIN users_by_id ON logins_by_id.user_id = users_by_id.id
    WINDOW TUMBLING (SIZE 1 SECONDS)
    GROUP BY user_id, name;
```

Create a table from this query:

```ksql
CREATE TABLE user_logins WITH (PARTITIONS=12) AS
  SELECT CAST(windowStart() AS BIGINT), user_id, name, count(time) as count
      FROM logins_by_id LEFT JOIN users_by_id ON logins_by_id.user_id = users_by_id.id
      WINDOW TUMBLING (SIZE 1 SECONDS)
      GROUP BY user_id, name;
```

20. Now, list the topics managed by kafka:

```ksql
list topics;
```

You should see the following. Notice the "USER_LOGINS" topic that has been created:

```
 Kafka Topic | Registered | Partitions | Partition Replicas | Consumers | ConsumerGroups
-----------------------------------------------------------------------------------------
 _schemas    | false      | 1          | 3                  | 0         | 0
 logins      | true       | 12         | 3                  | 12        | 1
 USER_LOGINS | true       | 12         | 1                  | 0         | 0
 users       | true       | 12         | 3                  | 12        | 1
-----------------------------------------------------------------------------------------
```

21. Find those users which are logging in very often:

```ksql
SELECT * FROM user_logins WHERE count >= 5;
```

Send this query to a topic via a table, so that we can use kafka connect to export the results to a CSV formatted file:

```ksql
CREATE TABLE user_logins_delimited
  WITH (KAFKA_TOPIC='user_logins_delimited', VALUE_FORMAT = 'DELIMITED')
  AS SELECT * FROM user_logins WHERE count >= 5;
```

From the kafka-tools cli, start kafka connect in the background:

```sh
./bin/connect-standalone.sh /root/data/connect-standalone.properties /root/data/connect-file-sink-csv.properties 2>&1 > kafka-connect-logs.txt &
```

A new file "logins.csv" should be created and populated with the results of our "user_logins_delimited" table:

```
cat logins.csv
```

You should see something like the following:

```
1557365998000,1,Jane Doe,5
1557365998000,2,John Smith,5
```

Now if we pipe more logins into the original "logins" topic:

```sh
./bin/kafka-console-producer.sh --broker-list kafka-1:19092 --topic logins --property "parse.key=true" --property "key.separator=:"
```

```
1:{"time": 210000, "user_id": 1}
1:{"time": 211000, "user_id": 1}
2:{"time": 212000, "user_id": 2}
2:{"time": 213000, "user_id": 2}
1:{"time": 214000, "user_id": 1}
1:{"time": 21000, "user_id": 1}
2:{"time": 22000, "user_id": 2}
3:{"time": 23000, "user_id": 3}
2:{"time": 24000, "user_id": 2}
2:{"time": 25000, "user_id": 2}
3:{"time": 26000, "user_id": 3}
3:{"time": 27000, "user_id": 3}
3:{"time": 28000, "user_id": 3}
1:{"time": 29000, "user_id": 1}
1:{"time": 210000, "user_id": 1}
1:{"time": 211000, "user_id": 1}
2:{"time": 212000, "user_id": 2}
2:{"time": 213000, "user_id": 2}
1:{"time": 214000, "user_id": 1}
```

And view the csv file again:

```
cat logins.csv
```

You will see that the logins.csv has accumulated more entries.







WIP WIP WIP below here WIP WIP WIP
WIP WIP WIP below here WIP WIP WIP
WIP WIP WIP below here WIP WIP WIP
WIP WIP WIP below here WIP WIP WIP
WIP WIP WIP below here WIP WIP WIP
WIP WIP WIP below here WIP WIP WIP
WIP WIP WIP below here WIP WIP WIP

# Schema registry

```sh
curl -s http://schema-registry:8081/subjects | jq .
```

[
  "_confluent-ksql-default_query_CTAS_J_TIME_GROUPED_4-Aggregate-aggregate-changelog-value",
  "J_TIME_GROUPED-value",
  "J_AVRO-value",
  "_confluent-ksql-default_query_CTAS_J_TIME_GROUPED2_5-Aggregate-aggregate-changelog-value",
  "_confluent-ksql-default_query_CTAS_J_TIME_GROUPED_4-Aggregate-groupby-repartition-value",
  "SMALL_NUMBERS-value",
  "J_TIMESTAMPED-value",
  "J_TIME_GROUPED2-value",
  "_confluent-ksql-default_query_CTAS_J_TIME_GROUPED2_5-Aggregate-groupby-repartition-value"
]

```sh
curl -s http://schema-registry:8081/subjects/J_TIMESTAMPED-value/versions/1 | jq '.schema | fromjson'
```

{
  "type": "record",
  "name": "KsqlDataSourceSchema",
  "namespace": "io.confluent.ksql.avro_schemas",
  "fields": [
    {
      "name": "TIME",
      "type": [
        "null",
        "long"
      ],
      "default": null
    },
    {
      "name": "NAME",
      "type": [
        "null",
        "string"
      ],
      "default": null
    },
    {
      "name": "NUMBER",
      "type": [
        "null",
        "long"
      ],
      "default": null
    }
  ]
}







curl -s http://localhost:8083/connectors/local-file-sink-json | jq .

{
  "name": "local-file-sink-json",
  "config": {
    "connector.class": "org.apache.kafka.connect.file.FileStreamSinkConnector",
    "file": "test.sink.csv",
    "tasks.max": "1",
    "topics": "csv-delim",
    "name": "local-file-sink-json"
  },
  "tasks": [
    {
      "connector": "local-file-sink-json",
      "task": 0
    }
  ],
  "type": "sink"
}




# joins

CREATE STREAM pageviews_transformed
  WITH (TIMESTAMP='time',
        PARTITIONS=5,
        VALUE_FORMAT='JSON') AS
  SELECT viewtime,
         userid,
         pageid,
         time
  FROM pageviews
  PARTITION BY userid;

CREATE TABLE users_5part
    WITH (PARTITIONS=5) AS
    SELECT * FROM USERS;

CREATE STREAM pageviews_enriched AS
  SELECT pv.viewtime,
         pv.userid AS userid,
         pv.pageid,
         pv.timestring,
         u.gender,
         u.regionid,
         u.interests,
         u.contactinfo
  FROM pageviews_transformed pv
  LEFT JOIN users_5part u ON pv.userid = u.userid;
