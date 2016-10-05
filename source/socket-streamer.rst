.. toctree::
    :maxdepth: 3

Kafka Socket Streamer
=====================

Akka Http with Reactive Kafka to stream topics to clients via Web sockets and Server Send Events.

**This is test and not yet intended for any serious use yet.**

Prerequisites
-------------

* Confluent Platform 3.0.0
* Scala 2.11.7

Setup
-----

Confluent Setup
~~~~~~~~~~~~~~~

.. sourcecode:: bash

    #make confluent home folder
    mkdir confluent

    #download confluent
    wget wget http://packages.confluent.io/archive/3.0/confluent-3.0.1-2.11.tar.gz

    #extract archive to confluent folder
    tar -xvf confluent-3.0.1-2.11.tar.gz -C confluent

    #setup variables
    export CONFLUENT_HOME=~/confluent/confluent-3.0.1

Enable topic deletion.

In ``/etc/kafka/server.properties`` add the following to we can delete
topics.

.. sourcecode:: bash

    delete.topic.enable=true

Start the Confluent platform.

.. sourcecode:: bash

    #Start the confluent platform, we need kafka, zookeeper and the schema registry
    bin/zookeeper-server-start etc/kafka/zookeeper.properties &
    bin/kafka-server-start etc/kafka/server.properties &
    bin/schema-registry-start etc/schema-registry/schema-registry.properties &

QuickStart
----------

The socket streamer pushes events out from Kafka to clients via websockets or server send events. Two different endpoints
are available. But first we need some data in Kafka. Start the console producer and send some events in:

.. sourcecode:: bash

    ➜   bin/kafka-avro-console-producer \
      --broker-list localhost:9092 --topic socket_streamer \
      --property value.schema='{"type":"record","name":"User","namespace":"com.datamountaineer.streamreactor.connect.redis" \
      ,"fields":[{"name":"firstName","type":"string"},{"name":"lastName","type":"string"},{"name":"age","type":"int"}, \
      {"name":"salary","type":"double"}]}'

Paste the following in at the console producer:

.. sourcecode:: bash

    {"firstName": "John", "lastName": "Smith", "age":30, "salary": 4830}
    {"firstName": "Max", "lastName": "Power", "age":30, "salary": 1000000}


Now start the socket streamer. We need to set some configurations first. The socket-streamer uses Typesafe's configuration
loader so we can create a file called ``application.conf`` and add the following.

.. sourcecode:: bash

    system-name = "streamreactor-socket-streamer"
    port = 8080

    kafka {
      bootstrap-servers = "localhost:9092"
      zookeeper-servers = "localhost:2181"
      schema-registry-url = "http://localhost:8081"
    }

To use the ``application.conf`` file, set its location as a Java property when starting the application like this
``-Dconfig.file=path_to_file/application.conf``

To start the socket streamer:

.. sourcecode:: bash

    ➜   java -Dconfig.file=path_to_file/application.conf -jar build/libs/kafka-socket-streamer-0.1-all.jar

    2016-05-12 15:57:39,712 INFO  [main] [c.d.s.s.Main$] [delayedEndpoint$com$datamountaineer$streamreactor$socketstreamer$Main$1:32]

        ____        __        __  ___                  __        _
       / __ \____ _/ /_____ _/  |/  /___  __  ______  / /_____ _(_)___  ___  ___  _____
      / / / / __ `/ __/ __ `/ /|_/ / __ \/ / / / __ \/ __/ __ `/ / __ \/ _ \/ _ \/ ___/
     / /_/ / /_/ / /_/ /_/ / /  / / /_/ / /_/ / / / / /_/ /_/ / / / / /  __/  __/ /
    /_____/\__,_/\__/\__,_/_/  /_/\____/\__,_/_/ /_/\__/\__,_/_/_/ /_/\___/\___/_/
      _____            __        __  _____ __
     / ___/____  _____/ /_____  / /_/ ___// /_________  ____ _____ ___  ___  _____
     \__ \/ __ \/ ___/ //_/ _ \/ __/\__ \/ __/ ___/ _ \/ __ `/ __ `__ \/ _ \/ ___/
     ___/ / /_/ / /__/ ,< /  __/ /_ ___/ / /_/ /  /  __/ /_/ / / / / / /  __/ /
    /____/\____/\___/_/|_|\___/\__//____/\__/_/   \___/\__,_/_/ /_/ /_/\___/_/

    by Andrew Stevenson

    2016-05-12 15:57:39,716 INFO  [main] [c.d.s.s.Main$] [delayedEndpoint$com$datamountaineer$streamreactor$socketstreamer$Main$1:49]
    System name      : streamreactor-socket-streamer
    Kafka brokers    : localhost:9092
    Zookeepers       : localhost:2181
    Schema registry  : http://localhost:8081
    Listening on port : 8080


Now lets have the socket streamer push using server send event by simply calling curl:

.. sourcecode:: bash

    ➜  curl 'http://localhost:8080/sse/topics?topic=socket_streamer&consumergroup=testcg'

    data:{"value":"{\"firstName\": \"John\", \"lastName\": \"Smith\", \"age\": 30, \"salary\": 4830.0}"}
    data:{"value":"{\"firstName\": \"Max\", \"Power\": \"Jones\", \"age\": 30, \"salary\": 1000000}"}
    data:{"timestamp":"Thu May 12 16:42:02 CEST 2016","system":"streamreactor-socket-streamer","message":"heartbeat"}

For websockets, install a websocket client, for example `Dark WebSocket Terminal <http://tinyurl.com/nqc9s3c>`_. Start
it and connect to the websocket endpoint.

.. note:: Dark Terminal, for some reason, needs a extra whitespace at the end of the connection url to work.

.. image:: ../images/dtws.png


.. sourcecode:: bash

    command:	/connect ws://localhost:8080/ws/topics?topic=socket_streamer&consumergroup=testcgws
    system:	connection established, ws://localhost:8080/ws/topics?topic=person_redis&consumergroup=testcgws
    received:	{"value":"{\"firstName\": \"John\", \"lastName\": \"Smith\", \"age\": 30, \"salary\": 4830.0}"}


Features
--------

1. Web Sockets
2. Server Send Events
3. HeartBeat Messages

Configurations
--------------

Endpoints
---------

.. http:get:: /ws/topics?topic=<topic_name>&consumergroup=<consumergroup>

    **WebSocket example request**

    .. sourcecode:: http

        GET /ws/topics?topic=orders&consumergroup=cg1 HTTP/1.1

    Stream via websockets the orders topic with consumer group cg1.

.. http:get:: /sse/topics?topic=<topic_name>&consumergroup=<consumergroup>

    **Send Server Events example request**

    .. sourcecode:: http

        GET /sse/topics?topic=orders&consumergroup=cg1 HTTP/1.1

    Stream via Send Server Events the orders topic with consumer group cg1.

Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO
