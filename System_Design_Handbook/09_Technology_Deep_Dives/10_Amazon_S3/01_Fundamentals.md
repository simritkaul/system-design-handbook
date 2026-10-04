# Module 9 — Chapter 10: Amazon S3

We've just finished messaging:

```text
RabbitMQ → route and process work
Kafka    → retain and stream events
SQS      → managed work queue
```

Now we move to a completely different storage problem.

So far, most of our databases have dealt with **structured application data**:

```text
Users
Orders
Payments
Products
Relationships
Events
```

But real systems also need to store:

```text
Images
Videos
PDFs
Backups
Logs
Datasets
Machine-learning models
Static assets
Documents
```

A 10 MB image or a 5 GB video is a very different storage problem from:

```text
User {
    id,
    name,
    email
}
```

This is where **Amazon S3** comes in.

---

# 1. The Problem

Imagine we're building Instagram.

A user uploads:

```text
vacation.jpg
```

It's 8 MB.

Then:

```text
User A → 500 photos
User B → 2,000 photos
User C → 10,000 photos
```

Eventually we're storing billions of files.

A naive design might be:

```text
Application Server
       ↓
Database
       ↓
Store image bytes
```

This is usually a terrible idea.

Why?

---

## Problem 1 — Files are large

Databases are excellent at managing structured records:

```text
photo_id
user_id
caption
created_at
```

But storing enormous binary objects directly alongside those records can create:

- huge database sizes
- expensive backups
- increased I/O
- difficult scaling
- contention between transactional data and large-file workloads

We don't want our database doing this:

```text
Users + Orders + Payments + 5 TB of videos
```

---

## Problem 2 — Storage needs are enormous

A social media system could eventually need:

```text
10 TB
100 TB
1 PB
10 PB
```

We don't want engineers manually adding disks every time storage grows.

We want:

```text
Storage demand ↑
        ↓
Storage capacity scales with it
```

---

## Problem 3 — Files don't behave like rows

Suppose we have:

```text
video.mp4
```

We don't normally need:

```sql
SELECT *
FROM video
WHERE ...
```

We primarily need:

> **"Give me this object."**

That's a fundamentally different access model.

---

# 2. The Big Idea

> **Amazon S3 is a highly scalable object-storage service designed to store and retrieve arbitrary objects using a unique key.**

The key concept is:

# Object Storage

Instead of thinking:

```text
Table
 ├── Row
 ├── Row
 └── Row
```

think:

```text
Bucket
 ├── Object
 ├── Object
 ├── Object
 └── Object
```

An object consists conceptually of:

```text
Object
├── Data
├── Metadata
└── Key
```

---

# 3. The Fundamental Model

S3 has three concepts worth knowing:

```text
Bucket
Object
Key
```

Imagine:

```text
Bucket: my-app-images

photos/user123/profile.jpg
photos/user123/vacation.jpg
photos/user456/profile.jpg
```

The bucket is the container.

The object is the actual stored data.

The key identifies the object.

So:

```text
Bucket
   ↓
"my-app-images"

Key
   ↓
"photos/user123/vacation.jpg"

Object
   ↓
actual image bytes
```

---

# 4. Important: The Key Isn't a Real Directory Path

You might see:

```text
photos/user123/vacation.jpg
```

and think:

```text
photos/
   user123/
       vacation.jpg
```

as if S3 were a traditional filesystem.

Conceptually, that's misleading.

The key is essentially an identifier:

```text
"photos/user123/vacation.jpg"
```

The `/` is part of the key naming convention.

S3's underlying model is:

> **object + key**

not:

> **folders + files on a disk**

This distinction becomes useful when thinking about scalability.

---

# 5. Why Object Storage?

Let's compare the models we've learned.

### Relational database

```text
Rows
Columns
Relationships
Transactions
Queries
```

### Document database

```text
Documents
Flexible schemas
```

### Key-value store

```text
Key → Value
```

### Object storage

```text
Key → Large object
```

That's the simplest mental model:

> **"I have a blob of data. Give me a durable place to store it and retrieve it by key."**

---

# 6. S3 Is Not a Database

This distinction matters.

Suppose we have:

```text
photo.jpg
```

S3 is excellent at:

```text
GET photo.jpg
PUT photo.jpg
DELETE photo.jpg
```

But it isn't intended to replace your application database.

Suppose you want:

> "Find all photos uploaded by users from Delhi in the last 30 days with more than 500 likes."

