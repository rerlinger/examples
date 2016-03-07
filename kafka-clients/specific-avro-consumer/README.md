This example reads Avro records containing click information, as produced by AvroProducerExample .
(you can find the AvroProducerExample in specific-avro-producer directory),
We then enrich these events with session information and publish both to stdout and to another Kafka topic.

Sessionization is a very common step in preparing user click information collected from website logs for analysis. 
It involves grouping user clicks into visits and marking all the clicks that belong to the same visit with unique session identifier.
These sessions are later used to analyse what users are doing on the website during a visit, and answer questions such as "how many visits end in a purchase" or "what is the last page visited before purchasing".
The way we decide if a visit from a user is a new session is by examining the time gap between his current and previous visit. 
If the gap is below 30 minutes, we decide the user is still continuing their previous visit, if it is larger than we consider this a new visit.

A "click" event contains fields for IP, timestamp, url (current page user is visiting), referrer (previous page in visit), user agent (browser) and session ID.
When we first record (or generate) a field, the session timestamp is empty. Only by reading all events that belong to the same user, can we decide when an existing sessions continues and when a new one starts.

We make the assumption that the click information is partitioned based on user-identifying field (in this example, IP). 
So all events that belong to the same user will end up in the same partition and therefore the same consumer will receive all of them.
This allows the consumer to decide on when a new session starts without having to coordinate this with other applications.

To manage session state, we need an in-memory table with last connection time for each client.
Since this is a simple example, we don't worry about persisting or purging this table.
Maybe this will arrive later :)

This example demonstrates 3 important design patterns:

* Consuming records from a topic, filtering or modifying them and producing them to a new topic
* Partitioning data based on meaningful fields to allow more efficient processing later
* Consuming and producing Avro records using the Avro serializers

To build this producer:

    $ cd specific-avro-consumer
    $ mvn clean package
    
Quickstart
-----------

Before running the examples, make sure that Zookeeper, Kafka and Schema Registry are
running. In what follows, we assume that Zookeeper, Kafka and Schema Registry are
started with the default settings.

    # Start Zookeeper
    $ bin/zookeeper-server-start config/zookeeper.properties

    # Start Kafka
    $ bin/kafka-server-start config/server.properties

    # Start Schema Registry
    $ bin/schema-registry-start config/schema-registry.properties
    
If you don't already have a schema registry, you will need to install it.
Either from packages: http://www.confluent.io/developer
Or from source: https://github.com/confluentinc/schema-registry

Then create a topic called sessions:

    # Create page_visits topic
    $ bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 \
      --partitions 1 --topic sessions
      

This example works with data generated by AvroClicksProducer, so start by running that project first and start generating clicks into "clicks" topic.
If you run AvroClicksProducer and leave it running, you can run the sessionizing consumer at the same time in another window and see "real time updates" of user sessions.
   

Run the clickstream sessionizing consumer:

    $ java -cp target/uber-clickstream-sessionizing-consumer-1.0-SNAPSHOT.jar io.confluent.examples.consumer.AvroClicksSessionizer http://localhost:8081
    
You can validate the result by using the avro console consumer (part of the schema repository):

    $ bin/kafka-avro-console-consumer --zookeeper localhost:2181 --topic sessions --from-beginning