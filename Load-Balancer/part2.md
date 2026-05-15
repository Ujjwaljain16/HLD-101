# SECTION 2 — CORE LOAD DISTRIBUTION ALGORITHMS

---

# Why This Section Exists

Section 1 established:

* why reverse proxies emerge,
* why traffic coordination infrastructure exists.

But once multiple backend servers exist,
a new question immediately appears:

> HOW should traffic actually be distributed?

At first glance, this looks trivial:

* “just send equal traffic everywhere.”

But production systems quickly reveal:
equal request count does NOT mean equal work.

Examples:

* one request may take 1ms,
* another may take 2 seconds,
* one connection may carry 1 request,
* another may multiplex 1000 streams,
* one user may generate 2 requests,
* another may generate 10,000.

This section studies:

> how distributed systems attempt to balance work under incomplete information and constantly changing conditions.

This is where load balancing evolves from:

* “traffic splitting”
  into:
* “real-time workload coordination.”

---

# The Fundamental Problem

The central challenge is deceptively difficult.

A load balancer must continuously answer:

> Which backend should receive the next request?

But it must answer using:

* incomplete metrics,
* delayed observations,
* partial failures,
* changing traffic patterns,
* heterogeneous servers,
* and non-uniform workloads.

A critical insight:

> load balancing algorithms are fundamentally prediction systems.

They attempt to predict:

* which backend will provide the best outcome,
  using imperfect signals.

This means:
all algorithms are approximations.

There is NO perfect balancing algorithm.

Only:

* different trade-offs,
* under different workload realities.

---

# What Is ACTUALLY Being Balanced?

Before studying algorithms,
we must establish a foundational concept.

One of the biggest beginner mistakes is assuming:

> “all requests are equal.”

They are not.

Different systems balance different units.

---

# Possible Units of “Load”

| Unit           | Example               |
| -------------- | --------------------- |
| Packets        | L4 packet forwarding  |
| Flows          | TCP sessions          |
| Connections    | HTTP keepalive        |
| Requests       | HTTP request routing  |
| Streams        | HTTP/2 multiplexing   |
| Sessions       | Sticky affinity       |
| Queue depth    | Least-request systems |
| CPU time       | Adaptive balancing    |
| Latency budget | EWMA routing          |

This distinction fundamentally changes algorithm correctness.

Example:

A server with:

* 10 TCP connections,
  may actually be handling:
* 1000 HTTP/2 streams.

An L4 least-connections algorithm sees:

* “10 connections.”

An L7 least-requests algorithm sees:

* “1000 active requests.”

The routing decisions become radically different.

---

# The Hidden Queueing Theory Narrative

Another critical systems insight:

> load balancing is largely queue management.

Servers rarely fail instantly.

Instead:

* queues grow,
* waiting time increases,
* latency explodes,
* retries amplify pressure,
* and throughput collapses.

Most advanced algorithms exist to:

* prevent queue buildup,
* reduce tail latency,
* smooth concurrency,
* and stabilize overload behavior.

This queue-centric mental model is essential.

---

# Static vs Dynamic Algorithms

All balancing algorithms fall into two major categories.

---

# Static Algorithms

Static algorithms:

* do NOT use live backend feedback.

Routing decisions are predetermined.

Examples:

* round robin,
* weighted round robin,
* hashing.

---

## Advantages

* simple,
* extremely fast,
* minimal coordination overhead,
* predictable behavior,
* scalable.

Often:

* <1ms routing overhead. 

---

## Weaknesses

They assume:

* servers are equal,
* requests cost similar amounts,
* infrastructure is stable.

Production systems violate ALL these assumptions.

---

# Dynamic Algorithms

Dynamic algorithms:

* observe backend state,
* adapt routing decisions in real time.

Examples:

* least requests,
* least connections,
* least response time,
* EWMA latency balancing.

---

## Advantages

They adapt to:

* uneven workloads,
* degraded servers,
* variable request cost,
* changing infrastructure conditions.

They can reduce:

* tail latency,
* queue buildup,
* overload hotspots.

---

## Weaknesses

They require:

* fresh metrics,
* coordination,
* state propagation,
* and feedback loops.

This introduces:

* instability risk,
* oscillation,
* stale metric problems,
* and operational complexity.

---

# Round Robin — The Simplest Algorithm

---

# Definition

Round robin distributes requests sequentially:

Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A

And so on.

---

# Why It Exists

Round robin assumes:

* all servers are equivalent,
* requests have similar cost.

This makes it:

* extremely simple,
* stateless,
* and computationally cheap.

---

# Why It Works Surprisingly Well Initially

If:

* servers are homogeneous,
* requests are uniform,
* traffic is stable,

then round robin achieves:

* near-even distribution,
* minimal overhead,
* excellent throughput.

This simplicity is why many systems begin here.

---

