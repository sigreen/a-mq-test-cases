A-MQ Test Cases
=============================

This project contains various test case scenarios for ActiveMQ

Notes:

* all instructions assume you are executing from the top level directory of this project
* it is assumed you have Apache Maven installed, and that you are familiar with its usage
* assumes you are using Apache ActiveMQ 5.11.0 or later, or JBoss A-MQ 6.2.0 or later

Note: All of this code will run correctly against either Apache ActiveMQ 5.9.0 or JBoss A-MQ 6.2.0 as both
internally use the same ActiveMQ 5.9.0 code base. The instructions on how to start (command line) the brokers from an
ActiveMQ install will **not** work with JBoss A-MQ (e.g. `bin/amq`) as JBoss A-MQ has ActiveMQ
deployed within an Apache Karaf container to allow for runtime updates to configuration information (versus needing to
restart the broker in the case on Apache ActiveMQ). There are additional steps required to use these provided broker
configuration files and deploy them correctly to JBoss A-MQ with its Fabric based configuration system.
JBoss A-MQ's install contains an `extras` directory with a support version of the Apache ActiveMQ 5.9.0 binary
install.

### Prerequisites ###

Please run the following command  to build the project before running these examples:

    shell1> mvn clean install


Test Case One: Composite Topics Forwarding to Multiple Durable Queues
---------------------------------------------------------------------

### Referenced A-MQ Config Files ###

The following config files are referenced in this example:

- conf/testcase1-compositeTopicForwarding.xml

### Run ###
    
(1) Start the ActiveMQ broker from Maven:

    shell1> mvn -P broker-testCaseOne

(2) Start the message consumer in another shell:

    shell2> mvn -P consumer

(3) Start the second message consumer in another shell:

    shell3> mvn -P consumer2

(4) Start the topic publisher in another shell:

    shell3> mvn -P publisher

The message publisher is coded to send 100 messages to the topic. The broker will forward these messages to the simple and simple2 durable queues. Each consumer will log the messages it receives, and will timeout and exit after 120 seconds of inactivity (no messages received).

Test Case Two: Network of Brokers (Topic to Queue bridging)
-----------------------------------------------------------

The Network of Brokers configuration allows you to scale out the processing of messages across multiple brokers. In
this configuration, messages are published to a topic then forwarded to a second broker. The second broker bridges the topic to a queue where a Consumer reads the messages.

### Referenced A-MQ Config Files ###

The following config files are referenced in this example:

- conf/testcase2-nwob1.xml
- conf/testcase2-nwob2.xml

### Run ###

To run this sample,

(1) Start ActiveMQ from Maven:

    shell1> mvn -P broker-nwob1
    shell2> mvn -P broker-nwob2

(2) Start the message consumer in another shell:

    shell3> mvn -P consumer-nwob

(3) Start the message publisher in another shell:

    shell4> mvn -P publisher-nwob

You can test the brokers forwarding messages by randomly killing and restarting broker instances. You'll see messages
stop when the broker is stopped, and you'll see any sent messages forwarded when the brokers are restarted and the
network connection is re-established. This shows how ActiveMQ is saving persistent messages in its store for delivery
when the broker re-starts. You can especially see this if you kill the broker the consumer is connected to
(testcase2-nwob1.xml) while the producer continues sending messages to the other broker.

Test Case Two: Fanout Topology
-----------------------------------------------------------

The Fanout configuration demonstrates a client publishing to a topic on a local broker.  The local broker forwards the message from a topic to a queue, then broadcasts it to any connected broker in the federation. Each consumer reads the message from their assigned broker queue.

In addition, the configuration demonstrates the forwarding of messages over 2 separate independent topics over a network configuration bridge.

### Referenced A-MQ Config Files ###

The following config files are referenced in this example:

- conf/testcase2-fanout1.xml
- conf/testcase2-fanout2.xml
- conf/testcase2-fanout3.xml

### Run ###

To run this sample,

(1) Start ActiveMQ from Maven:

    shell1> mvn -P broker-fanout1
    shell2> mvn -P broker-fanout2
    shell3> mvn -P broker-fanout3

(2) Start the message consumers in another shell:

    shell4> mvn -P consumer-fanout1
    shell5> mvn -P consumer-fanout2


(3) Start the message producer in another shell:

    shell6> mvn -P publisher


Test Case Three: Master / Slave Failover (with Topics)
------------------------------------------------------

The Master-Slave configuration enables quick activation of a broker instance to continue processing of messages stored
within a message persistence store. It utilizes a shared file system, in this case the same local directory, with both
brokers referencing this same directory. The first broker instance to acquire a file lock is considered the 'master',
and all subsequent broker instances waiting for the file lock are considered 'slave'.

### Referenced A-MQ Config Files ###

The following config files are referenced in this example:

- conf/testcase3-failover1.xml
- conf/testcase3-failover2.xml

### Run ###

To run this sample,

(1) Start ActiveMQ from Maven:

    shell1> mvn -P broker-failover1
    shell2> mvn -P broker-failover2

(2) Start the message consumer in another shell:

    shell3> mvn -P publisher

(3) Start the message producer in another shell:

    shell4> mvn -P subscriber

The message publisher is coded to send 100 messages. The subscriber will log the messages it receives, and will timeout
and exit after 120 seconds of inactivity (no messages received). To test failover, kill (Ctrl+C) the current master
broker while the Producer is sending messages. You will see a log entry about the clients reconnecting. You will
eventually see log entries in the slave broker about it starting up; it may take up to 30 seconds for the slave to
detect that the master has failed and startup.

To test fail back, restart the failed broker, see the log entries about it not acquiring the file lock, kill (Ctrl+C)
the currently active broker, and see the client automatically fail back. The automatic reconnect be the clients assumes,
as is the case with this example, that the clients are using the fail over transport.

Test Case Four: Virtual Topics
------------------------------------------------------

The virtual topic configuration enables forwarding of messages from broker A to broker B while offering message persistence
in the event broker B goes down. It utilizes a shared file system, with each broker referencing its own directory. 

### Referenced A-MQ Config Files ###

The following config files are referenced in this example:

- conf/testcase4-virtualDestinations1.xml
- conf/testcase4-virtualDestinations2.xml

### Run ###

To run this sample,

(1) Start ActiveMQ from Maven:

    shell1> mvn -P broker-vt1
    shell2> mvn -P broker-vt2
    
(2) Start the message consumer (using queues) in another shell:

    shell3> mvn -P consumer-vt1

(3) Start the message producer in another shell:

    shell4> mvn -P publisher-vt1
    
** Altenative ** (4) Instead of step (2), start the message subscriber in another shell to consume from the original topic name:

    shell5> mvn -P subscriber-vt1

The message publisher is coded to send 100 messages. The consumer will log the messages it receives, and will timeout
and exit after 120 seconds of inactivity (no messages received).  In this case, the consumer reads messages from a generated
queue - not topic.  It references Consumer.<clientID>.<topic> allowing broker B to route on the original topic name if required.
Alternatively, step (4) allows a subscriber to read from the original topic name but unfortunately does not offer the same
durable subscription functionality as the consumer does when reading from a queue. To test message durability, kill (Ctrl+C) 
broker-failover2 while the Publisher is sending messages. You will see a log entry about the clients reconnecting. You will
eventually see log entries in the broker B about it starting up.

To test fail back, restart the failed broker.  Any messages that were sent while broker B was down should be forwarded to the consumer.
In the case of the subscriber scenario, all messages sent during the time the failed broker was down will be lost.