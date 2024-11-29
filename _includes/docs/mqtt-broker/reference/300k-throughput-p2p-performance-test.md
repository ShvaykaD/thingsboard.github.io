
* TOC
{:toc}

[Point-to-point](https://docs.aws.amazon.com/whitepapers/latest/designing-mqtt-topics-aws-iot-core/mqtt-communication-patterns.html#point-to-point) (P2P) communication is a core pattern in MQTT, 
enabling devices to exchange messages directly in a one-to-one manner. This pattern is especially relevant for IoT scenarios requiring reliable, targeted messaging.
In our previous tests, we benchmarked fan-in and fan-out patterns in our [100 million connections test](/docs/mqtt-broker/reference/100m-connections-performance-test/) 
and [3 million msg/sec single-node throughput test](/docs/mqtt-broker/reference/3m-throughput-single-node-performance-test/),
both of which utilized [persistent APPLICATION clients](/docs/mqtt-broker/architecture/?utm_source=chatgpt.com#persistent-application-client).
Following these tests, we aimed to evaluate how well the broker performs with [persistent DEVICE clients](/docs/mqtt-broker/architecture/?utm_source=chatgpt.com#persistent-device-client) in a P2P communication scenario.
Persistent DEVICE clients are well-suited for P2P messaging because they use a shared Kafka topic, reducing the load on Kafka, while leveraging Redis to ensure reliable message delivery even if the client is temporarily offline.
This combination makes them efficient and effective for this type of communication. Scaling from 100,000 to 300,000 msg/sec across multiple brokers, this test highlights TBMQ’s scalability and reliable performance under increasing workloads.

![image](/images/mqtt-broker/reference/single-node-test/tbmq-perf-test-diagram.png)

### Test methodology

To assess the ThingsBoard MQTT Broker's (TBMQ) ability to handle point-to-point communication at scale, we conducted tests to measure performance, efficiency, and latency under increasing workloads, with a maximum throughput of 300,000 msg/sec.
We deployed the performance test environment on an AWS EKS cluster and scaled it horizontally as the workload increased. This allowed us to evaluate how TBMQ handles growing demands while maintaining reliable performance.
The following sections detail the configurations and results of the tests across different workload phases.

#### 100K Msg/Sec Test

The first phase of testing established a baseline for TBMQ's performance under moderate workloads. We configured the test environment as follows:

##### Cluster Setup
  
  - 1 TBMQ Nodes: c7a.4xlarge
  - 3 Kafka Nodes: c7a.large
  - 6 Redis Nodes: r7a.large (3 masters and 3 replicas)
  - 1 RDS Instance: TODO set instance type.

##### Results

  - Throughput: 100,000 msg/sec
  - Latency: ~32 ms
  - Broker CPU Usage: ~88%

This test demonstrated that a single broker could efficiently handle a one-to-one messaging workload at 100,000 msg/sec with acceptable latency and resource usage.

#### 200K Msg/Sec Test

In the second phase, we doubled the workload to 200,000 msg/sec, requiring a horizontally scaled cluster to sustain performance. The configuration was updated as follows:

##### Cluster Setup

- 2 TBMQ Nodes: c7a.4xlarge
- 3 Kafka Nodes: c7a.large
- 10 Redis Nodes: r7a.large (5 masters and 5 replicas)
- 1 RDS Instance: TODO set instance type.

##### Results

- Throughput: 200,000 msg/sec
- Latency: ~50 ms
- Broker CPU Usage: ~90%

The results showed that by adding a second broker and scaling Redis nodes, the system maintained reliable performance while handling twice the workload.

#### 300K Msg/Sec Test

The final phase scaled the system to a workload of 300,000 messages per second, demonstrating the broker's ability to handle significant throughput while maintaining reliable performance.

##### Cluster Setup

- 3 TBMQ Nodes: c7a.4xlarge
- 3 Kafka Nodes: c7a.large
- 7 Redis Nodes: r7a.large (masters only)
- 1 RDS Instance: TODO set instance type.

##### Results

- Throughput: 300,000 msg/sec
- Latency: ~57 ms
- Broker nodes CPU Usage: ~90%

This test highlighted TBMQ's ability to handle high-throughput workloads in a point-to-point messaging scenario with persistent DEVICE clients. 
Despite the significant load, the broker maintained reliable performance and acceptable latency.

### Test Agent Configuration
The test agent was designed to simulate publishers and subscribers, ensuring a realistic evaluation of TBMQ's performance under increasing workloads. It consisted of a cluster of performance test nodes (runners) and an orchestrator pod to supervise their operation.

 - Publisher Pods: For the 100k and 200k msg/sec tests, 5 publisher pods were deployed on a single EC2 instance. 
   For the 300k msg/sec test, the number of publisher pods was increased to 10 due to port limitations within a single pod for simulating clients.
 - Subscriber Pods: Similarly, for the 100k and 200k msg/sec tests, 5 subscriber pods were deployed on another EC2 instance. 
   For the 300k msg/sec test, this was scaled to 10 subscriber pods for the same reason.
 - Orchestrator Pod: The orchestrator pod was deployed on a separate EC2 instance, which also hosted additional components, including Kafka Redpanda and Redis Insights pods, to facilitate monitoring and coordination.

This flexible configuration allowed the test agent to adapt to the increasing workload requirements while addressing infrastructure constraints such as port limitations.