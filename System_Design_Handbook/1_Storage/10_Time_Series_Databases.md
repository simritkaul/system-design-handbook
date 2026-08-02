# Chapter 10 — Time-Series Databases

> **Goal:** Understand what time-series databases are, why they exist, what problems they solve, and when they are preferred over traditional databases.

---

# 1. The Problem

Imagine you're monitoring your servers.

Every second, every server reports:

```text
CPU Usage

Memory Usage

Disk Usage

Network Traffic
```

One second later

```text
CPU Usage

Memory Usage

Disk Usage

Network Traffic
```

Again.

Again.

Again.

Millions of times.

Notice something.

Every record has one thing in common.

**Time.**

---

# 2. Another Example

Imagine a smartwatch.

Every second it records

```text
Heart Rate

Steps

Calories
```

Tomorrow

Millions more readings.

After one year

Billions of records.

---

# 3. The Big Idea

A Time-Series Database is built for one kind of data:

> **Measurements that are continuously recorded over time.**

Every record has

```text
Timestamp

↓

Measurement

↓

Value
```

Time is the most important field.

---

# 4. What Is Time-Series Data?

Examples

```text
10:00 → CPU = 20%

10:01 → CPU = 24%

10:02 → CPU = 31%

10:03 → CPU = 28%
```

Unlike SQL,

you're usually not asking

> Find User 123.

You're asking

> Show CPU usage over the last hour.

---

# 5. Common Queries

Imagine monitoring a web server.

Typical questions are:

```text
Average CPU yesterday

Maximum memory today

Temperature over the last week

Disk usage every minute

Network traffic this month
```

Notice how almost every query includes a time range.

---

# 6. Why SQL Isn't Ideal

Could SQL store this?

Absolutely.

But after years of continuous measurements,

you may have trillions of rows.

Now ask

```text
Last 30 days

Grouped every minute

Average CPU
```

A general-purpose relational database wasn't specifically optimized for this workload.

Time-series databases are.

---

# 7. Optimizations

Time-series databases make assumptions.

They know

- Data arrives in time order.
- Most queries are based on time ranges.
- Old data is accessed less frequently.

Because of these assumptions, they can optimize storage, indexing, compression, and querying for time-based workloads.

---

# 8. Retention Policies

Suppose you collect metrics every second.

After

```text
5 years
```

Do you really need every single second forever?

Often,

No.

A Time-Series Database lets you define retention.

Example

```text
Keep raw data

30 days

↓

Delete automatically
```

Or

```text
Keep 1-second data

30 days

↓

Store hourly averages

5 years
```

This helps control storage costs.

---

# 9. Downsampling

Imagine this data.

```text
Every second

↓

CPU readings
```

After one year,

there are millions of points.

Instead,

store

```text
Hourly Average

Daily Average

Monthly Average
```

Older data becomes smaller while still preserving useful trends.

This process is called **downsampling**.

---

# 10. Where Time-Series Databases Shine

### Monitoring

CPU

Memory

Disk

---

### IoT

Temperature

Humidity

Pressure

---

### Finance

Stock prices

Crypto prices

---

### Smart Devices

Heart rate

Fitness trackers

---

### Manufacturing

Machine sensors

Factory metrics

---

### Application Metrics

Response time

Request count

Error rate

Latency

---

# 11. Where They Perform Poorly

Suppose you're building Amazon Orders.

Need

Customers

Products

Payments

Transactions

Relationships.

SQL is better.

---

Suppose you're building LinkedIn.

Need

Connections.

Relationships.

Graph databases are better.

---

# 12. Time-Series vs SQL

| SQL               | Time-Series                      |
| ----------------- | -------------------------------- |
| General-purpose   | Time-based workloads             |
| Tables            | Measurements over time           |
| Any query pattern | Optimized for time-range queries |
| Manual retention  | Built-in retention policies      |
| General indexing  | Time-optimized indexing          |

---

# 13. Time-Series vs Wide Column

This is the confusing one.

Both handle huge amounts of writes.

The difference is their goal.

### Wide Column

Optimized for

```text
Massive write throughput

Known query patterns

General distributed storage
```

