# SECTION 8 — EVICTION, ADMISSION, AND MODERN CACHE MEMORY MANAGEMENT

---

# SECTION GOAL

This section explains one of the deepest truths in caching systems:

> memory is the most valuable constrained resource in large-scale infrastructure.

At beginner level:
cache eviction appears simple:

* “remove old entries.”

At production scale:
eviction becomes:

* probabilistic prediction,
* workload modeling,
* thermal working-set protection,
* and memory economics engineering.

This section explains:

* why eviction exists,
* how classic policies work,
* why traditional policies fail under modern workloads,
* and why modern caches increasingly prioritize:

  * admission,
  * frequency estimation,
  * pollution resistance,
  * and working-set preservation.

This is where caching begins to resemble:
an intelligent memory-management system.

---

# Core Mental Model For This Section

Caches fundamentally answer one question repeatedly:

> “Given finite memory,
> which objects deserve to stay?”

Everything in this section exists because:
memory is finite,
while potential data is effectively infinite.

Eviction and admission policies attempt to maximize:

\text{Future Cache Utility}

But:
future access patterns are uncertain.

Thus:
cache memory management becomes:
a prediction problem under uncertainty.

---

# Hidden Systems Insight

Modern cache systems are fundamentally:

> probabilistic future-reuse estimators.

This is the conceptual leap most beginners miss.

Caches are not just:
“hash maps with expiration.”

They are:

* statistical systems,
* locality exploitation engines,
* and predictive working-set preservation mechanisms.

---

# The Evolution Narrative

Cache memory management evolved roughly like this:

| Era                    | Focus                         |
| ---------------------- | ----------------------------- |
| Early caches           | Simple eviction               |
| Larger systems         | Recency optimization          |
| Internet-scale systems | Frequency optimization        |
| Modern systems         | Pollution-resistant admission |
| Advanced systems       | Predictive thermal management |

This evolution happened because:
real-world workloads became:

* noisier,
* burstier,
* larger,
* and more adversarial.

---

─────────────────────────────────────────────

# 8.1 Why Eviction Exists

─────────────────────────────────────────────

# The One-Line Definition

Eviction removes cache entries when memory capacity becomes full so higher-value objects can remain resident.

---

# Intuition First

[Intuition]

Imagine:
your study desk has limited space.

Eventually:
new materials arrive.

You must decide:
which old materials to remove.

The challenge:
you do not know exactly which notes you will need tomorrow.

Caches face the same uncertainty continuously.

---

# The Problem It Solves

Memory is finite.

Working sets evolve continuously.

Eventually:
new entries arrive after cache fills.

Without eviction:
cache stops accepting new objects entirely.

Systems need:
a replacement strategy.

---

# The Core Idea (Precise)

When cache capacity exceeded:
system selects existing entries for removal.

Goal:
maximize future hit probability.

Eviction attempts to preserve:
high future reuse objects.

---

# Hidden Systems Insight

Eviction is fundamentally:

> a prediction problem.

The cache tries to estimate:
which objects are least likely to be useful soon.

This prediction is imperfect.

Therefore:
all eviction algorithms are heuristics.

---

# How It Works — Step By Step

1. New object arrives
2. Cache full
3. Eviction policy selects victim
4. Victim removed
5. New object inserted

This process happens continuously under load.

---

# Request Lifecycle Reality

At production scale:
eviction itself consumes resources:

* CPU cycles,
* metadata tracking,
* synchronization,
* memory writes,
* background cleanup.

Eviction policies therefore must balance:

* quality,
* overhead,
* scalability.

---

# Worked Example

Suppose:
cache capacity:

* 1 million objects.

New hot object arrives.

Cache full.

Eviction policy removes:
least valuable entry.

Future hit rate depends heavily on:
whether this choice was correct.

---

# Visual / Diagram Description

[Diagram]

Cache Full
↓
New Entry Arrives
↓
Eviction Policy Chooses Victim
↓
Old Entry Removed
↓
New Entry Inserted

---

# Key Properties and Characteristics

* Finite memory requires replacement
* Eviction continuous under load
* Policies estimate future usefulness
* Prediction always imperfect
* Policy quality strongly affects hit rates

---

# Trade-offs

