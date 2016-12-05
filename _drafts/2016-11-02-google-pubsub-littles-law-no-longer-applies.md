---
layout: post
title:  "Footnote: Kafka to Cloud Pub/Sub"
date:   2016-10-01 00:00:00
categories: golang data streams functional programming queueing theory
---

My last few projects have greatly benefited from using Google Pub/Sub. Nevertheless I thought I would denote some of the caveats for using .including performance benchmarks (they now have clients backed by gRPC). However, I thought it'd be helpful to keep some of the 

# <3 Cloud

Google Cloud Pub/Sub makes it easy to get up and running with a durable message bus that provides loose coupling for data senders and receivers. The fact that it 
doesn't require any cluster management (bye Ansible scripts and broker configs!) makes it a leightweight alternative to Kafka for some scenarios. 
It even automatically garbage collects topics that haven't been used in 30 days, pressuring developers to keep things manageable.

### Ordering is tough stuff

* intra-topic partitioning for semi-ordering of messages.

### We must always move forward 

* specify a starting point for new subscriptions or rewind checkpoints for existing

### Data Loss, & no more Little's Law

No log compaction and ability to store messages for all time.

At one of my last gigs, some engineers had been playing with Kafka before setting up and sort of checkpoint lag monitoring. They noticed that some of their service logging events were seemingly not getting delivered to downstream consumers. 
I remember taking a look, and noticing events were being produced much more quickly than consumed. This was easily explained via Litte's Law
One of the engineers at Jet.com once asked me why some of there events were not being delivered in Kafka. 
While teaching other engineers at Jet.com about Kafka, I used [Little's Law](https://en.wikipedia.org/wiki/Litte's_law), a theorem in Queueing theory, to explain the reason's
they could "fall behind" with standard configurations set.

In Google Pubsub, your data is persisted for 7 days, regardless of throughput. While their are tradeoffs to both approaches, "you lose your data after 7 days" is definitely easier to swallow at first.

