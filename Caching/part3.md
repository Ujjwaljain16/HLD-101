# SECTION 3 — WHERE CACHING HAPPENS (CACHE PLACEMENT LAYERS)

---

# SECTION GOAL

Most beginners think:
“cache = Redis between app and database.”

That is only one layer.

Real systems contain:
multiple overlapping cache layers operating at different distances from users, computation, and storage.

This section explains:

* where caches exist,
* why different layers exist,
* what each layer optimizes,
* how layers interact,
* and how cache placement changes latency, consistency, and operational complexity.

This section is critical because:
cache placement fundamentally determines:

* latency profile,
* scalability limits,
* coherence difficulty,
* blast radius,
* and infrastructure cost.

---

# Core Mental Model For This Section

The closer a cache is to:

* the user,
* the CPU,
* or the application,

the faster it usually becomes.

But:
closer caches are also:

* smaller,
* less globally coherent,
* and harder to synchronize.

Caching architecture is fundamentally:
a hierarchy of speed vs coordination tradeoffs.

---

# High-Level Cache Hierarchy

Typical large-scale systems contain multiple cache layers simultaneously:

[Diagram]

User Device
↓
Browser Cache / Client Cache
↓
CDN Edge Cache
↓
Reverse Proxy / Gateway Cache
↓
Application-Level Distributed Cache
↓
In-Process Cache
↓
Database Buffer Pool
↓
Disk Storage

Each layer exists because:
avoiding work earlier in the request path is usually cheaper.

---

─────────────────────────────────────────────

# 3.1 Client-Side Caching

─────────────────────────────────────────────

# The One-Line Definition

Client-side caching stores reusable data directly on the user’s device to avoid unnecessary network requests entirely.

---

# Intuition First

[Intuition]

Imagine repeatedly downloading the same textbook every time you open it.

That would be absurd.

Instead:
your laptop stores a local copy.

Future reads become instant because:
network retrieval is avoided entirely.

Browsers behave exactly this way.

---

# [Analogy]

Think of carrying a water bottle.

Without it:
you walk to the water station repeatedly.

With local storage:
you eliminate most trips entirely.

Client-side caching eliminates server trips the same way.

---

# The Problem It Solves

Without client caching:
every interaction requires:

* network RTT,
* server processing,
* backend infrastructure work,
* bandwidth consumption.

This creates:

* unnecessary latency,
* repeated traffic,
* origin overload,
* and poor user experience.

Especially harmful for:

* static assets,
* repeated API responses,
* mobile networks,
* offline-capable apps.

---

# The Core Idea (Precise)

The client device stores previously fetched resources locally so future accesses can bypass the network entirely.

Common client-side cache types:

* browser HTTP cache
* localStorage
* IndexedDB
* mobile app local memory
* service-worker caches
* SDK metadata caches

Resources commonly cached:

* images
* CSS
* JavaScript bundles
* API responses
* sessions
* user preferences
* offline data

---

# How It Works — Step By Step

## Browser Cache Example

1. User requests webpage

2. Browser downloads:

   * HTML
   * CSS
   * JS
   * images

3. Browser stores resources locally

4. User revisits page

5. Browser checks local cache

6. Cached assets reused directly

7. Network requests avoided or minimized

---

# Conditional Revalidation Flow

1. Cached item expires

2. Browser sends:

   * ETag
   * If-Modified-Since

3. Server checks freshness

4. If unchanged:

   * returns 304 Not Modified

5. Browser reuses local copy

This dramatically reduces bandwidth.

---

# Worked Example

Suppose:
homepage logo:

* 500 KB image.

Daily active users:

* 10 million.

Without browser caching:
all users repeatedly redownload image.

Bandwidth cost enormous.

With browser caching:
logo downloaded once,
reused repeatedly.

Infrastructure savings become massive.

---

# Visual / Diagram Description

[Diagram]

First Request:

Browser
↓
Origin Server
↓
Asset Download
↓
Stored Locally

