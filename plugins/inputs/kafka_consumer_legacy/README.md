# Kafka Consumer Legacy Input Plugin

## Deprecated in version 1.4. Please use [Kafka Consumer input plugin][]

The [Kafka](http://kafka.apache.org/) consumer plugin polls a specified Kafka
topic and adds messages to InfluxDB. The plugin assumes messages follow the
line protocol. [Consumer Group](http://godoc.org/github.com/wvanbergen/kafka/consumergroup)
is used to talk to the Kafka cluster so multiple instances of telegraf can read
from the same topic in parallel.

## Configuration

```toml @sample.conf
## DEPRECATED: The 'kafka_consumer_legacy' plugin is deprecated in version 1.4.0, use 'inputs.kafka_consumer' instead, NOTE: 'kafka_consumer' only supports Kafka v0.8+.
# Read metrics from Kafka topic(s)
[[inputs.kafka_consumer_legacy]]
  ## topic(s) to consume
  topics = ["telegraf"]

  ## an array of Zookeeper connection strings
  zookeeper_peers = ["localhost:2181"]

  ## Zookeeper Chroot
  zookeeper_chroot = ""

  ## the name of the consumer group
  consumer_group = "telegraf_metrics_consumers"

  ## Offset (must be either "oldest" or "newest")
  offset = "oldest"

  ## Data format to consume.
  ## Each data format has its own unique set of configuration options, read
  ## more about them here:
  ## https://github.com/influxdata/telegraf/blob/master/docs/DATA_FORMATS_INPUT.md
  data_format = "influx"

  ## Maximum length of a message to consume, in bytes (default 0/unlimited);
  ## larger messages are dropped
  max_message_len = 65536
```

## Testing

Running integration tests requires running Zookeeper & Kafka. See Makefile
for kafka container command.

[Kafka Consumer input plugin]: /plugins/inputs/kafka_consumer
