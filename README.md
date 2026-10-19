# Travel Social Network — System Design

## Overview

System design for a social network where travelers publish photo posts tied to places, follow each other, react, comment and discover popular destinations.

| | |
|---|---|
| Scale | 10M DAU after one year |
| Load | ~1.5k write RPS, ~1.6k read RPS, ~5.8k posts/s served to feeds |
| Traffic | 8.4 MB/s metadata + 526 MB/s media (98% of egress is images, served via CDN) |
| Storage | 4.6 TB/year metadata, 532 TB/year media |
| Footprint | 35 hosts for year one: 10 for databases and caches, 25 for object storage |

What's inside: requirements and SLAs, back-of-the-envelope estimates (RPS, traffic, connections, capacity), disk selection, replication, sharding, host sizing, service decomposition and key data flows.

Repository layout:

- `architecture/` — C4 diagrams (PlantUML): System Context and Container
- `api/rest_api.yml` — REST API (OpenAPI)
- `db/` — database schemas (DBML): PostgreSQL and Redis

---

## Functional requirements

- publishing travel posts:
  - photos
  - short description
  - link to the travel location
- rating and commenting on other travelers' posts
- following other travelers
- viewing activity of followed users
- searching for popular travel destinations (most ratings)
- viewing posts for a specific place
- viewing the feed:
  - global feed of all users
  - following feed (reverse chronological order)
- access via:
  - mobile devices
  - web browser

---

## Non-functional requirements

- DAU after one year: **10,000,000**
- audience growth: linear, continuing after the first year
- geography: **CIS**
- availability: **high (99.999%)**
- UX: fast and convenient on mobile + web (see SLA)

---

## SLA

### Comments
- open comments section: P95 0.5s
- post a comment: P99 1s

### Posts (feed)
- publish a post: P99 1.5s
- load the feed: P95 2.0s

### Subscriptions
- follow a user: P99 0.5s
- view subscriptions: P99 0.5s

### Search
- search places: P95 2.0s


# Load assumptions

## Comments

| Parameter | Value |
|----------|----------|
| Comments per user | 3 / day |
| Comment views | 15 / day (3 feed opens) |
| Reactions (likes) | 10 / day |
| Average text | 200 characters |
| Image | 1 per 10 comments |
| Average image size | 200 KB |

---

## Posts

| Parameter | Value |
|----------|----------|
| Posts per user | 1 / week |
| Post views | 50 / day (5 feed opens) |
| Posts per feed request | 10 |
| Average text | 800 characters |
| Average number of images | 3 |
| Image size | 200 KB (compressed) |
| Feed preview | 40 KB (WebP), 1 per post |
| Full view | 10% of posts, 3 × 150 KB |

---

## Subscriptions

| Parameter | Value |
|----------|----------|
| New subscriptions | 1 / week |
| Subscription list views | 1 / day |
| Subscriptions per request | 20 |

---

## Search

| Parameter | Value |
|----------|----------|
| Searches | 5 / day |

---

# RPS estimation

DAU = 10,000,000  
Seconds/day = 86,400  

---

## Comments

### Write comment

```text
10,000,000 × 3 / 86,400 ≈ 347 RPS
```

### Read comments

Requests (comment section opens):

```text
10,000,000 × 3 / 86,400 ≈ 347 RPS
```

Comments served:

```text
10,000,000 × 15 / 86,400 ≈ 1,736 comments/s
```

### Reactions

```text
10,000,000 × 10 / 86,400 ≈ 1,157 RPS
```

---

## Posts

### Write post

```text
10,000,000 × 1 / 7 / 86,400 ≈ 17 RPS
```
### Read feed 

Requests (feed opens):

```text
10,000,000 × 5 / 86,400 ≈ 579 RPS
```

Posts served:

```text
10,000,000 × 50 / 86,400 ≈ 5,787 posts/s
```

### Search

```text
10,000,000 × 5 / 86,400 ≈ 579 RPS
```

---

## Subscriptions

### Write subscription

```text
10,000,000 × 1 / 7 / 86,400 ≈ 17 RPS
```

### Read subscriptions

```text
10,000,000 × 1 / 86,400 ≈ 116 RPS
```

---


# Total RPS

| System | RPS |
|--------|-----|
| Feed read (requests) | 579 |
| Feed read (posts/s) | 5,787 |
| Comments read (requests) | 347 |
| Comments read (comments/s) | 1,736 |
| Comments write | 347 |
| Reactions | 1,157 |
| Posts write | 17 |
| Subscriptions write | 17 |
| Subscriptions read | 116 |
| Search | 579 |