Future Request:

Browser
↓
Local Cache Hit
↓
Immediate Render

No network round trip required.

---

# Key Properties and Characteristics

* Avoids network entirely
* Extremely low latency
* Reduces bandwidth consumption
* Improves offline capability
* Reduces origin traffic
* Limited backend control
* Freshness harder to guarantee

---

# Trade-offs

| Advantage            | Cost / Limitation          |
| -------------------- | -------------------------- |
| Zero network latency | Hard invalidation          |
| Lower bandwidth cost | Stale client copies        |
| Better UX            | Browser/device differences |
| Offline support      | Limited storage            |
| Reduced server load  | Less centralized control   |

---

# Failure Modes

## Stale Assets

Old JS/CSS versions remain cached after deployment.

---

## Cache Poisoning

Incorrect responses cached locally.

---

## Version Drift

Different users operate on different cached versions.

---

## Offline Divergence

Local state becomes inconsistent with server truth.

---

# When To Use This

Critical for:

* static assets,
* mobile applications,
* offline-first systems,
* repeated API reads,
* media-heavy applications.

---

# When NOT To Use This

Avoid aggressive client caching for:

* highly dynamic sensitive data,
* strict consistency systems,
* rapidly changing financial data.

---

# Common Mistakes and Misconceptions

## Misconception

“Client cache is always safe.”

Reality:
invalidation becomes extremely difficult.

---

## Misconception

“Browser caching only matters for images.”

Reality:
entire applications often depend heavily on it.

---

# Connection To Other Concepts

Connects directly to:

* CDN caching
* HTTP cache headers
* ETags
* stale-while-revalidate
* offline systems
* service workers
* consistency tradeoffs

---

# Quick Summary

[Quick Summary]

* Client caching avoids network requests entirely.
* Browser caches dramatically reduce repeated traffic.
* Client-side invalidation is difficult.
* Local storage improves latency and offline capability.
* Freshness becomes harder to control centrally.

---

─────────────────────────────────────────────

# 3.2 CDN Caching (Edge Caching)

─────────────────────────────────────────────

# The One-Line Definition

A CDN cache stores content at geographically distributed edge servers close to users to reduce latency and origin traffic.

---

# Intuition First

[Intuition]

Suppose:
your company has one warehouse in New York,
but customers exist globally.

Shipping every product internationally:
creates massive delays.

Instead:
you place regional warehouses near users.

CDNs behave exactly this way.

They move data physically closer to consumers.

---

# The Problem It Solves

Without CDN caching:
every request travels to origin servers.

Problems:

* high latency,
* cross-continent RTTs,
* origin overload,
* bandwidth concentration,
* scalability bottlenecks.

Especially severe for:

* images,
* video,
* JS bundles,
* static assets,
* global traffic.

---

# The Core Idea (Precise)

A CDN (Content Delivery Network) distributes cached copies of content across globally distributed edge locations called PoPs (Points of Presence).

Users fetch content from:
nearest edge node
instead of origin infrastructure.

Common CDN providers:

* Cloudflare
* Akamai
* Fastly

CDNs commonly cache:

* images
* videos
* static assets
* API responses
* HTML pages

---

# How It Works — Step By Step

## CDN Cache Hit

1. User requests image
2. DNS routes request to nearest edge PoP
3. Edge checks cache
4. Asset exists
5. Response returned locally

Origin server never contacted.

---

## CDN Cache Miss

1. User requests content
2. Edge cache misses
3. Edge fetches content from origin
4. Content cached at edge
5. Response returned
6. Future nearby users become hits

---

# Worked Example

Suppose:
origin server located in Virginia.

User located in India.

Without CDN:
every request crosses continents:

* 250–300ms latency.

With CDN:
request served from nearby India PoP:

* 20–40ms latency.

Massive improvement.

---

# Visual / Diagram Description

[Diagram]

Without CDN:

User (India)
↓
Internet Backbone
↓
US Origin Server
↓
Long RTT

