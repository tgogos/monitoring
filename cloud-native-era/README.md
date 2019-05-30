# Monitoring in the time of Cloud Native

Tags & Buzzwords included in this document:  `LOGS` ,  `METRICS` ,   `TRACES` ,  `EVENTS` ,  `MONITORING` ,  `OBSERVABILITY` ,  `DevOps` ,  `ELASTIC SEARCH` ,  `ELK` ,  `KIBANA` ,   `KAFKA` ,  `STREAM-PROCESSING` ,  `SERVICE-MESH` 

The scope of this document is to help us understand more about the above [**👆**](https://emojipedia.org/white-up-pointing-backhand-index/) and prevent us from doing **Cargo Cult Engineering**:

![](https://paper-attachments.dropbox.com/s_1E611437D0329ADE41805C7EFCB29C0B037E418309021C6DBC2F23478C6155C6_1559127836789_PiNFw0F.png)


While reading, try to consider:
— the **strengths** and **weaknesses** of each category of tools 
— the **problems** they solve
— the **tradeoffs** they make
— their **ease of adoption**/integration into an existing infrastructure


## Doc structure
- [The  problem](https://paper.dropbox.com/doc/Monitoring-in-the-time-of-Cloud-Native--AeDxr2MEqbVHcgQwCabpWowTAg-O06bGQHG9kyXvvZtoLFFZ#:uid=262679896511911311533669&h2=The-problem%E2%80%A6)
- [Basics: metrics, logs, traces](https://paper.dropbox.com/doc/Monitoring-in-the-time-of-Cloud-Native--AeDxr2MEqbVHcgQwCabpWowTAg-O06bGQHG9kyXvvZtoLFFZ#:uid=124809189654263080686925&h2=Basics:-metrics%2C-logs%2C-traces)
- [Things to consider](https://paper.dropbox.com/doc/Monitoring-in-the-time-of-Cloud-Native--AeDxr2MEqbVHcgQwCabpWowTAg-O06bGQHG9kyXvvZtoLFFZ#:uid=117266842196421618091969&h2=Things-to-consider)
- [RANDOM things to be aware of](https://paper.dropbox.com/doc/Monitoring-in-the-time-of-Cloud-Native--AeDxr2MEqbVHcgQwCabpWowTAg-O06bGQHG9kyXvvZtoLFFZ#:uid=973051127132419617423874&h2=RANDOM-things-to-be-aware-of)




![A quick overview…](https://paper-attachments.dropbox.com/s_1E611437D0329ADE41805C7EFCB29C0B037E418309021C6DBC2F23478C6155C6_1559129084608_2019-05-29_14-24-20.gif)





# The problem…

Containers, Kubernetes, microservices, service meshes, immutable infrastructure and serverless are all incredibly *promising* ideas which fundamentally change the way we run software. As more and more organizations move toward these paradigms, the systems we build have become more distributed and in the case of containerization, more ephemeral.

The *nature* *of* *failure* is changing, the way our systems behave (or *misbehave*) as a whole is changing, the requirements these systems need to meet are changing, the guarantees these systems need to provide are changing.


![](https://paper-attachments.dropbox.com/s_1E611437D0329ADE41805C7EFCB29C0B037E418309021C6DBC2F23478C6155C6_1559123384107_system.png)








# Basics: metrics, logs, traces


## Metrics

Metrics are just *numbers* measured over intervals of time, and numbers are optimized for storage, processing, compression and retrieval. As such, metrics enable longer retention of data as well as easier querying, which can in turn be used to build dashboards to reflect historical trends. Additionally, metrics better allow for gradual reduction of data resolution over time, so that after a certain period of time data can be aggregated into daily or weekly frequency.

Modern monitoring systems like Prometheus represent every time series using a metric *name* as well as additional key-value pairs called *labels*. 

This allows for a high degree of *dimensionality* in the data model. A metric is identified using both the metric name and the labels. The actual data stored in the time-series is called a *sample* and it consists of two components — a *float64* value and a millisecond precision timestamp.


![The anatomy of a Prometheus metric](https://paper-attachments.dropbox.com/s_1E611437D0329ADE41805C7EFCB29C0B037E418309021C6DBC2F23478C6155C6_1559122896390_prometheus_metric.png)



## Logs

A log is an immutable record of discrete *events* that happened over time. Some people take the view that *events* are distinct compared to *logs*, but for all intents and purposes they can be used interchangeably.

Event logs in general come in three forms:
**1.** **Plaintext** — A log record might take the form of free-form text. This is also the most common format of logs.
**2.** **Structured** — Much evangelized and advocated for in recent days. Typically this is logs emitted in the JSON format.
**3.** **Binary** — think logs in the Protobuf format, MySQL *binlogs* used for replication and point-in-time recovery, systemd *journal* logs, the `pflog`format used by the BSD firewall `pf` which often serves as a frontend to `tcpdump`.

Logs shine when it comes to providing valuable insight along with *ample context* into the long tail that averages and percentiles don’t surface. Coming back to the example we saw above, let us assume that all of these various services also emit logs at varying degrees of granularity. Some services might emit more log information per request than others. Looking at logs alone, our data landscape might look like the following:

![](https://paper-attachments.dropbox.com/s_1E611437D0329ADE41805C7EFCB29C0B037E418309021C6DBC2F23478C6155C6_1559124305714_logs.png)


Recording anything and everything that might be of interest to us becomes incredibly useful when we are searching at a very fine level of granularity, but simply looking at this mass of data, it’s impossible to infer at a glance what the request lifecycle was or even which systems the request traversed through or even the overall health of any particular system. Sure, the data might be *rich* but without further processing, it’s pretty *impenetrable*. There is, quite frankly, an *endless* amount of data points we can collect and an *endless* number of questions we can answer, from the most trivial to the most difficult.



## Traces

A trace is a representation of a series of causally-related distributed events that encode the end-to-end request flow through a distributed system. A single trace can provide visibility into both the *path* traversed by a request as well as the *structure* of a request. The path of a request allows us to understand the services involved in the servicing of a request, and the *structure* of a trace helps one understand the junctures and effects of asynchrony in the execution of a request.


![](https://paper-attachments.dropbox.com/s_1E611437D0329ADE41805C7EFCB29C0B037E418309021C6DBC2F23478C6155C6_1559124250115_traces.png)


Albeit discussions around tracing pivot around their utility in a microservices environment, I think it’s fair to suggest that any sufficiently complex application that interacts with — or rather, *contends* for — resources such as the network or disk in a non-trivial manner can benefit from the benefits tracing can provide.

The way this is achieved is by adding instrumentation to specific points in code. When a request begins, it’s assigned a globally unique ID, which is then propagated throughout the request path, so that each point of instrumentation is able to insert or enrich metadata before passing the ID around to the next hop in the meandering flow of a request. When the execution flow reaches the instrumented point at one of these services, a record is emitted along with metadata. These records are usually asynchronously logged to disk before being submitted out of band to a collector, which then can reconstruct the flow of execution based on different records emitted by different parts of the system.

Collecting this information and reconstructing the flow of execution while preserving causality for retrospective analysis and troubleshooting enables one to understand the lifecycle of a request better. Most importantly, having an understanding of the entire request lifecycle makes it possible to *debug requests spanning multiple services* to pinpoint the source of increased response time or resource utilization. As such, traces largely help one understand the *which* and sometimes even the *why —* like *which* component of a system is even touched during the lifecycle of a request and is slowing the response?




# Things to consider

— Ease of generation/instrumentation
— Ease of processing 
— Ease of querying/searching
— Quality of information
— Cost Effectiveness


## Logs
- Logs are, by far, the easiest to generate since there is no initial processing involved
- Logs are also easy to instrument since adding a log line is quite as trivial as adding a print statement

**The utility of logs, unfortunately, ends right there. Everything else about logs is only going to be painful**


- Most performant logging libraries allocate very little, if any, and are extremely fast. However, the default logging libraries of many languages and frameworks are not the cream of the crop, which means the application as a whole becomes susceptible to suboptimal performance due to the overhead of logging
- raw logs are almost always normalized, filtered and processed by a tool like **Logstash, fluentd, Scribe or Heka** before they’re persisted in a data store like **Elasticsearch** or **BigQuery**
- while **Elasticsearch** might be a fantastic search engine, there’s a real operational cost involved in running it. Even if your organization is staffed with a team of Operations engineers who are experts in operating **ELK**, there might be other drawbacks



## Metrics
- the *biggest* advantage of metrics based monitoring over logs is the fact that unlike log generation and storage, metrics transfer and storage has a constant overhead. Unlike logs, the cost of metrics doesn’t increase in lockstep with user traffic or any other system activity that could result in a sharp uptick in data
- Metrics, once collected, are also more malleable to mathematical, probabilistic and statistical transformations such as sampling, aggregation, summarization and correlation, which make it better suited to report the overall health of a system
- Metrics are also better suited to trigger *alerts*, since running queries against an in-memory time series database is far more efficient, not to mention more reliable, than running a query against a distributed system like ELK and then aggregating the results before deciding if an alert needs to be triggered


- The biggest drawback with both logs and metrics is that they are *system* scoped, making it hard to understand anything else other than what’s happening inside of a particular system
- With logs, without fancy joins, a single log line or metric doesn’t give much information about what happened to a request across all components of a system. Together and when used optimally, logs and metrics give us complete omniscience into a *silo*, but nothing more




## Traces
- Tracing is, by far, the hardest to retrofit into an existing infrastructure, owing to the fact that for tracing to be truly effective, every component in the path of a request needs to be modified to propagate tracing information
- The second problem with tracing instrumentation is that it’s not sufficient for developers to instrument *their* code. A large number of applications in the wild are built using open source frameworks or libraries which might require additional instrumentation. This becomes all the more challenging at places with polyglot architectures, since every language, framework and wire protocol with widely disparate concurrency patterns and guarantees need to cooperate





# RANDOM things to be aware of


## 1. Logging is a Stream Processing Problem

Data isn’t only ever used for application performance and debugging use cases. It also forms the source of all analytics data as well. This data is often of tremendous utility from a business intelligence perspective, and usually businesses are willing to pay for both the technology and the personnel required to make sense of this data in order to make better product decisions. 

For example:

| **Business perspective**                                                                         | **Debugging perspective**                                                                          |
| ------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------- |
| *Filter to outlier countries from where users viewed this article fewer than 100 times in total* | *Filter to outlier page loads that performed more than 100 database queries*                       |
|                                                                                                  | *show me only page loads from Indonesia that took more than 10 seconds to load*                    |
| *User A viewed Product X*                                                                        | *User A viewed Product X and the page took 0.5s to load*                                           |
|                                                                                                  | *User A viewed Product X whose image was not served from cache*                                    |
|                                                                                                  | *User A viewed Product X and read review Z for which the response time for the API call was 300ms* |


Marrying business information along with information about the *lifetime* of the request (timers, durations and so forth) makes it possible to repurpose analytics tooling for observability purposes.



## 2. Most analytics pipelines use Kafka as an event bus.

Sending enriched event data to Kafka allows one to search in real time over streams with KSQL, [a streaming SQL engine for Kafka](https://www.confluent.io/blog/ksql-open-source-streaming-sql-for-apache-kafka/) from the fine folks at Confluent.

Enriching business events that go into Kafka anyway with additional timing and other metadata required for observability use cases can be helpful when repurposing existing stream processing infrastructures. A further benefit this pattern provides is that this data can be expired from the Kafka log regularly. Most event data required for debugging purposes are only valuable for a relatively short period of time after the event has been generated, unlike any business centric information that normally would’ve been evaluated and persisted by an ETL job. Of course, this makes most sense when Kafka already is an integral part of an organization. Introducing Kafka into a stack purely for real time log analytics is a bit of an overkill, especially in non-JVM shops.



## 3. Metrics & label-space

With metrics, however, it’s important to be careful not to explode the label space. Labels should be so chosen so that it remains limited to a small set of attributes that can remain somewhat uniform. It also becomes important to resist the temptation to alert on *everything*. For alerting to be *effective,* it becomes salient to be able to identify a small set of hard failure modes of a system. Some believe that the ideal number of signals to be “monitored” is anywhere between 3–5, and definitely no more than 7–10.




## 4. Observability & Service Meshes

While historically tracing has been difficult to implement, the rise of service meshes make integrating tracing functionality almost effortless. Lyft famously got tracing support for all of their applications without changing a single line of code by adopting the service mesh pattern. Service meshes help with the DRYing of observability by implementing tracing and stats collections at the mesh level, which allows one to treat individual services as blackboxes but still get incredible observability onto the *mesh* as a whole. Even with the caveat that the applications forming the mesh need to be able to forward headers to the next hop in the mesh, this pattern is incredibly useful for retrofitting tracing into existing infrastructures with the least amount of code change.




## 5. Elastic Search & Kibana

While Elasticsearch might be a fantastic search engine, there’s a real operational cost involved in running it. Even if your organization is staffed with a team of Operations engineers who are experts in operating ELK, there might be other drawbacks. Case in point — one of my friends was telling me about how he would often see a sharp downward slope in the graphs in Kibana, not because traffic to the service was dropping but because ELK couldn’t keep up with the indexing of the sheer volume of data being thrown at it. Even if log ingestion processing isn’t an issue with ELK, no one I know of seems to have fully figured out how to use Kibana’s UI, let alone *enjoy* using it.





# Links
- *source: https://medium.com/@copyconstruct/monitoring-in-the-time-of-cloud-native-c87c7a5bfa3e*
- *hands-on demo for observability (events / tracing): https://www.honeycomb.io/play/*

