## Mini-Scheduler – Process Scheduling Simulator

### Authors

**Fagan Fataliyev**, **Elnur Hagverdiyev**, **Tamerlan Imanov**

**Note: In the repository:**

* **Elnur Hagverdiyev**: branches: “elnur”, “ellight” ; nicknames: HAGVERDIYEV ELNUR, ellight
* **Imanov Tamerlan**: branch: `tamerlan`
* **Fagan Fataliyev**: branch: `f`

---

## 1. Introduction

This project is a simple, user-space simulation of a single-CPU process scheduler. Given a workload—defined as a trace of processes with start and finish times, along with priorities—the program produces a timeline that shows the state of each process (running, pending, or finished) at every discrete time step. Optionally, it can also generate a chronogram and a trace log that explain each scheduling decision.

The simulation strictly follows the rules specified in the course hand-out. It operates under the assumption of a single CPU with a fixed capacity of `C = 20`, which represents the maximum sum of priorities that can run simultaneously. The system manages two queues—**running** and **pending**—each storing process entries as pairs of priority and process ID. The time horizon is fixed at `N = 30` time steps (from 0 to 29). When the combined priorities of active tasks exceed the CPU’s capacity, the scheduler deschedules lower-priority tasks to the pending queue. Each such descheduling increases the idle time and delays the task's finish time by one step.

---

## 2. Build & Run

### 2.1 Prerequisites

To build and run this project, you will need:

* GCC 10+ or Clang 12+ (C17)
* `make`

### 2.2 Compilation

Clone the repository and build the scheduler:

```bash
$ git clone https://git.unistra.fr/<your​-group>/mini-scheduler.git
$ cd mini-scheduler
$ make            # builds `./sched` in ./bin
```

Extra targets:

* `make debug` (AddressSanitizer)
* `make clean`

### 2.3 Execution

Run with input from stdin (default mode):

```bash
$ ./bin/sched < workloads/sample.txt
```

With file argument:

```bash
$ ./bin/sched workloads/sample.txt
```

---

## 3. Workload File Format

Each line contains exactly **seven blank-separated fields**:

```
pid ppid ts tf idle cmd prio
```

* `pid`: Process ID
* `ppid`: Parent Process ID
* `ts`: Start time
* `tf`: Finish time
* `idle`: Initial idle time
* `cmd`: Command name
* `prio`: Priority (integer)

**Example:**

```
4 1 7 9 0 gcc 5
```

The simulator ignores `ppid` and `idle`. File parsing is tolerant to extra whitespace and `#` comments.

---

## 4. Scheduling Algorithm

The core routine `schedule_round()` in `src/scheduler.c` advances the simulation by a single timestep `t` (from 0 to N − 1). Its behaviour mirrors the specification but is deliberately organised around two small vectors—**running** and **pending**—rather than several ad-hoc lists so that each stage remains easy to reason about and to unit-test.

At the start of a round we retire processes whose planned finish date `tf` has already elapsed. When a process becomes finished we mark its state accordingly and remove it from both vectors; it will contribute `_` to the timeline for every remaining timestep.

We then admit newborn tasks. The immutable array `all[]`, filled by the parser, contains every `workload_item` from the input file. Whenever `item.ts == t`, a shallow copy of the item moves to the pending vector. At this point the union `running ∪ pending` corresponds exactly to the set of current processes, i.e. those satisfying `ts ≤ t ≤ tf`.

The selection phase is carried out in two passes:

* **Ordering.** A stable in-place sort (`stable_sort()` defined atop the tiny binary heap found in `heap.c`) arranges the pending vector in descending priority order, breaking ties by the natural PID order preserved from the input.
* **Packing the CPU.** With tasks neatly ordered we sweep the array once, keeping a running total capacity. A task joins the running vector if `capacity + task.prio ≤ C`, where `C = 20` is the fixed CPU bandwidth. Tasks that do not fit stay pending. When such a task was already running in the previous round `deschedule()` is invoked; it increments the task’s idle counter and pushes its `tf` back by one tick.

