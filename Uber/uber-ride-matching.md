# Design Uber (fare estimates, ride matching, driver locations)

Source: [Hello Interview, "Design Uber w/ a Ex-Meta Staff Engineer"](https://www.youtube.com/watch?v=lsKU38RKQSo) with Evan King, plus the [Hello Interview write-up](https://www.hellointerview.com/learn/system-design/problem-breakdowns/uber). High-level design starts at 21:00, deep dives at 32:30. Timestamps below point into the video.

Stack used in the video: AWS API Gateway, microservices (ride, ride matching, location, notification), a third-party maps API, a primary DB (DynamoDB by the end), Redis for driver locations (geohash) and driver locks (TTL), a ride request queue partitioned by region.

Sections marked **Beyond the video** add background the video skips or glosses over. Where the write-up adds something the video doesn't cover, it's labeled **From the write-up**.

Your timestamp (9:21) lands at the end of the non-functional requirements, where Evan lists what's out of scope (GDPR, monitoring, CI/CD) right before his case against early back-of-envelope math.

## Understanding the problem

**What is this system?** A ride-hailing app. A rider says where they want to go, gets a price, asks for a ride, and the system finds a nearby free driver who accepts and drives them there.

Evan calls it the hardest of the "proximity search" problems. If you can do Uber, you can do Yelp and Find My Friends. The two things that make it hard: millions of drivers reporting their location every few seconds, and many servers trying to hand out the same few drivers at the same moment.

### Functional requirements (3:00)

Core requirements

1. Riders can input a start location and a destination and get a fare estimate.
2. Riders can request a ride based on the estimate, and get matched with a nearby available driver in real time.
3. Drivers can accept or decline a request, then navigate to pickup and drop-off.

Below the line (out of scope)

1. Multiple car types (UberX, XL, Comfort). One estimate only.
2. Ratings for drivers and riders.
3. Scheduling rides in advance.

Evan's advice: at most three core features. Naming a few out-of-scope ones shows product sense, but interviews move fast, and focus wins.

### Non-functional requirements (6:00)

His complaint: candidates rush this part and write buzzwords ("scalable, available"). The non-functional requirements decide your deep dives, so make each one specific to this system and put a number on it.

1. **Low-latency matching.** Match in under 1 minute, or fail with "no drivers available".
2. **Consistency of matching.** A ride goes to exactly one driver, and a driver gets at most one ride offer at a time. In CAP terms, this is the part that picks consistency.
3. **High availability outside of matching.** Fare estimates, location updates and everything else should stay up 24/7. So the system is consistent where it matters and available everywhere else.
4. **High throughput during surges.** A concert or New Year's Eve can mean hundreds of thousands of requests in one region within minutes.

Out of scope: GDPR and privacy, resilience to failure (touched lightly), monitoring and alerting, CI/CD.

**From the write-up:** it lists the same requirements but drops availability, and quantifies the surge as "100k requests from the same location".

### Skip the math up front (10:00)

Evan skips back-of-envelope estimates at this point on purpose. His experience: candidates compute DAU, storage and bandwidth, conclude "it's a lot", and move on, which they knew already. His script for the interviewer: "I'll skip estimates for now and do them during the design when a number changes a decision. Is that okay?" He does exactly that later, when the location write rate decides the database (deep dive 1).

### Planning the approach

Same shape as every Hello Interview product question. Requirements, then core entities, then API. The high-level design walks the API one endpoint at a time until every functional requirement works. The deep dives go one by one through the non-functional requirements.

## Core entities (12:00)

- **Ride**: one trip, from fare estimate to drop-off. Holds rider, driver, fare, ETA, source, destination, status.
- **Driver**: profile, car, and a status (available, in ride, offline).
- **Rider**: profile and payment. Boring for this design, so he dot-dot-dots it.
- **Location**: the latest position of every driver. This one is less obvious. It exists so matching can ask "who is near this rider?" quickly, and its write rate drives the first deep dive.

He says "core entities" instead of "data model" because at this point you don't know the columns yet. You fill in fields during the high-level design, each time an arrow lands on the database.

**From the write-up:** it adds a separate **Fare** entity (estimate, ETA, pickup, destination) and creates the Ride only when the rider confirms. The video puts the estimate on the Ride row itself. Both work. The video's version keeps unbooked estimates around, which is useful for analytics ("what did people price and not book?").

## API (13:00)

```
POST  /ride/fare-estimate      { source, destination }  -> Partial<Ride> { id, eta, fare }
PATCH /ride/request            { rideId }               -> 200 (matching runs async)
POST  /location/update         { lat, long }            -> 200   // driver calls every ~5 s
PATCH /ride/driver/accept      { rideId, accept: bool } -> 200
PATCH /ride/driver/update      { rideId, status }       -> next { lat, long } | null
```

- **POST** for the estimate because it creates a row. **PATCH** for request and accept because they update that same row. Evan says don't get stuck on PATCH vs PUT vs POST.
- `ride/request` returns immediately. Matching can take up to a minute, so it's async.
- `location/update` is easy to miss up front. Matching needs driver positions, so something has to write them. He says it's fine to discover endpoints later and add them.
- `driver/update` with status `picked_up` returns the destination. With `dropped_off` it returns null.

Two interview tips from this section:

- **Skip types if you're senior.** Writing `lat: number` wastes time. Spell out a type only when it's not obvious, like the status enum.
- **Never put the user ID in the body.** The driver or rider ID comes from the JWT or session in the request header. If the body carried it, anyone could request a ride or accept one as someone else. **From the write-up:** the same goes for timestamps (server generates them) and the fare (read from the DB, never trusted from the client).

## High-level design

Each step satisfies one API endpoint. The design is deliberately simple; the deep dives fix what it leaves open.

### 1) Fare estimate: gateway, ride service, maps API (21:30)

![[01 - Fare estimate]]

**Decision:** rider client (iOS, Android, no web) calls an **API gateway**, which routes to a **ride service**. The ride service asks a **third-party mapping API** (Google Maps) for the ETA given current traffic, turns it into a price with a simple formula, and writes a Ride row with status `fare_estimated`.

**Why:** the real Uber prices with ML models. In an interview you either black-box that or simplify loudly. Evan simplifies and says so.

**What the API gateway does:** routing each request to the right microservice (its main job here), plus load balancing, authentication, SSL termination and rate limiting. He says list these briefly and move on.

Ride row so far:

```
Ride   id, riderId, fare, eta, source, destination, status (fare_estimated), driverId (added later)
Rider  id, ...
```

**What's open:** there's no way to request the ride. Step 2.

### 2) Request a ride: a separate matching service (26:00)

![[02 - Request ride and matching service]]

**Decision:** a new **ride matching service**, separate from the ride service.

**Why separate:**

- The two jobs are very different. Fare estimates are a quick request/response. Matching is slow (up to a minute), async and CPU heavy.
- They scale independently: during a surge you need many more matchers, not more fare estimators.
- Separate teams can own them.

The matching service needs "all drivers within N miles of the rider", so it needs a **location DB**. But nothing writes to it yet. Step 3.

### 3) Driver locations: a location service (28:00)

![[03 - Driver location updates]]

**Decision:** the **driver client** calls `location/update` every 5 seconds. A **location service** writes the latest lat/long to the location DB. Matching queries nearby drivers, then checks each one's status in the primary DB and drops anyone who isn't `available`.

```
Driver    id, car, plate, photo, status (available | in_ride | offline)
Location  driverId, lat, long, updatedAt
```

**What's suboptimal:** the location DB is a black box. It has to absorb every driver's updates and answer 2D "who is near me" queries fast. That's deep dive 1.

### 4) Notify, accept, navigate (30:00)

![[04 - Notify accept and navigate]]

**Decision:** matching sends the offer through a **notification service** that uses the native push systems: APNs on iOS, FCM (Firebase Cloud Messaging) on Android. Evan black-boxes it, since "design a notification service" is its own interview question.

Flow:

1. Matching picks a driver and pushes "do you want this ride?".
2. Driver taps accept, which calls `driver/accept`. The ride service sets `driverId` on the Ride and moves status to `matched`.
3. When the driver reaches the pickup, they call `driver/update` with `picked_up`. The ride service returns the destination.
4. At drop-off, `dropped_off` returns null. Nowhere else to go.

Status on the Ride: `fare_estimated -> requested -> matched -> picked_up -> dropped_off`.

**Beyond the video:** the rider also needs to hear "you're matched". Evan says "notification, polling, or otherwise" and moves on. Realistically: a push notification plus the app polling ride status, or a WebSocket/SSE while the app is open (the write-up points to the real-time updates pattern).

### Where the high-level design stands (32:00)

| Requirement | Status | Why |
| --- | --- | --- |
| Match in under 1 minute | Unknown | Depends on how fast the location DB answers "nearby drivers" |
| One driver per ride, one offer per driver | **Broken** | Nothing stops two matcher instances from offering the same driver |
| High availability outside matching | Partly | Services can scale out, but one region, one DB |
| Surges | **Broken** | Requests go straight to matchers; if they can't keep up, requests are lost |

Evan's level check here: for mid-level, this board plus decent answers to probing questions is close to a pass. Senior needs to go deep in about two places. Staff goes deeper in about three, and ideally teaches the interviewer something.

### The evolution at a glance

| Step | Decision | What it bought | What it broke or left open |
| --- | --- | --- | --- |
| 1 | Gateway, ride service, maps API, Ride row | Fare estimates | No booking |
| 2 | Separate ride matching service | Async matching that scales on its own | Needs driver locations |
| 3 | Location service, updates every 5 s | Matching knows where drivers are | 600k writes/s and 2D queries on a black box |
| 4 | Push notifications, accept and update endpoints | All functional requirements work | Double offers, surges, location speed |
| 5 | Postgres lat/long columns | Obvious and simple | 1D indexes, a few thousand writes/s |
| 6 | PostGIS quadtree + batching queue | Fast 2D queries, fewer writes | Stale positions, re-indexing cost |
| 7 | Redis geohash | 600k writes/s, fast radius search | Most writes are wasted |
| 8 | Adaptive update intervals on the client | Far fewer writes | Location done; matching still double-offers |
| 9 | One offer per ride via a local loop | Rule 1 solved | Rule 2 (one offer per driver) needs shared state |
| 10 | Driver status column + cron | Shared view of who has an offer | Ignored offers lock drivers until the next cron run |
| 11 | Redis lock with 10 s TTL | Exact, automatic expiry | Surges can outrun matcher scaling |
| 12 | Ride request queue, partitioned by region | Buffers spikes, survives crashes | The per-ride loop still dies with its process |
| 13 | Durable workflow (write-up) | Timeouts and progress survive crashes | Another system to run |
| 14 | Final architecture, one copy per region | Everything combined | |

## Potential deep dives

### 1) How do we store driver locations and search them fast? (34:30)

Two problems at once: a huge write rate, and "nearby" queries on two-dimensional data.

#### Start with numbers (35:00)

This is where Evan does the math he skipped earlier, because the result picks the database.

- Uber has about 6 million drivers. Say 3 million online at peak (he admits that's high).
- Each sends an update every 5 seconds.
- 3,000,000 / 5 = **600,000 writes per second**.

**From the write-up:** it uses 10 million drivers, so about 2 million writes/s, and notes DynamoDB on-demand would cost over $200k a day at that rate.

#### Bad solution: Postgres with lat and long columns (36:00)

![[05 - Location v1 Postgres lat long]]

Store `lat` and `long` columns, index each, and query a box:

```sql
SELECT driver_id FROM location
WHERE lat  BETWEEN :south AND :north
  AND long BETWEEN :west  AND :east;
```

**Challenges.**

- **B-tree indexes are one-dimensional.** The `lat` index finds every driver in a horizontal band that circles the whole earth. The database then filters that band by `long`. Wider searches scan far more rows.
- **Write rate.** Evan puts Postgres at 2k to 4k writes/s. 600k is two orders of magnitude away.

**Beyond the video:** 2-4k is low for a tuned modern Postgres node, which can do tens of thousands of small updates per second. Still far short of 600k. And an update-heavy table in Postgres creates a dead row version on every update (MVCC), so vacuum has to keep up with 600k dead tuples a second. The conclusion holds.

#### Good solution: a geospatial index plus a batching queue (38:00)

![[06 - Location v2 PostGIS quadtree and batching]]

Fix each problem separately:

1. **2D queries: a geospatial index.** Postgres has the **PostGIS** extension. Evan describes it as a quadtree.
2. **Write rate: a queue.** The location service puts updates on a queue, and a consumer writes them to Postgres in large batches, turning 600k small writes into a few thousand big ones.

**How a quadtree works (38:30).** Split the map into four squares. Any square holding more than *k* drivers (he uses k = 5) splits into four again, recursively. The result is a tree: the root is the world, each node has four children, and leaves hold a short list of drivers. To search, walk down to the leaf containing the rider, then check neighbouring leaves. Dense cities get tiny cells; empty oceans stay as one big cell.

**Challenges.**

- **Stale data.** Batches have to fill up before they're written, so matching sees positions that are seconds old.
- **Re-indexing.** Every batch of moved drivers reshapes the tree (cells split and merge). Costly under constant churn, and the tree lives in memory.

Evan: this is okay for mid-level, and for senior if they then spot the limits.

**Beyond the video: what PostGIS actually uses.** PostGIS's default spatial index is **GiST**, an R-tree style index (nested bounding rectangles), not a quadtree. Postgres also has **SP-GiST**, which does support quadtrees and k-d trees. The trade-offs Evan describes (tree maintenance under frequent updates) apply to both.

#### Great solution: Redis with geohashing (41:30)

![[07 - Location v3 Redis geohash]]

**Decision:** keep live driver locations in **Redis**, in memory. A Redis cluster handles 100k to about 1M operations per second, so 600k writes/s needs no queue and no batching. Redis supports geospatial queries built on **geohashing**.

**How geohashing works (42:30).** Divide the world into a grid, then subdivide each cell, recursively, down to the precision you want. Each cell gets a string ID where each extra character zooms in. Unlike a quadtree, the grid doesn't depend on where drivers are. It's the same fixed grid everywhere, so a driver's cell is just a function of their lat/long. No tree to maintain.

- Cheap to compute.
- Easy to store: a string (or number) per driver, no extra data structure.
- Shared prefix means same area: `9q8yu` and `9q8yv` are neighbouring cells inside `9q8y`.

**Beyond the video: how geohash is really built.** The video describes splitting into quarters labeled 0-3. That's closer to a quadkey (map tiles). Geohash **interleaves the bits** of longitude and latitude: bit 1 says east or west half, bit 2 north or south half, bit 3 east or west again, and so on. Every 5 bits become one base-32 character, so each character splits a cell 32 ways. Rough cell sizes at the equator: 5 characters is about 4.9 x 4.9 km, 6 characters about 1.2 x 0.6 km.

**Beyond the video: the edge problem.** Two points a few metres apart can sit in cells with different prefixes. In the diagram, the cell above `9q8yu` is `9q8zh`. So "same prefix" is not "nearby". Searches check the rider's cell **plus its 8 neighbours**, then filter by real distance. Redis does this for you.

**Beyond the video: the Redis commands.** A geo set is a sorted set where each member's score is a 52-bit interleaved geohash.

```
GEOADD drivers -122.42 37.77 driver:123       # add or move a driver (overwrites)
GEOSEARCH drivers FROMLONLAT -122.41 37.78 BYRADIUS 2 km ASC COUNT 10 WITHDIST
```

`GEOSEARCH` (Redis 6.2+) replaces the older `GEORADIUS`. Sorted sets have no per-member TTL, so a driver whose app died stays in the set. **From the write-up:** keep a second sorted set scored by last-update time and periodically remove anyone older than about 30 seconds from both.

**Memory:** 3M members at roughly 100 bytes each (member name, score, sorted set overhead) is a few hundred MB. The hard part here is the write rate, not the size.

**Durability:** Redis is in memory, so a crash loses locations. That's fine. Drivers re-send within 5 seconds, so the state rebuilds itself. (The write-up adds RDB/AOF persistence and Sentinel failover as options.)

#### Quadtree vs geohash (44:30)

| | Quadtree | Geohash |
| --- | --- | --- |
| Cells | Adapt to density: small where crowded | Fixed grid, same size everywhere |
| Best when | Uneven density, few updates (Yelp: businesses rarely move) | Many updates (Uber: drivers move every few seconds) |
| Cost of an update | May split or merge nodes, re-shape the tree | Recompute one number |
| Storage | A tree structure to maintain | A string/number per point |

Uber has uneven density too (Manhattan vs Kansas), but the write rate wins, so geohash.

**Beyond the video: Uber's H3.** Evan half-remembers Uber using hexagons. That's **H3**, Uber's open-source hexagonal grid (2018). A hexagon's six neighbours are all the same distance from its centre, while a square has four edge neighbours and four farther corner neighbours. That makes "rings" of nearby cells a better approximation of a circle, which helps for things like surge pricing zones.

**The key insight:** the location store's job is "hold the latest point for each driver and answer radius queries". That's hot, small, and disposable, which is exactly what in-memory stores are for. Durability can come from the drivers themselves resending.

### 2) Can we send fewer location updates? (46:30)

![[08 - Adaptive location updates]]

An interviewer may ask whether 600k/s is necessary.

- **Simple:** send every 10, 15, 20 seconds, or up to a minute. A product trade-off: less accuracy, less load.
- **Better: adaptive updates on the client.** The driver app decides:
  - not accepting rides: don't send at all
  - parked for 20 minutes: position isn't changing, don't send
  - far from any demand: send rarely
  - moving fast, turning, or near open requests: send often

His rough guess: 600k/s down to 100k/s or less. He'd keep Redis anyway.

**The write-up's lesson:** don't draw the client as a tiny box and forget it. Here the phone has the sensors and context, and client logic cuts backend load. Same idea as chunking and compression on the client in a file upload design.

### 3) How do we avoid double matching? (48:00)

Consistency of matching has two rules:

1. A ride is offered to **one driver at a time**.
2. A driver has **at most one outstanding offer** at a time.

#### Rule 1 is easy: a loop in one instance (48:30)

![[09 - Matching consistency the double send problem]]

Each ride request is handled by exactly one matching instance. So that instance can just loop:

```
while not matched:
    driver = next driver in ranked list
    lock(driver)
    send offer
    wait 10 s
```

No coordination needed, because no other instance works on that ride.

#### Rule 2 is hard: many instances, same drivers (49:30)

Matching is horizontally scaled. A concert lets out, 100,000 people request rides, requests land on different instances, and each asks Redis "who's closest?". They all get back `[driver 1, driver 2, ...]`. Every instance offers the ride to driver 1, whose phone now shows 100 pop-ups.

A single box can't fix this. The instances need shared state saying "driver 1 already has an offer".

#### Good solution: a status column plus a cron job (50:30)

![[10 - Lock v1 status column and cron job]]

**Idea:** add a driver status `request_sent`. Before offering, an instance atomically sets driver 1 to `request_sent` (only if they're `available`). Other instances see that and move on to driver 2. Evan points out this mirrors the Ticketmaster seat reservation problem.

**Challenge:** drivers get 10 seconds (the video says 5 or 10) to respond. If they ignore it, the status stays `request_sent` forever, and nobody can offer them anything.

Common fix: a **cron job** every N seconds or minutes that finds `request_sent` rows older than 10 seconds (`statusUpdatedAt`) and resets them to `available`.

**Challenge of the fix:** lag. With a 1-minute cron and a 10-second window, a driver can sit locked for about 50 extra seconds, exactly when demand is highest. Evan: fine for mid-level, maybe senior, not staff.

**From the write-up:** it adds a step before this one. The "bad" version keeps locks in each instance's memory with a timer. It fails because instances can't see each other's locks, and a crash loses the timer and leaves the lock stuck.

#### Great solution: a distributed lock with a TTL (54:00)

![[11 - Lock v2 distributed lock with TTL]]

**Idea:** before offering, write a key for the driver in Redis with a 10-second TTL. If the key exists, the driver is taken; skip them. If the driver never answers, Redis deletes the key at 10 seconds and the driver is free immediately. No cron.

Evan's alternatives: reuse the same Redis cluster as the locations, or put a `driver_lock` table in DynamoDB (which supports TTL) and skip adding a technology. He leans toward DynamoDB for the primary DB anyway: few relations, high availability, scales well.

**Beyond the video: make the lock atomic.** The video describes "check if the key exists, then set it". Two instances can both check, both see nothing, and both set. Use one atomic command:

```
SET lock:driver:1 <instanceId> NX EX 10    # OK = you own it, nil = taken
```

To release early (driver accepted or declined), delete only if the value is still yours, via a small Lua script. Otherwise a slow instance could delete a lock that expired and was re-acquired by someone else.

**Beyond the video: DynamoDB TTL is lazy.** DynamoDB deletes expired items in the background, typically within a few days, not at the second. A TTL lock table must store `expiresAt` and use a conditional write like `attribute_not_exists(driverId) OR expiresAt < :now`. Redis expiry, by contrast, is effectively immediate for reads.

**Beyond the video: the lock is advisory, the ride row is the truth.** The lock only prevents spamming a driver with offers. What actually guarantees "one driver per ride" is the accept path. If a driver accepts after the lock expired and the ride went to someone else, the ride service must refuse:

```sql
UPDATE ride SET driver_id = :d, status = 'matched'
WHERE id = :r AND driver_id IS NULL;      -- 0 rows updated => "ride already taken"
```

Set the driver's status to `in_ride` in the same transaction. This is why a rare lock failure (for example a Redis failover losing a key) only causes a duplicate pop-up, not a double booking. The write-up makes the same point: locks live 10 seconds, so losing them is cheap.

**The key insight:** anything with a timeout should expire by itself. Crons and in-memory timers both leave a window where state is wrong. A TTL puts the expiry in the data.

### 4) How do we handle surges without dropping requests? (57:30)

![[12 - Surges ride request queue by region]]

**Problem:** requests go straight to matching instances. A sudden surge can outpace autoscaling, and requests get dropped. A crashed instance also loses the rides it was matching.

**Decision:** a **ride request queue** between the request and matching. Matching instances pull when they have capacity.

- Surges wait in the queue instead of being dropped.
- An instance acks a message only after it finds a match. If it crashes mid-match, the message was never acked and gets redelivered to another instance.

**Challenge: head-of-line blocking.** In one FIFO line, a hard request (rural, no drivers nearby) blocks easy ones behind it (Manhattan, drivers everywhere). Fix: **partition the queue by region**, finer than boroughs in a city like New York. Slow regions no longer delay fast ones.

**From the write-up:** it names Kafka (commit the offset only after a match) and managed options (SQS, MSK, Confluent), and suggests a priority queue as another fix for head-of-line blocking.

**Beyond the video: Kafka vs SQS here.** Kafka consumer groups commit an **offset per partition**, not an ack per message. If message 5 takes 60 seconds to match, you can't commit messages 6 to 100 behind it without also committing 5, so one slow ride stalls its whole partition (or you build your own tracking). SQS-style queues have per-message visibility timeouts: a message is hidden while one worker holds it and reappears if not deleted in time. That fits "one ride, up to a minute, might fail" better. Newer Kafka versions add share groups (KIP-932) for queue-style acks.

### 5) What if a driver doesn't respond? (from the write-up)

![[13 - Driver timeouts durable workflow]]

The video's loop ("send offer, wait 10 s, try next") lives in one process's memory. If that process dies, the queue redelivers the ride, but matching starts over from the first driver.

#### Good solution: a delay queue

When an offer goes out, also schedule a delayed message (SQS delay queue) for 10 seconds later. When it fires, check if the ride is still unassigned; if so, offer the next driver and schedule another.

**Challenges.** Races between a late accept and the timer firing. The timer handler must check the ride state and do nothing if a driver already accepted.

#### Great solution: a durable workflow

Model matching as a workflow in **Temporal** or **AWS Step Functions**:

1. Offer to the first driver.
2. Wait for an accept signal or a 10-second timer.
3. Accepted: done. Declined or timed out: next driver.
4. List empty: tell the rider no drivers are nearby.

The engine persists every step. A crashed worker's workflow resumes on another worker at "driver 4, 6 seconds left".

**Challenges.** Another system to learn, run and monitor. The write-up argues it's worth it when dropped rides cost revenue. Fun fact: Uber built **Cadence**, which Temporal was forked from, for this kind of human-in-the-loop process.

### 6) High availability and scaling by region (60:00)

- Each service scales horizontally behind its own load balancer.
- DynamoDB scales largely without limit; Redis scales as a cluster.
- **Split by region.** The whole stack is copied per region (northeast, southwest, northwest), in different data centers. This spreads load, lowers latency, and makes queue partitioning natural.
- Users travel (New York to California), so user tables need copies in every region, for example read replicas.

**From the write-up:** vertical scaling is dismissed quickly. Geographic sharding applies to services, queues and databases. The only cross-shard work is a proximity search near a region border, which has to ask both shards (scatter-gather).

### Final architecture

![[14 - Final architecture]]

## Beyond the video: gaps worth knowing

- **Accept must be conditional.** Covered above: `WHERE driver_id IS NULL` on the ride, plus the driver's `in_ride` status in the same transaction. Interviewers love "what if two drivers accept at once?".
- **Fare estimate expiry.** Prices move with traffic and demand. Store an expiry on the estimate and reject `ride/request` on a stale one, or re-price.
- **Ranking, not just distance.** Straight-line distance isn't drive time: a driver across a river can be 200 m away and 15 minutes out. Real matching ranks by ETA on the road network, plus rating and other factors (the write-up mentions this).
- **Batched matching.** Greedy "closest free driver for each request" isn't globally optimal. Dispatch systems often collect requests for a couple of seconds and solve an assignment across all of them.
- **Stale drivers in Redis.** No per-member TTL in geo sets; you need the cleanup set described above, or drivers whose app crashed keep getting offers.
- **Idempotent requests.** A rider double-tapping "request" or retrying on a bad network shouldn't create two matching jobs. Key the queue message by `rideId` and ignore duplicates.
- **Live tracking during a trip.** The rider watching the car move is out of scope, but it's the same location stream fanned out to one rider, usually over a WebSocket or SSE.
- **Surge pricing.** Out of scope, but it's where H3 hexagons and per-cell supply/demand counts come in.

## What is expected at each level? (61:30)

### Mid-level

The high-level design, plus reasonable answers when the interviewer probes the deep dives. **From the write-up:** clear API and data model, knows a spatial index is needed (even without naming one), and reaches at least the "good" lock solution.

### Senior

Goes deep in at least two places, usually location storage and matching consistency. Landing on the cron job or the PostGIS + queue answer is still likely a hire if the rest is strong. **From the write-up:** move fast through the high-level design and discuss trade-offs in detail.

### Staff+

More depth, maybe in different places than Evan chose, drawn from real experience. A great sign is the interviewer learning something. **From the write-up:** three or more deep areas, interviewer steers only to focus.

## Check yourself

1. Why does Evan skip back-of-envelope math at the start, and where does he finally do it? What decision does the number drive?
2. The design is "consistent where it matters, available everywhere else". Which part needs consistency, and why?
3. Why is the user ID in the JWT and not the request body?
4. Why is ride matching a separate service from the ride service?
5. Why can't B-tree indexes on `lat` and `long` answer "drivers within 2 km" efficiently?
6. What are the two weaknesses of PostGIS plus a batching queue?
7. Quadtree vs geohash: when would you pick each? Why geohash for Uber even though density is uneven?
8. Why is "same geohash prefix" not the same as "nearby"? How do searches handle it?
9. Why is it fine that Redis loses driver locations on a crash?
10. How do you cut 600k writes/s without changing the backend?
11. Rule 1 (one offer per ride) needs no coordination. Rule 2 (one offer per driver) does. Why the difference?
12. What goes wrong with the status column plus cron approach?
13. Why is "check if the lock key exists, then set it" broken, and what's the fix?
14. If the lock can fail, what actually guarantees one driver per ride?
15. Why partition the ride request queue by region?
16. What does a durable workflow buy over the video's in-process loop?

<details>
<summary>Answers</summary>

1. Early estimates usually conclude "it's big", which everyone knew. He does it at the location deep dive: 3M drivers / 5 s = 600k writes/s, which rules out a plain Postgres table and points to Redis.
2. Matching. A ride must go to exactly one driver and a driver must get one offer at a time. Fare estimates and location updates can tolerate staleness, so they favor availability.
3. The server can't trust the body. If the ID came from the body, anyone could book or accept rides as someone else. The JWT is signed by the server.
4. Matching is slow, async and CPU heavy; fare estimation is a quick call. Separate services scale independently and can have separate owners.
5. A B-tree orders one dimension. The `lat` index returns a band of drivers across the whole earth, which then has to be filtered by `long`. Bigger radius, more rows scanned.
6. Batching delays writes, so matching uses stale positions. The spatial tree needs re-shaping as drivers move, which is costly at this update rate.
7. Quadtree for uneven density with rare updates (Yelp). Geohash for frequent updates (a driver move is recomputing a number, no tree changes). For Uber the write rate matters more than density.
8. Cells that touch can have different prefixes (`9q8yu` and `9q8zh`). Search the rider's cell plus its 8 neighbours, then filter by actual distance.
9. Drivers send a fresh location every few seconds, so the set rebuilds within one update interval.
10. Adaptive intervals on the driver app: nothing when offline, rarely when parked or far from demand, often when moving fast or near requests.
11. A ride is owned by one matching instance, so it can loop locally. A driver can be picked by any instance, so instances need shared state.
12. If a driver ignores an offer, they stay locked until the next cron run, up to about 50 s with a 1-minute cron and a 10 s window. Drivers sit idle during peaks.
13. Two instances can both check, see no key, and both set it. Use `SET key value NX EX 10`, which checks and sets atomically. Release with compare-and-delete.
14. A conditional update on the ride row (`WHERE driver_id IS NULL`) in the accept path, plus setting the driver `in_ride` in the same transaction. The lock only prevents duplicate pop-ups.
15. Head-of-line blocking: in one FIFO line, a slow rural request delays easy city requests. Region partitions isolate them, and regions map to the rest of the sharding.
16. The loop's state (which driver, how much time left) is persisted. A crashed worker's workflow resumes where it stopped instead of restarting or being lost.

</details>
