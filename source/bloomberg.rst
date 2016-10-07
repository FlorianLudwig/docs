Kafka Connect Bloomberg
=======================

**DOCS ARE WORKING IN PROGRESS**

Kafka Connect Bloomberg is a source connector to subscribe to Bloomberg feeds via the Bloomberg labs open API and write to Kafka.

Prerequisites
-------------

-  Bloomberg subscription
-  Confluent 2.0
-  Java 1.8
-  Scala 2.11

Setup
-----

Confluent Setup
~~~~~~~~~~~~~~~

.. sourcecode:: bash

    #make confluent home folder
    ➜  mkdir confluent

    #download confluent
    ➜  wget http://packages.confluent.io/archive/3.0/confluent-3.0.1-2.11.tar.gz

    #extract archive to confluent folder
    ➜  tar -xvf confluent-3.0.1-2.11.tar.gz -C confluent

    #setup variables
    ➜  export CONFLUENT_HOME=~/confluent/confluent-3.0.1

Start the Confluent platform.

.. sourcecode:: bash

    #Start the confluent platform, we need kafka, zookeeper and the schema registry
    bin/zookeeper-server-start etc/kafka/zookeeper.properties &
    bin/kafka-server-start etc/kafka/server.properties &
    bin/schema-registry-start etc/schema-registry/schema-registry.properties &

Build the Connector and CLI
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The prebuilt jars can be taken from `here <https://github.com/datamountaineer/stream-reactor/releases>`__ and
`here <https://github.com/datamountaineer/kafka-connect-tools/releases>`__
or from `Maven <http://search.maven.org/#search%7Cga%7C1%7Ca%3A%22kafka-connect-cli%22>`__

If you want to build the connector, clone the repo and build the jar.

.. sourcecode:: bash

    ##Build the connectors
    git clone https://github.com/datamountaineer/stream-reactor
    cd stream-reactor
    gradle fatJar

    ##Build the CLI for interacting with Kafka connectors
    git clone https://github.com/datamountaineer/kafka-connect-tools
    cd kafka-connect-tools
    gradle fatJar

Source Connector
----------------

Source Connector QuickStart
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Test data
^^^^^^^^^

Source Connector Configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Next we start the connector in standalone mode. This useful for testing
and one of jobs, usually you'd run in distributed mode to get fault
tolerance and better performance.

Before we can start the connector we need to setup it's configuration.
In standalone mode this is done by creating a properties file and
passing this to the connector at startup. In distributed mode you can
post in the configuration as json to the Connectors HTTP endpoint. Each
connector exposes a rest endpoint for stopping, starting and updating the
configuration.

Since we are in standalone mode we'll create a file called
bloomberg-source.properties with the contents below:

.. sourcecode:: bash

    name=bloomberg-source
    connector.class=com.datamountaineer.streamreactor.connect.bloomberg.BloombergSourceConnector
    tasks.max=1
    connect.bloomberg.server.host=localhost
    connect.bloomberg.server.port=8194
    connect.bloomberg.service.uri=//blp/mkdata
    connect.bloomberg.subscriptions=AAPL US Equity:LAST_PRICE,BID,ASK;IBM US Equity:BID,ASK,HIGH,LOW,OPEN
    kafka.topic=bloomberg
    connect.bloomberg.buffer.size=4096

This configuration defines:

1. The connector name.
2. The class containing the connector.
3. The number of tasks the connector is allowed to start.
4. The Bloomberg server host.
5. The Bloomberg server port.
6. The Bloomberg service uri.
7. The subscription keys to subscribe to.
8. The topic to write to.
9. The buffer size for the Bloomberg API to buffer events in.

Starting the Source Connector (Standalone)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Now we are ready to start the Bloomberg Source Connector in standalone mode.

.. note::

    You need to add the connector to your classpath or you can create a folder in share/java like
    kafka-connect-myconnector and the start scripts provided by Confluent will pick it up. The start script looks for
    folders beginning with kafka-connect.

