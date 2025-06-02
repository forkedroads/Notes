
| **Configuration**         | **Description** |
|--------------------------|-----------------|
| **acks (2:37)**           | Determines the durability of data. Setting it to `0` means no guarantee of storage, `1` ensures the lead broker has received the data, and `"all"` or `"-1"` guarantees the data has been written to the lead broker and all in-sync replicas. |
| **batch.size (3:14)**     | Target size for a single produced request. Increasing this can reduce network calls by combining several events into one request, increasing throughput but potentially increasing latency. |
| **linger.ms (3:30)**      | Determines how long a produced request will wait for the batch to reach the target batch size. Adjusting this in tandem with `batch.size` can optimize throughput. |
| **compression.type (3:41)** | Specifies the compression type to reduce the number of bytes sent in the producer request, which can improve throughput and latency. Options include `gzip`, `Snappy`, `lz4`, or `zstd`. Experimentation is recommended to determine the best option. |
| **retries (3:55)**        | Determines how many times the producer will retry sending a request before giving up. |
| **delivery.timeout.ms (4:07)** | Limits the total time of a produced request, including retries. If a request has not been successful by this limit, it will fail. |
| **transactional.id (4:17)** | Used to ensure the integrity of transactions across multiple producer sessions and is required for a producer to participate in a transaction. |
| **enable.idempotence (4:30)** | When set to `True` (default), the producer ensures that exactly one copy of each message is written to the stream by adding a sequence number to each produced message. This allows the broker to recognize and handle duplicate messages, preventing data loss or duplication. |