| Goal              | Tension                          |
| ----------------- | -------------------------------- |
| Better hit rates  | More algorithm complexity        |
| Better prediction | More metadata overhead           |
| Lower CPU cost    | Simpler but weaker policies      |
| Better stability  | Higher implementation complexity |

---

# Failure Modes

## Wrong Evictions

Hot objects removed accidentally.

---

## Thrashing

Objects constantly evicted/reloaded.

---

## Metadata Overhead Explosion

Tracking information consumes too much memory.

---

# When To Use Sophisticated Policies

Necessary when:

* memory constrained,
* workloads noisy,
* backend misses expensive,
* traffic massive.

---

# When Simpler Policies May Work

Simple eviction acceptable when:

* workloads stable,
* memory abundant,
* low concurrency.

---

# Common Mistakes and Misconceptions

## Misconception

“Eviction is only cleanup.”

Reality:
eviction quality fundamentally shapes scalability.

---

# Connection To Other Concepts

Connects directly to:

* working sets
* locality
* admission policies
* hit rates
* pollution
* memory economics

---

# Quick Summary

[Quick Summary]

* Eviction exists because memory is finite.
* Policies estimate future reuse value.
* Eviction quality strongly impacts hit rates.
* All eviction algorithms are predictive heuristics.
* Modern cache systems treat memory as a scarce resource.

---

─────────────────────────────────────────────

# 8.2 FIFO (First-In First-Out)

─────────────────────────────────────────────

# The One-Line Definition

FIFO evicts the oldest inserted cache entries first, regardless of access frequency or recency.

---

# Intuition First

[Intuition]

Imagine:
a waiting line.

The earliest arrival leaves first,
even if they are still important.

FIFO caches behave similarly.

---

# The Problem It Solves

Systems need:
extremely simple eviction mechanisms.

FIFO minimizes:

* bookkeeping,
* metadata,
* CPU overhead.

---

# The Core Idea (Precise)

Objects inserted earliest:
evicted earliest.

No tracking of:

* reuse,
* popularity,
* or recent access.

---

# How It Works — Step By Step

1. Object inserted
2. Added to tail of queue
3. Cache fills
4. Oldest queue entry evicted first

---

# Worked Example

Cache capacity:
3 objects.

Requests:

```text id="3ijj7u"
A → B → C → D
```

FIFO eviction:
A removed first.

Even if:
A still frequently accessed.

---

# Visual / Diagram Description

[Diagram]

Queue Order:

Front → [A] [B] [C] ← Back

New Entry D Arrives
↓
Evict A
↓
Queue:
[B] [C] [D]

---

# Key Properties and Characteristics

* Very simple
* Low overhead
* No access tracking
* Poor locality awareness
* Weak prediction quality

---

# Trade-offs

| Advantage            | Limitation                 |
| -------------------- | -------------------------- |
| Minimal overhead     | Ignores popularity         |
| Easy implementation  | Weak hit-rate performance  |
| Predictable behavior | Hot objects evicted easily |

---

# Failure Modes

## Hot Object Eviction

Frequently reused objects removed accidentally.

---

## Scan Pollution

Sequential scans destroy useful cache state.

---

# When To Use This

Useful for:

* simple systems,
* hardware caches,
* low-overhead environments.

---

# When NOT To Use This

Poor choice for:

* skewed workloads,
* internet-scale systems,
* high-value working sets.

---

# Common Mistakes and Misconceptions

## Misconception

“Oldest objects least useful.”

Reality:
old objects may still be extremely hot.

---

# Connection To Other Concepts

Connects directly to:

* locality
* pollution
* working sets
* eviction quality

---

# Quick Summary

[Quick Summary]

* FIFO removes oldest inserted entries first.
* Very simple but locality-unaware.
* Frequently evicts valuable hot objects.
* Weak performance under skewed workloads.
* Mostly historical/simple-policy relevance today.

---

─────────────────────────────────────────────

# 8.3 LRU (Least Recently Used)

─────────────────────────────────────────────

# The One-Line Definition

LRU evicts the object that has not been accessed for the longest recent time.

---

# Intuition First

[Intuition]

Imagine:
keeping recently used books on your desk.

Books untouched for long periods:
more likely removed first.

LRU follows this intuition.

---

# The Problem It Solves

FIFO ignores:
actual usage behavior.

Real workloads exhibit:
temporal locality.

Recently used objects often:
become useful again soon.