Requests drive server and connection sizing; items drive traffic and IOPS.

```text
Write RPS = 347 + 1,157 + 17 + 17 = 1,538
Read RPS  = 579 + 347 + 116 + 579 = 1,621
```

---

# Traffic estimation

## Post

### Post model

| Field | Size |
|------|--------|
| id | 8 bytes |
| user_id | 8 bytes |
| text | 800 bytes |
| images_url | 3 × 60 bytes |
| location_tag | 8 bytes |
| comments_num | 8 bytes |
| reactions | 8 bytes |
| created_at | 8 bytes |

### Result

- metadata = 6 × 8 = 48 bytes
- images_url = 60 * 3 = 180 bytes

Average post = **~1 KB**

---

## Comment

### Comment model

| Field | Size |
|------|--------|
| id | 8 bytes |
| post_id | 8 bytes |
| user_id | 8 bytes |
| text | 200 bytes |
| images_url | 60 bytes |
| created_at | 8 bytes |

### Variants

#### 1. Text-only comment
≈ 232 bytes

#### 2. Comment with an image (1/10)

232 + 60 ≈ 292 bytes

---

### Average comment size

```text
0.9 × 232 B + 0.1 × 292 B
≈ 238 B
```

Average comment = **~238 B**

---
# Reactions

### Reaction model

| Field | Size |
|------|--------|
| id | 8 bytes |
| post_id | 8 bytes |
| user_id | 8 bytes |
| type | 1 byte |
| created_at | 8 bytes |

### Result

Average reaction size ≈ 40 bytes (33 bytes before alignment)

---

# Subscriptions

### Subscription model

| Field | Size |
|------|--------|
| id | 8 bytes |
| follower_id | 8 bytes |
| followee_id | 8 bytes |
| created_at | 8 bytes |

### Result

Average subscription size = 32 bytes

### Response model

20 × 150 B ≈ 3 KB

---

# Search

### Response model

10 × 300 B ≈ 3 KB

Average search request + response ≈ ~3.2 KB

---

# Traffic usage

## Reactions

1157 RPS × 40 B ≈ 46 KB/s

---

## Subscriptions

Write:

17 RPS × 32 B ≈ 0.5 KB/s

Read:

116 RPS × 3 KB ≈ 348 KB/s

---

## Search

579 × 3.2 KB ≈ 1,853 KB/s ≈ 1.85 MB/s


## Post

Write:

17 RPS × 1 KB ≈ 17 KB/s

## Feed

5,787 posts/s × 1 KB ≈ 5,787 KB/s ≈ 5.8 MB/s

## Comments

Read:

1,736 RPS × 238 B ≈ 413 KB/s

Write

347 RPS × 238 B ≈ 83 KB/s

## Total traffic (metadata)

Write

0.5 + 17 + 83 + 46 = 146.5 KB/s

Read

348 + 1,853 + 5,787 + 413 = 8,401 KB/s ≈ 8.4 MB/s

## Images

The figures above exclude images: response bodies carry only links to them. But the files still reach the user, and they are the bulk of the system's traffic.

Originals are never served in the feed: on upload, renditions are generated from each image, the feed gets a cover preview, and the full size is loaded when the post is opened.

| Flow | Calculation | Traffic |
|-------|--------|--------|
| Feed, previews | 5,787 posts/s × 40 KB | 231 MB/s |
| Post open | 5,787 × 0.1 × 3 × 150 KB | 260 MB/s |
| Comments, read | 1,736 × 0.1 × 200 KB | 34.7 MB/s |
| Posts, write | 17 RPS × 3 × 200 KB | 10.2 MB/s |
| Comments, write | 347 × 0.1 × 200 KB | 6.9 MB/s |

```text
Read  ≈ 526 MB/s
Write ≈ 17 MB/s
```

If the feed served originals, reads would reach 5,787 × 3 × 200 KB ≈ 3.5 GB/s — 7× more. Images are immutable, so they are served through a CDN: with 95% offload, ≈ 26 MB/s remains on the origin.

## Result

```text
Read  = 8.4 MB/s metadata + 526 MB/s images
Write = 0.15 MB/s metadata + 17 MB/s images
```

Images account for 98% of egress. Database load is bound not by bandwidth but by operation count and latency.

# Concurrent connections

## Main estimate
```text
Connections = 10 000 000 * 0.1 = 1 000 000
```

