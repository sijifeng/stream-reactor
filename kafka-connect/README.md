### Kafka-Connect-Cassandra

A Connector and Sink to write events from Kafka to Cassandra. The connector converts the value from the Kafka Connect SinkRecords to Json and uses Cassandra's JSON insert functionality to insert the rows.

PreparedStatements are created and cached at startup for the topics assigned to the task.

The task expects a table in Cassandra per topic that the sink is configured to listen to. You could have different schemas in the same topic, Cassandra's JSON functioanlity will ignore missing columns.

The table and keyspace must be created before hand! The sink tries to write to a table matching the name of the topic. The prepared statements will blow if the corressponding table does not exist.

#### Prequisties

* Cassandra 2.2.4 - Json support was added in 2.2 but a bug wasn't fixed till 2.2.4 for rebinding prepared statements with JSON
* Cassandra-unit 2.2.2.1 - For Unit testing, currently built against 2.2 so I've included a version built againts 2.2.4 in the lib folder
  Install locally with `mvn install:install-file -Dfile=src/lib/cassandra-unit-2.2.2.2-SNAPSHOT.jar`
* Kafka 0.9.0

#### Properties

In addition to the default topics configuration the following options are added:

name | data type | required | description
-----|-----------|----------|------------
contact_points | string | yes | contact points (hosts) in Cassandra cluster
key_space | string | yes | key_space the tables to write to belong to
port | int | no | port for the native Java driver (default 9042)

Example connector.properties file

```bash 
name=cassandra-sink
connector.class=com.datamountaineer.streamreactor.connect.CassandraSinkConnector
tasks.max=1
topics=test_table
contact_points=localhost
port=9042
key_space=connect_test
```

You must also supply the `connector.class` as `com.datamountaineer.streamreactor.connect.CassandraSinkConnector`

#### Setup

* Clone and build the Connector and Sink

    ```bash
    git clone git@github.com:andrewstevenson/stream-reactor.git
    cd stream-reactor
    mvn package
    ```

* [Download and install Cassandra](http://cassandra.apache.org/)
* [Download and install Confluent](http://www.confluent.io/)
* Copy the Cassandra sink jar from your build location to `$CONFLUENT_HOME/share/java/kafka-connect-cassandra`

    ```bash
    mkdir $CONFLUENT_HOME/share/java/kafka-connect-cassandra
    cp target/kafka-connect-0.1-jar-with-dependencies.jar $CONFLUENT_HOME/share/java/kafka-connect-cassandra/
    ```
    
* Start Cassandra

    ```bash
   nohup $CASSANDRA_HOME/bin/cassandra &
    ```
    
* Start Confluents Zookeeper, Kafka and Schema registry

    ```bash
    nohup $CONFLUENT_HOME/bin/zookeeper-server-start $CONFLUENT_HOME/etc/kafka/zookeeper.properties &
    nohup $CONFLUENT_HOME/bin/kafka-server-start $CONFLUENT_HOME/etc/kafka/server.properties &
    nohup $CONFLUENT_HOME/bin/schema-registry-start $CONFLUENT_HOME/etc/schema-registry/schema-registry.properties &"
    ```
    
* Create keyspace in Cassandra

    ```sql
    $CASSANDRA_HOME/bin/cqlsh
    ```
    
    ```sql
    CREATE KEYSPACE connect_test WITH replication = {'class': 'SimpleStrategy', 'replication_factor': '3'}  
    AND durable_writes = true;
    
    CREATE TABLE connect_test.test_table (
    id int PRIMARY KEY,
    random_field text
    ); 
    ```
    
* Start Kafka Connect with the Cassandra sink

    ```bash
    $CONFLUENT_HOME/bin/connect-standalone etc/schema-registry/connect-avro-standalone.properties etc/kafka-connect-cassandra/cassandra.properties
    ```
    
* Test with avro console, start the console to create the topic and write values

    ```bash
    $CONFLUENT_HOME/bin/kafka-avro-console-producer \
      --broker-list localhost:9092 --topic test_table \
      --property value.schema='{"type":"record","name":"myrecord","fields":[{"name":"id","type":"int"}, {"name":"random_field", "type": "string"}]}'
    ```
    
    ```bash
    #insert at prompt
    {"id": 999, "random_field": "foo"}
    {"id": 888, "random_field": "bar"}
    ````
    
* Check in Cassandra for the records

    ```sql
    SELECT * FROM connect_test.test_table"
    ```

#### Improvements
* Add key of message to payload
* Auto create tables in Cassandra if they don't exists. Need a converter from Connect data types to Cassandra