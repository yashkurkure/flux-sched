# Flux Scheduling and Backfilling Exploration

## Overview
Flux's scheduler, often referred to as **Fluxion**, uses a graph-based approach to resource modeling. It separates the queuing logic (`sched-fluxion-qmanager`) from the resource matching logic (`sched-fluxion-resource`).

## How Scheduling Begins

1.  **Job Submission**: When a job is submitted to Flux, it is ingested and eventually handed off to the scheduler.
2.  **Queue Management**: The job enters the `PENDING` state within the Queue Manager (`qmanager`).
3.  **Scheduling Loop**: The core of the scheduling logic is driven by the `run_sched_loop` method implemented by specific queue policies.
    *   The loop iterates through jobs in the pending queue.
    *   It respects a `queue-depth` parameter to limit how many jobs are considered in one pass, ensuring the scheduler remains responsive even with massive queues.
4.  **Resource Request**: For each job considered, the Queue Manager contacts the Resource Module via the Resource API (`reapi`).
    *   It requests an allocation using `match_allocate` or `match_allocate_multi`.

## How Backfilling Works

Backfilling is the process of scheduling lower-priority jobs on resources that are currently available, provided they do not delay the start time of higher-priority jobs that are waiting for resources.

1.  **Allocation vs. Reservation**:
    *   When the scheduler attempts to schedule a job, it typically uses the `MATCH_ALLOCATE_ORELSE_RESERVE` operation.
    *   If resources are available immediately, the job is **Allocated** and starts running.
    *   If resources are *not* available, the scheduler calculates the earliest time in the future when the required resources will be free. It then **Reserves** those resources for that specific job at that future time.

2.  **Filling the Gaps**:
    *   Once a high-priority job has a reservation, the scheduler continues to look at subsequent jobs in the queue (up to the `queue-depth`).
    *   If a subsequent job is small enough and short enough to finish *before* the reserved start time of the high-priority job (and uses resources that don't conflict with the reservation), it is allowed to run immediately.

3.  **Backfill Policies**:
    Flux supports different backfilling strategies controlled by the `reserve-depth` parameter:
    *   **EASY Backfill** (`reserve-depth=1`): Only the *first* job that cannot run is granted a reservation. Subsequent jobs can backfill as long as they don't delay this one job.
    *   **Conservative Backfill** (`reserve-depth=-1`): *Every* job that cannot run is granted a reservation. A job can only backfill if it doesn't delay *any* ahead of it.
    *   **Hybrid** (`reserve-depth=N`): Reserves resources for the top N jobs.
    *   **Pure/None** (`reserve-depth=0`): No reservations are made (best effort scanning).

## Resource Representation and Optimizations

Fluxion employs several optimizations to make scheduling and backfilling efficient:

1.  **Graph-Based Resource Model**:
    *   Resources are modeled as a directed acyclic graph (DAG). This allows for flexible representation of hierarchical resources (Clusters, Racks, Nodes, GPUs, Cores).
    *   This graph model (defined via GRUG - GraphML Resource Update Generator) allows the scheduler to traverse relationships and constraints efficiently.

2.  **Temporal Plan Management**:
    *   Instead of just tracking current usage, Fluxion maintains a "temporal plan" to track resource usage over time segments.
    *   This allows for efficient queries about *when* a specific shape of resources will be available, which is critical for calculating reservations without expensive simulation.

3.  **Queue Depth Limits**:
    *   By enforcing a `queue-depth` (default often 32), the scheduler avoids the O(N) cost of scanning the entire queue every cycle.

4.  **Reactive Backfilling**:
    *   The scheduler is designed to be reactive. When a job completes (or is canceled), blocked jobs are "reconsidered" (`reconsider_blocked_jobs`), potentially moving them back to a state where they can be scheduled immediately if the reservation constraints allow.

## Relevant Flags and Configuration

### Scheduler Parameters
These are typically set via `flux module load` or configuration files for the `qmanager`.

*   `queue-depth=<N>`: Defines how deep into the pending queue the scheduler looks for work.
*   `max-queue-depth=<N>`: The absolute limit for queue depth.
*   `delay-scheduling=true`: Batches scheduling attempts to improve throughput at the cost of slight latency.

### Backfill Configuration
Controlled via parameters passed to the backfill policy module.

*   `reserve-depth=<N>`:
    *   `1`: EASY Backfill.
    *   `-1`: Conservative Backfill.
    *   `0`: No reservations.

*   `sched-params`: Can be used to pass generic key-value pairs to tune the policy.