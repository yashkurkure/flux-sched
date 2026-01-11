# Flux-Sched Scheduling and Backfilling

This document explains how scheduling and backfilling work in flux-sched, including the algorithms, optimizations, and configuration options.

## Table of Contents

1. [How Scheduling Begins](#how-scheduling-begins)
2. [How Backfilling Works](#how-backfilling-works)
   - [Backfill Policies](#backfill-policies)
   - [How Backfill Conflicts Are Detected](#how-backfill-conflicts-are-detected)
3. [Resource Representation and Optimizations](#resource-representation-and-optimizations)
4. [Scheduler Flags and Configuration](#scheduler-flags-and-configuration)

---

## How Scheduling Begins

### Module Initialization

The scheduling system is initialized in the **qmanager module**:

- **Entry Point**: `mod_main()` function in [qmanager.cpp:646](qmanager/modules/qmanager.cpp#L646)
- **Start Function**: `mod_start()` at [qmanager.cpp:589](qmanager/modules/qmanager.cpp#L589)

The qmanager module sets up three key watchers for the Flux reactor:

| Watcher | Callback | Purpose |
|---------|----------|---------|
| Prepare Watcher | `prep_watcher_cb()` | Checks if scheduling is needed |
| Check Watcher | `check_watcher_cb()` | Runs the actual scheduling loop |
| Idle Watcher | - | Ensures check watcher runs |

### Scheduling Loop Entry Points

#### 1. Job Submission (`jobmanager_alloc_cb`)

**File**: [qmanager_callbacks.cpp:290-365](qmanager/modules/qmanager_callbacks.cpp#L290-L365)

When a job is submitted to Flux:
1. The job-manager sends an allocation request
2. The callback unpacks the jobspec (lines 305-320)
3. Determines the target queue (default or from jobspec attributes)
4. Creates a `job_t` object and inserts it into the queue
5. Calls `queue->insert(job)` which sets `set_schedulability(true)`

#### 2. Reactor Loop Watchers

**File**: [qmanager_callbacks.cpp:491-530](qmanager/modules/qmanager_callbacks.cpp#L491-L530)

**`prep_watcher_cb()` (lines 491-504)**:
- Checks if any queue `is_schedulable()` or `is_scheduled()`
- If true, starts the idle watcher to ensure the check watcher runs

**`check_watcher_cb()` (lines 506-530)**:
- If scheduling loop is needed, calls `queue->run_sched_loop()` for each queue
- Calls `post_sched_loop()` to respond to job-manager with results

#### 3. Resource Events

**File**: [qmanager.cpp:212-234](qmanager/modules/qmanager.cpp#L212-L234)

The `update_on_resource_response()` callback:
- Receives notifications when resource state changes
- Calls `queue->reconsider_blocked_jobs()` to re-evaluate blocked jobs
- Sets `set_schedulability(true)` to trigger a new scheduling round

#### 4. Job Completion/Cancellation

**File**: [queue_policy_base.hpp:704-707](qmanager/policies/base/queue_policy_base.hpp#L704-L707)

When a job completes via the `remove()` method:
- Blocked jobs are reconsidered
- Calls `reconsider_blocked_jobs()` which triggers a new scheduling pass

### Queue Depth Limiting

**File**: [queue_system_defaults.hpp:16-19](qmanager/config/queue_system_defaults.hpp#L16-L19)

| Parameter | Default | Maximum |
|-----------|---------|---------|
| queue-depth | 32 | 1,000,000 |

This limits the scheduling loop to only consider the first N pending jobs per pass, preventing unbounded O(N) complexity.

---

## How Backfilling Works

### Core Concept

Backfilling allows lower-priority jobs to run while waiting for resources to become available for higher-priority jobs. A job can only backfill if it **doesn't delay** reserved high-priority jobs.

### Match Operations

**File**: [match_op.h:4-10](resource/policies/base/match_op.h#L4-L10)

Four match operations are defined:

| Operation | Description |
|-----------|-------------|
| `MATCH_ALLOCATE` | Allocate immediately or fail |
| `MATCH_ALLOCATE_ORELSE_RESERVE` | Allocate if possible, otherwise reserve for future |
| `MATCH_ALLOCATE_W_SATISFIABILITY` | Allocate + check if satisfiable |
| `MATCH_SATISFIABILITY` | Only check satisfiability (no allocation) |

### Backfill Algorithm Implementation

**File**: [queue_policy_bf_base_impl.hpp:73-234](qmanager/policies/queue_policy_bf_base_impl.hpp#L73-L234)

#### `allocate_orelse_reserve_jobs()` (lines 74-104)

```
For each pending job in queue order:
    1. Try to allocate using match_allocate_multi()
    2. Use m_try_reserve flag to control allocation vs. reservation
    3. Return EAGAIN if more work remains (resumable loop)
```

#### `handle_match_success()` (lines 133-177)

- **If job is RESERVED** (line 155):
  - Add to `m_reserved` map
  - Increment `m_reservation_cnt`
  - Continue to next job (as backfill candidate)
- **If job is ALLOCATED** (line 167):
  - Move to running queue immediately

#### `handle_match_failure()` (lines 180-221)

| Error Code | `m_try_reserve=true` | `m_try_reserve=false` |
|------------|---------------------|----------------------|
| `EBUSY` | Cannot reserve → move to blocked | Resources busy → continue |
| `ENODEV` | Unsatisfiable → reject job | Unsatisfiable → reject job |

Blocked jobs are reconsidered when resource state changes.

### Backfill Policies

#### EASY Backfill (`reserve-depth=1`)

**File**: [queue_policy_easy_impl.hpp:20-35](qmanager/policies/queue_policy_easy_impl.hpp#L20-L35)

```cpp
queue_policy_bf_base_t<reapi_type>::m_reservation_depth = 1;
```

- Only the **first job** that cannot run gets a reservation
- Subsequent jobs can backfill as long as they don't delay this **one** reserved job
- Simple and efficient for many workloads

#### Conservative Backfill (`reserve-depth=-1`)

**File**: [queue_policy_conservative_impl.hpp:56-59](qmanager/policies/queue_policy_conservative_impl.hpp#L56-L59)

```cpp
queue_policy_bf_base_t<reapi_type>::m_reservation_depth = MAX_RESERVATION_DEPTH;
```

- **Every** job that cannot run gets a reservation
- Subsequent jobs can only backfill if they don't delay **any** reserved job ahead of them
- Guarantees FCFS ordering but reduces backfill opportunities
- Maximum reservation depth can be limited via `max-reservation-depth` parameter

#### Hybrid Backfill (`reserve-depth=N`)

**File**: [queue_policy_hybrid_impl.hpp:26-29](qmanager/policies/queue_policy_hybrid_impl.hpp#L26-L29)

```cpp
queue_policy_bf_base_t<reapi_type>::m_reservation_depth = HYBRID_RESERVATION_DEPTH; // 64
```

- Reserves the top **N** jobs that cannot run
- Backfilling is allowed as long as it doesn't delay these N reserved jobs
- Tunable via `reservation-depth` and `max-reservation-depth` parameters
- Balance between EASY's flexibility and Conservative's fairness

### Backfill Configuration Defaults

**File**: [queue_system_defaults.hpp:16-19](qmanager/config/queue_system_defaults.hpp#L16-L19)

| Constant | Value | Description |
|----------|-------|-------------|
| `MAX_QUEUE_DEPTH` | 1,000,000 | Hard limit on queue-depth |
| `DEFAULT_QUEUE_DEPTH` | 32 | Jobs to consider per pass |
| `MAX_RESERVATION_DEPTH` | 100,000 | Hard limit on reservation-depth |
| `HYBRID_RESERVATION_DEPTH` | 64 | Default for hybrid policy |

### How Backfill Conflicts Are Detected

The key insight is that **reservations and allocations use the same data structure**. When resources are reserved for a future time, they're immediately marked as "in use" at that future time slot. This makes backfill conflict detection trivial.

#### The Planner Data Structures

Each resource vertex in the graph has a **planner** that tracks availability over time using two red-black trees:

**1. Scheduled Point Tree (SP Tree)**

**File**: [scheduled_point_tree.hpp](resource/planner/c++/scheduled_point_tree.hpp)

A red-black tree ordered by **time**, where each node represents a point where resource state changes:

```cpp
// From planner_internal_tree.hpp:21-33
struct scheduled_point_t {
    int64_t at;           // Time when resource state changes
    int64_t scheduled;    // Resources scheduled (in use) at this point
    int64_t remaining;    // Resources available at this point
    int ref_count;        // Number of spans referencing this point
};
```

**Purpose**: Answer "How many resources are available at time T?" in O(log N).

**2. Min-Time Resource Tree (MT Tree)**

**File**: [mintime_resource_tree.hpp](resource/planner/c++/mintime_resource_tree.hpp)

An **augmented** red-black tree ordered by **remaining resources**, with a special `subtree_min` field:

```cpp
// From mintime_resource_tree.hpp:20-28
struct mt_resource_rb_node_t {
    int64_t at;           // Time of this scheduled point
    int64_t subtree_min;  // MINIMUM time in subtree where these resources are available
    int64_t remaining;    // Available resources at this point
};
```

**Purpose**: Answer "What is the earliest time when N resources are available?" in O(log N).

#### How Resources Are Reserved for the Future

When a job gets a reservation via `planner_add_span()`:

**File**: [planner_c_interface.cpp:500-540](resource/planner/c/planner_c_interface.cpp#L500-L540)

```
planner_add_span(ctx, start_time=T+1000, duration=3600, request=8):

1. Create scheduled_point_t at start_time (T+1000)
2. Create scheduled_point_t at end_time (T+4600)
3. For all points in range [start, end):
     point->scheduled += 8;   // Mark 8 resources as scheduled
     point->remaining -= 8;   // Reduce available by 8
4. Update MT tree with new remaining values
```

The resources are now marked as "in use" during that future time window, even though the job hasn't started yet.

#### How Backfill Availability Is Checked

When checking if a backfill job can run via `avail_during()`:

**File**: [planner_c_interface.cpp:183-203](resource/planner/c/planner_c_interface.cpp#L183-L203)

```cpp
static bool avail_during(planner_t *ctx, int64_t at, uint64_t duration, int64_t request) {
    bool ok = true;
    scheduled_point_t *point = ctx->plan->sp_tree_get_state(at);  // O(log N)

    while (point) {
        if (point->at >= (at + duration)) {
            ok = true;   // Past the end of our window, we're good
            break;
        } else if (request > point->remaining) {
            ok = false;  // Not enough resources at this point!
            break;
        }
        point = ctx->plan->sp_tree_next(point);  // Walk to next time point
    }
    return ok;
}
```

The check walks through all scheduled points during the job's requested time window. If `remaining >= request` at every point, the job can run. If not, it would conflict with an existing allocation or reservation.

#### Visual Example

```
Time:     T=0        T=100      T=200      T=300      T=400
          |          |          |          |          |
Total:    10 cores   10 cores   10 cores   10 cores   10 cores

Job 1 (allocated):    [====== 8 cores ======]
                      T=0                   T=200
                      remaining=2           remaining=10

Job 2 (RESERVED):                           [==== 6 cores ====]
                                            T=200              T=400
                                            remaining=4        remaining=10

SP Tree points:
  T=0:   scheduled=8,  remaining=2
  T=200: scheduled=6,  remaining=4  (Job1 ended, Job2 reserved starts)
  T=400: scheduled=0,  remaining=10

Backfill Check Examples:

  Can Job 3 (4 cores, duration=150) backfill at T=50?
    avail_during(T=50, duration=150, request=4):
    - T=0 point: remaining=2 < 4 → FAILS! Cannot backfill.

  Can Job 4 (2 cores, duration=100) backfill at T=50?
    avail_during(T=50, duration=100, request=2):
    - T=0 point: remaining=2 >= 2 → OK
    - End at T=150, still in T=0-200 window, remaining=2 >= 2 → OK
    → SUCCESS! Job 4 can backfill without delaying Job 2's reservation.
```

#### Finding Earliest Available Time for Reservations

The MT tree's `subtree_min` augmentation enables O(log N) queries:

**File**: [mintime_resource_tree.cpp:60-87](resource/planner/c++/mintime_resource_tree.cpp#L60-L87)

```cpp
int64_t find_mintime_anchor(int64_t request, mt_resource_rb_node_t **anchor_p) {
    mt_resource_rb_node_t *node = m_tree.get_root();
    int64_t min_time = MAX;

    while (node) {
        if (request <= node->remaining) {
            // This node has enough resources - check its min time
            right_min_time = right_branch_mintime(node);
            if (right_min_time < min_time) {
                min_time = right_min_time;
                *anchor_p = node;
            }
            node = node->get_left();  // Search for even better times
        } else {
            // Not enough resources here, try right subtree
            node = node->get_right();
        }
    }
    return min_time;
}
```

#### Complexity Summary

| Question | Data Structure | Complexity |
|----------|---------------|------------|
| Resources available at time T? | SP Tree (`sp_tree_get_state`) | O(log N) |
| Can job run during [T, T+D]? | SP Tree (`avail_during`) | O(log N + K)* |
| Earliest time for N resources? | MT Tree (`mt_tree_get_mintime`) | O(log N) |

*K = number of scheduled points in the time range

The elegance of this design is that a backfill job doesn't explicitly check against each reserved job—it simply checks if resources are available during its window. If a reservation has claimed resources during that window, `remaining` will reflect that, and the check will fail automatically.

---

## Resource Representation and Optimizations

### Graph-Based Resource Model

**File**: [resource_graph.hpp:29-34](resource/schema/resource_graph.hpp#L29-L34)

```cpp
using resource_graph_t = boost::adjacency_list<
    boost::vecS,           // edges stored as vector
    boost::vecS,           // vertices stored as vector
    boost::bidirectionalS, // bidirectional edges
    resource_pool_t,       // vertex data (resource info)
    resource_relation_t,   // edge data (relationships)
    std::string>;          // graph property
```

- Uses **Boost Graph Library** for efficient DAG representation
- **Vertices** represent resources (Cluster, Rack, Node, GPU, Core, etc.)
- **Edges** represent hierarchical relationships and constraints

### Temporal Planning with Red-Black Trees

**File**: [resource/planner/README.md](resource/planner/README.md)

The planner uses **two highly efficient red-black binary search trees** per resource:

#### 1. Scheduled-Point Tree

- Tracks resource state at any instant in time
- **O(log N)** lookup to find resource state at time `t`
- Stores "scheduled points" where resource availability changes

#### 2. Min-Time Resource Tree

- Augmented red-black tree with resource requirements
- **O(log N)** lookup to find earliest time when N resources are available
- Enables efficient reservation time calculation **without simulation**

**File**: [planner.hpp:40-116](resource/planner/c++/planner.hpp#L40-L116)

Key methods:
- `mt_tree_get_mintime(int64_t request)`: Find earliest time for request
- `sp_tree_get_state(int64_t at)`: Get resource state at time `at`
- `sp_tree_insert/remove()`: Update scheduled points

### Scheduling Data Structure

**File**: [sched_data.hpp:23-33](resource/schema/sched_data.hpp#L23-L33)

```cpp
struct schedule_t {
    std::map<int64_t, int64_t> allocations;  // jobid -> start_time
    std::map<int64_t, int64_t> reservations; // jobid -> reserved_time
    planner_t *plans;  // One planner per resource type
};
```

### Depth-First-and-Up (DFU) Traversal

**File**: [dfu.hpp:22-115](resource/traversers/dfu.hpp#L22-L115)

The DFU traverser:
- Performs depth-first visit on the **dominant subsystem** (usually compute)
- **Upwalks** on auxiliary subsystems (power, storage, network)
- Uses callback-based scoring for best-fit matching

### Pruning for Efficiency

**File**: [matcher.hpp:103-121](resource/policies/base/matcher.hpp#L103-L121)

**Subtree planning** enables aggressive pruning:

1. Track aggregate resource availability at higher-level nodes
2. If a compute-node needs 4 cores but its subtree only has 2 available, skip entire subtree
3. Significantly reduces search space

Example spec: `rack:node,node:core`
- Tracks nodes available at rack level
- Tracks cores available at node level

### Performance Optimization Summary

| Optimization | Benefit |
|--------------|---------|
| Subtree Plans | Aggregate low-level resources at higher vertices for fast pruning |
| Queue Depth Limiting | Don't scan entire queue every iteration |
| Red-Black Trees | O(log N) time complexity for temporal queries |
| Reactive Backfilling | Only reconsider blocked jobs when resource state actually changes |
| Graph-Based Filtering | Subsystem and relationship-type filtering reduces search space |

---

## Scheduler Flags and Configuration

### Configuration Sources

**File**: [qmanager.cpp:78-100, 121-201](qmanager/modules/qmanager.cpp#L78-L201)

Configuration can be set via:
1. Command-line arguments
2. Configuration file at `sched-fluxion-qmanager` section

### Queue Parameters

**File**: [queue_policy_base.hpp:256-296](qmanager/policies/base/queue_policy_base.hpp#L256-L296)

| Parameter | Description | Default |
|-----------|-------------|---------|
| `queue-depth=<N>` | Number of pending jobs to schedule per pass | 32 |
| `max-queue-depth=<N>` | Hard upper limit on queue-depth | 1,000,000 |

### Queue Management Options

**File**: [qmanager_opts.hpp:57-67](qmanager/modules/qmanager_opts.hpp#L57-L67)

| Option | Description |
|--------|-------------|
| `queues=<q1,q2,...>` | Define named queues |
| `default-queue=<name>` | Default queue for jobs |
| `queue-policy=<policy>` | Policy per queue (fcfs, easy, conservative, hybrid) |
| `queue-params=<key=value>` | Parameters for queue |
| `policy-params=<key=value>` | Parameters for backfill policy |

### Policy-Specific Parameters

#### EASY Backfill
- No additional parameters needed
- Automatically sets `reservation-depth=1`

#### Conservative Backfill
| Parameter | Description |
|-----------|-------------|
| `max-reservation-depth` | Limit maximum reservations (default: unlimited up to MAX_RESERVATION_DEPTH) |

#### Hybrid Backfill

**File**: [queue_policy_hybrid_impl.hpp:32-67](qmanager/policies/queue_policy_hybrid_impl.hpp#L32-L67)

| Parameter | Description | Default |
|-----------|-------------|---------|
| `reservation-depth=<N>` | Number of jobs to reserve | 64 |
| `max-reservation-depth=<N>` | Hard upper limit | 100,000 |

---

## Key Files Reference

| File | Purpose |
|------|---------|
| [qmanager/modules/qmanager.cpp](qmanager/modules/qmanager.cpp) | QManager module entry point |
| [qmanager/modules/qmanager_callbacks.cpp](qmanager/modules/qmanager_callbacks.cpp) | Reactor callbacks for scheduling |
| [qmanager/policies/queue_policy_bf_base_impl.hpp](qmanager/policies/queue_policy_bf_base_impl.hpp) | Base backfill implementation |
| [qmanager/policies/queue_policy_easy_impl.hpp](qmanager/policies/queue_policy_easy_impl.hpp) | EASY backfill (reserve-depth=1) |
| [qmanager/policies/queue_policy_conservative_impl.hpp](qmanager/policies/queue_policy_conservative_impl.hpp) | Conservative backfill |
| [qmanager/policies/queue_policy_hybrid_impl.hpp](qmanager/policies/queue_policy_hybrid_impl.hpp) | Hybrid backfill (tunable) |
| [resource/planner/c++/planner.hpp](resource/planner/c++/planner.hpp) | Temporal planning with RB-trees |
| [resource/traversers/dfu.hpp](resource/traversers/dfu.hpp) | Depth-first traversal |
| [resource/policies/base/matcher.hpp](resource/policies/base/matcher.hpp) | Resource matching logic |
| [resource/schema/resource_graph.hpp](resource/schema/resource_graph.hpp) | Graph representation |

---

## Summary

Flux-sched is a sophisticated scheduler combining:

1. **Reactive event-driven scheduling** via watchers and callbacks
2. **Multiple backfill strategies** (EASY, Conservative, Hybrid) with configurable depth limits
3. **Temporal planning** using red-black trees for O(log N) time queries
4. **Graph-based resource modeling** with Boost Graph Library
5. **Aggressive pruning** via subtree planning to reduce search space
6. **Queue depth limiting** to prevent O(N^2) scheduling behavior
