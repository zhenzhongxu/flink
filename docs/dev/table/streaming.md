---
title: "Streaming Concepts"
nav-parent_id: tableapi
nav-pos: 10
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

**TO BE DONE:** Intro

* This will be replaced by the TOC
{:toc}

Dynamic Table
-------------

This section will be reworked soon. Until then, please read the [introductory blog post about Dynamic Tables](http://flink.apache.org/news/2017/04/04/dynamic-tables.html).

**TO BE DONE:**

* Stream -> Table
* Table -> Stream
* update changes / retraction

{% top %}

Time Attributes
---------------

Flink is able to process streaming data based on different notions of *time*.

- *Processing time* refers to the system time of the machine (also known as "wall-clock time") that is executing the respective operation.
- *Event time* refers to the processing of streaming data based on timestamps which are attached to each row. The timestamps can encode when an event happened.
- *Ingestion time* is the time that events enter Flink; internally, it is treated similarly to event time.

For more information about time handling in Flink, see the introduction about [Event Time and Watermarks]({{ site.baseurl }}/dev/event_time.html).

Table programs require that the corresponding time characteristic has been specified for the streaming environment:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime); // default

// alternatively:
// env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime);
// env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment

env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime) // default

// alternatively:
// env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime)
// env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
{% endhighlight %}
</div>
</div>

Time-based operations such as windows in both the [Table API]({{ site.baseurl }}/dev/table/tableApi.html#group-windows) and [SQL]({{ site.baseurl }}/dev/table/sql.html#group-windows) require information about the notion of time and its origin. Therefore, tables can offer *logical time attributes* for indicating time and accessing corresponding timestamps in table programs.

Time attributes can be part of every table schema. They are defined when creating a table from a `DataStream` or are pre-defined when using a `TableSource`. Once a time attribute has been defined at the beginning, it can be referenced as a field and can used in time-based operations.

As long as a time attribute is not modified and is simply forwarded from one part of the query to another, it remains a valid time attribute. Time attributes behave like regular timestamps and can be accessed for calculations. If a time attribute is used in a calculation, it will be materialized and becomes a regular timestamp. Regular timestamps do not cooperate with Flink's time and watermarking system and thus can not be used for time-based operations anymore.

### Processing time

Processing time allows a table program to produce results based on the time of the local machine. It is the simplest notion of time but does not provide determinism. It neither requires timestamp extraction nor watermark generation.

There are two ways to define a processing time attribute.

#### During DataStream-to-Table Conversion

The processing time attribute is defined with the `.proctime` property during schema definition. The time attribute must only extend the physical schema by an additional logical field. Thus, it can only be defined at the end of the schema definition.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Tuple2<String, String>> stream = ...;

// declare an additional logical field as a processing time attribute
Table table = tEnv.fromDataStream(stream, "Username, Data, UserActionTime.proctime");

WindowedTable windowedTable = table.window(Tumble.over("10.minutes").on("UserActionTime").as("userActionWindow"));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val stream: DataStream[(String, String)] = ...

// declare an additional logical field as a processing time attribute
val table = tEnv.fromDataStream(stream, 'UserActionTimestamp, 'Username, 'Data, 'UserActionTime.proctime)

val windowedTable = table.window(Tumble over 10.minutes on 'UserActionTime as 'userActionWindow)
{% endhighlight %}
</div>
</div>

#### Using a TableSource

The processing time attribute is defined by a `TableSource` that implements the `DefinedProctimeAttribute` interface. The logical time attribute is appended to the physical schema defined by the return type of the `TableSource`.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// define a table source with a processing attribute
public class UserActionSource implements StreamTableSource<Row>, DefinedProctimeAttribute {

	@Override
	public TypeInformation<Row> getReturnType() {
		String[] names = new String[] {"Username" , "Data"};
		TypeInformation[] types = new TypeInformation[] {Types.STRING(), Types.STRING()};
		return Types.ROW(names, types);
	}

	@Override
	public DataStream<Row> getDataStream(StreamExecutionEnvironment execEnv) {
		// create stream 
		DataStream<Row> stream = ...;
		return stream;
	}

	@Override
	public String getProctimeAttribute() {
		// field with this name will be appended as a third field 
		return "UserActionTime";
	}
}

// register table source
tEnv.registerTableSource("UserActions", new UserActionSource());

WindowedTable windowedTable = tEnv
	.scan("UserActions")
	.window(Tumble.over("10.minutes").on("UserActionTime").as("userActionWindow"));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
// define a table source with a processing attribute
class UserActionSource extends StreamTableSource[Row] with DefinedProctimeAttribute {

	override def getReturnType = {
		val names = Array[String]("Username" , "Data")
		val types = Array[TypeInformation[_]](Types.STRING, Types.STRING)
		Types.ROW(names, types)
	}

	override def getDataStream(execEnv: StreamExecutionEnvironment): DataStream[Row] = {
		// create stream
		val stream = ...
		stream
	}

	override def getProctimeAttribute = {
		// field with this name will be appended as a third field 
		"UserActionTime"
	}
}

// register table source
tEnv.registerTableSource("UserActions", new UserActionSource)

val windowedTable = tEnv
	.scan("UserActions")
	.window(Tumble over 10.minutes on 'UserActionTime as 'userActionWindow)
{% endhighlight %}
</div>
</div>

### Event time

Event time allows a table program to produce results based on the time that is contained in every record. This allows for consistent results even in case of out-of-order events or late events. It also ensures replayable results of the table program when reading records from persistent storage. 

Additionally, event time allows for unified syntax for table programs in both batch and streaming environments. A time attribute in a streaming environment can be a regular field of a record in a batch environment.

In order to handle out-of-order events and distinguish between on-time and late events in streaming, Flink needs to extract timestamps from events and make some kind of progress in time (so-called [watermarks]({{ site.baseurl }}/dev/event_time.html)).

An event time attribute can be defined either during DataStream-to-Table conversion or by using a TableSource. 

The Table API & SQL assume that in both cases timestamps and watermarks have been generated in the [underlying DataStream API]({{ site.baseurl }}/dev/event_timestamps_watermarks.html) before. Ideally, this happens within a `TableSource` with knowledge about the incoming data's characteristics and is hidden from the end user of the API.


#### During DataStream-to-Table Conversion

The event time attribute is defined with the `.rowtime` property during schema definition. 

Timestamps and watermarks must have been assigned in the `DataStream` that is converted.

There are two ways of defining the time attribute when converting a `DataStream` into a `Table`:

- Extending the physical schema by an additional logical field
- Replacing a physical field by a logical field (e.g. because it is no longer needed after timestamp extraction).

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

// Option 1:

// extract timestamp and assign watermarks based on knowledge of the stream
DataStream<Tuple3<String, String>> stream = inputStream.assignTimestampsAndWatermarks(...);

// declare an additional logical field as an event time attribute
Table table = tEnv.fromDataStream(stream, "Username, Data, UserActionTime.rowtime");


// Option 2:

// extract timestamp from first field, and assign watermarks based on knowledge of the stream
DataStream<Tuple3<Long, String, String>> stream = inputStream.assignTimestampsAndWatermarks(...);

// the first field has been used for timestamp extraction, and is no longer necessary
// replace first field with a logical event time attribute
Table table = tEnv.fromDataStream(stream, "UserActionTime.rowtime, Username, Data");

// Usage:

WindowedTable windowedTable = table.window(Tumble.over("10.minutes").on("UserActionTime").as("userActionWindow"));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}

// Option 1:

// extract timestamp and assign watermarks based on knowledge of the stream
val stream: DataStream[(String, String)] = inputStream.assignTimestampsAndWatermarks(...)

// declare an additional logical field as an event time attribute
val table = tEnv.fromDataStream(stream, 'Username, 'Data, 'UserActionTime.rowtime)


// Option 2:

// extract timestamp from first field, and assign watermarks based on knowledge of the stream
val stream: DataStream[(Long, String, String)] = inputStream.assignTimestampsAndWatermarks(...)

// the first field has been used for timestamp extraction, and is no longer necessary
// replace first field with a logical event time attribute
val table = tEnv.fromDataStream(stream, 'UserActionTime.rowtime, 'Username, 'Data)

// Usage:

val windowedTable = table.window(Tumble over 10.minutes on 'UserActionTime as 'userActionWindow)
{% endhighlight %}
</div>
</div>

#### Using a TableSource

The event time attribute is defined by a `TableSource` that implements the `DefinedRowtimeAttribute` interface. The logical time attribute is appended to the physical schema defined by the return type of the `TableSource`.

Timestamps and watermarks must be assigned in the stream that is returned by the `getDataStream()` method.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
// define a table source with a rowtime attribute
public class UserActionSource implements StreamTableSource<Row>, DefinedRowtimeAttribute {

	@Override
	public TypeInformation<Row> getReturnType() {
		String[] names = new String[] {"Username" , "Data"};
		TypeInformation[] types = new TypeInformation[] {Types.STRING(), Types.STRING()};
		return Types.ROW(names, types);
	}

	@Override
	public DataStream<Row> getDataStream(StreamExecutionEnvironment execEnv) {
		// create stream 
		// ...
		// extract timestamp and assign watermarks based on knowledge of the stream
		DataStream<Row> stream = inputStream.assignTimestampsAndWatermarks(...);
		return stream;
	}

	@Override
	public String getRowtimeAttribute() {
		// field with this name will be appended as a third field 
		return "UserActionTime";
	}
}

// register the table source
tEnv.registerTableSource("UserActions", new UserActionSource());

WindowedTable windowedTable = tEnv
	.scan("UserActions")
	.window(Tumble.over("10.minutes").on("UserActionTime").as("userActionWindow"));
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
// define a table source with a rowtime attribute
class UserActionSource extends StreamTableSource[Row] with DefinedRowtimeAttribute {

	override def getReturnType = {
		val names = Array[String]("Username" , "Data")
		val types = Array[TypeInformation[_]](Types.STRING, Types.STRING)
		Types.ROW(names, types)
	}

	override def getDataStream(execEnv: StreamExecutionEnvironment): DataStream[Row] = {
		// create stream 
		// ...
		// extract timestamp and assign watermarks based on knowledge of the stream
		val stream = inputStream.assignTimestampsAndWatermarks(...)
		stream
	}

	override def getRowtimeAttribute = {
		// field with this name will be appended as a third field
		"UserActionTime"
	}
}

// register the table source
tEnv.registerTableSource("UserActions", new UserActionSource)

val windowedTable = tEnv
	.scan("UserActions")
	.window(Tumble over 10.minutes on 'UserActionTime as 'userActionWindow)
{% endhighlight %}
</div>
</div>

{% top %}

Query Configuration
-------------------

Table API and SQL queries have the same semantics regardless whether their input is bounded batch input or unbounded stream input. In many cases, continuous queries on streaming input are capable of computing accurate results that are identical to offline computed results. However, this is not possible in general case because continuous queries have to restrict the size of the state they are maintaining in order to avoid to run out of storage and to be able to process unbounded streaming data over a long period of time. As a result, a continuous query might only be able to provide approximated results depending on the characteristics of the input data and the query itself.

Flink's Table API and SQL interface provide parameters to tune the accuracy and resource consumption of continuous queries. The parameters are specified via a `QueryConfig` object. The `QueryConfig` can be obtained from the `TableEnvironment` and is passed back when a `Table` is translated, i.e., when it is [transformed into a DataStream](common.html#convert-a-table-into-a-datastream-or-dataset) or [emitted via a TableSink](common.html#emit-a-table).

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// obtain query configuration from TableEnvironment
StreamQueryConfig qConfig = tableEnv.queryConfig();
// set query parameters
qConfig.withIdleStateRetentionTime(Time.hours(12));

// define query
Table result = ...

// create TableSink
TableSink<Row> sink = ...

// emit result Table via a TableSink
result.writeToSink(sink, qConfig);

// convert result Table into a DataStream<Row>
DataStream<Row> stream = tableEnv.toAppendStream(result, Row.class, qConfig);

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val env = StreamExecutionEnvironment.getExecutionEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

// obtain query configuration from TableEnvironment
val qConfig: StreamQueryConfig = tableEnv.queryConfig
// set query parameters
qConfig.withIdleStateRetentionTime(Time.hours(12))

// define query
val result: Table = ???

// create TableSink
val sink: TableSink[Row] = ???

// emit result Table via a TableSink
result.writeToSink(sink, qConfig)

// convert result Table into a DataStream[Row]
val stream: DataStream[Row] = result.toAppendStream[Row](qConfig)

{% endhighlight %}
</div>
</div>

In the the following we describe the parameters of the `QueryConfig` and how they affect the accuracy and resource consumption of a query.

### Idle State Retention Time

Many queries aggregate or join records on one or more key attributes. When such a query is executed on a stream, the continuous query needs to collect records or maintain partial results per key. If the key domain of the input stream is evolving, i.e., the active key values are changing over time, the continuous query accumulates more and more state as more and more distinct keys are observed. However, often keys become inactive after some time and their corresponding state becomes stale and useless.

For example the following query computes the number of clicks per session.

```
SELECT sessionId, COUNT(*) FROM clicks GROUP BY sessionId;
```

The `sessionId` attribute is used as a grouping key and the continuous query maintains a count for each `sessionId` it observes. The `sessionId` attribute is evolving over time and `sessionId` values are only active until the session ends, i.e., for a limited period of time. However, the continuous query cannot know about this property of `sessionId` and expects that every `sessionId` value can occur at any point of time. It maintains a count for each observed `sessionId` value. Consequently, the total state size of the query is continuously growing as more and more `sessionId` values are observed. 

The *Idle State Retention Time* parameters define for how long the state of a key is retained without being updated before it is removed. For the previous example query, the count of a `sessionId` would be removed as soon as it has not been updated for the configured period of time.

By removing the state of a key, the continuous query completely forgets that it has seen this key before. If a record with a key, whose state has been removed before, is processed, the record will be treated as if it was the first record with the respective key. For the example above this means that the count of a `sessionId` would start again at `0`.

There are two parameters to configure the idle state retention time:
- The *minimum idle state retention time* defines how long the state of an inactive key is at least kept before it is removed.
- The *maximum idle state retention time* defines how long the state of an inactive key is at most kept before it is removed.

The parameters are specified as follows:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

StreamQueryConfig qConfig = ...

// set idle state retention time: min = 12 hour, max = 16 hours
qConfig.withIdleStateRetentionTime(Time.hours(12), Time.hours(16));
// set idle state retention time. min = max = 12 hours
qConfig.withIdleStateRetentionTime(Time.hours(12);

{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}

val qConfig: StreamQueryConfig = ???

// set idle state retention time: min = 12 hour, max = 16 hours
qConfig.withIdleStateRetentionTime(Time.hours(12), Time.hours(16))
// set idle state retention time. min = max = 12 hours
qConfig.withIdleStateRetentionTime(Time.hours(12)

{% endhighlight %}
</div>
</div>

Configuring different minimum and maximum idle state retention times is more efficient because it reduces the internal book-keeping of a query for when to remove state.

{% top %}


