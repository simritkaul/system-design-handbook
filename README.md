# System Design Handbook

This repository is a structured learning path for understanding modern system design, storage systems, and distributed architecture.

## Repository structure

- `System_Design_Handbook/01_Storage/` — storage fundamentals and database categories
  - `01_Storage_Fundamentals.md`
  - `02_Relational_Databases.md`
  - `03_No_SQL_Databases.md`
  - `04_Key_Value_Databases.md`
  - `05_Document_Databases.md`
  - `06_Wide_Column_Databases.md`
  - `07_Graph_Databases.md`
  - `08_Object_Storage.md`
  - `09_Search_Databases.md`
  - `10_Time_Series_Databases.md`
- `System_Design_Handbook/02_Distributed_Systems/` — distributed systems topics and architecture patterns
  - `01_Replication.md`
  - `02_Sharding.md`
  - `03_Consistent_Hashing.md`
  - `04_CAP_Theorem.md`
  - `05_Consistency_Models.md`
- `System_Design_Handbook/03_Caching/` — caching strategies and optimization
  - `01_Introduction_To_Caching.md`
  - `02_Cache_Patterns.md`
  - `03_Cache_Eviction_Policies.md`
  - `04_Cache_Invalidation_And_Failure.md`
  - `05_Distributed_Caching.md`
  - `06_Browser_Reverse_Proxy_Cache_And_CDN.md`
- `System_Design_Handbook/04_Traffic_Management_Request_Routing/` — routing and request handling
  - `01_Domain_Name_System.md`
  - `02_Load_Balancers.md`
  - `03_Reverse_Proxy.md`
  - `04_API_Gateway.md`
  - `05_Service_Discovery.md`
- `System_Design_Handbook/05_Networking_Foundations/` — networking concepts and protocols
  - `01_Networking_Fundamentals.md`
  - `02_OSI_vs_TCPIP_Model.md`
  - `03_TCP_vs_UDP.md`
  - `04_IP_Address_Ports_And_Sockets.md`
  - `05_TLS_SSL_(HTTPS).md`
  - `06_HTTP_Request_Lifecycle.md`
  - `07_HTTP_Keep-Alive.md`
  - `08_Connection_Pooling.md`
- `System_Design_Handbook/06_Communication_Protocols/` — communication standards and APIs
  - `01_HTTP.md`
  - `02_REST.md`
  - `03_GraphQL.md`
  - `04_Websockets.md`
  - `05_Long_Polling.md`
  - `06_Server_Sent_Events.md`
  - `07_gRPC.md`
  - `08_HTTP2.md`
  - `09_HTTP3.md`
- `System_Design_Handbook/07_Messaging_Event_Driven_Systems/` — asynchronous communication and event handling
  - `01_Why_Async_Communication.md`
  - `02_Message_Queues.md`
  - `03_Publish_Subscribe.md`
  - `04_Event_Streaming.md`
  - `05_Delivery_Guarantees.md`
  - `06_Idempotency.md`
  - `07_Retry.md`
  - `08_Dead_Letter_Queues.md`
  - `09_Message_Ordering.md`
- `System_Design_Handbook/08_Reliability_Distributed_Architecture_Patterns/` — reliability and resilience patterns
  - `01_Timeout.md`
  - `02_Retry_Exponential_Backoff.md`
  - `03_Circuit_Breaker.md`
  - `04_Bulkhead.md`
  - `05_Rate_Limiting.md`
  - `06_Distributed_Locking.md`
  - `07_Leader_Election.md`
  - `08_Saga_Pattern.md`
  - `09_2PC_vs_Saga.md`
  - `10_CQRS.md`
  - `11_Event_Sourcing.md`
  - `12_Outbox_Pattern.md`

## What this repo covers

This handbook provides a comprehensive foundation for understanding modern system design:

- **Storage & Databases**: Models, trade-offs, and characteristics of relational, NoSQL, search, object, and time-series systems
- **Distributed Systems**: Replication, sharding, consistency models, and consensus algorithms
- **Caching**: Strategies, patterns, eviction policies, and distributed caching at scale
- **Traffic Management**: Load balancing, DNS, reverse proxies, API gateways, and service discovery
- **Networking**: Fundamentals, OSI/TCP-IP models, TCP/UDP, TLS/SSL, and HTTP lifecycle
- **Communication Protocols**: REST, GraphQL, WebSockets, gRPC, and HTTP/2/3
- **Asynchronous Systems**: Message queues, pub/sub, event streaming, delivery guarantees, and message ordering
- **Reliability Patterns**: Timeouts, retries, circuit breakers, rate limiting, distributed locking, sagas, CQRS, and event sourcing

## Suggested learning order

1. **Networking Foundations** — understand the underlying communication layer
2. **Storage & Databases** — learn storage fundamentals and compare database families
3. **Distributed Systems** — explore scalability, replication, and consistency
4. **Caching** — optimize performance through caching strategies
5. **Traffic Management** — implement routing and request handling
6. **Communication Protocols** — understand API design and synchronous/asynchronous patterns
7. **Messaging & Event-Driven Systems** — design asynchronous, scalable systems
8. **Reliability Patterns** — add resilience and fault tolerance to distributed systems

## Notes

This repository is intended as a study and reference guide for system design interviews, architecture exploration, and technical learning.
