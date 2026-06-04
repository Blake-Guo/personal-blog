---
title: "3-Min Summary: How OpenAI Scaled PostgreSQL to Millions of QPS"
description: "A concise note on OpenAI's PostgreSQL scaling work: one primary, many replicas, and disciplined operations."
pubDate: 2026-06-03
heroImage: "../../assets/openai-postgresql-scaling-hero.png"
---

I read OpenAI's post, [Scaling PostgreSQL to power 800 million ChatGPT users](https://openai.com/index/scaling-postgresql/), and found it inspiring for a simple reason: it is a reminder that a familiar database can go much further than expected when the workload shape and operational constraints are understood clearly.

The headline is impressive: one PostgreSQL primary, nearly 50 read replicas, and millions of QPS for a product at ChatGPT scale. But the lesson is not that PostgreSQL can scale without limits. The more precise lesson is that PostgreSQL can go very far when reads are scaled out aggressively, writes are kept under control, and the primary is treated as the resource to protect.

My main takeaways:

- Offload read traffic to many replicas across regions to reduce primary load and reserve the primary mainly for write traffic.
- Run the primary in High-Availability (HA) mode with a hot standby so failover can happen quickly.
- The primary streams Write Ahead Log (WAL) data to every read replica. Today this works for nearly 50 replicas, while cascading replication is being tested for further scale.
- Use other databases to complement PostgreSQL for workloads that are naturally sharded or write-heavy.
- Constrain schema changes. A good lesson: always think carefully about schema design up front to avoid costly changes later.

The rest of the article reinforces familiar production lessons at a much larger scale: cache carefully, use connection pooling, isolate noisy workloads, rate-limit overload paths, and continuously review expensive queries.

The broader lesson is that PostgreSQL can go surprisingly far when the system is designed around its constraints: protect the primary, scale reads through replicas, reduce unnecessary writes, constrain schema changes, and use complementary systems for separate or independent workloads when needed.