---

With CDN:

User (India)
↓
Nearby Edge PoP
↓
Immediate Local Response

Origin avoided entirely.

---

# Key Properties and Characteristics

* Geographic distribution
* Reduces origin traffic
* Dramatically lowers RTT
* Excellent for static content
* Improves global scalability
* Often hierarchical
* Can absorb massive traffic spikes

---

# Trade-offs

| Advantage              | Cost / Limitation               |
| ---------------------- | ------------------------------- |
| Lower global latency   | Stale edge copies               |
| Origin protection      | Invalidations propagate slowly  |
| Better scalability     | Extra infrastructure complexity |
| Lower bandwidth cost   | Personalized content harder     |
| Better DDoS absorption | Cache fragmentation             |

---

# Failure Modes

## Edge Staleness

Different regions serve different versions temporarily.

---

## Cache Fragmentation

Too many cache-key variants reduce hit rates.

---

## Global Invalidation Delays

Purges propagate slowly across edge network.

---

## Origin Stampedes

Simultaneous misses overload origin.

---

# When To Use This

Critical for:

* global applications,
* media delivery,
* static assets,
* large read-heavy workloads,
* video streaming.

---

# When NOT To Use This

Less useful for:

* highly personalized responses,
* rapidly changing transactional data,
* user-specific private state.

---

# Common Mistakes and Misconceptions

## Misconception

“CDNs only cache images.”

Reality:
modern CDNs cache:

* APIs,
* HTML,
* edge-computed responses.

---

## Misconception

“CDN eliminates origin traffic.”

Reality:
cache misses and invalidations still reach origin.

---

# Connection To Other Concepts

Connects directly to:

* edge computing
* stale-while-revalidate
* origin shielding
* hierarchical caching
* cache keys
* geographic routing
* hot content distribution

---

# Quick Summary

[Quick Summary]

* CDNs move cached content geographically closer to users.
* Edge hits dramatically reduce RTT.
* CDNs reduce origin traffic massively.
* Invalidation and coherence become globally distributed problems.
* CDNs are specialized distributed cache systems.

---

─────────────────────────────────────────────

# 3.3 Reverse Proxy / Gateway Caching

─────────────────────────────────────────────

# The One-Line Definition

Reverse proxy caching stores reusable responses at infrastructure gateways before requests reach application servers.

---

# Intuition First

[Intuition]

Imagine:
a receptionist answering repetitive questions before customers reach specialists.

Without receptionist filtering:
specialists become overloaded with repetitive work.

Reverse proxy caches behave similarly.

They absorb repeated requests before backend services see them.

---

# The Problem It Solves

Without gateway caching:
every request reaches:

* application servers,
* service meshes,
* backend APIs.

This creates unnecessary:

* CPU usage,
* request processing,
* thread allocation,
* serialization overhead.

---

# The Core Idea (Precise)

A reverse proxy cache sits between:
clients
and
application servers.

It caches reusable HTTP responses and serves them directly without invoking backend logic.

Common technologies:

* Varnish
* NGINX
* Envoy
* API gateways

Frequently cached:

* public APIs
* rendered pages
* static responses
* metadata endpoints

---

# How It Works — Step By Step

1. Client request arrives

2. Reverse proxy checks cache

3. If hit:
   response returned directly

4. If miss:
   backend request forwarded

5. Response cached

6. Future requests become hits

---

# Worked Example

Suppose:
homepage API:

* 200,000 RPS.

Most users receive identical response.

Gateway cache absorbs:

* 95% traffic.

Application servers now process:

* only 10,000 RPS.

Massive infrastructure savings.

---

# Visual / Diagram Description

[Diagram]

Client
↓
Reverse Proxy Cache
├── Hit → Immediate Response
└── Miss → App Servers
↓
Database

---

# Key Properties and Characteristics

* Infrastructure-level caching
* Reduces app-server load
* Fast HTTP-level reuse
* Centralized policy enforcement
* Good for public/shared responses