# Capacity 
(for one year)

## Main estimate

146.5 KB * 86400 * 365 = 4,620,024,000 KB ≈ 4.6 TB


## Images

### Posts
10,000,000 × 1/7 * 365 = 521,428,571 posts per year => 521,428,571 × 3 (images per post) = 1,564,285,713 images per year

### Comments (1/10)

10,000,000 × 3 × 365 = 10,950,000,000 comments per year => 10,950,000,000 × 0.1 = 1,095,000,000

### Total images 
1,564,285,713 + 1,095,000,000 = 2,659,285,713 images per year

### Capacity 

2,659,285,713 × 200KB = 531,857,142,600 KB ≈ 531,857 GB ≈ 532 TB/year

# Linking posts to locations

The source of truth is coordinates, not a places directory. A post always stores `coordinates` (the photo's geotag or a manually picked point) and a `geohash` for prefix-indexed lookup. `location_id` is an optional reference to a known place, not a mandatory foreign key.

Matching at publish time: candidates are taken from `locations` by `geohash`, then the nearest point is selected via the GiST index and its distance is compared to that place's `match_radius_m` (tens of kilometers for a city, hundreds of meters for a specific waterfall).

- within the radius of a known place → `location_id` is set;
- outside any radius (a photo on a highway, in a field, at sea) → `location_id` stays `NULL`, and the post is not lost: it is still found by `coordinates` and `geohash`, and the user's caption is kept in `location_title`.

That is why "posts for a specific place" works as two different queries: by `location_id` for places in the directory, and by bounding box / `geohash` prefix for an arbitrary point on the map. Only posts with a non-null `location_id` count toward popular places — a point on a highway is not a travel destination by itself.

# Disk selection

## IOPS

## POSTS

Write + Read (posts + feed + search)
IOPS = 5,787 + 17 + 579 = 6,383 ops/sec
Traffic = 7.7 MB/s
Storage = 4.6 TB/year (metadata only)

HDD
IOPS: 6,383 ops/sec / 100 IOPS = 64 disks
Bandwidth: 7.7 MB/s / 100 MB/s = 1 disk
Capacity: 4.6 TB / 32 TB = 1 disk
Result: max(64, 1, 1) = 64 disks

SSD (SATA)
IOPS: 6,383 ops/sec / 1,000 IOPS = 7 disks
Bandwidth: 7.7 MB/s / 500 MB/s = 1 disk
Capacity: 4.6 TB / 100 TB = 1 disk
Result: max(7, 1, 1) = 7 disks

SSD (NVMe)
IOPS: 6,383 ops/sec / 10,000 IOPS = 1 disk
Bandwidth: 7.7 MB/s / 3000 MB/s = 1 disk
Capacity: 4.6 TB / 30 TB = 1 disk
Result: max(1, 1, 1) = 1 disk

### Choice: SSD NVMe

The deciding factor is latency, not disk count. The required IOPS could be reached on HDD (an array of 64 disks), but reading the feed means random index lookups, where an HDD costs ~10 ms per seek. With a feed-load SLA of P95 2.0s and 10 posts per request, the whole budget goes to the disk. NVMe gives ~0.1 ms and handles the load with a single disk, with headroom for peaks.

## COMMENTS

IOPS = 1,736 + 347 = 2,083 ops/sec
Traffic = 0.5 MB/s
Storage = 0.85 TB/year (metadata only)

HDD
IOPS: 2,083 ops/sec / 100 IOPS = 21 disks
Bandwidth: 0.5 MB/s / 100 MB/s = 1 disk
Capacity: 0.85 TB / 32 TB = 1 disk
Result: max(21, 1, 1) = 21 disks

SSD (SATA)
IOPS: 2,083 ops/sec / 1,000 IOPS = 3 disks
Bandwidth: 0.5 MB/s / 500 MB/s = 1 disk
Capacity: 0.85 TB / 100 TB = 1 disk
Result: max(3, 1, 1) = 3 disks

SSD (NVMe)
IOPS: 2,083 ops/sec / 10,000 IOPS = 1 disk
Bandwidth: 0.5 MB/s / 3000 MB/s = 1 disk
Capacity: 0.85 TB / 30 TB = 1 disk
Result: max(1, 1, 1) = 1 disk

### Choice: SSD NVMe

The strictest SLA in the system is opening the comments section at P95 0.5s. By volume, SATA SSD would suffice (3 disks), but the price difference at these volumes is negligible, and a 0.5s latency budget leaves no room for 10 ms seeks. NVMe it is.

## SOCIAL (reactions + subscriptions + followers)

IOPS = 1,157 + 17 + 17 + 116 = 1,307 ops/sec
Traffic = 0.4 MB/s
Storage = ~0.1 TB/year (metadata only)

HDD
IOPS: 1,307 ops/sec / 100 IOPS = 14 disks
Bandwidth: 0.4 MB/s / 100 MB/s = 1 disk
Capacity: 0.1 TB / 32 TB = 1 disk
Result: max(14, 1, 1) = 14 disks

SSD (SATA)
IOPS: 1,307 ops/sec / 1,000 IOPS = 2 disks
Bandwidth: 0.4 MB/s / 500 MB/s = 1 disk
Capacity: 0.1 TB / 100 TB = 1 disk
Result: max(2, 1, 1) = 2 disks

SSD (NVMe)
IOPS: 1,307 ops/sec / 10,000 IOPS = 1 disk
Bandwidth: 0.4 MB/s / 3000 MB/s = 1 disk
Capacity: 0.1 TB / 30 TB = 1 disk
Result: max(1, 1, 1) = 1 disk

### Choice: SSD NVMe

Reactions are 1,157 RPS of small random writes — the most frequent write in the system. That is the worst possible workload profile for HDD (14 disks just to sustain the IOPS), plus the follow SLA is P99 0.5s. NVMe.

## MEDIA (post and comment images)

Storage = 532 TB/year
Workload profile: write once, read sequentially as a whole file, no random access within a file.

HDD
Capacity: 532 TB / 32 TB = 17 disks

### Choice: HDD

The only group where HDD is the right choice. Images are not stored in the database — they live in object storage and are served via CDN, so IOPS is not a constraint: each file is read in a single sequential request. The only constraint is capacity, and HDD is roughly an order of magnitude cheaper per terabyte than NVMe. Putting 532 TB/year on NVMe means paying for latency that is invisible behind the CDN anyway.

## TOTAL

HDD: 64 + 21 + 14 = 99 disks
SSD SATA: 7 + 3 + 2 = 12 disks
SSD NVMe: 1 + 1 + 1 = 3 disks

### Final choice

| Group | Disk type | Disks |
|--------|-----------|--------|
| Posts | SSD NVMe | 1 |
| Comments | SSD NVMe | 1 |
| Social | SSD NVMe | 1 |
| Media | HDD | 17 |

Total: **3 NVMe** for metadata + **17 HDD** for media in the first year.

No HDD for metadata anywhere: 99 disks versus 3, given that the bottleneck is random-access latency, not volume. SATA SSD remains a budget alternative (12 disks), but the price saving doesn't justify a fourfold larger fleet and the risk of missing the SLA at peak.

# Distributed data storage

## What is stored where

| Subsystem | Storage | Data | Volume per year |
|------------|-----------|--------|-------------:|
| Posts | PostgreSQL | posts, locations | 4.6 TB |
| Comments | PostgreSQL | comments | 0.85 TB |
| Reactions | PostgreSQL | reactions | 0.08 TB |
| Subscriptions | PostgreSQL | subscriptions, followers | 0.02 TB |
| Counters | Redis | reaction, comment and follower counters | ~50 GB RAM |
| Media | Object storage | post and comment images | 532 TB |

The Social group from the disk estimate is split into two clusters: reactions and subscriptions are unrelated (a reaction belongs to a post, a subscription to another user) and are served by different services. Load-wise, each cluster still fits on a single NVMe: 1,157 and 133 operations per second respectively.

## Replication

| Subsystem | Type | Master | Replica | RF |
|------------|-----|-------:|--------:|---:|
| PostgreSQL (Posts) | Master-Slave, asynchronous | 1 | 1 | 2 |
| PostgreSQL (Comments) | Master-Slave, asynchronous | 1 | 1 | 2 |
| PostgreSQL (Reactions) | Master-Slave, asynchronous | 1 | 1 | 2 |
| PostgreSQL (Subscriptions) | Master-Slave, asynchronous | 1 | 1 | 2 |
| Redis | Master-Replica + Sentinel | 1 | 1 | 2 |
| Object storage | Erasure Coding 4+2 | — | — | 1.5 |

**Asynchronous PostgreSQL replication.** Synchronous replication adds a network round-trip to every commit, and across data centers that is tens of milliseconds against a post-publish SLA of 1.5s.

**RF=2.** After a failover, the system runs without a standby until a new replica is up. This window is accepted as tolerable.

**Redis replica.** The replica ensures there is never a period without cache: counters are read on every feed render, and with a cold cache ~11,500 reads per second would fall onto PostgreSQL, which cannot handle that on top of its regular load.

```text
532 TB × 3   = 1,596 TB
532 TB × 1.5 =   798 TB
```

## Sharding

No sharding in the first year: 5.5 TB of metadata fits on one disk per group. Shard keys are fixed upfront so that splitting later won't require rewriting queries.

| Subsystem | Shard key | Rationale |
|------------|-------------------|-------------|
| Posts | `user_id` | an author's posts and their feed are read from one shard |
| Comments | `post_id` | comments are always read in the context of a post |
| Reactions | `post_id` | reactions are always read in the context of a post |
| Subscriptions | `follower_id` | "who I follow" is the main subscription-list query |
| Media | `media_id` | hashing inside the storage, the application is not involved |

The reverse index "author's followers", read by feed fan-out, is sharded by `followee_id`: the query directions are opposite, and one key cannot serve both.

## Host estimate for year one

One disk per host.

```text
Hosts = disks / disks_per_host
Hosts_with_replication = hosts × RF
```

### System

| Subsystem | Disk type | Disks | Hosts | RF | Hosts with replication |
|------------|-----------|-------:|-------:|---:|---------------------:|
| Posts | SSD NVMe | 1 | 1 | 2 | 2 |
| Comments | SSD NVMe | 1 | 1 | 2 | 2 |
| Reactions | SSD NVMe | 1 | 1 | 2 | 2 |
| Subscriptions | SSD NVMe | 1 | 1 | 2 | 2 |
| Redis | SSD NVMe | 1 | 1 | 2 | 2 |
| **Total** | | | **5** | | **10** |

Redis nodes need 64 GB RAM: counters for 521M posts per year take ~50 GB; with a TTL on old posts the working set is considerably smaller.

### Images

```text
532 TB × 1.5 (EC 4+2) = 798 TB
798 TB / 32 TB        = 25 disks
25 / 1                = 25 hosts
```

### Summary

| | Hosts |
|---|------:|
| System | 10 |
| Images | 25 |
| **Total** | **35** |

# High-level architecture

Diagrams are in `architecture/`, C4-PlantUML:

- `c4-context.puml` — Level 1, System Context
- `c4-container.puml` — Level 2, Container

## Services

| Service | Owns data |
|--------|-----------------|
| Auth service | users DB |
| Publications service | posts & locations DB, comments DB, reactions DB |
| Subscriptions service | subscriptions DB |
| Feed service | feed cache |
| Media service | object storage |

Posts, comments and reactions are served by one service: a comment or a reaction does not exist without a post and is read together with it, so splitting them would add a cross-service call to every post render. The storage is still separate — 1,157 reaction writes per second cannot share a disk with feed reads.

Subscriptions are a separate service: a subscription links two users, involves no post, and the follow graph is sharded by a different key.

The feed is separated not by data but by mode of operation. It has a background fan-out worker: with an average of 52 followers, publishing one post produces 52 cache writes, i.e. ~884 background writes per second. A fan-out backlog must not compete for resources with requests under the P95 2.0s SLA.

There is no separate places service: popular places are `locations` sorted by rating count, and posts for a place are a lookup by `location_id` and geohash. Both queries go to the posts DB.

## Key flows

**Photo upload.** The client requests a presigned URL from the media service and uploads the file directly to object storage, bypassing the API. The media service publishes a "media uploaded" event; a worker strips EXIF and generates previews and renditions. The 17 MB/s of uploads never pass through application services.

**Publishing a post.** The publications service writes the post and publishes a "new post" event. The fan-out worker fetches the author's followers from the subscriptions service and appends the post id to their feeds in Redis.

**Reading the feed.** The feed service takes a precomputed list of ids from the cache and fetches the posts with counters from the publications service. The client gets images directly from the CDN — the system is not involved in that flow, hence 8.4 MB/s of metadata versus 526 MB/s of media.

**Reaction.** The reaction row is written to PostgreSQL in batches — this is the source of truth. The counter is incremented in Redis rather than via an `UPDATE` on the post row: 1,157 reactions per second land on the same popular posts, and in the database that would mean a row lock, WAL and fsync for each one. An in-memory `INCR` costs none of that, and the counter can always be recomputed from the rows.