That's an application-data query.

You'd likely store metadata in a database:

```text
Photo DB

photo_id
user_id
upload_time
likes
s3_key
```

and the actual image in S3:

```text
S3
 ↓
actual bytes
```

So the architecture becomes:

```text
               Application
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
      Database               S3
    metadata              actual object
```

This separation is extremely common.

---

# 7. A Real Example: Instagram

Suppose a user uploads:

```text
vacation.jpg
```

We might store:

### Database

```text
Photo
-----------------------
id = 8472
user_id = 123
caption = "Goa!"
s3_key = photos/123/8472.jpg
created_at = ...
```

### S3

```text
Bucket:
instagram-media

Key:
photos/123/8472.jpg

Object:
[actual image bytes]
```

The database answers:

> "What is this photo?"

S3 answers:

> "Give me the actual photo."

This is an extremely useful separation.

---

# 8. Why Not Store Everything in MySQL?

You could technically store binary data in a relational database.

But at large scale, you're mixing two very different workloads:

```text
Transactional data
+
Large-object storage
```

Imagine:

```text
Orders DB
    │
    ├── 1 KB order records
    ├── 2 KB payment records
    ├── 500 B user records
    └── 500 MB videos
```

Now database operations, backups, replication, storage costs, and I/O become much harder to reason about.

Instead:

```text
MySQL
 ↓
structured application state

S3
 ↓
large binary objects
```

Each system does what it is good at.

---

# 9. S3's Scalability Model

One of the biggest reasons S3 is important is that you don't typically think about:

```text
Which physical disk?
Which server?
Which storage node?
How many disks?
```

Instead:

```text
Application
     ↓
     S3
     ↓
object stored
```

The underlying infrastructure handles the physical storage distribution.

This is one of the biggest differences between object storage and managing your own filesystem/storage cluster.

---

# 10. Durability vs Availability

This distinction matters enormously.

### Durability

> **How likely is my data to survive?**

### Availability

> **How likely is the service to be able to serve my request right now?**

These are not the same.

For example:

```text
Durability
"I won't lose your file."

Availability
"I can give you the file right now."
```

S3 is designed with extremely high durability, while availability depends on the specific storage/service configuration.

For HLD, don't casually use:

> "S3 is 100% reliable."

Instead understand:

> **S3 is designed for extremely high durability, but availability and failure behavior are separate architectural properties.**

---

# 11. Why Is Object Storage So Durable?

At a high level, cloud object storage doesn't rely on:

```text
one disk
```

or:

```text
one server
```

Your object is redundantly stored across underlying infrastructure.

Conceptually:

```text
             Object
                │
       ┌────────┼────────┐
       ▼        ▼        ▼
    Storage   Storage  Storage
      A         B        C
```

The exact implementation details are AWS-internal and vary by service configuration.

For HLD, the important idea is:

> **The system is designed so individual hardware failures don't normally mean losing the object.**

---

# 12. Storage Classes

Not every object has the same access pattern.

Consider:

### Profile pictures

```text
Read frequently
```

### Old invoices

```text
Read occasionally
```

### Compliance archives

```text
Almost never read
```

Storing everything using the same economics would be wasteful.

S3 therefore provides different **storage classes** optimized for different access patterns.

Conceptually:

```text
Frequently accessed
        ↓
higher availability / faster access
        ↓
higher storage economics

Rarely accessed
        ↓
cheaper storage
        ↓
different retrieval economics
```

The important HLD question is:

> **How frequently will this data be accessed?**

---

# 13. Standard Storage

For frequently accessed data, you use a general-purpose storage class.

Think:

```text
Profile images
Product images
Frequently accessed documents
Static assets
```

The key idea is:

> **Pay for storage and access characteristics appropriate for frequently accessed data.**

---

# 14. Infrequent Access

Suppose:

```text
Invoice archive
```

is accessed:

```text
once every six months
```

It doesn't make sense to optimize its storage economics exactly like:

```text
Instagram profile pictures
```

S3 provides storage classes for infrequently accessed data.

This creates a classic storage tradeoff:

```text
Cheaper storage
      ↕
Retrieval/access cost and characteristics
```

---

# 15. Glacier / Archive Storage

Now imagine:

```text
Legal records
Compliance backups
Old historical data
```

which might be accessed:

```text
once every few years
```

Archive-oriented S3 storage classes are designed for such workloads.

The tradeoff is:

```text
Very cheap long-term storage
        +
different retrieval characteristics
```

