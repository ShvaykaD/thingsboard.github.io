---
layout: docwithnav-mqtt-broker
title: Scaling the Point-to-Point Messaging Communication Pattern
description: 600k msg/sec throughput One-to-one messaging utilizing persistent DEVICE clients as subscribers

tbmq-p2p-test-aws-infrastructure:
  0:
    image: /images/mqtt-broker/reference/p2p-test/aws-instances.png
    title: 'AWS EC2 instances deployed'
  1:
    image: /images/mqtt-broker/reference/p2p-test/eks-pods.png
    title: 'AWS EKS cluster pods running'  

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
    image: /images/mqtt-broker/reference/p2p-test/tbmq-total-connected-clients.png
    title: 'TBMQ 600K connected clients'
  4:
    image: /images/mqtt-broker/reference/p2p-test/tbmq-total-subscriptions.png
    title: 'TBMQ 300K total subscriptions'  

tbmq-p2p-test-redis-stats:
  0:
    image: /images/mqtt-broker/reference/p2p-test/redis-throughput.png
    title: 'Redis throughput stats'
  1:
    image: /images/mqtt-broker/reference/p2p-test/redis-total-keys.png
    title: 'Redis total stored keys'

---

{% include docs/mqtt-broker/reference/600k-throughput-p2p-performance-test.md %}