---

# Trade-offs

| Advantage           | Limitation             |
| ------------------- | ---------------------- |
| Lower backend CPU   | Harder personalization |
| Better scalability  | Stale response risk    |
| Simpler app servers | Complex cache rules    |
| Lower latency       | Cache coherence issues |

---

# Failure Modes

## Incorrect Cache Keys

Users receive wrong cached responses.

---

## Personalized Data Leakage

Shared cache accidentally exposes private content.

---

## Cache Bypass Storms

Large miss bursts overload app layer.

---

# When To Use This

Useful for:

* public APIs,
* shared content,
* expensive rendering,
* high read traffic.

---

# When NOT To Use This

Avoid for:

* highly user-specific responses,
* rapidly mutating state,
* strongly personalized feeds.

---

# Common Mistakes and Misconceptions

## Misconception

“Gateway cache replaces app caching.”

Reality:
multiple cache layers usually coexist.

---

# Connection To Other Concepts

Connects directly to:

* API gateways
* HTTP caching
* CDN architecture
* shared-cache semantics
* edge infrastructure

---

# Quick Summary

[Quick Summary]

* Reverse proxy caches absorb requests before app servers.
* Shared responses become extremely scalable.
* Gateway caches reduce backend CPU usage.
* Incorrect cache-key design is dangerous.
* Multi-layer caching is common in production systems.

---

─────────────────────────────────────────────

# 3.4 Distributed Application Caching

─────────────────────────────────────────────

# The One-Line Definition

Distributed application caching uses shared cache clusters accessed by multiple application servers to store reusable data centrally.

---

# Intuition First

[Intuition]

Imagine:
multiple office workers sharing a central reference library.

Instead of each worker maintaining separate copies,
everyone accesses:
one shared fast-access knowledge system.

Distributed caches work similarly.

---

# The Problem It Solves

Single-machine caches fail because:

* app servers scale horizontally,
* local memory is isolated,
* cache state becomes fragmented,
* deployments lose local cache state.

Systems need:
shared reusable memory across fleet instances.

---

# The Core Idea (Precise)

A distributed cache cluster:

* stores shared reusable state,
* partitions data across nodes,
* serves multiple application instances simultaneously.

Common technologies:

* Redis
* Memcached

Common cached objects:

* sessions
* user profiles
* feed results
* recommendation objects
* metadata
* precomputed views

---

# How It Works — Step By Step

1. App receives request
2. Cache key generated
3. Request sent to distributed cache cluster
4. Correct node located
5. Cache checked
6. Hit returned OR backend queried

---

# Worked Example

Suppose:
100 application servers exist.

Without distributed cache:
each server independently queries DB.

With Redis cluster:
all servers share reusable hot data.

Backend pressure drops dramatically.

---

# Visual / Diagram Description

[Diagram]

Application Servers
├── App 1
├── App 2
├── App 3
↓
Shared Distributed Cache Cluster
↓
Database

---

# Key Properties and Characteristics

* Shared reusable state
* Horizontally scalable
* Centralized cache layer
* High throughput
* Network-based access
* Multi-node architecture

---

# Trade-offs

| Advantage                | Cost                   |
| ------------------------ | ---------------------- |
| Shared reuse             | Network RTT            |
| Better scalability       | Operational complexity |
| Centralized invalidation | Cluster coordination   |
| Higher hit rates         | Cache dependency risk  |

---

# Failure Modes

## Cluster Partitioning

Cache nodes become unreachable.

---

## Hot-Key Saturation

Single node overloaded.

---

## Stampede Events

Massive concurrent misses overload backend.

---

## Cache Dependency Collapse

Backend overwhelmed after cache outage.

---

# When To Use This

Critical when:

* horizontally scaled applications exist,
* shared reuse valuable,
* backend load high,
* hot objects reused globally.

---

# When NOT To Use This

Less useful when:

* application tiny,
* local memory sufficient,
* shared reuse minimal.

