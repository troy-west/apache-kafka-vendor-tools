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

## Creating and populating topics with the CLI tools

2. In a new terminal, start the Kafka CLI tools, as follows. You will be dropped into a bash shell
from where you can interact with the Kafka brokers:

```sh
docker-compose -f docker-compose.tools.yml run kafka-tools
```

3. Create a new topic "radio-logs" with the following command:

```sh
./bin/kafka-topics.sh --bootstrap-server kafka-1:19092 --create --topic radio-logs --partitions 12 --replication-factor 3
```

4. Populate the "radio-logs" topic by using the code from the streaming compute section in the Apache Kafka workshop.

5. Populate the "logins" topic by using the kafka-console-producer:

```sh
./bin/kafka-console-producer.sh --broker-list kafka-1:19092 --topic logins --property "parse.key=true" --property "key.separator=:"
```

## Starting KSQL and querying Kafka topics via tables and streams

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
 radio-logs  | false      | 12         | 3                  | 0         | 0
-----------------------------------------------------------------------------------------
```

8. View the messsages on the "radio-logs" topic, then hit Ctrl+C to get back to the prompt:

```ksql
print 'radio-logs' from beginning LIMIT 10;
```

You will see something like the following:

```
Format:JSON
{"ROWTIME":1557380525600,"ROWKEY":"353","time":1557125670796,"type":"MOR","name":"353","long":41,"lat":13,"content":[".----"]}
{"ROWTIME":1557380525604,"ROWKEY":"353","time":1557125670821,"type":"MOR","name":"353","long":41,"lat":13,"content":[".----"]}
{"ROWTIME":1557380525605,"ROWKEY":"353","time":1557125670846,"type":"MOR","name":"353","long":41,"lat":13,"content":[".----"]}
{"ROWTIME":1557380525607,"ROWKEY":"095","time":1557125670915,"type":"MOR","name":"095","long":-87,"lat":-29,"content":[".----"]}
{"ROWTIME":1557380525608,"ROWKEY":"095","time":1557125670940,"type":"MOR","name":"095","long":-87,"lat":-29,"content":[".----"]}
{"ROWTIME":1557380525609,"ROWKEY":"095","time":1557125670965,"type":"MOR","name":"095","long":-87,"lat":-29,"content":[".----"]}
{"ROWTIME":1557380525612,"ROWKEY":"032","time":1557125671068,"type":"MOR","name":"032","long":-119,"lat":-39,"content":[".----"]}
{"ROWTIME":1557380525613,"ROWKEY":"032","time":1557125671093,"type":"MOR","name":"032","long":-119,"lat":-39,"content":[".----"]}
{"ROWTIME":1557380525615,"ROWKEY":"032","time":1557125671118,"type":"MOR","name":"032","long":-119,"lat":-39,"content":[".----"]}
{"ROWTIME":1557380525616,"ROWKEY":"122","time":1557125671157,"type":"MOR","name":"122","long":-74,"lat":-24,"content":[".----"]}
```

11. Create a KSQL stream around the "radio-logins" topic:

```ksql
CREATE STREAM radio_logs (time BIGINT, type VARCHAR, name VARCHAR, long VARCHAR, lat VARCHAR, content ARRAY<VARCHAR>)
  WITH (KAFKA_TOPIC='radio-logs', VALUE_FORMAT='JSON', KEY='name', TIMESTAMP='time');
```

12. List the existing KSQL streams:

```ksql
show streams;
```

You should see the following:

```
 Stream Name | Kafka Topic | Format
------------------------------------
 RADIO_LOGS  | radio-logs  | JSON
------------------------------------
```

14. View the details of the "users_by_id" table:

```ksql
describe radio_logs;
```

You should see the following:

```
Name                 : RADIO_LOGS
 Field   | Type
-------------------------------------
 ROWTIME | BIGINT           (system)
 ROWKEY  | VARCHAR(STRING)  (system)
 TIME    | BIGINT
 TYPE    | VARCHAR(STRING)
 NAME    | VARCHAR(STRING)
 LONG    | VARCHAR(STRING)
 LAT     | VARCHAR(STRING)
 CONTENT | ARRAY<VARCHAR(STRING)>
