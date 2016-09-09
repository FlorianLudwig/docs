Kafka Connect Redis
===================

A Connector and Sink to write events from Kafka to Redis. The connector takes the value from the Kafka Connect
SinkRecords and inserts a new entry to Redis.

Prerequisites
-------------

- Confluent 3.0.0
- Jedis 2.8.1
- Java 1.8
- Scala 2.11

Setup
-----

Redis Setup
~~~~~~~~~~~

Download and install Redis.

.. sourcecode:: bash

    ➜  wget http://download.redis.io/redis-stable.tar.gz
    ➜  tar xvzf redis-stable.tar.gz
    ➜  cd redis-stable
    ➜  sudo make install


Start Redis

.. sourcecode:: bash

    ➜  bin/redis-server

Check Redis is running:

.. sourcecode:: bash

    ➜  redis-cli ping
        PONG
    ➜  sudo service redis-server status

Confluent Setup
~~~~~~~~~~~~~~~

.. sourcecode:: bash

    #make confluent home folder
    ➜  mkdir confluent

    #download confluent
    ➜  wget http://packages.confluent.io/archive/3.0/confluent-3.0.0-2.11.tar.gz

    #extract archive to confluent folder
    ➜  tar -xvf confluent-3.0.0-2.11.tar.gz -C confluent

    #setup variables
    ➜  export CONFLUENT_HOME=~/confluent/confluent-3.0.0

Start the Confluent platform.

.. sourcecode:: bash

    #Start the confluent platform, we need kafka, zookeeper and the schema registry
    ➜  bin/zookeeper-server-start etc/kafka/zookeeper.properties &
    ➜  bin/kafka-server-start etc/kafka/server.properties &
    ➜  bin/schema-registry-start etc/schema-registry/schema-registry.properties &

Build the Connector and CLI
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The prebuilt jars can be taken from here and
`here <https://github.com/datamountaineer/kafka-connect-tools/releases>`__
or from `Maven <http://search.maven.org/#search%7Cga%7C1%7Ca%3A%22kafka-connect-cli%22>`__

If you want to build the connector, clone the repo and build the jar.

.. sourcecode:: bash

    ##Build the connectors
    ➜  git clone https://github.com/datamountaineer/stream-reactor
    ➜  cd stream-reactor
    ➜  gradle fatJar

    ##Build the CLI for interacting with Kafka connectors
    ➜  git clone https://github.com/datamountaineer/kafka-connect-tools
    ➜  cd kafka-connect-tools
    ➜  gradle fatJar

Sink Connector QuickStart
-------------------------

Next we will start the connector in distributed mode. Connect has two modes, standalone where the tasks run on only one host
and distributed mode. Usually you'd run in distributed mode to get fault tolerance and better performance. In distributed mode
you start Connect on multiple hosts and they join together to form a cluster. Connectors which are then submitted are
distributed across the cluster.

Before we can start the connector we need to setup it's configuration. In standalone mode this is done by creating a
properties file and passing this to the connector at startup. In distributed mode you can post in the configuration as
json to the Connectors HTTP endpoint. Each connector exposes a rest endpoint for stopping, starting and updating the
configuration.

Sink Connector Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Create a file called ``redis-sink.properties`` with the contents below:

.. sourcecode:: bash

    name=redis-sink
    connect.redis.connection.host=localhost
    connect.redis.connection.port=6379
    connector.class=com.datamountaineer.streamreactor.connect.redis.sink.RedisSinkConnector
    tasks.max=1
    topics=person_redis
    connect.redis.export.route.query=INSERT INTO TABLE1 SELECT * FROM person_redis

This configuration defines:

1.  The name of the sink.
2.  The name of the redis host to connect to.
3.  The redis port to connect to.
4.  The sink class.
5.  The max number of tasks the connector is allowed to created. Should not be greater than the number of partitions in
    the source topics otherwise tasks will be idle.
6.  The source kafka topics to take events from.
7.  The KCQL statement for topic routing and field selection.

Starting the Connector (Distributed)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Connectors can be deployed distributed mode. In this mode one or many connectors are started on the same or different
hosts with the same cluster id. The cluster id can be found in ``etc/schema-registry/connect-avro-distributed.properties.``

.. sourcecode:: bash

    # The group ID is a unique identifier for the set of workers that form a single Kafka Connect
    # cluster
    group.id=connect-cluster

Now start the connector in distributed mode. We only give it one properties file for the kafka, zookeeper and
schema registry configurations.

First add the connector jar to the CLASSPATH and then start Connect.

.. note::

    You need to add the connector to your classpath or you can create a folder in ``share/java`` of the Confluent
    install location like, kafka-connect-myconnector and the start scripts provided by Confluent will pick it up.
    The start script looks for folders beginning with kafka-connect.

.. sourcecode:: bash

    #Add the Connector to the class path
    ➜  export CLASSPATH=kafka-connect-redis-0.2-cp-3.0.0.all.jar