---

# The Core Idea (Precise)

LRU assumes:

> recent past access predicts near-future access.

Objects accessed recently:
protected.

Objects unused recently:
evicted first.

---

# Hidden Systems Insight

LRU fundamentally exploits:
temporal locality.

This works surprisingly well because:
real-world access patterns are highly clustered in time.

---

# How It Works — Step By Step

1. Object accessed
2. Object moved to MRU (Most Recently Used) position
3. Cache fills
4. Least recently used entry evicted

---

# Worked Example

Cache capacity:
3 entries.

Access pattern:

```text id="l0a3o8"
A → B → C → A → D
```

When D arrives:
B evicted,
because B least recently used.

---

# Visual / Diagram Description

[Diagram]

Most Recently Used
↓
[A] [C] [B]
↑
Least Recently Used

New Entry D Arrives
↓
Evict B

---

# Key Properties and Characteristics

* Exploits temporal locality
* Widely used historically
* Better than FIFO
* Simple mental model
* Still vulnerable to scans

---

# Trade-offs

| Advantage                           | Limitation                    |
| ----------------------------------- | ----------------------------- |
| Good temporal locality exploitation | Scan pollution vulnerability  |
| Better hit rates than FIFO          | Metadata maintenance overhead |
| Simple reasoning                    | Frequency blindness           |

---

# Failure Modes

## Sequential Scan Destruction

Large scans evict useful working set.

---

## Frequency Blindness

Frequently reused but temporarily inactive objects removed.

---

# When To Use This

Good general-purpose choice for:

* moderate workloads,
* recency-dominated traffic.

---

# When NOT To Use This

Weak under:

* scan-heavy workloads,
* mixed burst traffic,
* highly noisy systems.

---

# Common Mistakes and Misconceptions

## Misconception

“Recently used means frequently useful.”

Reality:
single temporary scans can distort recency heavily.

---

# Connection To Other Concepts

Connects directly to:

* temporal locality
* scan pollution
* working sets
* hot/cold separation

---

# Quick Summary

[Quick Summary]

* LRU evicts least recently accessed entries.
* Exploits temporal locality effectively.
* Better than FIFO for most workloads.
* Vulnerable to scans and burst pollution.
* Historically one of the most influential policies.

---

─────────────────────────────────────────────

# 8.4 LFU (Least Frequently Used)

─────────────────────────────────────────────

# The One-Line Definition

LFU evicts objects with the lowest long-term access frequency.

---

# Intuition First

[Intuition]

Imagine:
a library tracking how often books are borrowed.

Rarely borrowed books:
removed first.

LFU behaves similarly.

---

# The Problem It Solves

LRU overreacts to:
recent temporary traffic.

Some objects are:
globally valuable long-term,
even if not accessed recently.

LFU attempts to preserve:
historically important objects.

---

# The Core Idea (Precise)

Each object tracks:
access count/frequency.

Eviction targets:
lowest-frequency objects.

---

# Hidden Systems Insight

LFU exploits:
popularity locality,
rather than:
pure recency locality.

This better models:
long-term hot objects.

---

# How It Works — Step By Step

1. Object accessed
2. Frequency counter increments
3. Cache fills
4. Lowest-frequency object evicted

---

# Worked Example

Suppose:

| Object | Access Count |
| ------ | ------------ |
| A      | 1000         |
| B      | 50           |
| C      | 2            |

New object arrives.

LFU evicts:
C.

---

# Visual / Diagram Description

[Diagram]

Objects Ranked By Frequency:

High Frequency
↓
[A:1000]
[B:50]
[C:2]
↑
Eviction Target

---

# Key Properties and Characteristics

* Preserves globally hot objects
* Better long-term popularity modeling
* More resistant to temporary scans
* Requires frequency tracking

---

# Trade-offs

| Advantage                      | Limitation        |
| ------------------------------ | ----------------- |
| Better popularity preservation | Metadata overhead |
| Better scan resistance         | Slow adaptation   |
| Stable hot-object retention    | Historical bias   |

---

# Failure Modes

## Stale Popularity Bias

Old once-popular objects remain forever.

---

## Slow Adaptation

Traffic shifts detected slowly.

---

## Counter Explosion

Frequency metadata grows expensive.

---

# When To Use This

Useful when:

