
* TOC
{:toc}

Point-to-point (P2P) communication is one of the core MQTT patterns, enabling devices to exchange messages directly in a one-to-one manner.
This pattern is especially relevant for IoT scenarios requiring reliable, targeted messaging. In our previous benchmarks, we focused on two additional core MQTT communication patterns: 
fan-in, leveraging [persistent APPLICATION clients](/docs/mqtt-broker/architecture/?utm_source=chatgpt.com#persistent-application-client),
and fan-out. Following these tests, we aimed to evaluate how well the broker performs with [persistent DEVICE clients](/docs/mqtt-broker/architecture/?utm_source=chatgpt.com#persistent-device-client) in a P2P communication scenario.
Persistent DEVICE clients are well-suited for P2P messaging because they use a shared Kafka topic, reducing the load on [Kafka](/docs/mqtt-broker/architecture/#kafka-topics), while leveraging [Redis](/docs/mqtt-broker/architecture/#redis) 
to ensure reliable message delivery even if the client is temporarily offline. This combination makes them efficient for this type of communication. 
The tests involved scaling the infrastructure while progressively increasing the load from 200,000 to 600,000 msg/sec, demonstrating TBMQ’s scalability and consistent performance.

![image](/images/mqtt-broker/reference/p2p-test/tbmq-perf-test-diagram.png)

### Test methodology

To assess the TBMQ's ability to handle point-to-point communication at scale, we conducted three tests to measure performance, efficiency, and latency, 
with a maximum throughput of 600,000 msg/sec. We deployed the performance test environment on an AWS EKS cluster and scaled it horizontally as the workload increased. 
This allowed us to evaluate how TBMQ handles growing demands while maintaining reliable performance.

> **Note:** Throughput represents the total number of messages per second, including both incoming and outgoing messages.

Each test ran for 10 minutes, using an equal number of publishers and subscribers.
Publishers sent 62-byte messages to unique topics `"europe/ua/kyiv/$number"` once per second, 
while subscribers subscribed to corresponding topics `"europe/ua/kyiv/$number/+"`. Here, `$number` served as the unique identifier for each pair of publisher and subscriber. 
Both publishers and subscribers operated with QoS 1, ensuring reliable message delivery. This configuration simulated one-to-one communication with precise topic mapping.

#### Test agent setup

The [test agent](/docs/mqtt-broker/reference/600k-throughput-p2p-performance-test/#how-to-repeat-the-600k-msgsec-throughput-test) 
was designed to simulate publishers and subscribers, ensuring a realistic evaluation of TBMQ's performance under growing message traffic. 
It consisted of a cluster of performance test nodes (runners) and an orchestrator pod to supervise their operation.

- Publisher Pods: For the 200k and 400k msg/sec tests, 5 publisher pods were deployed on a single EC2 instance.
  For the 600k msg/sec test, the number of publisher pods was increased to 10 due to port limitations within a single pod for simulating clients.
- Subscriber Pods: Similarly, for the 200k and 400k msg/sec tests, 5 subscriber pods were deployed on another EC2 instance.
  For the 600k msg/sec test, this was scaled to 10 subscriber pods for the same reason.
- Orchestrator Pod: The orchestrator pod was deployed on a separate EC2 instance, which also hosted additional components, including [Kafka Redpanda console](https://www.redpanda.com/redpanda-console-kafka-ui) and [Redis Insight](https://redis.io/docs/latest/operate/redisinsight/) pods, to facilitate monitoring and coordination.

This flexible configuration enabled the test agent to adapt to rising traffic demands while addressing infrastructure constraints, such as port limitations.

#### Infrastructure Overview

To provide a clear understanding of our test environment, this section details the hardware specifications of the services used and illustrates how the EKS cluster pods were distributed across the AWS EC2 instances.

**Hardware Specifications**

| Service Name              | **TBMQ**    | **AWS RDS (PostgreSQL)** | **Kafka** | **Redis** |
|---------------------------|-------------|--------------------------|-----------|-----------|
| Instance Type             | c7a.4xlarge | db.m6i.large             | c7a.large | r7a.large |
| vCPU                      | 16          | 2                        | 2         | 2         |
| Memory (GiB)              | 32          | 8                        | 4         | 16        |
| Storage (GiB)             | 20          | 20                       | 30        | 8         |
| Network bandwidth (Gibps) | 12.5        | 12.5                     | 12.5      | 12.5      |

> **Note:** For all tests, we used only Redis master nodes without replicas to reduce costs during load testing. 
  This configuration allowed us to focus on achieving the target throughput without over-provisioning resources.

**EC2 Instances and Pod Distribution**

The following images provide a detailed overview of our AWS EC2 instances and how the EKS cluster pods were distributed across them in our test setup:

{% include images-gallery.html imageCollection="tbmq-p2p-test-aws-infrastructure" %}

> **Note:** Provided images illustrate the infrastructure setup used in the final test, achieving a throughput of 600,000 msg/sec.

Instance scaling was adjusted during each test to match workload demands, as described in the next section.

### Performance tests

TBMQ's performance was evaluated through three testing phases at 200k, 400k, and 600k msg/sec throughput.
In each phase, we scaled the number of TBMQ brokers and Redis nodes to handle the corresponding workload. The test configurations and results are summarized in the table below.

| Throughput (msg/sec) | TBMQ Nodes | Redis Nodes | Publishers | Subscribers | Avg Msg Processing Latency | TBMQ CPU |
|----------------------|------------|-------------|------------|-------------|----------------------------|----------|
| 200k msg/sec         | 1          | 3           | 100k       | 100k        | ~32ms                      | ~88%     |
| 400k msg/sec         | 2          | 5           | 200k       | 200k        | ~50ms                      | ~90%     |
| 600k msg/sec         | 3          | 7           | 300k       | 300k        | ~57ms                      | ~90%     |

These tests demonstrated that TBMQ could efficiently handle one-to-one messaging workloads at varying levels of throughput while maintaining optimal latency and resource usage.

**Key takeaways from the tests include:**

 - Scalability: TBMQ exhibited linear scalability. By incrementally adding TBMQ nodes and Redis nodes, we maintained reliable performance as the workload doubled and tripled.
 - Efficient Resource Utilization: CPU utilization on TBMQ nodes remained consistently around ~90%, indicating that the system effectively leveraged available resources without overconsumption.
 - Latency Management: As throughput increased to 600k msg/sec, the average message processing latency grew. This growth reflects the robust lifecycle of a message in TBMQ, designed to prioritize reliability through multiple stages of persistence:
    - Initial Storage in `tbmq.msg.all` Kafka Topic: Each message published to the broker is first stored in this topic, ensuring all incoming messages are reliably logged and tracked.
    - Redirection to `tbmq.msg.persisted` Kafka Topic: Messages intended for DEVICE persistent clients are routed to this topic, where they are matched with client subscriptions for delivery.
    - Persistence in Redis: For DEVICE persistent clients, messages are persisted in Redis. This ensures high-performance retrieval and reliable delivery, even if the subscriber is temporarily offline.

   These stages safeguard messages at every step, providing a dependable mechanism for recovery in the event of a system failure. 
   While this comprehensive approach naturally increases latency compared to a simple message-forwarding system, it significantly enhances reliability—a critical priority for IoT scenarios requiring guaranteed message delivery.
   Importantly, even with this growth, the latency remains within efficient bounds balancing reliability and performance. We remain committed to optimizing performance without compromising reliability. 
   For more details on potential improvements, see the [Future optimizations](/docs/mqtt-broker/reference/600k-throughput-p2p-performance-test/#future-optimizations) section.

The following screenshots illustrate the insights obtained for the final test at 600k msg/sec throughput:

{% include images-gallery.html imageCollection="tbmq-p2p-test-monitoring" %}

These visualizations provide deeper insights into the system's behavior, corroborating the numerical performance metrics.
While the system maintained reliability and efficiency during the tests, the data indicates that the current setup is operating near
its limits and will require scaling to accommodate higher loads or additional performance demands in the future.

### How to repeat the 600k msg/sec throughput test

We recommend referring to our [installation guide](/docs/mqtt-broker/install/cluster/aws-cluster-setup/), which provides step-by-step instructions on how to deploy TBMQ on AWS.
In addition, you may explore the [branch](https://github.com/thingsboard/tbmq/tree/p2p-perf-test/k8s/aws#readme) containing the scripts and parameters used for running TBMQ during this performance test,
enabling you to gain deeper insights into our configuration. For the practical execution of performance tests, we offer a dedicated [performance testing tool](https://github.com/thingsboard/tb-mqtt-perf-tests/tree/p2p-perf-test)
capable of generating MQTT clients and simulating the desired message load. Especially for the P2P scenario testing, we improved our testing tool to have the ability to autogenerate the configuration for publishers and subscribers instead of loading it from a JSON configuration file.
Please refer to the [README.md](https://github.com/thingsboard/tb-mqtt-perf-tests/tree/p2p-perf-test/k8s#readme) for more details about P2P testing configuration.

### Migrating from Jedis to Lettuce: Overcoming a Key Testing Challenge

One of the most significant challenges during performance testing was overcoming the limitations of the [Jedis](https://github.com/redis/jedis) library, whose synchronous nature became a bottleneck in high-throughput scenarios.
With Jedis, each Redis command is sent and processed sequentially, meaning the system has to wait for each command to complete before issuing the next one.
This approach significantly limited Redis’s potential to handle concurrent operations and fully utilize available system resources.

To address this, we migrated to the [Lettuce](https://github.com/redis/lettuce) client, which leverages [Netty](https://github.com/netty/netty) under the hood for efficient asynchronous communication.
Unlike Jedis, Lettuce allows multiple commands to be sent in parallel without waiting for their completion, enabling non-blocking operations and better resource utilization.
This architecture made it possible to fully exploit Redis’s performance capabilities, especially under high message loads. Migration to Lettuce, however, was not trivial. 
It required rewriting a substantial portion of the codebase to transition from synchronous to asynchronous workflows. This included restructuring how Redis commands were issued and handled. 
Careful planning and rigorous testing ensured that these changes maintained system reliability and correctness.

### Future optimizations

Currently, we use [Lua scripts](https://redis.io/docs/latest/develop/interact/programmability/eval-intro/) in Redis to process messages for DEVICE persistent clients.
These scripts ensure atomic operations for saving, updating, and deleting messages, which is crucial for maintaining data consistency.
However, we execute one script per client to comply with Redis Cluster constraints, where all keys accessed in a script must reside in the same hash slot.

To optimize performance, we are considering adjusting the hashing mechanism for client IDs to intentionally group more clients into the same Redis hash slots.
By doing so, we aim to increase the likelihood of batching operations into a single script execution per hash slot, allowing the Lua scripts to handle multiple clients simultaneously.
This approach could reduce overhead and improve Redis efficiency, while still adhering to the cluster’s constraints.

### Conclusion

Across all three tests, TBMQ demonstrated linear scalability and efficient resource utilization. As the workload increased from 200,000 to 600,000 msg/sec, 
the system ensured reliable message delivery and maintained sufficiently low latency, remaining within acceptable bounds for point-to-point messaging scenarios.

These capabilities make TBMQ a dependable solution for IoT use cases requiring high-performance one-to-one communication with guaranteed delivery. 
While we plan to explore further optimizations to enhance performance, TBMQ has proven its suitability for high-throughput scenarios.

We value your feedback and encourage you to stay connected with our project on [GitHub](https://github.com/thingsboard/tbmq) and [Twitter](https://twitter.com/thingsboard).