.. sourcecode:: bash

    ➜  confluent-3.0.0/bin/connect-distributed confluent-3.0.0/etc/schema-registry/connect-avro-distributed.properties

Once the connector has started lets use the kafka-connect-tools cli to post in our distributed properties file.

.. sourcecode:: bash

    ➜  java -jar build/libs/kafka-connect-cli-0.5-all.jar create redis-sink < redis-sink.properties
    #Connector name=`redis-sink`
    connect.redis.connection.host=localhost
    connect.redis.connection.port=6379
    connector.class=com.datamountaineer.streamreactor.connect.redis.sink.RedisSinkConnector
    tasks.max=1
    topics=person_redis
    connect.redis.export.route.query=INSERT INTO TABLE1 SELECT * FROM person_redis
    #task ids: 0

If you switch back to the terminal you started the Connector in you should see the Redis sink being accepted and the
task starting.

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    #check for running connectors with the CLI
    ➜ java -jar build/libs/kafka-connect-cli-0.5-all.jar ps
    redis-sink

.. sourcecode:: bash

    [2016-05-08 22:37:05,616] INFO
        ____        __        __  ___                  __        _
       / __ \____ _/ /_____ _/  |/  /___  __  ______  / /_____ _(_)___  ___  ___  _____
      / / / / __ `/ __/ __ `/ /|_/ / __ \/ / / / __ \/ __/ __ `/ / __ \/ _ \/ _ \/ ___/
     / /_/ / /_/ / /_/ /_/ / /  / / /_/ / /_/ / / / / /_/ /_/ / / / / /  __/  __/ /
    /_____/\__,_/\__/\__,_/_/  /_/\____/\__,_/_/ /_/\__/\__,_/_/_/ /_/\___/\___/_/
        ____           ___      _____ _       __
       / __ \___  ____/ (_)____/ ___/(_)___  / /__
      / /_/ / _ \/ __  / / ___/\__ \/ / __ \/ //_/
     / _, _/  __/ /_/ / (__  )___/ / / / / / ,<
    /_/ |_|\___/\__,_/_/____//____/_/_/ /_/_/|_|


     (com.datamountaineer.streamreactor.connect.redis.sink.config.RedisSinkConfig:165)
    [2016-05-08 22:37:05,641] INFO Settings:
    RedisSinkSettings(RedisConnectionInfo(localhost,6379,None),RedisKey(FIELDS,WrappedArray(firstName, lastName)),PayloadFields(false,Map(firstName -> firstName, lastName -> lastName, age -> age, salary -> income)))
           (com.datamountaineer.streamreactor.connect.redis.sink.RedisSinkTask:65)
    [2016-05-08 22:37:05,687] INFO Sink task org.apache.kafka.connect.runtime.WorkerSinkTask@44b24eaa finished initialization and start (org.apache.kafka.connect.runtime.WorkerSinkTask:155)


Test Records
^^^^^^^^^^^^

Now we need to put some records it to the test_table topics. We can use the ``kafka-avro-console-producer`` to do this.

Start the producer and pass in a schema to register in the Schema Registry. The schema has a ``firstname`` field of type
string a ``lastname`` field of type string, an ``age`` field of type int and a ``salary`` field of type double.

.. sourcecode:: bash

    bin/kafka-avro-console-producer \
      --broker-list localhost:9092 --topic person_redis \
      --property value.schema='{"type":"record","name":"User","namespace":"com.datamountaineer.streamreactor.connect.redis"
      ,"fields":[{"name":"firstName","type":"string"},{"name":"lastName","type":"string"},{"name":"age","type":"int"},{"name":"salary","type":"double"}]}'

Now the producer is waiting for input. Paste in the following:

.. sourcecode:: bash

    {"firstName": "John", "lastName": "Smith", "age":30, "salary": 4830}

Check for records in Redis
~~~~~~~~~~~~~~~~~~~~~~~~~~

Now check the logs of the connector you should see this:

.. sourcecode:: bash

    INFO Received record from topic:person_redis partition:0 and offset:0 (com.datamountaineer.streamreactor.connect.redis.sink.writer.RedisDbWriter:48)
    INFO Empty list of records received. (com.datamountaineer.streamreactor.connect.redis.sink.RedisSinkTask:75)

Check the Redis.

.. sourcecode:: bash

    redis-cli

    127.0.0.1:6379> keys *
    1) "John.Smith"
    2) "11"
    3) "10"
    127.0.0.1:6379>
    127.0.0.1:6379> get "John.Smith"
    "{\"firstName\":\"John\",\"lastName\":\"Smith\",\"age\":30,\"income\":4830.0}"


Now stop the connector.

Features
--------

The Redis sink writes records from Kafka to Redis.

The sink supports:

1. Field selection - Kafka topic payload field selection is supported, allowing you to select fields written to Redis.
2. Topic to table routing.
3. RowKey selection - Selection of fields to use as the row key, if none specified the topic name, partition and offset are
   used.
4. Error policies for handling failures.

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L** anguage found here `GitHub repo <https://github.com/datamountaineer/kafka-connector-query-language>`_
allows for routing and mapping using a SQL like syntax, consolidating typically features in to one configuration option.

