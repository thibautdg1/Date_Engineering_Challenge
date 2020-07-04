# Hiring_Challenge @ Doodle 

## Challenge's context 

The asked challenge needs to work with Apache Kafka. Kafka is an open-source and real-time stream-processing software platform written in Scala and Java. This technology works with APIs to stream data between a producer and a consumer. The basic architecture with this tool is the following: 

![](https://github.com/thibautdg1/Hiring_Challenge/blob/master/apache-kafka-apis.png) 

A Kafka cluster is basically built with some brokers and a Zookeeper server. Each broker in a Kafka cluster is a single server that can receive messages from producers, assigns offsets to them, and commits the messages to storage on disk. It also services consumers, responding to fetch requests for partitions and responding with the messages that have been committed to disk. 

Secondly, Apache Kafka uses Zookeeper to store metadata about the Kafka cluster, as well as consumer client details.  

These details about the Kafka cluster are summarized in the diagram below: 

![](https://github.com/thibautdg1/Hiring_Challenge/blob/master/Kafka-Architecture.png) 

Within a cluster of brokers, one broker will also function as the cluster controller (elected automatically from the live members of the cluster). The controller is responsible for administrative operations, including assigning partitions to brokers and monitoring for broker failures. A partition is owned by a single broker in the cluster, and that broker is called the leader of the partition. A partition may be assigned to multiple brokers, which will result in the partition being replicated to provide a reduncy of messages within the partition. 

The following figure shows a replication of partitions in a cluster: 

![](https://github.com/thibautdg1/Hiring_Challenge/blob/master/Replication%20of%20partitions.png) 

To stream data with Kafka, we need to add to this cluster bloc a streams bloc. Kafka Stream processes event in real time (no micro-batching like Spark Streaming i.e.) and allows you handle the late arrival of data seamlessly by using different logical components: 

1. Stream: Unbounded, continuously updated dataset, composed of stream partition 
2. Stream partition: a part of the stream of data. 
3. Processor topology: a graph of stream and stream processor which describes how the streams are to be processed. For those familiar with Spark, it resembles a logical plan. 
4. Stream processor: a step to be performed on a stream (map, filter, etc…). There is two special stream processor in a topology, the source processor and the sink processor (read/write data). Two APIs are available, the Kafka Stream DSL or the low-level Processor API. 
5. Stream task: an instance of the processor topology attached to a partition of the input topic and stream processor’s steps on that subset only. 
6. Stream thread: can run one or more stream task, is used to scale out the application, if the number of stream threads is greater than the number of partition, some instances will stay idle. If a streaming task fails, an idle instance will take its place. 
7. Record: A key-value pair, streams are composed of records 
8. Tables: used to maintain a state inside a Kafka Stream application 
(Source: confluent) 

The following figure describes this Streams architecture: 

![](https://github.com/thibautdg1/Hiring_Challenge/blob/master/streams-architecture.jpg) 

### Goal 

The goal of the challenge is to have a tool that is able to stream data from kafka and count unique things within this data. The simplest use case is that we want to calculate unique users per minute, day, week, month, year. For a very first version, business wants us to provide just the unique users per minute. 
- The data consists of (Log)-Frames of JSON data that are streamed into/from apache kafka. 
- Each frame has a timestamp property which is unixtime, the name of the property is `ts`. 
- Each frame has a user id property calls `uid`. 
- we can assume that 99.9% of the frames arrive with a maximum latency of 5 seconds. 
- we want to display the results as soon as possible. 
- the results should be forwarded to a new kafka topic (again as json.) choose a suitable structure. 
- for an advanced solution you should assume that you can not guarantee that events are always strictly ordered. 

### Data 

To build the solution, I could use either a sample data generated by a data generator or generate my own data by using a data-generator tool to introduce random-ness and to also test the built application's robustness.  

### Suggested steps to get this goal 

#### basic solution 

1. install kafka 
2. create a topic 
3. use the kafka producer from kafka itself to send our test data to your topic 
4. create a small app that reads this data from kafka and prints it to stdout 
5. find a suitable data structure for counting and implement a simple counting mechanism, output the results to stdout 

#### advanced solution 
6. benchmark 
7. Output to a new Kafka Topic instead of stdout 
8. try to measure performance and optimize 
9. write about how you could scale 
10. only now think about the edge cases, options and other things 

## Install Kafka 

As it was my first use of Kafka, I did different steps to familiarize myself with this tool. To achieve it, I used a Terminal on a Linux VM: 
1. Download and untar Kafka 
2. Start the Zookeeper and the Kafka servers. 
3. Create a topic for trying to send message between producer and consumer. I could process 13 messages between both consoles. 
4. Delete this topic. 

## Create a topic for the challenge 

To take the benefits from the replication within the cluster, I chose to set up a multi-broker cluster. 

Once Kafka has been fully installed on my terminal, I created a topic calls `test` with the command `bin/kafka-topics.sh --create --bootstrap-server localhost: 9092 --replication-factor 3 --partitions 1 --topic test`   

## Send the test data to the topic 

First of all, I needed to launch the producer console of my topic `test` using the command `./bin/kafka-console-producer.sh --broker-list localhost: 9092 -topic test`. And to see if I can execute in the binary now, I launched the consumer console of my topic with the command `./bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning`. 

From my producer console and after dowloading the sample data, I could import it into my kafka cluster using the command `zcat stream.gz | ./bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test`.  

## Create a small app to read the data and to count before to print it to stdout 

Reading data means to configure a Kafka consumer with a group ID (to gain in scalability) to receive and process records, and the logging setup. Basically, we can imagine a code in Java to read data from the file of data imported into our cluster. On a Java IDE, we can have a code like that: 

`package com.cloudurable.kafka; 

import org.apache.kafka.clients.consumer.*; 

import org.apache.kafka.clients.consumer.Consumer; 

import org.apache.kafka.common.serialization.LongDeserializer; 

import org.apache.kafka.common.serialization.StringDeserializer; 

import java.util.Collections; 

import java.util.Properties; 

 
public class KafkaConsumerTest { 
      private final static String TOPIC = "test"; 
      private final static String BOOTSTRAP_SERVER = "localhost:9092"; 

      private static Consumer<Long, String> createConsumer() { 
      '''Creation of consumer to process records'''
          final Properties props = new Properties(); 
          props.put(ConsumerConfig.BOOTSTRAP_SERVER_CONFIG, BOOTSTRAP_SERVER); 
          props.put(ConsumerConfig.GROUP_ID_CONFIG, "KafkaTestConsumer"); 
          props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, 
          LongDeserializer.class.getName()); 
          props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, 
          StringDeserializer.class.getName()); 

          final Consumer<Long, String> consumer =   # Create the consumer using props. 
                new KafkaConsumer<>(props); 

          consumer.subscribe(Collections.singletonList(TOPIC));    #Subscribe to the topic. 
                return consumer; 
                
          } 


          static void runConsumer() throws InterruptedException { 
          ''' Process Records from Consumer '''
          final Consumer<Long, String> consumer = createConsumer(); 
          final int maxRecordsCount = 1000 int noRecordsCount = 0; 

          while (true) { 
            final ConsumerRecords<Long, String> consumerRecords = consumer.poll(1000); 
            if (consumerRecords.count()==0) { 
            noRecordsCount++;
            if (noRecordsCount > giveUp) break; 
            else continue; 
            }           

        consumerRecords.forEach(record -> { 
            System.stdout.printf("Consumer Record:(%d, %s, %d, %d)\n", 
                record.key(), record.value(), 
                record.partition(), record.offset()); 
            }); 

        consumer.commitAsync(); 
            } 

        consumer.close(); 
        System.stdout.println("DONE"); 

        } 

}` 


To run the consumer: 

`public class KafkaConsumerTest { 

        public static void main(String... args) throws Exception { 
                runConsumer(); 
        } 

}` 

The `main` method is basically calls `runConsumer` here. 

## To store data 

Then, we will use KSQL to aggregate the data ingested by our cluster and transformed by the Stream Processor. The JSON format is the best format for our sample because JSON is a structured format and it is easy to query and to access. 

According to the challenge description, each frame of our sample data file have got a timestamp property and an user id property. We also need to create a table with both information for each raw.  

The main advantages of this storage are: flexible, scalability and easy to manipulate and to extract for analytics.  

If we want to get in scalability within our Kafka environment, it could be interesting to multiple nodes in the cluster and the consumer groups or to use big data tools in the consumer part (instead of our stdout. 