---

# Common Mistakes and Misconceptions

## Misconception

“Distributed caches are simple.”

Reality:
they become full distributed systems.

---

# Connection To Other Concepts

Connects directly to:

* consistent hashing
* replication
* Redis clusters
* hot keys
* invalidation
* coherence
* failover

---

# Quick Summary

[Quick Summary]

* Distributed caches provide shared reusable memory.
* Multiple app servers share hot data centrally.
* Redis/Memcached are common implementations.
* Distributed caches become distributed systems themselves.
* Cache outages can become catastrophic infrastructure events.

---

─────────────────────────────────────────────

# 3.5 In-Process Caching

─────────────────────────────────────────────

# The One-Line Definition

In-process caching stores reusable data directly inside application memory for ultra-low-latency local access.

---

# Intuition First

[Intuition]

Instead of:
walking to the shared office library every time,
you keep frequently used notes directly on your desk.

That avoids even internal travel overhead.

In-process caching works similarly.

---

# The Problem It Solves

Even distributed caches require:

* network RTT,
* serialization,
* socket communication,
* remote lookups.

Very hot tiny objects may justify:
keeping data directly inside process memory.

---

# The Core Idea (Precise)

Application process stores frequently accessed objects directly in local RAM structures.

Examples:

* feature flags
* config values
* rate-limiter counters
* tiny metadata
* reference tables

Common implementations:

* in-memory hash maps
* LRU maps
* Guava Cache
* Caffeine

---

# How It Works — Step By Step

1. Request arrives

2. App checks local memory map

3. If hit:
   immediate response

4. If miss:
   distributed cache or DB queried

5. Local cache updated

---

# Worked Example

Suppose:
feature-flag lookup happens:

* millions of times/second.

Redis lookup:

* 1ms.

Local memory lookup:

* nanoseconds to microseconds.

Massive CPU savings.

---

# Visual / Diagram Description

[Diagram]

Request
↓
Application Process
├── Local Memory Cache Hit
│     ↓
│   Immediate Response
│
└── Miss
↓
Distributed Cache / DB

---

# Key Properties and Characteristics

* Fastest cache layer
* Zero network overhead
* Extremely low latency
* Per-process isolated state
* Small capacity
* Hard synchronization

---

# Trade-offs

| Advantage               | Limitation             |
| ----------------------- | ---------------------- |
| Fastest possible access | No shared coherence    |
| No network RTT          | Duplicate cache copies |
| Lower CPU overhead      | Hard invalidation      |
| Reduced Redis load      | Memory duplication     |

---

# Failure Modes

## Version Drift

Different instances hold different values.

---

## Stale Local State

Invalidation propagation delayed.

---

## Memory Explosion

Large local caches multiply across instances.

---

# When To Use This

Useful for:

* tiny hot objects,
* feature flags,
* configs,
* reference metadata,
* extremely hot counters.

---

# When NOT To Use This

Avoid for:

* highly dynamic shared state,
* large datasets,
* strong consistency requirements.

---

# Common Mistakes and Misconceptions

## Misconception

“In-process cache replaces Redis.”

Reality:
it usually supplements distributed caching.

---

# Connection To Other Concepts

Connects directly to:

* CPU locality
* memory hierarchy
* distributed invalidation
* hot objects
* multi-tier caching

---

# Quick Summary

[Quick Summary]

* In-process caches are the fastest cache layer.
* They avoid all network overhead.
* Each process maintains isolated state.
* Synchronization becomes difficult.
* Usually used as L1 cache above distributed caches.

---

─────────────────────────────────────────────

# SHORT BRIDGE

So far we established:

* caching exists across many architectural layers,
* each layer optimizes different latency and coordination costs,
* and cache placement fundamentally shapes scalability and coherence behavior.

Next sections will build on this foundation to explain:

* HOW applications interact with caches,
* WHO owns loading and invalidation,
* and WHY different cache interaction patterns create different consistency and operational tradeoffs.