* long-term popularity stable,
* hot objects persist,
* scans common.

---

# When NOT To Use This

Weak when:

* workloads shift rapidly,
* recency matters heavily.

---

# Common Mistakes and Misconceptions

## Misconception

“Most frequent always best.”

Reality:
old historical popularity may become irrelevant.

---

# Connection To Other Concepts

Connects directly to:

* hot objects
* popularity skew
* Zipfian distributions
* admission policies

---

# Quick Summary

[Quick Summary]

* LFU preserves frequently reused objects.
* Better at long-term popularity retention.
* More resistant to scans than LRU.
* Adapts slowly to changing workloads.
* Metadata tracking becomes more expensive.

---

─────────────────────────────────────────────

# 8.5 Why Traditional Policies Fail Under Modern Workloads

─────────────────────────────────────────────

# The One-Line Definition

Traditional eviction policies fail because modern workloads contain scans, bursts, skew, and noisy one-hit traffic that destroy naive locality assumptions.

---

# Intuition First

[Intuition]

Imagine:
trying to organize airport traffic using rules designed for small-town roads.

The environment changed fundamentally.

Old assumptions no longer hold.

Modern internet traffic behaves similarly.

---

# The Problem It Solves

Modern workloads contain:

* viral traffic,
* recommendation bursts,
* crawlers,
* scans,
* bots,
* adversarial requests,
* massive skew.

Traditional policies:

* overreact to scans,
* admit useless entries,
* destroy working sets.

---

# The Core Idea (Precise)

Classic policies assume:
past access strongly predicts future value.

Modern workloads violate this frequently.

Especially dangerous:
one-hit traffic.

Objects accessed once:
often never reused again.

Yet naive caches still admit them.

This creates pollution.

---

# Hidden Systems Insight

The biggest modern cache problem often becomes:

> not eviction quality,
> but admission quality.

Protecting memory from bad objects matters more than optimizing removal afterward.

This is a major conceptual shift in modern caching.

---

# How It Works — Step By Step

1. Large noisy traffic burst arrives
2. Naive cache admits all entries
3. Valuable working set displaced
4. Hit rate collapses
5. Backend pressure surges

---

# Worked Example

Suppose:
analytics scan accesses:
50 million unique keys once.

LRU admits all scan entries.

Existing hot working set evicted.

Hit rate collapses catastrophically.

---

# Visual / Diagram Description

[Diagram]

Before Scan:
Hot Working Set Protected

After Scan:
Millions of One-Hit Objects
↓
Hot Data Evicted
↓
Backend Misses Increase

---

# Key Properties and Characteristics

* Modern traffic highly noisy
* One-hit traffic common
* Traditional recency assumptions break
* Admission becomes critical
* Pollution resistance essential

---

# Trade-offs

| Traditional Simplicity      | Modern Challenge          |
| --------------------------- | ------------------------- |
| Easy implementation         | Weak pollution resistance |
| Low metadata overhead       | Poor skew handling        |
| Simple locality assumptions | Burst traffic instability |

---

# Failure Modes

## Working Set Destruction

Useful hot data displaced.

---

## Scan Pollution

Sequential access destroys locality.

---

## Thrashing

Cache constantly churns useless objects.

---

# When To Use Modern Policies

Necessary when:

* workloads internet-scale,
* traffic noisy,
* backend misses expensive.

---

# When Simpler Policies May Work

Simpler policies acceptable when:

* workloads stable,
* low concurrency,
* predictable locality.

---

# Common Mistakes and Misconceptions

## Misconception

“LRU solves caching sufficiently.”

Reality:
modern workloads often defeat naive recency models.

---

# Connection To Other Concepts

Connects directly to:

* pollution
* admission
* TinyLFU
* working sets
* skewed traffic

---

# Quick Summary

[Quick Summary]

* Modern workloads break traditional cache assumptions.
* One-hit traffic heavily pollutes caches.
* Admission quality increasingly matters more than eviction.
* Working-set protection becomes critical.
* Modern caches evolve beyond simple recency heuristics.

---

─────────────────────────────────────────────

# 8.6 TinyLFU and Modern Admission-Based Caching

─────────────────────────────────────────────

# The One-Line Definition

TinyLFU uses approximate frequency estimation to admit only high-value objects into cache, protecting working sets from pollution.

---

# Intuition First

