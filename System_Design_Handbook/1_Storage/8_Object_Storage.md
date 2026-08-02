# Chapter 8 — Object Storage

> **Goal:** Understand what object storage is, why it exists, what problems it solves, and when it is preferred over traditional databases.

---

# 1. The Problem

Imagine you're building Instagram.

Users upload

- Photos
- Videos
- Stories
- Profile Pictures

Suppose one user uploads

```text
vacation.jpg
```

Size

```text
8 MB
```

Then another uploads

```text
birthday.mp4
```

```text
450 MB
```

Millions of users continue uploading.

Soon you're storing

```text
Petabytes of media
```

Where should it go?

---

# 2. The First Thought

Many beginners think

> "Let's store the image inside MySQL."

Example

```text
Users

ID

Name

Photo
```

Technically,

you can.

SQL databases support BLOBs (Binary Large Objects).

So why doesn't Instagram do this?

---

# 3. Why SQL Is a Poor Fit

Imagine MySQL storing

```text
User

↓

500 MB Video
```

Now thousands of people upload videos simultaneously.

Problems appear.

### Huge Storage

Databases become enormous.

---

### Slow Backups

Backing up terabytes or petabytes of media becomes expensive and slow.

---

### Expensive Reads

Fetching a profile picture now involves reading binary media from the database.

---

### Poor Scalability

Relational databases are optimized for structured records.

Not gigantic files.

---

# 4. The Big Idea

Separate

**metadata**

from

**file contents**.

Store

```text
Photo

↓

Object Storage
```

Store

```text
Photo URL

↓

Database
```

Example

Users Table

| ID  | Name  | ProfilePhotoURL       |
| --- | ----- | --------------------- |
| 1   | Alice | images/profile123.jpg |

The actual image lives somewhere else.

---

# 5. What Is an Object?

An object has three parts.

### The Data

The actual file.

```text
vacation.jpg
```

---

### Metadata

Information about the file.

Example

```text
Size

Owner

Content Type

Upload Time
```

---

### Unique Identifier

Every object has a unique key.

Example

```text
photos/user123/profile.jpg
```

The object store uses this key to retrieve the file.

---

# 6. Buckets

Instead of tables,

object storage uses

**Buckets**.

Think of a bucket as a top-level container.

Example

```text
instagram-images

user-videos

backups

documents
```

Each bucket contains many objects.

---

# 7. How Retrieval Works

Suppose Alice uploads

```text
profile.jpg
```

The flow is

```text
User

↓

Upload

↓

Object Storage

↓

Returns Object Key

↓

Store Key in Database
```

Later,

the application asks the database for the key,

then retrieves the file from object storage.

---

# 8. Why Is This Better?

The database stays small.

It stores only

```text
User

↓

Object Key
```

instead of

```text
User

↓

500 MB Video
```

Structured data and large media each live in systems optimized for their own workloads.

---

# 9. Where Object Storage Shines

### Images

Instagram

Facebook

---

### Videos

YouTube

Netflix (original uploaded assets)

---

### Documents

Google Drive

Dropbox

---

### Backups

Database snapshots

Server backups

---

### Machine Learning

Training datasets

Model checkpoints

---

### Static Website Assets

HTML

CSS

JavaScript

Fonts

---

# 10. Where It Performs Poorly

Suppose you ask

> Find every employee earning more than ₹20 lakh.

Object storage cannot answer that.

It isn't a database for structured querying.

It simply stores and retrieves objects by key.

---

# 11. Object Storage vs SQL

| SQL                | Object Storage          |
| ------------------ | ----------------------- |
| Structured records | Files                   |
| Tables             | Buckets                 |
| Rows               | Objects                 |
| Rich queries       | Retrieval by object key |
| Transactions       | File storage            |

---

# 12. Object Storage vs File Systems

A normal file system

```text
Folders

↓

Files
```

works well on one machine.

Object storage is designed for

- massive scale
- durability
- distribution
- internet access

It can store billions of objects across many servers.

---

# 13. Why It Scales So Well

Suppose YouTube stores

```text
20 billion videos
```

Each video is just another object.

Objects are independent.

No joins.

No relationships.

This simplicity makes it easier to distribute storage across many machines and data centers.

---

# 14. Durability

Imagine a disk fails.

You don't want users to lose photos forever.

Object storage systems typically keep multiple copies of data across different disks, servers, or even data centers.

The goal is extremely high durability.

---

# 15. Popular Technologies

Examples include:

- Amazon S3
- Google Cloud Storage
- Azure Blob Storage
- MinIO (self-hosted object storage)

We'll study their individual features later.

---

# 16. Mental Model

Imagine a giant warehouse.

Every package has

- a unique barcode
- some metadata
- the package itself

You don't ask

> "Find every package with a blue shirt."

You ask

> "Bring me package #A83921."

The warehouse retrieves it immediately.

That's object storage.

---

# 17. Tradeoffs

### Advantages

- Excellent for large files
- Highly durable
- Easy to scale
- Cost-effective for massive storage
- Ideal for media and backups

---

### Disadvantages

- Not suitable for relational data
- No joins
- Limited querying
- Higher latency than in-memory systems like Redis
- Not designed for frequent updates to small portions of a file (objects are generally replaced as a whole)

---

# 18. Real-World Examples

### Instagram

Photos

Stories

Videos

---

### YouTube

Original uploaded videos

Thumbnails

Captions

---

### Google Drive

User documents

---

### Netflix

Movie assets

Subtitles

Images

---

### Dropbox

Files

Folders

Versions

---

# 19. Common Interview Questions

- Why don't companies store videos in MySQL?
- What is object storage?
- What is an object?
- Why is object storage so scalable?
- Why store only the object key in the database?
- How is object storage different from a traditional file system?
- Why is it ideal for media-heavy applications?

---

# 20. Connections

A typical architecture looks like this:

```text
User uploads photo
        │
        ▼
Object Storage
        │
        ▼
Returns Object Key
        │
        ▼
Store Object Key in SQL Database
```

Notice how multiple storage systems work together.

- **SQL** stores metadata (user, caption, upload time, object key).
- **Object Storage** stores the actual file.
- Later we'll see **CDNs** cache those files close to users for faster delivery.

---

# Key Takeaways

- Object storage is designed for **large, unstructured files**, not structured records.
- An object consists of **data + metadata + a unique key**.
- Applications typically store **file metadata in a database** and **the file itself in object storage**.
- Object storage provides excellent scalability, durability, and cost efficiency for media, backups, and documents.
- It complements databases rather than replacing them.

---

## One small improvement I'd suggest

I think every concept chapter should end with a simple **"Decision Rule."**

For this one:

> **If your primary concern is storing and serving large files, think Object Storage.**

The earlier chapters become:

- **Need relationships?** → SQL
- **Know the key?** → Key-Value
- **Need flexible objects?** → Document
- **Need massive write throughput?** → Wide Column
- **Need relationship traversal?** → Graph
- **Need to store files?** → Object Storage

Those one-line rules become incredibly useful during interviews because they let you quickly map a problem to the right category before thinking about specific technologies like Amazon S3 or Google Cloud Storage.