-------------------------------------
```

16. View the content in the "radio_logs" stream, beginning with the earliest entry. The query can take a little time to return the initial results:

```ksql
SET 'auto.offset.reset' = 'earliest';
SELECT * from radio_logs LIMIT 10;
```

The "earliest" setting tells KSQL that every query in this KSQL session should begin from the earliest
offset on each topic, table, and stream.

You should see something like the following:

```
1557125670921 | 040 | 1557125670921 | GER | 040 | -115 | -38 | [eins]
1557125671114 | 485 | 1557125671114 | MOR | 485 | 107 | 35 | [.----]
1557125670806 | 344 | 1557125670806 | MOR | 344 | 37 | 12 | [.----]
1557125670781 | 185 | 1557125670781 | MOR | 185 | -42 | -14 | [.----]
1557125670946 | 040 | 1557125670946 | GER | 040 | -115 | -38 | [eins]
1557125670971 | 040 | 1557125670971 | GER | 040 | -115 | -38 | [eins]
1557125671223 | 502 | 1557125671223 | GER | 502 | 116 | 38 | [eins]
1557125671248 | 502 | 1557125671248 | GER | 502 | 116 | 38 | [eins]
1557125671273 | 502 | 1557125671273 | GER | 502 | 116 | 38 | [eins]
1557125671868 | 477 | 1557125671868 | ENG | 477 | 103 | 34 | [one]
```

Press Ctrl+c to exit the query.

17. Now, let's search for a specific entry in "radio_logs":

```ksql
SELECT * FROM radio_logs WHERE type='ENG' and name='324' LIMIT 10;
```

We should get multiple results, as follows:

```
1557125672954 | 324 | 1557125672954 | ENG | 324 | 27 | 9 | [one]
1557125672979 | 324 | 1557125672979 | ENG | 324 | 27 | 9 | [one]
1557125673004 | 324 | 1557125673004 | ENG | 324 | 27 | 9 | [one]
1557125680792 | 324 | 1557125680792 | ENG | 324 | 27 | 9 | [one]
1557125680817 | 324 | 1557125680817 | ENG | 324 | 27 | 9 | [one]
1557125680842 | 324 | 1557125680842 | ENG | 324 | 27 | 9 | [one]
1557125695496 | 324 | 1557125695496 | ENG | 324 | 27 | 9 | [one]
1557125695521 | 324 | 1557125695521 | ENG | 324 | 27 | 9 | [one]
1557125695546 | 324 | 1557125695546 | ENG | 324 | 27 | 9 | [one]
1557125705652 | 324 | 1557125705652 | ENG | 324 | 27 | 9 | [one]
Limit Reached
```

18. Now, limit the query to only those with content having at least 3 items:

```ksql
select * from radio_logs WHERE content[2] IS NOT NULL limit 10;
```

You should see something like the following:

```
1557126942198 | 423 | 1557126942198 | ENG | 423 | 76 | 25 | [one, three, six]
1557126942223 | 423 | 1557126942223 | ENG | 423 | 76 | 25 | [one, three, nine]
1557126942248 | 423 | 1557126942248 | ENG | 423 | 76 | 25 | [one, two, seven]
1557126957782 | 423 | 1557126957782 | ENG | 423 | 76 | 25 | [one, five, nine]
1557126957807 | 423 | 1557126957807 | ENG | 423 | 76 | 25 | [one, six, three]
1557126957832 | 423 | 1557126957832 | ENG | 423 | 76 | 25 | [one, four, nine]
1557126965396 | 423 | 1557126965396 | ENG | 423 | 76 | 25 | [one, five, eight]
1557126965421 | 423 | 1557126965421 | ENG | 423 | 76 | 25 | [one, six, two]
1557126965446 | 423 | 1557126965446 | ENG | 423 | 76 | 25 | [one, four, eight]
1557126971643 | 423 | 1557126971643 | ENG | 423 | 76 | 25 | [one, three, five]
Limit Reached
```

19. Count how many messages each radio station is sending each minute:

```ksql
SELECT CAST(windowStart() AS BIGINT), type, name, count(*)
    FROM radio_logs
    WINDOW TUMBLING (SIZE 1 MINUTE)
    GROUP BY type, name;
```

You may see something like the following:

```
1557126840000 | MOR | 470 | 18
1557126420000 | MOR | 416 | 18
1557126420000 | ENG | 444 | 18
1557126420000 | ENG | 117 | 18
1557126420000 | MOR | 044 | 18
1557126480000 | ENG | 117 | 3
1557126840000 | GER | 523 | 18
1557126900000 | GER | 523 | 6
1557126900000 | GER | 445 | 6
```

20. From the select, create a table from this query:

```ksql
CREATE TABLE radio_log_count
  WITH (KAFKA_TOPIC='radio_log_count', VALUE_FORMAT = 'DELIMITED')
  AS SELECT CAST(windowStart() AS BIGINT), type, name, count(*)
        FROM radio_logs
        WINDOW TUMBLING (SIZE 1 MINUTE)
        GROUP BY type, name;
```

21. Now, list the topics managed by kafka:

```ksql
list topics;
```

You should see the following. Notice the "radio_log_count" topic that has been created:

```
 Kafka Topic     | Registered | Partitions | Partition Replicas | Consumers | ConsumerGroups
---------------------------------------------------------------------------------------------
 _schemas        | false      | 1          | 3                  | 0         | 0
 radio-logs      | true       | 12         | 3                  | 24        | 2
 radio_log_count | true       | 4          | 1                  | 4         | 1
---------------------------------------------------------------------------------------------
```

It has used a default value for the number of partitions.

## Exporting Kafka topics using Kafka Connect

25. From the kafka-tools cli, start Kafka Connect in the background:

```sh
./bin/connect-standalone.sh /root/data/connect-standalone.properties /root/data/connect-file-sink-csv.properties &> kafka-connect-logs.txt &
```

26. A new file "radio_log_count.csv" should be created and populated with the results of our "radio_log_count" table. Run the following to view its content:

```
cat radio_log_counts.csv
```

You should see something like the following:

```
1557134280000,GER,268,18
1557134280000,MOR,050,18
1557134280000,MOR,251,18
1557134280000,MOR,524,18
1557134340000,GER,268,9
1557134340000,MOR,050,9
1557134340000,MOR,251,12
1557134340000,MOR,524,12
```
