# RSKafka_wasi

This crate aims to be a minimal Kafka implementation for simple workloads that wish to use Kafka as a distributed
write-ahead log. This is a fork from the original RSKafka with support for WebAssembly compilation target. 
That allows Kafka apps to run inside the WasmEdge Runtime as a lightweight and secure alternative to natively compiled apps in Linux container.

It is **not** a general-purpose Kafka implementation, instead it is heavily optimised for simplicity, both in terms of
implementation and its emergent operational characteristics. In particular, it aims to meet the needs
of [IOx].

This crate has:

* No support for offset tracking, consumer groups, transactions, etc...
* No built-in buffering, aggregation, linger timeouts, etc...
* Independent write streams per partition

It will be a good fit for workloads that:

* Perform offset tracking independently of Kafka
* Read/Write reasonably sized payloads per-partition
* Have a low number of high-throughput partitions [^1]


## Usage

```rust,no_run
# async fn test() {
use rskafka::{
    client::{
        ClientBuilder,
        partition::{Compression, UnknownTopicHandling},
    },
    record::Record,
};
use chrono::{TimeZone, Utc};
use std::collections::BTreeMap;

// setup client
let connection = "localhost:9093".to_owned();
let client = ClientBuilder::new(vec![connection]).build().await.unwrap();

// create a topic
let topic = "my_topic";
let controller_client = client.controller_client().unwrap();
controller_client.create_topic(
    topic,
    2,      // partitions
    1,      // replication factor
    5_000,  // timeout (ms)
).await.unwrap();

// get a partition-bound client
let partition_client = client
    .partition_client(
        topic.to_owned(),
        0,  // partition
        UnknownTopicHandling::Retry,
     )
     .await
    .unwrap();

// produce some data
let record = Record {
    key: None,
    value: Some(b"hello kafka".to_vec()),
    headers: BTreeMap::from([
        ("foo".to_owned(), b"bar".to_vec()),
    ]),
    timestamp: Utc.timestamp_millis(42),
};
partition_client.produce(vec![record], Compression::default()).await.unwrap();

// consume data
let (records, high_watermark) = partition_client
    .fetch_records(
        0,  // offset
        1..1_000_000,  // min..max bytes
        1_000,  // max wait time
    )
   .await
   .unwrap();
# }
```

For more advanced production and consumption, see [`crate::client::producer`] and [`crate::client::consumer`].


## Features

- **`compression-gzip` (default):** Support compression and decompression of messages using [gzip].
- **`compression-lz4` (default):** Support compression and decompression of messages using [LZ4].
- **`compression-snappy` (default):** Support compression and decompression of messages using [Snappy].
- **`compression-zstd` (default):** Support compression and decompression of messages using [zstd].
- **`full`:** Includes all stable features (`compression-gzip`, `compression-lz4`, `compression-snappy`,
  `compression-zstd`, `transport-socks5`, `transport-tls`).
- **`transport-socks5`:** Allow transport via SOCKS5 proxy.

## Testing

### Redpanda

To run integration tests against [Redpanda], run:

```console
$ docker-compose -f docker-compose-redpanda.yml up
```

in one session, and then run:

```console
$ TEST_INTEGRATION=1 TEST_BROKER_IMPL=redpanda KAFKA_CONNECT=0.0.0.0:9011 cargo test
```

in another session.

### Apache Kafka

To run integration tests against [Apache Kafka], run:

```console
$ docker-compose -f docker-compose-kafka.yml up
```

in one session, and then run:

```console
$ TEST_INTEGRATION=1 TEST_BROKER_IMPL=kafka KAFKA_CONNECT=localhost:9011 cargo test
```

in another session. Note that Apache Kafka supports a different set of features then redpanda, so we pass other
environment variables.

## License

Licensed under either of these:

 * Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or <https://www.apache.org/licenses/LICENSE-2.0>)
 * MIT License ([LICENSE-MIT](LICENSE-MIT) or <https://opensource.org/licenses/MIT>)

### Contributing

Unless you explicitly state otherwise, any contribution you intentionally submit for inclusion in the work, as defined
in the Apache-2.0 license, shall be dual-licensed as above, without any additional terms or conditions.


[^1]: Kafka's design makes it hard for any client to support the converse, as ultimately each partition is an
independent write stream within the broker. However, this crate makes no attempt to mitigate per-partition overheads
e.g. by batching writes to multiple partitions in a single ProduceRequest


[Apache Kafka]: https://kafka.apache.org/
[cargo-criterion]: https://github.com/bheisler/cargo-criterion
[cargo-fuzz]: https://github.com/rust-fuzz/cargo-fuzz
[cargo-with]: https://github.com/cbourjau/cargo-with
[gzip]: https://en.wikipedia.org/wiki/Gzip
[IOx]: https://github.com/influxdata/influxdb_iox/
[LLDB]: https://lldb.llvm.org/
[LZ4]: https://lz4.github.io/lz4/
[perf]: https://perf.wiki.kernel.org/index.php/Main_Page
[Redpanda]: https://vectorized.io/redpanda
[rustls]: https://github.com/rustls/rustls
[Snappy]: https://github.com/google/snappy
[zstd]: https://github.com/facebook/zstd
