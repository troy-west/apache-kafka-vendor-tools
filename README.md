# Exploring Apache Kafka vendor tooling: KSQL, Kafka connect

## References

- https://www.confluent.io/product/ksql/
- https://kafka.apache.org/documentation/#connect

## Goals

- Use the KSQL DSL to perform streaming queries.
- Use Kafka Connect to export CSV results from KSQL to a file.

## Running Kafka, KSQL, and the Schema Registry

Shutdown any kafka brokers that you may have run from previous exercise, then follow the steps below.

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

4. Populate the "radio-logs" topic by using the code from the streaming compute section in previous section of the Apache Kafka workshop.

## Starting KSQL and querying Kafka topics via tables and streams

5. In a new terminal, start the KSQL CLI tool:

```sh
docker-compose -f docker-compose.ksql.yml run ksql
```

You should see the following appear:

```
                  ===========================================
                  =        _  __ _____  ____  _             =
                  =       | |/ // ____|/ __ \| |            =
                  =       | ' /| (___ | |  | | |            =
                  =       |  <  \___ \| |  | | |            =
                  =       | . \ ____) | |__| | |____        =
                  =       |_|\_\_____/ \___\_\______|       =
                  =                                         =
                  =  Streaming SQL Engine for Apache Kafka® =
                  ===========================================

Copyright 2017-2018 Confluent Inc.

CLI v5.2.1, Server v5.2.1 located at http://ksql-server:8088

Having trouble? Type 'help' (case-insensitive) for a rundown of how things work!

ksql>
```

6. At the KSQL command prompt, enter the following command to list the existing topics:

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

7. View the messsages on the "radio-logs" topic:

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

Hit Ctrl+c to get back to the prompt.

8. Create a KSQL stream around the "radio-logins" topic:

```ksql
CREATE STREAM radio_logs (time BIGINT, type VARCHAR, name VARCHAR, long VARCHAR, lat VARCHAR, content ARRAY<VARCHAR>)
  WITH (KAFKA_TOPIC='radio-logs', VALUE_FORMAT='JSON', KEY='name', TIMESTAMP='time');
```

9. List the existing KSQL streams:

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

10. View the details of the "radio_logs" stream:

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

11. View the content in the "radio_logs" stream, beginning with the earliest entry. The query can take a little time to return the initial results:

```ksql
SET 'auto.offset.reset' = 'earliest';
SELECT * from radio_logs LIMIT 10;
```

The "earliest" setting tells KSQL that every query in this KSQL session should begin from the earliest offset on each topic, table, and stream.

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

12. Now, let's search for messages coming from a particular radio station:

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

13. Now, limit the query to only those with content having at least 3 items:

```ksql
select * from radio_logs WHERE type='ENG' and name='324' and content[2] IS NOT NULL limit 10;
```

After a while, you should see something like the following:

```
1557132917575 | 324 | 1557132917575 | ENG | 324 | 27 | 9 | [one, zero, zero]
1557132923187 | 324 | 1557132923187 | ENG | 324 | 27 | 9 | [one, zero, zero]
Limit Reached
```

14. We can limit it even further to a specific timestamp:

```ksql
select * from radio_logs WHERE type='ENG' and name='324' and content[2] IS NOT NULL AND time = 1557132923187 limit 10;
```

Resulting in:

```ksql
1557132923187 | 324 | 1557132923187 | ENG | 324 | 27 | 9 | [one, zero, zero]
```

15. We can count how many messages each radio station is sending each minute:

```ksql
SELECT CAST(windowStart() AS BIGINT), type, name, count(*)
    FROM radio_logs
    WINDOW TUMBLING (SIZE 1 MINUTE)
    GROUP BY type, name LIMIT 10;
```

You should see something like the following:

```
1557125640000 | MOR | 407 | 9
1557125640000 | GER | 028 | 9
1557125640000 | MOR | 437 | 9
1557125640000 | GER | 169 | 9
1557125640000 | ENG | 522 | 9
1557125640000 | ENG | 261 | 9
1557125640000 | ENG | 507 | 9
1557125640000 | GER | 178 | 9
1557125640000 | GER | 064 | 9
1557125640000 | ENG | 279 | 9
```

16. From the select, create a table from this query, so that we can then export as a csv file:

```ksql
CREATE TABLE radio_log_count
  WITH (KAFKA_TOPIC='radio_log_count', VALUE_FORMAT = 'DELIMITED')
  AS SELECT CAST(windowStart() AS BIGINT), type, name, count(*)
        FROM radio_logs
        WINDOW TUMBLING (SIZE 1 MINUTE)
        GROUP BY type, name;
```

17. Now, list the topics managed by kafka:

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

Also notice that a default value has been assigned to the partition count.

## Exporting Kafka topics using Kafka Connect

18. From the kafka-tools cli, start Kafka Connect in the background:

```sh
./bin/connect-standalone.sh /root/data/connect-standalone.properties /root/data/connect-file-sink-csv.properties &> kafka-connect-logs.txt &
```

19. A new file "radio_log_count.csv" should be created and populated with the results of our "radio_log_count" table. Run the following to view its content:

```
cat radio_log_count.csv
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