So your architecture should ask:

> **How often will we need this data?**

Not:

> "Which S3 storage class is the coolest?"

---

# 16. Lifecycle Policies

Here's where S3 becomes especially useful.

Imagine:

```text
New video
   ↓
Frequently accessed
   ↓
30 days
   ↓
Rarely accessed
   ↓
180 days
   ↓
Archive
```

Instead of manually moving objects:

```text
S3 lifecycle policy
        ↓
automatically transition objects
```

Conceptually:

```text
Object created
      │
      ▼
Standard storage
      │
      │ after X days
      ▼
Infrequent access
      │
      │ after Y days
      ▼
Archive
```

This allows the system to optimize storage economics automatically.

---

# 17. Versioning

Suppose:

```text
profile.jpg
```

is uploaded.

Then someone accidentally overwrites it.

Without versioning:

```text
old object
   ↓
overwritten
```

The previous version may be gone.

With versioning:

```text
profile.jpg
   │
   ├── Version 1
   ├── Version 2
   └── Version 3
```

This can help with:

- accidental deletion
- accidental overwrites
- recovery
- auditability

But it has a cost:

> **More versions mean more stored data.**

Again: capability → tradeoff.

---

# 18. Object Immutability Mental Model

Object storage often works well when objects are treated as relatively independent blobs.

For example:

```text
video_123.mp4
```

rather than constantly modifying individual fields inside the object.

If the application's logical state changes, you often update metadata in a database:

```text
Database
status = PROCESSED
```

while the actual object remains:

```text
S3
video_123.mp4
```

This is another reason the DB + S3 combination works so well.

---

# 19. Multipart Upload

Now imagine a 20 GB video.

Trying to upload it as one enormous request is fragile.

What happens if the network dies at 19 GB?

Do we start again?

That's inefficient.

S3 supports **multipart upload**.

Conceptually:

```text
20 GB file

        ↓

┌──────┬──────┬──────┬──────┐
│ Part │ Part │ Part │ Part │ ...
└──────┴──────┴──────┴──────┘
   ↓      ↓      ↓      ↓
  upload independently
            ↓
          S3
            ↓
      assemble object
```

Benefits include:

- parallel uploads
- better recovery
- large-object support
- retrying only failed parts

This becomes particularly important for large files.

---

# 20. Presigned URLs

Here's an important HLD pattern.

Suppose users upload images.

Naively:

```text
User
  ↓
Application Server
  ↓
S3
```

The application server becomes a middleman for every byte.

For large files:

```text
User ───────→ Application ───────→ S3
```

means your application infrastructure is carrying potentially huge amounts of traffic.

Instead, we can use a **presigned URL**.

Conceptually:

```text
User
  │
  │ "I want to upload image"
  ▼
Application
  │
  │ generate temporary authorization
  ▼
Presigned URL
  │
  ▼
User ─────────────────────→ S3
```

The application controls authorization without having to proxy the actual file contents.

This is an extremely important HLD pattern.

---

# 21. Direct Upload Architecture

Instead of:

```text
                    100 MB
User ─────────────→ API ─────────────→ S3
```

we can do:

```text
User
  │
  │ request upload permission
  ▼
API
  │
  │ presigned URL
  ▼
User
  │
  │ 100 MB
  ▼
S3
```

Now:

```text
API traffic
↓
tiny metadata request

S3 traffic
↓
large file transfer
```

This keeps your application servers focused on application logic rather than acting as file proxies.

---

# 22. Download Architecture

The same pattern can work for downloads.

Instead of:

```text
S3
 ↓
Application Server
 ↓
User
```

you can give the user a temporary authorized URL:

```text
Application
    ↓
temporary URL
    ↓
User ─────────────→ S3
```

Again:

> **Application handles authorization; object storage handles the bytes.**

This is a powerful architectural separation.

---

# 23. Security

S3 objects can contain extremely sensitive data.

The architecture needs to consider:

```text
Authentication
Authorization
Encryption
Access policies
Temporary access
```

A common HLD principle is:

> **Don't make your entire bucket publicly readable just because some objects need to be accessible to users.**

Instead, control access carefully.

For example:

```text
User
 ↓
Application
 ↓
Authorize
 ↓
Presigned URL
 ↓
S3
```

This keeps authorization decisions in the application while S3 handles object transfer.

---

# 24. S3 Is Not a CDN

Another common misconception.

S3:

```text
Object storage
```

CDN:

```text
Edge caching / content delivery
```

You can combine them:

```text
User
 ↓
CDN
 ↓
S3
```

The CDN caches frequently requested content closer to users.

S3 remains the durable origin storage.

This is particularly useful for:

```text
Images
Videos
CSS
JavaScript
Downloads
Static assets
```

---

# 25. S3 + CDN

Imagine your website has:

```text
logo.png
```

Millions of users request it.

Without caching:

```text
Users
  ↓
S3
```

With a CDN:

```text
              ┌→ User
              │
Users → CDN ──┼→ User
              │
              └→ User
                 │
            cache miss
                 ↓
                 S3
```

Most requests can be served from the edge.

This reduces:

- latency
- origin traffic
- repeated reads from S3

Again, notice how Module 3 comes back.

```text
Module 3
Caching / CDN
       ↓
Module 9
S3 + CDN in a real architecture
```

---

# 26. Event Notifications

Another useful S3 capability is that object changes can trigger downstream processing.

For example:

```text
User uploads image
        ↓
S3
        ↓
ObjectCreated event
        ↓
Processing
```

This can feed asynchronous workflows.

For example:

```text
S3
 ↓
event
 ↓
queue
 ↓
image processor
 ↓
S3
```

So you can build:

```text
Upload
  ↓
S3
  ↓
Queue
  ↓
Worker
  ↓
Thumbnail
  ↓
S3
```

Notice how S3 and SQS can work together naturally.

---

# 27. S3 + SQS + Workers

Let's build a photo-processing system.

```text
                 User
                   │
                   │ upload
                   ▼
                  S3
                   │
                   │ object-created event
                   ▼
                  SQS
                   │
          ┌────────┼────────┐
          ▼        ▼        ▼
       Worker   Worker   Worker
          │        │        │
          └────────┼────────┘
                   ▼
                  S3
             thumbnails
```

Now each technology has a clear responsibility:

```text
S3
→ store objects

SQS
→ distribute asynchronous work

Workers
→ process objects
```

This is exactly the kind of architecture you should be able to derive in an HLD interview.

---

# 28. Failure Scenario

Suppose a worker crashes:

```text
SQS
 ↓
Worker
 ↓
download image from S3
 ↓
process
 X
crash
```

Because the work was represented as a queue message, the message can be retried.

The original image remains safely stored in S3.

So:

```text
S3
→ durable object

SQS
→ durable work request
```

The two systems provide different forms of durability.

This separation is powerful.

---

# 29. Object Storage vs Database

Let's make the distinction explicit.

|                            | Database             | S3                                             |
| -------------------------- | -------------------- | ---------------------------------------------- |
| Primary model              | Structured records   | Objects                                        |
| Querying                   | Rich queries         | Key/object access                              |
| Transactions               | Core capability      | Not the same model                             |
| Relationships              | Strong support       | Not the purpose                                |
| Huge files                 | Poorer fit           | Excellent                                      |
| Massive unstructured data  | Not ideal            | Excellent                                      |
| Metadata                   | Strong               | Basic object metadata + external DB often used |
| Cost at huge storage scale | Can become expensive | Designed for large object storage              |

The important architectural principle:

> **Use a database to manage application state; use object storage to hold large blobs when that is the better fit.**

---

# 30. S3 vs Traditional Filesystem

A traditional filesystem thinks in terms of:

```text
Directories
Files
Paths
Disk
Permissions
```

Object storage thinks:

```text
Bucket
Key
Object
```

This difference allows object storage to operate at enormous scale without exposing a filesystem abstraction to your application.

You don't care:

```text
Which disk?
Which server?
Which storage node?
```

You care:

```text
bucket + key
```

That's the abstraction S3 gives you.

---

# 31. What S3 Is Particularly Good At

Strong signals include:

### Large media

```text
Images
Videos
Audio
```

### Documents

```text
PDFs
Reports
Contracts
```

### Backups

```text
Database backups
Application backups
```

### Data lakes

```text
Raw events
Logs
Analytics datasets
```

### Static assets

```text
Images
JS
CSS
Downloads
```

### ML artifacts

```text
Models
Training datasets
Feature datasets
```

The common characteristic is:

> **Large, independently addressable objects.**

---

# 32. What S3 Is Not Good At

Don't use S3 when you need:

### Complex relational queries

```text
JOIN users
WITH orders
WITH payments
```

That's a database problem.

### Frequent fine-grained updates

If you're constantly modifying individual fields:

```text
counter += 1
status = ...
balance = ...
```

S3 isn't the right abstraction.

### Low-latency transactional state

For:

```text
balance
inventory
payment status
```

you generally want a database or specialized datastore.

S3 is primarily:

> **object storage, not transactional application state.**

---

# 33. The Key HLD Tradeoffs

### Advantages

- Massive scalability
- Extremely high durability
- Designed for large objects
- Simple object-based API
- Multiple storage classes
- Lifecycle automation
- Versioning
- Multipart uploads
- Direct client uploads/downloads
- Integrates naturally with other cloud services

### Disadvantages

- Not a relational database
- Limited query model compared with databases
- Object-level access rather than arbitrary fine-grained mutation
- Retrieval/access economics vary by storage class
- Architecture often requires a separate metadata database
- Public/direct access must be carefully secured

---

# 34. A Complete HLD Example

Let's design a YouTube-like video upload system.

The user uploads:

```text
2 GB video
```

We don't want:

```text
User
  ↓
API Server
  ↓
2 GB
  ↓
API Server
  ↓
Storage
```

Instead:

```text
                ┌───────────────┐
                │ Application   │
                │ API           │
                └───────┬───────┘
                        │
                 Generate URL
                        │
                        ▼
User ──────────────────→ S3
       2 GB upload
                        │
                        │ event
                        ▼
                       SQS
                        │
                        ▼
                 Video Workers
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
        Transcoded Video      Thumbnail
              │                   │
              └─────────┬─────────┘
                        ▼
                       S3
```

Metadata:

```text
Database
----------------------
video_id
user_id
title
status
original_s3_key
processed_s3_key
thumbnail_s3_key
```

Now every component has a clear responsibility.

---

# 35. The Architecture Journey

Notice how this architecture emerged from requirements rather than technology names.

We started with:

> "Users upload huge videos."

That leads to:

```text
Huge objects
   ↓
Object storage
   ↓
S3
```

Then:

> "Processing is expensive."

That leads to:

```text
Asynchronous work
   ↓
Queue
   ↓
SQS
```

Then:

> "Workers need to scale."

That leads to:

```text
Competing consumers
   ↓
multiple workers
```

Then:

> "We shouldn't proxy 2 GB through our API."

That leads to:

```text
Presigned URLs
   ↓
Direct client → S3 upload
```

This is exactly the reasoning pattern we want from Module 9.

---

# 36. The S3 Mental Model

Keep this:

```text
                    S3
                     │
             ┌───────┴───────┐
             ▼               ▼
          Bucket           Bucket
             │
             ▼
        Object + Key
             │
       ┌─────┼─────┐
       ▼     ▼     ▼
     Data  Metadata Version
```

And the simplest mental model:

> **S3 is a massive object store: give it a key and an object, and it gives you highly durable storage without making you manage the underlying storage infrastructure.**

---

# 37. One Interview Question You Should Be Ready For

### "Why store images in S3 instead of MySQL?"

A strong answer:

> "Images are large binary objects and don't benefit from the relational querying and transactional capabilities of MySQL. Storing them in S3 gives us scalable object storage with high durability and storage options optimized for different access patterns. I'd typically keep the image metadata and S3 object key in MySQL, while the actual bytes live in S3. For large uploads, I'd use presigned URLs so clients can upload directly to S3 rather than routing the file through application servers."

That's the level we're aiming for.

---

# 38. The Bigger Picture

We've now connected several technologies:

```text
                    Application
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
       Database        Queue         S3
          │             │             │
      App state       Work          Objects
                        │
                        ▼
                     Workers
```

And the technologies we've learned give us concrete implementations:

```text
MySQL / PostgreSQL
        ↓
Structured state

MongoDB / Cassandra / DynamoDB
        ↓
Alternative data models / scaling needs

RabbitMQ / SQS
        ↓
Asynchronous work

Kafka
        ↓
Durable event streams

S3
        ↓
Large objects
```

We're slowly building the technology-selection vocabulary needed to assemble real systems.

---

## Next: Neo4j

The next problem is very different again:

> **What if the important part of our data isn't the individual record, but the relationships between records?**

For example:

```text
Alice
 ├── FRIEND_OF → Bob
 │                 └── WORKS_AT → Google
 └── FOLLOWS → Charlie
                  └── LIKES → Product X
```

At that point, repeatedly joining tables can become the wrong mental model.

That's where **Neo4j and graph databases** enter the picture.