[Intuition]

Imagine:
a VIP club checking whether guests are actually popular before granting entry.

Unknown random visitors:
rejected.

Regular valuable guests:
admitted.

TinyLFU behaves similarly for cache memory.

---

# The Problem It Solves

Traditional caches:
admit nearly everything.

This allows:

* scans,
* bots,
* one-hit traffic,
* random bursts

to destroy working sets.

Systems need:
predictive admission filtering.

---

# The Core Idea (Precise)

TinyLFU estimates:
future reuse probability
using approximate frequency statistics.

Before admitting object:
cache asks:

> “Has this object demonstrated enough value to deserve memory?”

Low-frequency objects:
often rejected entirely.

---

# Hidden Systems Insight

Modern caching increasingly treats RAM as:

> protected thermodynamic territory.

Only sufficiently “hot” objects deserve residency.

This is the conceptual foundation of modern cache engineering.

---

# How It Works — Step By Step

1. Request arrives
2. Frequency sketch updated
3. Candidate object evaluated
4. Existing victim compared
5. Higher-value object retained
6. Low-value object rejected

---

# Request Lifecycle Reality

TinyLFU commonly uses:
approximate probabilistic data structures:

* Count-Min Sketch
* compact frequency estimators

This reduces:
metadata overhead dramatically.

Exact tracking would be too expensive.

---

# Worked Example

Suppose:
cache receives:
10 million one-hit scan requests.

TinyLFU detects:
objects have near-zero reuse probability.

Most entries:
never admitted.

Hot working set preserved.

---

# Visual / Diagram Description

[Diagram]

Incoming Object
↓
Frequency Estimator
├── High Reuse Probability → Admit
└── Low Reuse Probability → Reject

Protected Working Set Remains Stable

---

# Key Properties and Characteristics

* Admission-focused
* Strong pollution resistance
* Frequency-aware
* Approximate probabilistic estimation
* Excellent modern hit rates

---

# Trade-offs

| Advantage                     | Cost                             |
| ----------------------------- | -------------------------------- |
| Better hit rates              | More algorithm complexity        |
| Strong scan resistance        | Metadata structures required     |
| Better working-set protection | Approximation errors             |
| Lower backend misses          | Higher implementation difficulty |

---

# Failure Modes

## Estimation Errors

Future-hot objects rejected mistakenly.

---

## Frequency Drift

Historical patterns become outdated.

---

## Metadata Corruption

Sketch structures become inaccurate.

---

# When To Use This

Excellent for:

* internet-scale systems,
* noisy workloads,
* scan-heavy traffic,
* high miss cost environments.

---

# When NOT To Use This

Possibly excessive for:

* tiny caches,
* simple applications,
* low traffic systems.

---

# Common Mistakes and Misconceptions

## Misconception

“Modern caches mainly improve eviction.”

Reality:
modern advances focus heavily on admission quality.

---

# Connection To Other Concepts

Connects directly to:

* working sets
* pollution
* frequency estimation
* Zipfian traffic
* probabilistic data structures

---

# Quick Summary

[Quick Summary]

* TinyLFU protects memory using predictive admission.
* One-hit traffic often rejected entirely.
* Modern caches prioritize working-set preservation.
* Frequency estimation becomes probabilistic.
* Admission increasingly matters more than eviction.

---

─────────────────────────────────────────────

# SECTION SYNTHESIS — THE DEEPER ENGINEERING NARRATIVE

This section reveals a major evolution in caching philosophy:

Early caches asked:

> “What should we remove?”

Modern caches increasingly ask:

> “What deserves memory at all?”

That shift is profound.

The deeper systems lesson:

> memory is too valuable to waste on low-probability future reuse.

Thus:
modern caching evolved from:
simple storage optimization
into:
probabilistic working-set protection engineering.

Caching systems increasingly behave like:
predictive thermodynamic managers of hot data.

That is the correct abstraction level for understanding modern cache architecture.

---

# SHORT BRIDGE

So far we established:

* how caches manage finite memory,
* why traditional eviction policies fail under modern workloads,
* and why modern systems increasingly focus on predictive admission and pollution resistance.

Next sections will build on this foundation to explain:

* how caches interact with databases internally,
* how buffer pools differ from application caches,
* and why database storage engines themselves are fundamentally sophisticated caching systems.
