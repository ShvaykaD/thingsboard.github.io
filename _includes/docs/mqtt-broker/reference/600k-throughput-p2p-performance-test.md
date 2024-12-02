
* TOC
{:toc}

[Point-to-point](https://docs.aws.amazon.com/whitepapers/latest/designing-mqtt-topics-aws-iot-core/mqtt-communication-patterns.html#point-to-point) (P2P) communication is a core pattern in MQTT, 
enabling devices to exchange messages directly in a one-to-one manner. This pattern is especially relevant for IoT scenarios requiring reliable, targeted messaging.
In our previous benchmarks, such as [100 million connections test](/docs/mqtt-broker/reference/100m-connections-performance-test/) 
and [3 million msg/sec single-node throughput test](/docs/mqtt-broker/reference/3m-throughput-single-node-performance-test/),
we focused on fan-in and fan-out patterns, utilizing [persistent APPLICATION clients](/docs/mqtt-broker/architecture/?utm_source=chatgpt.com#persistent-application-client).
Following these tests, we aimed to evaluate how well the broker performs with [persistent DEVICE clients](/docs/mqtt-broker/architecture/?utm_source=chatgpt.com#persistent-device-client) in a P2P communication scenario.
Persistent DEVICE clients are well-suited for P2P messaging because they use a shared Kafka topic, reducing the load on [Kafka](/docs/mqtt-broker/architecture/#kafka-topics), while leveraging [Redis](/docs/mqtt-broker/architecture/#redis) to ensure reliable message delivery even if the client is temporarily offline.
This combination makes them efficient for this type of communication. Scaling from 200,000 to 600,000 msg/sec across multiple brokers, this test highlights TBMQ’s scalability and reliable performance under increasing workloads.

![image](/images/mqtt-broker/reference/single-node-test/tbmq-perf-test-diagram.png)

### Test methodology

To assess the ThingsBoard MQTT Broker's (TBMQ) ability to handle point-to-point communication at scale, we conducted tests to measure performance, efficiency, and latency under increasing workloads, 
with a maximum throughput of 600,000 msg/sec (incoming and outgoing). We deployed the performance test environment on an AWS EKS cluster and scaled it horizontally as the workload increased. 
This allowed us to evaluate how TBMQ handles growing demands while maintaining reliable performance.

Each test ran for 10 minutes, using an equal number of publishers and subscribers. Publishers sent 62-byte messages to unique topics `"europe/ua/kyiv/$number"`, 
while subscribers subscribed to corresponding topics `"europe/ua/kyiv/$number/+"`. Here, `$number` served as the unique identifier for each pair of publisher and subscriber. 
Both publishers and subscribers operated with QoS 1, ensuring reliable message delivery. This configuration simulated one-to-one communication with precise topic mapping.

#### Test agent setup

The test agent was designed to simulate publishers and subscribers, ensuring a realistic evaluation of TBMQ's performance under increasing workloads. It consisted of a cluster of performance test nodes (runners) and an orchestrator pod to supervise their operation.

- Publisher Pods: For the 200k and 400k msg/sec tests, 5 publisher pods were deployed on a single EC2 instance.
  For the 600k msg/sec test, the number of publisher pods was increased to 10 due to port limitations within a single pod for simulating clients.

- Subscriber Pods: Similarly, for the 200k and 400k msg/sec tests, 5 subscriber pods were deployed on another EC2 instance.
  For the 600k msg/sec test, this was scaled to 10 subscriber pods for the same reason.

- Orchestrator Pod: The orchestrator pod was deployed on a separate EC2 instance, which also hosted additional components, including [Kafka Redpanda console](https://www.redpanda.com/redpanda-console-kafka-ui) and [Redis Insight](https://redis.io/docs/latest/operate/redisinsight/) pods, to facilitate monitoring and coordination.

This flexible configuration allowed the test agent to adapt to increasing workload requirements while addressing test infrastructure constraints such as port limitations.

#### Hardware used

| Service Name              | **TBMQ**    | **AWS RDS (PostgreSQL)** | **Kafka** | **Redis**                    |
|---------------------------|-------------|--------------------------|-----------|------------------------------|
| Instance Type             | c7a.4xlarge | db.m6i.large             | c7a.large | r7a.large                    |
| vCPU                      | 16          | 2                        | 2         | 2                            |
| Memory (GiB)              | 32          | 8                        | 4         | 16                           |
| Storage (GiB)             | 10          | 10                       | 10        | 10                           |
| Network bandwidth (Gibps) | 12.5        | 12.5                     | 12.5      | 12.5                         |

Instance scaling was adjusted during each test to match workload demands, as described in the next section.

### Performance tests

#### 200K msg/sec throughput test

The first phase of testing established a baseline for TBMQ's performance under moderate workloads. We configured the test environment as follows:

 - 1 TBMQ broker
 - 6 Redis nodes (3 masters and 3 replicas)
 - Other components remained unchanged.

This test demonstrated that a single broker could efficiently handle a one-to-one messaging workload at 200,000 msg/sec with acceptable latency and resource usage.
Please check the test results below:

| Publishers | Subscribers | Msg/sec | Throughput (incoming and outgoing) | Msg processing latency(avg) | QoS | Payload  | TBMQ CPU |
|------------|-------------|---------|------------------------------------|-----------------------------|-----|----------|----------|
| 100K       | 100K        | 1       | 200K msg/sec                       | ~32ms                       | 1   | 62 bytes | ~88%     |

#### 400K msg/sec throughput test

In the second phase, we doubled the workload to 400,000 msg/sec, requiring a horizontally scaled cluster to sustain performance. The configuration was updated as follows:

 - 2 TBMQ brokers
 - 10 Redis nodes (5 masters and 5 replicas)
 - Other components remained unchanged.

The results showed that by adding a second broker and scaling Redis nodes, the system maintained reliable performance while handling twice the workload.
Please check the test results below:

| Publishers | Subscribers | Msg/sec | Throughput       | Msg processing latency(avg) | QoS | Payload  | TBMQ CPU |
|------------|-------------|---------|------------------|-----------------------------|-----|----------|----------|
| **200K**   | **200K**    | 1       | **400K msg/sec** | **~50ms**                   | 1   | 62 bytes | **~90%** |

#### 600K msg/sec throughput test

The final phase scaled the system to a workload of 600,000 messages per second, demonstrating the broker's ability to handle significant throughput while maintaining reliable performance.
For this test, we decided to remove Redis replicas from the setup, opting for a master-only configuration to reduce costs during the load tests. 
This change allowed us to focus on achieving the target throughput without over-provisioning resources.

- 3 TBMQ brokers
- 7 Redis nodes (Masters only)
- Other components remained unchanged.

At this workload, average processing latency increased slightly to **~57ms**,
showcasing TBMQ’s ability to handle a threefold increase in throughput with only a moderate impact on latency. Please check the test results below:

| Publishers | Subscribers | Msg/sec | Throughput       | Msg processing latency(avg) | QoS | Payload  | TBMQ CPU |
|------------|-------------|---------|------------------|-----------------------------|-----|----------|----------|
| **300K**   | **300K**    | 1       | **600K msg/sec** | **~57ms**                   | 1   | 62 bytes | ~90%     |

The following screenshots illustrate the insights obtained for the last test:

{% include images-gallery.html imageCollection="tbmq-p2p-test-monitoring" %}

These visualizations provide deeper insights into the system's behavior, corroborating the numerical performance metrics.
While the system maintained reliability and efficiency during the tests, the data indicates that the current setup is operating near
its limits and will require scaling to accommodate higher loads or additional performance demands in the future.

### How to repeat the 600K msg/sec throughput test

We recommend referring to our [installation guide](/docs/mqtt-broker/install/cluster/aws-cluster-setup/), which provides step-by-step instructions on how to deploy TBMQ on AWS.
In addition, you may explore the [branch](https://github.com/thingsboard/tbmq/tree/p2p-perf-test/k8s/aws#readme) containing the scripts and parameters used for running TBMQ during this performance test,
enabling you to gain deeper insights into our configuration. For the practical execution of performance tests, we offer a dedicated [performance testing tool](https://github.com/thingsboard/tb-mqtt-perf-tests/tree/p2p-perf-test)
capable of generating MQTT clients and simulating the desired message load. Especially for the P2P scenario testing we improved our testing tool
to have ability autogenerate the configuration for publishers and subscribers instead of loading it from json configuration file. Please refer to the [README.md](https://github.com/thingsboard/tb-mqtt-perf-tests/tree/p2p-perf-test/k8s#readme) for more details about P2P testing configuration.

### Migrating from Jedis to Lettuce: Overcoming a Key Testing Challenge

One of the most significant challenges during performance testing was overcoming the limitations of the [Jedis](https://github.com/redis/jedis) library, whose synchronous nature became a bottleneck in high-throughput scenarios.
With Jedis, each Redis command is sent and processed sequentially, meaning the system had to wait for each command to complete before issuing the next one.
This approach significantly limited Redis’s potential to handle concurrent operations and fully utilize available system resources.

To address this, we migrated to the [Lettuce](https://github.com/redis/lettuce) client, which leverages [Netty](https://github.com/netty/netty) under the hood for efficient asynchronous communication.
Unlike Jedis, Lettuce allows multiple commands to be sent in parallel without waiting for their completion, enabling non-blocking operations and better resource utilization.
This architecture made it possible to fully exploit Redis’s performance capabilities, especially under high message loads.

The migration to Lettuce, however, was not trivial. It required rewriting a substantial portion of the codebase to transition from synchronous to asynchronous workflows.
This included restructuring how Redis commands were issued and handled. Careful planning and rigorous testing ensured that these changes maintained system reliability and correctness.

### Conclusion

Across all three tests, TBMQ demonstrated linear scalability and efficient resource utilization. 
As the workload increased from 200,000 to 600,000 msg/sec, latency grew only modestly, from ~32ms to ~57ms, remaining within acceptable bounds. 
This predictable behavior highlights TBMQ’s robustness and suitability for point-to-point messaging scenarios, even at high throughput levels.

While these tests focused on existing capabilities, we have already identified several theoretical optimizations that can be implemented in future releases. 
These enhancements aim to further reduce latency and improve overall system performance, paving the way for even better results in upcoming iterations.

The system’s ability to scale horizontally while maintaining reliability and efficiency makes it a valuable solution for IoT use cases that demand high-performance, one-to-one communication.