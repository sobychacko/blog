+++
title = "Basic stateful streaming application using Spring Cloud Stream and Kafka Streams"
description = ""
tags = [
    "spring-cloud-stream"
]
date = "2019-03-20"
categories = [
    "Development",
]
menu = "main"
+++

Continuing on what we started [here](https://sobychacko.github.io/posts/kafka-streams-binder-basic/), in this blog,
we are looking at how a basic stateful stream application can be written using Spring Cloud Stream and Kafka Streams.

Writing a stateless streaming application is very straightforward in Spring Cloud Stream, but when you have to
include state about your data, then we have more things to worry about. Kafka Streams make it really easy
to write stateful applications and Spring Cloud Stream provides tighter integrations for that.

Let's focus on a single, simple use case.

Imagine that your company is a credit card processing gateway and your business depends on the number of transactions
you approve through your gateway. If there is a higher rate of denied transactions due to failures on your end, then
that is a concern for you. In this situation, what most customers do is to re-route the request to a different
gateway and that means lost revenue for you. So, you want to provide a quicker way to monitor denied transactions.
You want to find this information relatively sooner so that you can take proactive measures. In reality,
these kind of gateway systems are very complex and involve many parameters. Often times, the transactions per minute
are in the hundreds of thousands of ranges, if not more. For the purposes of this blog, lets limit the scope to a very minimum. You want to find out the number of denied transactions due to issues that you could have prevented. You also want to find this information in little chunks of time windows such as 1 minute, 5 minute, 1 hour etc.

As you can notice, this requirement involves the need for some state. Why? Because you need to store the count of something based
on the data you received over a period of time window. Where is your app going to store this state? Traditionally, applications use some kind of
in-memory store, other database solutions, caches etc. for this purpose. Kafka Streams handles this natively and uses a built in
RocksDB database behind the scenes to store the state. All these are completely hidden from the user and handled behind the scenes.
You express your intention to store the state using the API.

Although this is a contrived use case, it is not hard to imagine seeing these kinds of situations regularly in the enterprise.

Lets write some code and see this in action.

We are going to make some assumptions about the data structures used in this example.
Following is the structure of the incoming data and we call it as `TransactionStatus`.
In the stateful processor we will write, the incoming data is a stream of `TransactionStatus` objects.

```java
class TransactionStatus {

  private int status; // 0 (approved) or 1 (denied)
  private int failCause;

  //Other fields are omitted
  ...
  //Setters and Getters are omitted

}
```

This is a very minimal basic class that satisfies our use case.
Among many possible other things, it gives two important details - one is the overall status of the
transaction whether it was approved or denied and the other is the reason for the failure.
Lets assume that if we get a status of 1, that means the transaction was denied.
Similarly, if the failCause is from 0 to 5 it means that it is a gateway failure.
Each of these various fail cause could mean that a particular type of failure at the gateway occurred such as network connection issues, software failures, hardware malfunction etc.
You want to find out the aggregate for each of these category of failures so that you can track if and take actions if it crosses a particular threshold that causes concern for you.

In a nutshell, you want to capture all those transactions with overall status of 1
and the failCause between 0 and 5. This will effectively give you the transactions those have failed because of issues on your end.

We want to write a processor that counts the number of such transactions every minute - i.e. a one minute tumbling window.
Every minute, each time a transaction failure occurs which is due to a gateway failure, you want to find out the failure count so far in that minute for that category and write that information to
a Kafka topic. This value will be reset to zero after the window is expired.

This is going to be the blueprint for the Java object that is going to be returned from the processor.

```java
class TransactionFailCounter {

  private int failCause;
  private int failCount;
  private long windowStartTime;
  private long windowEndTime;

  //other details are omitted

}
```

This will represent the number of failures within a range of time window.
Each record of this object will tell you the fail count so far for a particular category of fail cause.
It also includes the current time window to which the record belongs.

Here is the main processor code:

```java
@StreamListener("input")
@SendTo("output")
public KStream<?, TransactionFailMonitor> process(KStream<?, TransactionStatus> tansactionStream) {

	JsonSerde<TransactionStatus> jsonSerde = new JsonSerde<>(TransactionStatus.class);

	return tansactionStream
      .filter((k,v) -> v.getStatus() == 1 && v.getFailCause() <= 5)
			.map((k, v) -> new KeyValue<>(v.getFailCause(), v))
			.groupByKey(Serialized.with(Serdes.Integer(), jsonSerde))
			.windowedBy(TimeWindows.of(60_000))
			.count(Materialized.as("transactions-failure-counter-store"))
			.toStream()
			.map((k,v) -> new KeyValue<>(null,
					new TransactionFailMonitor(k.key(), v, k.window().start(), k.window().end())));
}
```

Once you have this processor ready, you can send some test data to the destination topic to see this in action.
In the real world scenario, you will most likely receive this information from some source data, but for testing this
we can use the console producer and consumer scripts that come with Kafka.

Start the console producer (assuming that your input topic is named as `transactions`).

```
kafka-console-producer.sh --broker-list localhost:9092 --topic transactions
```

Similarly, start the console consumer (assuming that your output topic is named as `failures`)

```
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic failures
```

Enter some input data with properly formatted JSON as shown below.

```
>{"status":1,"failCause":0}
>{"status":1,"failCause":1}
>{"status":1,"failCause":3}
>{"status":1,"failCause":5}
>{"status":1,"failCause":2}
>{"status":1,"failCause":5}
>{"status":1,"failCause":4}
>{"status":1,"failCause":5}
>{"status":1,"failCause":2}
>{"status":1,"failCause":1}
....
....
```

Spring Cloud Stream will automatically convert this JSON data to the proper Java objects using a message conversion strategy before
handing this to the processor above.

On the consumer console, you should see data as below:

```
{"failCause":0,"failCount":2,"windowStartTime":1553120520000,"windowEndTime":1553120580000}
{"failCause":1,"failCount":1,"windowStartTime":1553120520000,"windowEndTime":1553120580000}
{"failCause":2,"failCount":1,"windowStartTime":1553120520000,"windowEndTime":1553120580000}
{"failCause":3,"failCount":1,"windowStartTime":1553120520000,"windowEndTime":1553120580000}
{"failCause":5,"failCount":1,"windowStartTime":1553120520000,"windowEndTime":1553120580000}
{"failCause":2,"failCount":1,"windowStartTime":1553120580000,"windowEndTime":1553120640000}
{"failCause":5,"failCount":1,"windowStartTime":1553120580000,"windowEndTime":1553120640000}
{"failCause":4,"failCount":1,"windowStartTime":1553120580000,"windowEndTime":1553120640000}
{"failCause":5,"failCount":2,"windowStartTime":1553120580000,"windowEndTime":1553120640000}
{"failCause":2,"failCount":2,"windowStartTime":1553120580000,"windowEndTime":1553120640000}
{"failCause":1,"failCount":1,"windowStartTime":1553120580000,"windowEndTime":1553120640000}
```

On the outbound also, Spring Cloud Stream will send the data as a JSON string.

The code used in this blog and instructions for running the application can be found [here](https://github.com/schacko-samples/basic-stateful-sample).