After selection, `record_timeline()` writes a single character for each task into the global timeline matrix: `'R'` for currently running, `'.'` for pending, and `'_'` for finished. Because this happens exactly once per round, producing the mandatory `[===Results===]` section at the end is a trivial dump of that matrix.

**Complexity:**

* Only `P` tasks (with `P ≤ 10` in the instructor’s workloads) are sorted each round.
* Per-round cost is `O(P log P)`, overall `O(N · P log P)` for `N = 30`.
* Simulation completes in under 0.5 ms.

---

## 5. Output Specification

The last lines of stdout must follow exactly:

```
[===Results===]
<pid0> <tab> <30 chars>
<pid1> <tab> <30 chars>
...
```

Each char ∈ {`R`, `.`, `_`} represents running, pending, finished at time `t` (0-based).

**Optional extras before `[===Results===]`:**

* **Chronogram** – vertical list with X spanning `ts … tf`
* **Trace** – detailed log of queue contents after each round

---

## 6. Testing

We exercised the scheduler with the two official harnesses packaged in `test-sched/`.

### Harness Table

| Harness                                | What it checks                                                                                          | Result summary                                                                                                                                                                                        |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **test-sched.sh (reference-based)**    | Timeline must byte-match the instructor’s `.ref` file.                                                  | 5 / 11 workloads match (test-0,1,2,3,5). The other six diverge even though most still obey the formal constraints.                                                                                    |
| **test-sched-2.sh (constraint-based)** | Only two invariants: • CPU load ≤ 20 at every timestep • All processes finish by the prescribed horizon | 9 / 11 workloads satisfy both invariants. Tests 6 and 9 violate the duration limit (finish at timestep 22 and 26 respectively, while the limits are 21 and 24). CPU-capacity is respected everywhere. |

### Execution:

```bash
$ make test
cd test-sched && ./test-sched.sh   ../src/sched
… OK on tests 0 1 2 3 5
… NOK on 4 6 7 8 9 10          # timeline mismatch only

cd test-sched && ./test-sched-2.sh ../src/sched
… CPU capacity: OK  on all 11
… Schedule duration: NOK on 6 9   (exceeds limit by 1-2 steps)
```

**Interpretation:**

* **Reference mismatches (tests 4, 6, 7, 8, 9, 10).**
  Our scheduler makes different—but still legal—preemption choices whenever multiple traces satisfy the rules. The strict byte-comparison therefore flags them as failures even though `test-sched-2.sh` accepts four of those six cases.

* **True violations (tests 6 and 9).**
  Both overrun the deadline because low-priority jobs accumulate extra idle time near the horizon. The fix is to tighten the tie-breaker in `schedule_round()` so that jobs with imminent deadlines are preferred whenever priorities are equal.

At present the implementation always respects CPU capacity, and converges within the required 30 ticks for 9/11 public workloads. Addressing the overrun edge-cases will bring the constraint score to 11/11 and, incidentally, make the timelines line up with the instructor’s reference for the remaining tests.

---

## 8. Repository Layout

```
dsa2-scheduler/
├── obj/                     # (Optional) Object files directory
├── out/                     # (Optional) Test results or logs
├── src/                     # Scheduler source code
│   ├── heap.c               # Max-heap implementation
│   ├── heap.h               # Heap headers
│   ├── sched.c              # Main scheduling logic
│   ├── sched.h              # Structs and shared types
│   ├── trace.c              # Timeline and visualization
│   ├── trace.h              # Trace interface
├── test-sched/              # Test framework and cases
│   ├── in/                  # Input workloads
│   ├── out/                 # Test outputs
│   ├── ref/                 # Reference outputs
│   ├── src/                 # Teacher reference code
│   ├── test-sched.sh        # Test script
│   ├── test-sched-2.sh      # New tests
│   └── README.md            # Test instructions
├── Makefile                 # Build config
└── README.md                # Main report
```
