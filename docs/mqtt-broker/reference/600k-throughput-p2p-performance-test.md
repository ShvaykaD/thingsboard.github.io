---
layout: docwithnav-mqtt-broker
title: TBMQ Scaling with One-to-one Messaging Communication Pattern
description: 600k msg/sec throughput One-to-one messaging utilizing persistent DEVICE clients as subscribers

tbmq-p2p-test-aws-instances:
  0:
    image: /images/mqtt-broker/reference/p2p-test/aws-instances.png
    title: 'AWS EC2 instances deployed'

tbmq-p2p-test-monitoring:
  0:
    image: /images/mqtt-broker/reference/p2p-test/tbmq-aws.png
    title: 'AWS EC2 TBMQ monitoring'
  1:
    image: /images/mqtt-broker/reference/p2p-test/tbmq-visual-vm-jmx.png
    title: 'Visual VM JMX TBMQ monitoring'
  2:
    image: /images/mqtt-broker/reference/p2p-test/redis-monitoring.png
    title: 'Redis Insight monitoring'
  3:
    image: /images/mqtt-broker/reference/p2p-test/kafka-monitoring.png
    title: 'Kafka Redpanda console monitoring'

---

{% include docs/mqtt-broker/reference/600k-throughput-p2p-performance-test.md %}