Examples

Facebook inbox

Event storage

Activity feeds

---

### Time-Series

Optimized for

```text
Continuous measurements

Time-range queries

Retention

Downsampling
```

Examples

CPU monitoring

Stock prices

Weather

Server metrics

A Time-Series Database is specialized for one very common kind of data: measurements over time.

---

# 14. Popular Technologies

Examples include:

- InfluxDB
- TimescaleDB
- OpenTSDB
- Prometheus (for monitoring metrics)

We'll study them individually later.

---

# 15. Mental Model

Imagine a weather station.

Every minute it writes

```text
Time

Temperature

Humidity

Wind Speed
```

Nobody asks

> "Show me record #18293."

Instead they ask

> "Show the temperature over the last month."

Time is the organizing principle.

---

# 16. Tradeoffs

### Advantages

- Excellent for time-based queries
- High write throughput
- Efficient storage for sequential measurements
- Built-in retention policies
- Downsampling support
- Optimized aggregation over time windows

---

### Disadvantages

- Specialized workload
- Poor fit for relational data
- Limited joins
- Not intended as a general-purpose application database

---

# 17. Real-World Examples

### Prometheus

Kubernetes monitoring.

---

### Grafana Dashboards

Visualizing metrics from time-series databases.

---

### Smart Watches

Heart rate history.

---

### Stock Exchanges

Price history.

---

### Tesla

Vehicle telemetry.

---

### IoT Platforms

Sensor data.

---

# 18. Common Interview Questions

- What is time-series data?
- Why not use SQL for monitoring?
- What is downsampling?
- What are retention policies?
- Why are time-range queries important?
- How are time-series databases different from wide column databases?
- Why are they common in monitoring systems?

---

# 19. Connections

Imagine monitoring an e-commerce application.

```text
Application
      │
      ▼
Time-Series Database
      │
      ▼
Grafana Dashboard
```

Meanwhile

```text
Orders

↓

SQL
```

```text
Product Images

↓

Object Storage
```

```text
Search

↓

Search Database
```

Different systems solve different problems.

---

# Key Takeaways

- Time-series databases are specialized for **continuous measurements over time**.
- Every record revolves around a **timestamp**.
- They excel at **time-range queries**, **aggregations**, **retention**, and **downsampling**.
- They are commonly used for monitoring, IoT, finance, telemetry, and analytics.
- They complement general-purpose databases rather than replacing them.

---

# 🎉 Storage Phase Complete

I think this is a good place to stop and appreciate what we've built.

You now understand the **major storage models** that appear in system design:

| Storage Type    | Primary Question It Solves                                     |
| --------------- | -------------------------------------------------------------- |
| SQL             | How do I store structured, relational data safely?             |
| Key-Value       | How do I retrieve data instantly when I know the key?          |
| Document        | How do I store flexible, evolving objects?                     |
| Wide Column     | How do I store and write enormous amounts of distributed data? |
| Graph           | How do I efficiently navigate relationships?                   |
| Object Storage  | How do I store huge files like images and videos?              |
| Search Database | How do I search text quickly and intelligently?                |
| Time-Series     | How do I store and analyze measurements over time?             |

---

## I think this is where the handbook becomes really exciting.

Everything we've studied so far answers **"Where should the data live?"**

The next phase answers a different question:

> **"How do we make these systems scale, stay available, and work together?"**

That means we're leaving the world of storage categories and entering the world of **distributed systems**.

If I were designing this as a university course, the next sequence would be:

1. **Replication** (Why copy data?)
2. **Partitioning & Sharding** (How do we split data?)
3. **Consistent Hashing** (How do we distribute data gracefully?)
4. **CAP Theorem** (What tradeoffs are unavoidable?)
5. **Consistency Models** (What does "eventual consistency" actually mean?)
6. **Caching** (How do we avoid unnecessary work?)
7. **Indexes** (How do databases find data quickly?)

Once you understand those concepts, technologies like Redis, Cassandra, MongoDB, DynamoDB, and Elasticsearch will feel much more intuitive because you'll already know the problems they're solving.