The Redis sink supports the following:

.. sourcecode:: bash

    INSERT INTO <table> SELECT <fields> FROM <source topic> <PK> primary_key_cols

Example:

.. sourcecode:: sql

    #Insert mode, select all fields from topicA and write to tableA and use the default rowkey (topic name, partition, offset)
    INSERT INTO tableA SELECT * FROM topicA

    #Insert mode, select 3 fields and rename from topicB and write to tableB, use field y from the topic as the primary key
    INSERT INTO tableB SELECT x AS a, y AS b and z AS c FROM topicB PK y

This is set in the ``connect.redis.export.route.query`` option.

Error Polices
~~~~~~~~~~~~~

The sink has three error policies that determine how failed writes to the target database are handled. The error policies
affect the behaviour of the schema evolution characteristics of the sink. See the schema evolution section for more
information.

**Throw**

Any error on write to the target database will be propagated up and processing is stopped. This is the default
behaviour.

**Noop**

Any error on write to the target database is ignored and processing continues.

.. warning::

    This can lead to missed errors if you don't have adequate monitoring. Data is not lost as it's still in Kafka
    subject to Kafka's retention policy. The sink currently does **not** distinguish between integrity constraint
    violations and or other expections thrown by drivers.

**Retry**

Any error on write to the target database causes the RetryIterable exception to be thrown. This causes the
Kafka connect framework to pause and replay the message. Offsets are not committed. For example, if the table is offline
it will cause a write failure, the message can be replayed. With the Retry policy the issue can be fixed without stopping
the sink.

The length of time the sink will retry can be controlled by using the ``connect.redis.sink.max.retries`` and the
``connect.redis.sink.retry.interval``.

Configurations
--------------

``connect.redis.export.route.query``

Kafka connect query language expression. Allows for expressive topic to table routing, field selection and renaming. Fields
to be used as the row key can be set by specifing the ``PK``. The below example uses field1 as the primary key.

* Data type : string
* Importance: high
* Optional  : no

Examples:

.. sourcecode:: sql

    INSERT INTO TABLE1 SELECT * FROM TOPIC1;INSERT INTO TABLE2 SELECT * FROM TOPIC2 PK field1

Examples:

.. sourcecode:: sql

    INSERT INTO TABLE1 SELECT * FROM TOPIC1;INSERT INTO TABLE2 SELECT * FROM TOPIC2 PK field1, field2

``connect.redis.sink.connection.host``

Specifies the Redis server.

* Data type : string
* Importance: high
* Optional  : no

``connect.redis.sink.connection.port``

Specifies the Redis server port number.

* Data type : int
* Importance: high
* Optional  : no

``connect.redis.sink.connection.password``

Specifies the authorization password.

* Data type : string
* Importance: high
* Optional  : yes

``connect.redis.sink.error.policy``

Specifies the action to be taken if an error occurs while inserting the data.

There are three available options, **noop**, the error is swallowed, **throw**, the error is allowed to propagate and retry.
For **retry** the Kafka message is redelivered up to a maximum number of times specified by the ``connect.redis.sink.max.retries``
option. The ``connect.redis.sink.retry.interval`` option specifies the interval between retries.

The errors will be logged automatically.

* Type: string
* Importance: medium
* Optional: yes
* Default: RETRY


``connect.redis.sink.max.retries``

The maximum number of times a message is retried. Only valid when the ``connect.redis.sink.error.policy`` is set to ``retry``.

* Type: string
* Importance: medium
* Optional: yes
* Default: 10


``connect.redis.sink.retry.interval``

The interval, in milliseconds between retries if the sink is using ``connect.redis.sink.error.policy`` set to **RETRY**.

* Type: int
* Importance: high
* Optional: no
* Default : 60000 (1 minute)

Example
~~~~~~~

.. sourcecode:: bash

    name=redis-sink
    connect.redis.connection.host=localhost
    connect.redis.connection.port=6379
    connector.class=com.datamountaineer.streamreactor.connect.redis.sink.RedisSinkConnector
    tasks.max=1
    topics=person_redis
    connect.redis.export.route.query=INSERT INTO TABLE1 SELECT * FROM person_redis

Schema Evolution
----------------

Upstream changes to schemas are handled by Schema registry which will validate the addition and removal
or fields, data type changes and if defaults are set. The Schema Registry enforces Avro schema evolution rules.
More information can be found `here <http://docs.confluent.io/3.0.1/schema-registry/docs/api.html#compatibility>`_.

The Redis sink will automatically write and update the Redis table if new fields are added to the source topic,
if fields are removed the Kafka Connect framework will return the default value for this field, dependent of the
compatibility settings of the Schema registry. This value will be put into the Redis column family cell based on the
``connect.redis.export.route.query`` mappings.

Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO
