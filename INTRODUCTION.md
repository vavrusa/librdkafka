//@file INTRODUCTION.md
# Introduction to librdkafka - the Apache Kafka C/C++ client library


librdkafka is a high performance C implementation of the Apache
Kafka client, providing a reliable and performant client for production use.
librdkafka also provides a native C++ interface.

## Contents

The following chapters are available in this document

  * [Performance](#performance)
    * [Performance numbers](#performance-numbers)
    * [High throughput](#high-throughput)
    * [Low latency](#low-latency)
    * [Compression](#compression)
  * [Message reliability](#message-reliability)
    * [Idempotent Producer](#idempotent-producer)
  * [Usage](#usage)
    * [Documentation](#documentation)
    * [Initialization](#initialization)
    * [Configuration](#configuration)
    * [Threads and callbacks](#threads-and-callbacks)
    * [Brokers](#brokers)
    * [Producer API](#producer-api)
    * [Consumer API](#simple-consumer-api-legacy)
  * [Appendix](#appendix)
    * [Test details](#test-details)
  



## Performance

librdkafka is a multi-threaded library designed for use on modern hardware and
it attempts to keep memory copying at a minimal. The payload of produced or
consumed messages may pass through without any copying
(if so desired by the application) putting no limit on message sizes.

librdkafka allows you to decide if high throughput is the name of the game,
or if a low latency service is required, all through the configuration
property interface.

The two most important configuration properties for performance tuning are:

  * `batch.num.messages` - the maximum number of messages to wait for to
	  accumulate in the local queue before sending off a message set.
  * `queue.buffering.max.ms` - how long to wait for batch.num.messages to
	  fill up in the local queue. A lower value improves latency at the
          cost of lower throughput and higher per-message overhead.
          A higher value improves throughput at the expense of latency.
          The recommended value for high throughput is > 50ms.


### Performance numbers

The following performance numbers stem from tests using the following setup:

  * Intel Quad Core i7 at 3.4GHz, 8GB of memory
  * Disk performance has been shortcut by setting the brokers' flush
	configuration properties as so:
	* `log.flush.interval.messages=10000000`
	* `log.flush.interval.ms=100000`
  * Two brokers running on the same machine as librdkafka.
  * One topic with two partitions.
  * Each broker is leader for one partition each.
  * Using `rdkafka_performance` program available in the `examples` subdir.



	

**Test results**

  * **Test1**: 2 brokers, 2 partitions, required.acks=2, 100 byte messages: 
	  **850000 messages/second**, **85 MB/second**

  * **Test2**: 1 broker, 1 partition, required.acks=0, 100 byte messages: 
	  **710000 messages/second**, **71 MB/second**
	  
  * **Test3**: 2 broker2, 2 partitions, required.acks=2, 100 byte messages,
	  snappy compression:
	  **300000 messages/second**, **30 MB/second**

  * **Test4**: 2 broker2, 2 partitions, required.acks=2, 100 byte messages,
	  gzip compression:
	  **230000 messages/second**, **23 MB/second**



**Note**: See the *Test details* chapter at the end of this document for
	information about the commands executed, etc.

**Note**: Consumer performance tests will be announced soon.


### High throughput

The key to high throughput is message batching - waiting for a certain amount
of messages to accumulate in the local queue before sending them off in
one large message set or batch to the peer. This amortizes the messaging
overhead and eliminates the adverse effect of the round trip time (rtt).

`queue.buffering.max.ms` (also called `linger.ms`) allows librdkafka to
wait up to the specified amount of time to accumulate up to
`batch.num.messages` in a single batch (MessageSet) before sending
to the broker. The larger the batch the higher the throughput.
Enabling `msg` debugging (set `debug` property to `msg`) will emit log
messages for the accumulation process which lets you see what batch sizes
are being produced.

Example using `queue.buffering.max.ms=1`:

```
... test [0]: MessageSet with 1514 message(s) delivered
... test [3]: MessageSet with 1690 message(s) delivered
... test [0]: MessageSet with 1720 message(s) delivered
... test [3]: MessageSet with 2 message(s) delivered
... test [3]: MessageSet with 4 message(s) delivered
... test [0]: MessageSet with 4 message(s) delivered
... test [3]: MessageSet with 11 message(s) delivered
```

Example using `queue.buffering.max.ms=1000`:
```
... test [0]: MessageSet with 10000 message(s) delivered
... test [0]: MessageSet with 10000 message(s) delivered
... test [0]: MessageSet with 4667 message(s) delivered
... test [3]: MessageSet with 10000 message(s) delivered
... test [3]: MessageSet with 10000 message(s) delivered
... test [3]: MessageSet with 4476 message(s) delivered

```


The default setting of `queue.buffering.max.ms=1` is not suitable for
high throughput, it is recommended to set this value to >50ms, with
throughput leveling out somewhere around 100-1000ms depending on
message produce pattern and sizes.

These setting are set globally (`rd_kafka_conf_t`) but applies on a
per topic+partition basis.


### Low latency

When low latency messaging is required the `queue.buffering.max.ms` should be
tuned to the maximum permitted producer-side latency.
Setting queue.buffering.max.ms to 1 will make sure messages are sent as
soon as possible. You could check out [How to decrease message latency](https://github.com/edenhill/librdkafka/wiki/How-to-decrease-message-latency)
to find more details.
Lower buffering time leads to smaller batches and larger per-message overheads,
increasing network, memory and CPU usage for producers, brokers and consumers.

#### Latency measurement

End-to-end latency is preferably measured by synchronizing clocks on producers
and consumers and using the message timestamp on the consumer to calculate
the full latency. Make sure the topic's `log.message.timestamp.type` is set to
the default `CreateTime` (Kafka topic configuration, not librdkafka topic).

Latencies are typically incurred by the producer, network and broker, the
consumer effect on end-to-end latency is minimal.

To break down the end-to-end latencies and find where latencies are adding up
there are a number of metrics available through librdkafka statistics
on the producer:

 * `brokers[].int_latency` is the time, per message, between produce()
   and the message being written to a MessageSet and ProduceRequest.
   High `int_latency` indicates CPU core contention: check CPU load and,
   involuntary context switches (`/proc/<..>/status`).
   Consider using a machine/instance with more CPU cores.
   This metric is only relevant on the producer.

 * `brokers[].outbuf_latency` is the time, per protocol request
   (such as ProduceRequest), between the request being enqueued (which happens
   right after int_latency) and the time the request is written to the
   TCP socket connected to the broker.
   High `outbuf_latency` indicates CPU core contention or network congestion:
   check CPU load and socket SendQ (`netstat -anp | grep :9092`).

 * `brokers[].rtt` is the time, per protocol request, between the request being
   written to the TCP socket and the time the response is received from
   the broker.
   High `rtt` indicates broker load or network congestion:
   check broker metrics, local socket SendQ, network performance, etc.

 * `brokers[].throttle` is the time, per throttled protocol request, the
   broker throttled/delayed handling of a request due to usage quotas.
   The throttle time will also be reflected in `rtt`.

 * `topics[].batchsize` is the size of individual Producer MessageSet batches.
   See below.

 * `topics[].batchcnt` is the number of messages in individual Producer
   MessageSet batches. Due to Kafka protocol overhead a batch with few messages
   will have a higher relative processing and size overhead than a batch
   with many messages.
   Use the `linger.ms` client configuration property to set the maximum
   amount of time allowed for accumulating a single batch, the larger the
   value the larger the batches will grow, thus increasing efficiency.
   When producing messages at a high rate it is recommended to increase
   linger.ms, which will improve throughput and in some cases also latency.


See [STATISTICS.md](STATISTICS.md) for the full definition of metrics.
A JSON schema for the statistics is available in
[statistics-schema.json](src/statistics-schema.json).


### Compression

Producer message compression is enabled through the `compression.codec`
configuration property.

Compression is performed on the batch of messages in the local queue, the
larger the batch the higher likelyhood of a higher compression ratio.
The local batch queue size is controlled through the `batch.num.messages` and
`queue.buffering.max.ms` configuration properties as described in the
**High throughput** chapter above.



## Message reliability

Message reliability is an important factor of librdkafka - an application
can rely fully on librdkafka to deliver a message according to the specified
configuration (`request.required.acks` and `message.send.max.retries`, etc).

If the topic configuration property `request.required.acks` is set to wait
for message commit acknowledgements from brokers (any value but 0, see
[`CONFIGURATION.md`](https://github.com/edenhill/librdkafka/blob/master/CONFIGURATION.md)
for specifics) then librdkafka will hold on to the message until
all expected acks have been received, gracefully handling the following events:

  * Broker connection failure
  * Topic leader change
  * Produce errors signaled by the broker
  * Network problems

We recommend `request.required.acks` to be set to `all` to make sure
produced messages are acknowledged by all in-sync replica brokers.

This is handled automatically by librdkafka and the application does not need
to take any action at any of the above events.
The message will be resent up to `message.send.max.retries` times before
reporting a failure back to the application.

The delivery report callback is used by librdkafka to signal the status of
a message back to the application, it will be called once for each message
to report the status of message delivery:

  * If `error_code` is non-zero the message delivery failed and the error_code
    indicates the nature of the failure (`rd_kafka_resp_err_t` enum).
  * If `error_code` is zero the message has been successfully delivered.

See Producer API chapter for more details on delivery report callback usage.

The delivery report callback is optional but highly recommended.


### Producer message delivery success

When a ProduceRequest is successfully handled by the broker and a
ProduceResponse is received (also called the ack) without an error code
the messages from the ProduceRequest are enqueued on the delivery report
queue (if a delivery report callback has been set) and will be passed to
the application on the next invocation rd_kafka_poll().


### Producer message delivery failure

The following sub-chapters explains how different produce errors
are handled.

If the error is retryable and there are remaining retry attempts for
the given message(s), an automatic retry will be scheduled by librdkafka,
these retries are not visible to the application.

Only permanent errors and temporary errors that have reached their maximum
retry count will generate a delivery report event to the application with an
error code set.

The application should typically not attempt to retry producing the message
on failure, but instead configure librdkafka to perform these retries
using the `retries` and `retry.backoff.ms` configuration properties.


#### Error: Timed out in transmission queue

Internal error ERR__TIMED_OUT_QUEUE.

The connectivity to the broker may be stalled due to networking contention,
local or remote system issues, etc, and the request has not yet been sent.

The producer can be certain that the message has not been sent to the broker.

This is a retryable error, but is not counted as a retry attempt
since the message was never actually transmitted.

A retry by librdkafka at this point will not cause duplicate messages.


#### Error: Timed out in flight to/from broker

Internal error ERR__TIMED_OUT, ERR__TRANSPORT.

Same reasons as for `Timed out in transmission queue` above, with the
difference that the message may have been sent to the broker and might
be stalling waiting for broker replicas to ack the message, or the response
could be stalled due to networking issues.
At this point the producer can't know if the message reached the broker,
nor if the broker wrote the message to disk and replicas.

This is a retryable error.

A retry by librdkafka at this point may cause duplicate messages.


#### Error: Temporary broker-side error

Broker errors ERR_REQUEST_TIMED_OUT, ERR_NOT_ENOUGH_REPLICAS,
ERR_NOT_ENOUGH_REPLICAS_AFTER_APPEND.

These errors are considered temporary and librdkafka is will retry them
if permitted by configuration.


#### Error: Temporary errors due to stale metadata

Broker errors ERR_LEADER_NOT_AVAILABLE, ERR_NOT_LEADER_FOR_PARTITION.

These errors are considered temporary and a retry is warranted, a metadata
request is automatically sent to find a new leader for the partition.

A retry by librdkafka at this point will not cause duplicate messages.


#### Error: Local time out

Internal error ERR__MSG_TIMED_OUT.

The message could not be successfully transmitted before `message.timeout.ms`
expired, typically due to no leader being available or no broker connection.
The message may have been retried due to other errors but
those error messages are abstracted by the ERR__MSG_TIMED_OUT error code.

Since the `message.timeout.ms` has passed there will be no more retries
by librdkafka.


#### Error: Permanent errors

Any other error is considered a permanent error and the message
will fail immediately, generating a delivery report event with the
distinctive error code.

The full list of permanent errors depend on the broker version and
will likely grow in the future.

Typical permanent broker errors are:
 * ERR_CORRUPT_MESSAGE
 * ERR_MSG_SIZE_TOO_LARGE  - adjust client's or broker's `message.max.bytes`.
 * ERR_UNKNOWN_TOPIC_OR_PART - topic or partition does not exist,
                               automatic topic creation is disabled on the
                               broker or the application is specifying a
                               partition that does not exist.
 * ERR_RECORD_LIST_TOO_LARGE
 * ERR_INVALID_REQUIRED_ACKS
 * ERR_TOPIC_AUTHORIZATION_FAILED
 * ERR_UNSUPPORTED_FOR_MESSAGE_FORMAT
 * ERR_CLUSTER_AUTHORIZATION_FAILED


### Producer retries

The ProduceRequest itself is not retried, instead the messages
are put back on the internal partition queue by an insert sort
that maintains their original position (the message order is defined
at the time a message is initially appended to a partition queue, i.e., after
partitioning).
A backoff time (`retry.backoff.ms`) is set on the retried messages which
effectively blocks retry attempts until the backoff time has expired.


### Reordering

As for all retries, if `max.in.flight` > 1 and `retries` > 0, retried messages
may be produced out of order, since a sub-sequent message in a sub-sequent
ProduceRequest may already be in-flight (and accepted by the broker)
by the time the retry for the failing message is sent.

Using the Idempotent Producer prevents reordering even with `max.in.flight` > 1,
see [Idempotent Producer](#idempotent-producer) below for more information.


### Idempotent Producer

librdkafka supports the idempotent producer which provides strict ordering and
and exactly-once producer guarantees.
The idempotent producer is enabled by setting the `enable.idempotence`
configuration property to `true`, this will automatically adjust a number of
other configuration properties to adhere to the idempotency requirements,
see the documentation of `enable.idempotence` in [CONFIGURATION.md] for
more information.
Producer instantiation will fail if the user supplied an incompatible value
for any of the automatically adjusted properties, e.g., it is an error to
explicitly set `acks=1` when `enable.idempotence=true` is set.


#### Guarantees

There are three types of guarantees that the idempotent producer can satisfy:

 * Exactly-once - a message is only written to the log once.
                  Does NOT cover the exactly-once consumer case.
 * Ordering - a series of messages are written to the log in the
              order they were produced.
 * Gap-less - a series of messages are written once and in order
              without risk of skipping messages. The sequence
              of messages may be cut short and fail before all
              messages are written, but may not fail individual
              messages in the series.
              This guarantee is enabled by default, but may be disabled
              by setting `enable.gapless.guarantee` if individual message
              failure is not a concern.


All three guarantees are in effect when idempotence is enabled, only
gap-less may be disabled individually.


#### Ordering and message sequence numbers

librdkafka maintains the original produce() ordering per-partition for all
messages produced, using an internal per-partition 64-bit counter
called the msgseq which starts at 1. This msgseq allows messages to be
re-inserted in the partition message queue in the original order in the
case of retries.

The Idempotent Producer functionality in the Kafka protocol also has
a per-message sequence number, which is a signed 32-bit wrapping counter that is
reset each time the Producer's ID (PID) or Epoch changes.

The librdkafka msgseq is used, along with a base msgseq value stored
at the time the PID/Epoch was bumped, to calculate the Kafka protocol's
message sequence number.

With Idempotent Producer enabled there is no risk of reordering despite
`max.in.flight` > 1 (capped at 5).

**Note**: "msgseq" in log messages refer to the librdkafka msgseq, while "seq"
          refers to the protocol message sequence, "baseseq" is the seq of
          the first message in a batch.


The producer also maintains two next expected sequence counters:

 * `next_ack_seq` - the next sequence to expect an acknowledgement for, which
                    is the last successfully produced MessageSet's last
                    sequence + 1.
 * `next_err_seq` - the next sequence to expect an error for, which is typically
                    the same as `next_ack_seq` until an error occurs, in which
                    case the `next_ack_seq` can't be incremented (since no
                    messages were acked on error). `next_err_seq` is used to
                    properly handle sub-sequent errors due to a failing
                    first request.

**Note**: Both are exposed in partition statistics.



#### Partitioner considerations

Strict ordering is guaranteed on a **per partition** basis.

An application utilizing the idempotent producer should not mix
producing to explicit partitions with partitioner-based partitions
since messages produced for the latter are queued separately until
a topic's partition count is known, which would insert these messages
after the partition-explicit messages regardless of produce order.


#### Leader change

There are corner cases where an Idempotent Producer has outstanding
ProduceRequests in-flight to the previous leader while a new leader is elected.

A leader change is typically triggered by the original leader
failing or terminating, which has the risk of also failing (some of) the
in-flight ProduceRequests to that broker. To recover the producer to a
consistent state it will not send any ProduceRequests for these partitions to
the new leader broker until all responses for any outstanding ProduceRequests
to the previous partition leader has been received, or these requests have
timed out.
This drain may take up to `min(socket.timeout.ms, message.timeout.ms)`.
If the connection to the previous broker goes down the outstanding requests
are failed immediately.


#### Error handling

Background:
The error handling for the Idempotent Producer, as initially proposed
in the [EOS design document](https://docs.google.com/document/d/11Jqy_GjUGtdXJK94XGsEIK7CP1SnQGdp2eF0wSw9ra8),
missed some corner cases which are now being addressed in [KIP-360](https://cwiki.apache.org/confluence/display/KAFKA/KIP-360%3A+Improve+handling+of+unknown+producer).
There were some intermediate fixes and workarounds prior to KIP-360 that proved
to be incomplete and made the error handling in the client overly complex.
With the benefit of hindsight the librdkafka implementation will attempt
to provide correctness from the lessons learned in the Java client and
provide stricter and less complex error handling.

Note: At the time of this writing KIP-360 has not been accepted.


The follow sections describe librdkafka's handling of the
Idempotent Producer specific errors that may be returned by the broker.


##### RD_KAFKA_RESP_ERR_OUT_OF_ORDER_SEQUENCE_NUMBER

This error is returned by the broker when the sequence number in the
ProduceRequest is larger than the expected next sequence
for the given PID+Epoch+Partition (last BaseSeq + msgcount + 1).
Note: sequence 0 is always accepted.

If the failed request is the head-of-line (next expected sequence to be acked)
it indicates desynchronization between the client and broker:
the client thinks the sequence number is correct but the broker disagrees.
There is no way for the client to recover from this scenario without
risking message loss or duplication, and it is not safe for the
application to manually retry messages.
A fatal error (`RD_KAFKA_RESP_ERR_OUT_OF_ORDER_SEQUENCE_NUMBER`) is raised.

When the request is not head-of-line the previous request failed
(for any reason), which means the messages in the current request
can be retried after waiting for all outstanding requests for this
partition to drain and then reset the Producer ID and start over.


**Java Producer behaviour**:
Fail the batch, reset the pid, and then continue producing
(and retrying sub-sequent) messages. This will lead to gaps
in the message series.



##### RD_KAFKA_RESP_ERR_DUPLICATE_SEQUENCE_NUMBER

Returned by broker when the request's base sequence number is
less than the expected sequence number (which is the last written
sequence + msgcount).
Note: sequence 0 is always accepted.

This error is typically benign and occurs upon retrying a previously successful
send that was not acknowledged.

The messages will be considered successfully produced but will have neither
timestamp or offset set.


**Java Producer behaviour:**
Treats the message as successfully delivered.


##### RD_KAFKA_RESP_ERR_UNKNOWN_PRODUCER_ID

Returned by broker when the PID+Epoch is unknown, which may occur when
the PID's state has expired (due to topic retention, DeleteRercords,
or compaction).

The Java producer added quite a bit of error handling for this case,
extending the ProduceRequest protocol to return the logStartOffset
to give the producer a chance to differentiate between an actual
UNKNOWN_PRODUCER_ID or topic retention having deleted the last
message for this producer (effectively voiding the Producer ID cache).
This workaround proved to be error prone (see explanation in KIP-360)
when the partition leader changed.

KIP-360 suggests removing this error checking in favour of failing fast,
librdkafka follows suite.


If the response is for the first ProduceRequest in-flight
and there are no messages waiting to be retried nor any ProduceRequests
unaccounted for, then the error is ignored and the epoch is incremented,
this is likely to happen for an idle producer who's last written
message has been deleted from the log, and thus its PID state.
Otherwise the producer raises a fatal error
(RD_KAFKA_RESP_ERR_UNKNOWN_PRODUCER_ID) since the delivery guarantees can't
be satisfied.


**Java Producer behaviour:**
Retries the send in some cases (but KIP-360 will change this).
Not a fatal error in any case.


##### Standard errors

All the standard Produce errors are handled in the usual way,
permanent errors will fail the messages in the batch, while
temporary errors will be retried (if retry count permits).

If a permanent error is returned for a batch in a series of in-flight batches,
the sub-sequent batches will fail with
RD_KAFKA_RESP_ERR_OUT_OF_ORDER_SEQUENCE_NUMBER since the sequence number of the
failed batched was never written to the topic log and next expected sequence
thus not incremented on the broker.

A fatal error (RD_KAFKA_RESP_ERR__GAPLESS_GUARANTEE) is raised to satisfy
the gap-less guarantee (if `enable.gapless.guarantee` is set) by failing all
queued messages.


##### Message persistance status

To help the application decide what to do in these error cases, a new
per-message API is introduced, `rd_kafka_message_persistance_status()`,
which returns one of the following values:

 * `RD_KAFKA_MSG_STATUS_NOT_PERSISTED` - the message has never
   been transmitted to the broker, or failed with an error indicating
   it was not written to the log.
   Application retry will risk ordering, but not duplication.
 * `RD_KAFKA_MSG_STATUS_POSSIBLY_PERSISTED` - the message was transmitted
   to the broker, but no acknowledgement was received.
   Application retry will risk ordering and duplication.
 * `RD_KAFKA_MSG_STATUS_PERSISTED` - the message was written to the log by
   the broker and fully acknowledged.
   No reason for application to retry.

This method should be called by the application on delivery report error.





## Usage

### Documentation

The librdkafka API is documented in the
[`rdkafka.h`](https://github.com/edenhill/librdkafka/blob/master/src/rdkafka.h)
header file, the configuration properties are documented in
[`CONFIGURATION.md`](https://github.com/edenhill/librdkafka/blob/master/CONFIGURATION.md)

### Initialization

The application needs to instantiate a top-level object `rd_kafka_t` which is
the base container, providing global configuration and shared state.
It is created by calling `rd_kafka_new()`.

It also needs to instantiate one or more topics (`rd_kafka_topic_t`) to be used
for producing to or consuming from. The topic object holds topic-specific
configuration and will be internally populated with a mapping of all available
partitions and their leader brokers.
It is created by calling `rd_kafka_topic_new()`.

Both `rd_kafka_t` and `rd_kafka_topic_t` comes with a configuration API which
is optional.
Not using the API will cause librdkafka to use its default values which are
documented in [`CONFIGURATION.md`](https://github.com/edenhill/librdkafka/blob/master/CONFIGURATION.md).

**Note**: An application may create multiple `rd_kafka_t` objects and
	they share no state.

**Note**: An `rd_kafka_topic_t` object may only be used with the `rd_kafka_t`
	object it was created from.



### Configuration

To ease integration with the official Apache Kafka software and lower
the learning curve, librdkafka implements identical configuration
properties as found in the official clients of Apache Kafka.

Configuration is applied prior to object creation using the
`rd_kafka_conf_set()` and `rd_kafka_topic_conf_set()` APIs.

**Note**: The `rd_kafka.._conf_t` objects are not reusable after they have been
	passed to `rd_kafka.._new()`.
	The application does not need to free any config resources after a
	`rd_kafka.._new()` call.

#### Example

    rd_kafka_conf_t *conf;
    char errstr[512];

    conf = rd_kafka_conf_new();
    rd_kafka_conf_set(conf, "compression.codec", "snappy", errstr, sizeof(errstr));
    rd_kafka_conf_set(conf, "batch.num.messages", "100", errstr, sizeof(errstr));

    rd_kafka_new(RD_KAFKA_PRODUCER, conf);


### Threads and callbacks

librdkafka uses multiple threads internally to fully utilize modern hardware.
The API is completely thread-safe and the calling application may call any
of the API functions from any of its own threads at any time.

A poll-based API is used to provide signaling back to the application,
the application should call rd_kafka_poll() at regular intervals.
The poll API will call the following configured callbacks (optional):

  * `dr_msg_cb` - Message delivery report callback - signals that a message has
    been delivered or failed delivery, allowing the application to take action
    and to release any application resources used in the message.
  * `error_cb` - Error callback - signals an error. These errors are usually of
    an informational nature, i.e., failure to connect to a broker, and the
    application usually does not need to take any action.
    The type of error is passed as a rd_kafka_resp_err_t enum value,
    including both remote broker errors as well as local failures.
    An application typically does not have to perform any action when
    an error is raised through the error callback, the client will
    automatically try to recover from all errors, given that the
    client and cluster is correctly configured.
    In some specific cases a fatal error may occur which will render
    the client more or less inoperable for further use:
    if the error code in the error callback is set to
    `RD_KAFKA_RESP_ERR__FATAL` the application should retrieve the
    underlying fatal error and reason using the `rd_kafka_fatal_error()` call,
    and then begin terminating the instance.
    The Event API's EVENT_ERROR has a `rd_kafka_event_error_is_fatal()`
    function, and the C++ EventCb has a `fatal()` method, to help the
    application determine if an error is fatal or not.
  * `stats_cb` - Statistics callback - triggered if `statistics.interval.ms`
    is configured to a non-zero value, emitting metrics and internal state
    in JSON format, see [STATISTICS.md].
  * `throttle_cb` - Throttle callback - triggered whenever a broker has
    throttled (delayed) a request.

These callbacks will also be triggered by `rd_kafka_flush()`,
`rd_kafka_consumer_poll()`, and any other functions that serve queues.


Optional callbacks not triggered by poll, these may be called spontaneously
from any thread at any time:

  * `log_cb` - Logging callback - allows the application to output log messages
    generated by librdkafka.
  * `partitioner` - Partitioner callback - application provided message partitioner.
    The partitioner may be called in any thread at any time, it may be
    called multiple times for the same key.
    Partitioner function contraints:
      - MUST NOT call any rd_kafka_*() functions
      - MUST NOT block or execute for prolonged periods of time.
      - MUST return a value between 0 and partition_cnt-1, or the
        special RD_KAFKA_PARTITION_UA value if partitioning
        could not be performed.



### Brokers

On initialization, librdkafka only needs a partial list of
brokers (at least one), called the bootstrap brokers.
The client will connect to the bootstrap brokers, specified by the
`bootstrap.servers` (or `metadata.broker.list`) configuration property or
by `rd_kafka_brokers_add()`, and query cluster Metadata information
which contains the full list of brokers, topic, partitions and their
leaders in the Kafka cluster.

Broker names are specified as `host[:port]` where the port is optional
(default 9092) and the host is either a resolvable hostname or an IPv4 or IPv6
address.
If host resolves to multiple addresses librdkafka will round-robin the
addresses for each connection attempt.
A DNS record containing all broker address can thus be used to provide a
reliable bootstrap broker.


#### Sparse connections

Maintaining an open connection to each broker in the cluster is problematic
for clusters with a large number of brokers, as well as for clusters with
large number of clients.

To alleviate this, the `enable.sparse.connections=true` configuration property
can be set to `true` (**default**), in which case the client will only connect
to brokers it needs to communicate with, and only when necessary.

Examples of needed broker connections are:

 * leaders for partitions being consumed from
 * leaders for partitions being produced to
 * consumer group coordinator broker
 * cluster controller for Admin API operations


##### Random broker selection

When there is no broker connection and a connection to any broker
is needed, such as on startup to retrieve metadata, the client randomly selects
a broker from its list of brokers, which includes both the configure bootstrap
brokers (including brokers manually added with `rd_kafka_brokers_add()`), as
well as the brokers discovered from cluster metadata.
Brokers with no prior connection attempt are tried first.

If there is already an available broker connection to any broker it is used,
rather than connecting to a new one.

The random broker selection and connection scheduling is triggered when:
 * bootstrap servers are configured (`rd_kafka_new()`)
 * brokers are manually added (`rd_kafka_brokers_add()`).
 * a consumer group coordinator needs to be found.
 * acquiring a ProducerID for the Idempotent Producer.
 * cluster or topic metadata is being refreshed.

A single connection attempt will be performed, and the broker will
return to an idle INIT state on failure to connect.

The random broker selection is rate-limited to:
10 < `reconnect.backoff.ms`/2 < 1000 milliseconds.

**Note**: The broker connection will be maintained until it is closed
          by the broker (idle connection reaper).

##### Persistent broker connections

While the random broker selection is useful for one-off queries, there
is need for the client to maintain persistent connections to certain brokers:
 * Consumer: the group coordinator.
 * Consumer: partition leader for topics being fetched from.
 * Producer: partition leader for topics being produced to.

These dependencies are discovered and maintained automatically, marking
matching brokers as persistent, which will make the client maintain connections
to these brokers at all times, reconnecting as necessary.


#### Connection close

A broker connection may be closed by the broker, intermediary network gear,
due to network errors, timeouts, etc.
When a broker connection is closed, librdkafka will back off the next reconnect
attempt (to the given broker) for `reconnect.backoff.ms` -25% to +50% jitter,
this value is increased exponentially for each connect attempt until
`reconnect.backoff.max.ms` is reached, at which time the value is reset
to `reconnect.backoff.ms`.

The broker will disconnect clients that have not sent any protocol requests
within `connections.max.idle.ms` (broker configuration propertion, defaults
to 10 minutes), but there is no fool proof way for the client to know that it
was a deliberate close by the broker and not an error. To avoid logging these
deliberate idle disconnects as errors the client employs some logic to try to
classify a disconnect as an idle disconnect if no requests have been sent in
the last `socket.timeout.ms` or there are no outstanding, or
queued, requests waiting to be sent. In this case the standard "Disconnect"
error log is silenced (will only be seen with debug enabled).

`log.connection.close=false` may be used to silence all disconnect logs,
but it is recommended to instead rely on the above heuristics.





### Feature discovery

Apache Kafka broker version 0.10.0 added support for the ApiVersionRequest API
which allows a client to query a broker for its range of supported API versions.

librdkafka supports this functionality and will query each broker on connect
for this information (if `api.version.request=true`) and use it to enable or disable
various protocol features, such as MessageVersion 1 (timestamps), KafkaConsumer, etc.

If the broker fails to respond to the ApiVersionRequest librdkafka will
assume the broker is too old to support the API and fall back to an older
broker version's API. These fallback versions are hardcoded in librdkafka
and is controlled by the `broker.version.fallback` configuration property.



### Producer API

After setting up the `rd_kafka_t` object with type `RD_KAFKA_PRODUCER` and one
or more `rd_kafka_topic_t` objects librdkafka is ready for accepting messages
to be produced and sent to brokers.

The `rd_kafka_produce()` function takes the following arguments:

  * `rkt` - the topic to produce to, previously created with
	  `rd_kafka_topic_new()`
  * `partition` - partition to produce to. If this is set to
	  `RD_KAFKA_PARTITION_UA` (UnAssigned) then the configured partitioner
		  function will be used to select a target partition.
  * `msgflags` - 0, or one of:
	  * `RD_KAFKA_MSG_F_COPY` - librdkafka will immediately make a copy of
	    the payload. Use this when the payload is in non-persistent
	    memory, such as the stack.
	  * `RD_KAFKA_MSG_F_FREE` - let librdkafka free the payload using
	    `free(3)` when it is done with it.

	These two flags are mutually exclusive and neither need to be set in
	which case the payload is neither copied nor freed by librdkafka.

	If `RD_KAFKA_MSG_F_COPY` flag is not set no data copying will be
	performed and librdkafka will hold on the payload pointer until
	the message has been delivered or fails.
	The delivery report callback will be called when librdkafka is done
	with the message to let the application regain ownership of the
	payload memory.
	The application must not free the payload in the delivery report
	callback if `RD_KAFKA_MSG_F_FREE is set`.
  * `payload`,`len` - the message payload
  * `key`,`keylen` - an optional message key which can be used for partitioning.
	  It will be passed to the topic partitioner callback, if any, and
	  will be attached to the message when sending to the broker.
  * `msg_opaque` - an optional application-provided per-message opaque pointer
	  that will be provided in the message delivery callback to let
	  the application reference a specific message.


`rd_kafka_produce()` is a non-blocking API, it will enqueue the message
on an internal queue and return immediately.
If the number of queued messages would exceed the `queue.buffering.max.messages`
configuration property then `rd_kafka_produce()` returns -1 and sets errno
to `ENOBUFS` and last_error to `RD_KAFKA_RESP_ERR__QUEUE_FULL`, thus
providing a backpressure mechanism.


`rd_kafka_producev()` provides an alternative produce API that does not
require a topic `rkt` object and also provides support for extended
message fields, such as timestamp and headers.


**Note**: See `examples/rdkafka_performance.c` for a producer implementation.


### Simple Consumer API (legacy)

NOTE: For the high-level KafkaConsumer interface see rd_kafka_subscribe (rdkafka.h) or KafkaConsumer (rdkafkacpp.h)

The consumer API is a bit more stateful than the producer API.
After creating `rd_kafka_t` with type `RD_KAFKA_CONSUMER` and
`rd_kafka_topic_t` instances the application must also start the consumer
for a given partition by calling `rd_kafka_consume_start()`.

`rd_kafka_consume_start()` arguments:

  * `rkt` - the topic to start consuming from, previously created with
    	  `rd_kafka_topic_new()`.
  * `partition` - partition to consume from.
  * `offset` - message offset to start consuming from. This may either be an
    	     absolute message offset or one of the two special offsets:
	     `RD_KAFKA_OFFSET_BEGINNING` to start consuming from the beginning
	     of the partition's queue (oldest message), or
	     `RD_KAFKA_OFFSET_END` to start consuming at the next message to be
	     produced to the partition, or
	     `RD_KAFKA_OFFSET_STORED` to use the offset store.

After a topic+partition consumer has been started librdkafka will attempt
to keep `queued.min.messages` messages in the local queue by repeatedly
fetching batches of messages from the broker. librdkafka will fetch all
consumed partitions for which that broker is a leader, through a single
request.

This local message queue is then served to the application through three
different consume APIs:

  * `rd_kafka_consume()` - consumes a single message
  * `rd_kafka_consume_batch()` - consumes one or more messages
  * `rd_kafka_consume_callback()` - consumes all messages in the local
    queue and calls a callback function for each one.

These three APIs are listed above the ascending order of performance,
`rd_kafka_consume()` being the slowest and `rd_kafka_consume_callback()` being
the fastest. The different consume variants are provided to cater for different
application needs.

A consumed message, as provided or returned by each of the consume functions,
is represented by the `rd_kafka_message_t` type.

`rd_kafka_message_t` members:

  * `err` - Error signaling back to the application. If this field is non-zero
    	  the `payload` field should be considered an error message and
	  `err` is an error code (`rd_kafka_resp_err_t`).
	  If `err` is zero then the message is a proper fetched message
	  and `payload` et.al contains message payload data.
  * `rkt`,`partition` - Topic and partition for this message or error.
  * `payload`,`len` - Message payload data or error message (err!=0).
  * `key`,`key_len` - Optional message key as specified by the producer
  * `offset` - Message offset

Both the `payload` and `key` memory, as well as the message as a whole, is
owned by librdkafka and must not be used after an `rd_kafka_message_destroy()`
call. librdkafka will share the same messageset receive buffer memory for all
message payloads of that messageset to avoid excessive copying which means
that if the application decides to hang on to a single `rd_kafka_message_t`
it will hinder the backing memory to be released for all other messages
from the same messageset.

When the application is done consuming messages from a topic+partition it
should call `rd_kafka_consume_stop()` to stop the consumer. This will also
purge any messages currently in the local queue.


**Note**: See `examples/rdkafka_performance.c` for a consumer implementation.


#### Offset management

Broker based offset management is available for broker version >= 0.9.0
in conjunction with using the high-level KafkaConsumer interface (see
rdkafka.h or rdkafkacpp.h)

Offset management is also available through a local offset file store, where the
offset is periodically written to a local file for each topic+partition
according to the following topic configuration properties:

  * `auto.commit.enable`
  * `auto.commit.interval.ms`
  * `offset.store.path`
  * `offset.store.sync.interval.ms`

There is currently no support for offset management with ZooKeeper.



#### Consumer groups

Broker based consumer groups (requires Apache Kafka broker >=0.9) are supported,
see KafkaConsumer in rdkafka.h or rdkafkacpp.h


### Topics

#### Topic auto creation

Topic auto creation is supported by librdkafka.
The broker needs to be configured with `auto.create.topics.enable=true`.



### Metadata

#### < 0.9.3
Previous to the 0.9.3 release librdkafka's metadata handling
was chatty and excessive, which usually isn't a problem in small
to medium-sized clusters, but in large clusters with a large amount
of librdkafka clients the metadata requests could hog broker CPU and bandwidth.

#### > 0.9.3

The remaining Metadata sections describe the current behaviour.

**Note:** "Known topics" in the following section means topics for
          locally created `rd_kafka_topic_t` objects.


#### Query reasons

There are four reasons to query metadata:

 * brokers - update/populate cluster broker list, so the client can
             find and connect to any new brokers added.

 * specific topic - find leader or partition count for specific topic

 * known topics - same, but for all locally known topics.

 * all topics - get topic names for consumer group wildcard subscription
                matching

The above list is sorted so that the sub-sequent entries contain the
information above, e.g., 'known topics' contains enough information to
also satisfy 'specific topic' and 'brokers'.


#### Caching strategy

The prevalent cache timeout is `metadata.max.age.ms`, any cached entry
will remain authoritative for this long or until a relevant broker error
is returned.


 * brokers - eternally cached, the broker list is additative.

 * topics - cached for `metadata.max.age.ms`



#### Fatal errors

The added guarantee of ordering and no duplicates also requires a way for
the client to fail gracefully when these guarantees can't be satisfied.
If an unresolvable error occurs a fatal error is triggered in one
or more of the follow ways depending on what APIs the application is utilizing:

 * C: the `error_cb` is triggered with error code `RD_KAFKA_RESP_ERR__FATAL`,
   the application should call `rd_kafka_fatal_error()` to retrieve the
   underlying fatal error code and error string.
 * C: an `RD_KAFKA_EVENT_ERROR` event is triggered and
   `rd_kafka_event_error_is_fatal()` returns true: the fatal error code
   and string are available through `rd_kafka_event_error()`, and `.._string()`.
 * C++: an `EVENT_ERROR` event is triggered and `event.fatal()` returns true:
   the fatal error code and string are available through `event.err()` and
   `event.str()`.

An application may call `rd_kafka_fatal_error()` at any time to check if
a fatal error has been raised.

If a fatal error has been raised, sub-sequent use of the following API calls
will fail:

 * `rd_kafka_produce()`
 * `rd_kafka_producev()`
 * `rd_kafka_produce_batch()`

The underlying fatal error code will be returned, depending on the error
reporting scheme for each of those APIs.


When a fatal error has occurred the application should call `rd_kafka_flush()`
to wait for all outstanding and queued messages to drain before terminating
the application.
`rd_kafka_purge(RD_KAFKA_PURGE_F_QUEUE)` is automatically called by the client
when a producer fatal error has occurred, messages in-flight are not purged
automatically to allow waiting for the proper acknowledgement from the broker.
The purged messages in queue will fail with error code set to
`RD_KAFKA_RESP_ERR__PURGE_QUEUE`.





## Appendix

### Test details

#### Test1: Produce to two brokers, two partitions, required.acks=2, 100 byte messages

Each broker is leader for one of the two partitions.
The random partitioner is used (default) and each broker and partition is
assigned approximately 250000 messages each.

**Command:**

    # examples/rdkafka_performance -P -t test2 -s 100 -c 500000 -m "_____________Test1:TwoBrokers:500kmsgs:100bytes" -S 1 -a 2
	....
    % 500000 messages and 50000000 bytes sent in 587ms: 851531 msgs/s and 85.15 Mb/s, 0 messages failed, no compression

**Result:**

Message transfer rate is approximately **850000 messages per second**,
**85 megabytes per second**.



#### Test2: Produce to one broker, one partition, required.acks=0, 100 byte messages

**Command:**

    # examples/rdkafka_performance -P -t test2 -s 100 -c 500000 -m "_____________Test2:OneBrokers:500kmsgs:100bytes" -S 1 -a 0 -p 1
	....
	% 500000 messages and 50000000 bytes sent in 698ms: 715994 msgs/s and 71.60 Mb/s, 0 messages failed, no compression

**Result:**

Message transfer rate is approximately **710000 messages per second**,
**71 megabytes per second**.



#### Test3: Produce to two brokers, two partitions, required.acks=2, 100 byte messages, snappy compression

**Command:**

	# examples/rdkafka_performance -P -t test2 -s 100 -c 500000 -m "_____________Test3:TwoBrokers:500kmsgs:100bytes:snappy" -S 1 -a 2 -z snappy
	....
	% 500000 messages and 50000000 bytes sent in 1672ms: 298915 msgs/s and 29.89 Mb/s, 0 messages failed, snappy compression

**Result:**

Message transfer rate is approximately **300000 messages per second**,
**30 megabytes per second**.


#### Test4: Produce to two brokers, two partitions, required.acks=2, 100 byte messages, gzip compression

**Command:**

	# examples/rdkafka_performance -P -t test2 -s 100 -c 500000 -m "_____________Test3:TwoBrokers:500kmsgs:100bytes:gzip" -S 1 -a 2 -z gzip
	....
	% 500000 messages and 50000000 bytes sent in 2111ms: 236812 msgs/s and 23.68 Mb/s, 0 messages failed, gzip compression

**Result:**

Message transfer rate is approximately **230000 messages per second**,
**23 megabytes per second**.