# Hidden Assumption

Round robin secretly assumes:

> request count ≈ work performed

This assumption breaks badly in production.

---

# Failure Example — Uneven Request Cost

Suppose:

* 9 servers handle:

  * 2000 RPS capacity each.
* 1 degraded server handles:

  * only 1000 RPS.

Traffic:

* 15,000 RPS total.

Round robin sends:

* 1500 RPS to every server.

Result:

* degraded server overloads,
* queue grows by 500 RPS continuously,
* p99 latency explodes,
* retries amplify pressure,
* failures spread outward. 

This demonstrates a critical distributed systems reality:

> equal distribution can create unequal overload.

---

# Weighted Round Robin

---

# Definition

Weighted round robin distributes traffic proportionally.

Example:

* Server A weight = 2
* Server B weight = 1

Traffic pattern:
A → A → B → A → A → B

---

# Why It Exists

Real fleets are often heterogeneous:

* different CPU generations,
* different memory,
* different hardware,
* different regional capacity.

Weighted RR approximates:

* proportional load distribution.

---

# Hidden Limitation

Weights are:

* static assumptions.

They do NOT adapt dynamically to:

* sudden degradation,
* GC pauses,
* noisy neighbors,
* partial failures.

Thus:
weighted RR still lacks runtime awareness.

---

# Least Connections

---

# Definition

Route traffic to:

> the backend with the fewest active connections.

---

# Intuition

A server with:

* fewer active connections,
  is assumed to:
* have more spare capacity.

---

# Why It Improves on Round Robin

Least connections partially accounts for:

* variable request duration.

Example:

* long-running uploads,
* streaming responses,
* WebSockets.

Servers handling many long-lived requests naturally accumulate:

* more active connections.

The algorithm shifts traffic elsewhere.

---

# Hidden Problem — Connection Count ≠ Actual Load

Modern protocols break this assumption.

Example:
HTTP/2 multiplexing.

One TCP connection may carry:

* hundreds of concurrent streams.

L4 least-connections sees:

* “1 connection.”

But backend reality may be:

* severe overload.

This creates massive imbalance.

---

# Least Requests / Least In-Flight

---

# Definition

Route requests to:

> the backend with the fewest active requests.

---

# Why This Is Better

Least requests measures:

* actual concurrent work,
  not merely:
* connection ownership.

This better reflects:

* backend pressure,
* queue buildup,
* resource utilization.

Especially important for:

* HTTP/2,
* gRPC,
* multiplexed systems.

---

# Queueing Theory Connection

Least requests attempts to:

* minimize queue growth.

Why?

Because:
queue wait time increases non-linearly under saturation.

Once queues grow:

* tail latency explodes rapidly.

A backend at:

* 95% utilization,
  can experience vastly worse latency than:
* one at 70%.

---

# The Hidden Problem — Metric Freshness

Dynamic balancing depends on:

* live state accuracy.

But metrics propagate with delay.

Example:

* metrics refresh every 5 seconds.

During those 5 seconds:
many proxies may observe:

* “Server B is idle!”

All simultaneously route traffic there.

Result:

* sudden overload spike,
* oscillation,
* backend thrashing. 

This introduces a major distributed systems law:

> delayed feedback creates instability.

---

# Power of Two Choices — One of the Most Important Algorithms

This is one of the deepest and most elegant production algorithms.

---

# The Core Problem

Naive least-load systems require:

* querying ALL backends.

At:

* 1000 servers,
  this becomes expensive and coordination-heavy.

---

# The Algorithm

For each request:

1. Randomly sample TWO servers.
2. Compare load.
3. Choose the less-loaded one.

That’s it.

---

# Why This Is Brilliant

A single extra random choice produces:

* exponentially better distribution.

Mathematically:
maximum load drops dramatically:

* from roughly:
  log(n)/log(log(n))
  to:
  log(log(n)).

For:

* 1000 servers,
  worst-case imbalance drops roughly:
* ~145 → ~3. 

This is an astonishing result.

---

# Why It Works Operationally

Benefits:

* O(1) decision cost,
* no global coordination,
* local proxy state only,
* scalable,
* excellent distribution.

This becomes extremely attractive at scale.

---

# Hidden Insight

This algorithm reveals a profound systems-design principle:

> small amounts of randomness dramatically improve large distributed systems.

Randomization helps prevent:

* synchronization,
* hotspots,
* coordinated collapse.

This principle appears repeatedly later:

* jitter,
* retry randomization,
* backoff spreading,
* load shedding.

---

# Consistent Hashing — Stability Over Perfect Balance

---

# The Core Problem

Suppose:
hash(user_id) % N

With:

* 10 servers → 11 servers,
  almost ALL mappings change.

Result:

* cache invalidation,
* session migration,
* massive cold-cache storms.

---

# Consistent Hashing Idea

Map:

* servers,
* and keys,
  onto:
* a circular hash ring.

A key routes to:

* the next clockwise server.

Adding/removing servers only remaps:

* neighboring keys.

Thus:

* only ~1/N keys move. 

---

# Why This Matters

This preserves:

* cache locality,
* session affinity,
* routing stability.

Consistency here means:

* stable mapping under topology changes,
  NOT:
* distributed consistency models.

---

# Virtual Nodes

Physical server placement may be uneven.

Solution:

* each server appears multiple times on the ring.

Example:
serverA_1
serverA_2
serverA_3
...

This smooths imbalance.

---

# Hidden Trade-Off

Consistent hashing prioritizes:

* stability,
* locality.

NOT:

* perfect load fairness.

Hot keys become dangerous.

Example:

* celebrity profile,
* viral video,
* huge tenant.

All traffic pins to:

* one backend.

This creates:

* hotspot collapse.

---

# Locality vs Uniformity — A Fundamental Distributed Trade-Off

This is a recurring systems law.

Systems often choose between:

| Goal                 | Benefit                    | Cost           |
| -------------------- | -------------------------- | -------------- |
| Uniform distribution | Better fairness            | Worse locality |
| Locality/stickiness  | Better cache/session reuse | Hotspots       |

This trade-off appears everywhere:

* sticky sessions,
* caches,
* CDNs,
* partitioning,
* consistent hashing.

---

# Tail Latency — The Real Enemy

A critical production insight:

> average latency is usually misleading.

Users experience:

* tail latency,
  not averages.

Example:

* p50 = 20ms
* p99 = 10 seconds

System “looks healthy” statistically,
while users suffer massively.

Most modern balancing algorithms primarily exist to:

* reduce p95/p99 latency,
* prevent queue explosion,
* isolate overload early.

This is one of the deepest hidden narratives across all production infrastructure.

---

# Control Theory Hidden Narrative

Modern balancing algorithms are fundamentally:

> distributed feedback-control systems.

Examples:

* least requests,
* EWMA routing,
* adaptive weighting,
* slow start,
* retry budgets,
* autoscaling.

All attempt to:

* stabilize systems under changing conditions.

But delayed metrics create:

* oscillation,
* overshoot,
* feedback instability.

This hidden control-theory perspective is essential for advanced systems thinking.

---

# Observability Distortion

Another advanced operational reality:

Metrics themselves become misleading.

Examples:

| Metric              | Hidden Distortion         |
| ------------------- | ------------------------- |
| Average latency     | Hides tail collapse       |
| Connection count    | Hides multiplexed streams |
| Cluster CPU average | Hides hotspots            |
| Retry success rate  | Hides original failures   |
| Cache hit ratio     | Hides backend pressure    |

Thus:
routing decisions may optimize for:

* incorrect reality.

This becomes a major production challenge.

---

# Algorithm Evolution Narrative

The evolution across algorithms follows a clear systems progression.

---

# Phase 1 — Static Simplicity

Round robin.

Problem:
uneven workloads.

---

# Phase 2 — Runtime Awareness

Least requests.

Problem:
metric freshness and instability.

---

# Phase 3 — Scalable Adaptation

Power of two choices.

Problem:
still imperfect under hotspots.

---

# Phase 4 — Locality Preservation

Consistent hashing.

Problem:
fairness vs locality trade-offs.

---

# Phase 5 — Adaptive Stability Systems

EWMA,
adaptive weighting,
queue-aware balancing,
capacity signals.

Problem:
distributed control instability.

This evolution is driven by:

> increasingly difficult coordination problems under scale.

---

# Connection to Next Sections

This section explained:
HOW systems distribute work.

But another major problem emerges naturally:

> sometimes systems intentionally BREAK uniform distribution.

Why?

Because:

* locality,
* session reuse,
* cache affinity,
* user stickiness,
  can improve performance dramatically.

That leads directly into:

* sticky sessions,
* affinity,
* consistent locality,
* and stateful routing.

The next section studies:

> why systems intentionally trade fairness for locality.

---

# Quick Summary

* Load balancing algorithms attempt to distribute work under incomplete information.
* Equal request count does NOT mean equal backend load.
* Static algorithms are simple and fast but cannot adapt to runtime conditions.
* Dynamic algorithms reduce overload and tail latency but introduce coordination complexity.
* Least requests better reflects actual work than least connections in modern multiplexed protocols.
* Power of two choices achieves near-optimal balancing using only local random sampling.
* Consistent hashing prioritizes locality and stable mappings over perfect fairness.
* Most balancing problems are actually queue-management and stability problems.
* Delayed metrics create oscillation and instability in adaptive systems.
* Tail latency, not average latency, dominates real production experience.
* Modern load balancing algorithms are fundamentally distributed feedback-control systems operating under incomplete and distorted observability.