.. sourcecode:: bash

    #Add the Connector to the class path
    ➜  export CLASSPATH=kafka-connect-bloomberg-0.1-all.jar
    #Start the connector in standalone mode, passing in two properties files, the first for the schema registry, kafka
    #and zookeeper and the second with the connector properties.
    ➜  bin/connect-standalone etc/schema-registry/connect-avro-standalone.properties bloomberg-source.properties

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    ➜ java -jar build/libs/kafka-connect-cli-0.6-all.jar get bloomberg-source


Check for Source Records in Kafka
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Now check the logs of the connector you should see this

... sourcecode:: bash


We can then use the kafka-avro-console-consumer to see what's in the kafka topic we have routed the subscription to.

... sourcecode:: bash

Now stop the connector.

Starting the Connector (Distributed)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Connectors can be deployed distributed mode. In this mode one or many
connectors are started on the same or different hosts with the same cluster id.
The cluster id can be found in ``etc/schema-registry/connect-avro-distributed.properties.``

.. sourcecode:: bash

    # The group ID is a unique identifier for the set of workers that form a single Kafka Connect
    # cluster
    group.id=connect-cluster

For this quick-start we will just use one host.

Now start the connector in distributed mode, this time we only give it
one properties file for the kafka, zookeeper and schema registry
configurations.

.. sourcecode:: bash

    ➜  confluent-3.0.1/bin/connect-distributed confluent-3.0.1/etc/schema-registry/connect-avro-distributed.properties

Once the connector has started lets use the kafka-connect-tools cli to
post in our distributed properties file.

.. sourcecode:: bash

    ➜  java -jar build/libs/kafka-connect-cli-0.6-all.jar create bloomberg-source < bloomberg-source.properties

If you switch back to the terminal you started the Connector in you
should see the Bloomberg Source being accepted and the task starting.

Check the logs.

Check Kafka.



Features
--------

Source Connector
~~~~~~~~~~~~~~~~

Data Types
^^^^^^^^^^


Configurations
--------------

``connect.bloomberg.server.host``

The bloomberg endpoint to connect to.

* Data type : string
* Optional  : no

``connect.bloomberg.server.port``

The Bloomberg endpoint to connect to.

* Data type : string
* Optional  : no

``connect.bloomberg.service.uri``

Which Bloomberg service to connect to. Can be //blp/mkdata or //blp/refdata.

* Data type : string
* Optional  : no

``connect.bloomberg.authentication.mode``

The mode to authentication against the Bloomberg server. Either APPLICATION_ONLY or USER_AND_APPLICATION.

* Data type : string
* Optional  : no


``connect.bloomberg.subscriptions``

* Data type : string
* Optional  : no

Specifies which ticker subscription to make. The format is TICKER:FIELD,FIELD,..;
e.g.AAPL US Equity:LAST_PRICE;IBM US Equity:BID

``connect.bloomberg.buffer.size``

* Data type : int
* Optional  : yes
* Default   : 2048

The buffer accumulating the data updates received from Bloomberg. If not provided it will default to 2048. If the
buffer is full and a new update will be received it won't be added to the buffer until it is first drained.

``connect.bloomberg.kafka.topic``

The topic to write to.

* Data type : string
* Optional  : no


Example
~~~~~~~

.. sourcecode:: bash

    name=bloomberg-source
    connector.class=com.datamountaineer.streamreactor.connect.bloomberg.BloombergSourceConnector
    tasks.max=1
    connect.bloomberg.server.host=localhost
    connect.bloomberg.server.port=8194
    connect.bloomberg.service.uri=//blp/mkdata
    connect.bloomberg.subscriptions=AAPL US Equity:LAST_PRICE,BID,ASK;IBM US Equity:BID,ASK,HIGH,LOW,OPEN
    kafka.topic=bloomberg
    connect.bloomberg.buffer.size=4096

Schema Evolution
----------------

TODO

Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO
