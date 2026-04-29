COM 274, Linux Kernel Internals: Data Structures for Open Computing
===================================================================
# Project #4 (Bonus): Bring Your Own Sabotage

## General Information

In this bonus project, you pick a subsystem we have not yet tried to sabotage. Swap one of its data structures for a worse one, implement the swap behind a `CONFIG_*` flag in the Linux kernel, benchmark the damage, and present the results in person. There is no autograder, no submission portal, and no fixed deadline. Grading happens in a 5-minute defense session.

* No GitHub Actions, no Classroom, no pull requests, no commit URLs. The course server already has every tool you need (`virtme-ng`, `perf`, `trace-cmd`). We also have some Ubuntu-compatible kernels in `/usr/local/src`, though you can download a specific version yourself too.
* No commit hash to submit. The defense itself is the submission.
* The deadline is the last class of the course (which is May 11 this year).
* Try to follow the format and structure used in Projects #1, #2, and #3. Some artistic deviations are acceptable if justified.

## What You Build

1. Pick a sabotage. Use one of the three examples below or invent your own. The constraint: you must replace one kernel data structure with another that is _plausibly an optimization_ and _actually a deoptimization_ under realistic workloads.
2. Implement it behind a `CONFIG_*` flag. Both the stock and the sabotaged kernels must build from the same source tree. Use `vng --kconfig --configitem CONFIG_*=y` to enable the sabotaged build, the same way you did in Projects 1–3.
3. Benchmark stock vs sabotaged. Pick a workload that exposes the slowdown. Capture the numbers. Plot them.
4. Write the story. Three short paragraphs: the sales pitch (why a junior reviewer might wave it through), the design (which data structure you swapped and why the substitution is plausibly better), the evidence (your charts).

Try to keep the same structure where you store the source code to be added to the kernel with the patch files (e.g., `kernel`), put the outcomes such as measurements, write-up, charts into `results`. Bring whatever else you need to the defense to convince the instructor.

## The Defense (5 minutes)

Bring:

* A `virtme-ng` setup that boots both kernels (stock and sabotaged) on the course server.
* A `git diff` or `changes.patch` showing exactly what you modified.
* Your benchmark scripts, ready to re-run live if the instructor asks.
* You charts with the benchmarking results to show and explain.

## Grading

* Plausibility of the story. Could a junior reviewer reasonably wave the patch through? Sabotage that's obvious on inspection isn't interesting.
* Quality of the benchmark. Is the slowdown measurable, reproducible, and isolated to the data structure you changed? Does it survive small workload perturbations?
* Technical depth. Can you defend the trade-offs? Do you know why the original design was chosen and what the swap really costs?
* Presentation. Are the plots readable? Does the narrative land? Can a non-expert come away understanding the sabotage?

## Required Tools

All available on the course server, the same as in Projects 1 through 3: `virtme-ng`, `perf`, `trace-cmd`, `slabtop`, `cyclictest`, `hackbench`, with `git` for the diff. Write to your instructor if you need something special installed or configured.

---

## Examples

The three examples below are starting points. You may pick one verbatim, adapt one, or invent your own.

### 1. The Timid Table

Subsystem: VFS, file descriptor table.

Data structure swap: dynamic array (geometric doubling) -> dynamic array (+1 growth).

Sabotage in one sentence: modify `expand_fdtable()` to grow by exactly one slot instead of doubling, framing it as "memory savings on processes that only open a handful of files."

Algorithmic complexity: amortized _O(1)_ append -> _O(n)_ append; cumulative _O(n)_ -> _O(n^2)_. Opening 1000 files (probably) goes from ~10 reallocations to 936.

What to do:

* Locate `expand_fdtable()` in `fs/file.c`.
* Replace the doubling growth policy with +1 (or a small constant like +4) growth, gated on `CONFIG_*`.
* Be careful with the locking around `dup_fd()` and `__fd_install()`; the growth path runs under the file table spinlock.

How to benchmark: a C program that opens N file descriptors in a tight loop (`open("/dev/null", O_RDONLY)` is fine) for N ∈ {100, 1000, 10000}. Measure per-fd latency and the total memory copied (sum of old-table sizes during reallocations).

Tools: `perf mem record`, `perf stat -e cache-misses,cache-references`, `time`.

Story angle: "We grow the table only as needed. For the median process that opens fewer than ten files, this saves a few KB of slab memory."

---

### 2. Unfair Scheduler

Subsystem: CFS scheduler runqueue.

Data structure swap: red-black tree -> unsorted linked list.

Sabotage in one sentence: replace `cfs_rq.tasks_timeline` rb-tree with an unsorted linked list, framing it as "removing unnecessary tree-balancing overhead from the hot path."

Algorithmic complexity: _O(log n)_ insert/remove and _O(1)_ find-min (`rb_first_cached`) -> _O(1)_ insert and _O(n)_ find-min. Scheduler overhead now scales with runqueue depth.

What to do:

* Locate `cfs_rq` and the rb-tree operations in `kernel/sched/fair.c`.
* Replace `__enqueue_entity` and `__dequeue_entity` with `list_add` / `list_del`.
* Replace `__pick_first_entity` with a linear scan that finds the smallest `vruntime`. Gate everything on `CONFIG_*`.

How to benchmark: run with 1, 10, 100, and 500 CPU-bound tasks (busy-loop processes pinned to a single CPU). Measure scheduling latency, context switch overhead, and (qualitatively) interactive responsiveness. Type into a terminal under each load and feel the lag yourself.

Tools: `perf sched record`, `perf sched latency`, `cyclictest`, `hackbench`.

Story angle: "Trees are over-engineered for runqueues that fit in cache. A list keeps the fast path branch-free."

---

### 3. The Forgetful Slab

Subsystem: kernel allocator (slab -> kmalloc).

Data structure swap: per-object `kmem_cache` (e.g. `task_struct_cachep`, `inode_cachep`, `filp_cachep`) -> raw `kmalloc` / `kfree`.

Sabotage in one sentence: bypass the per-object slab cache and route allocations through general-purpose `kmalloc`, framing it as "fewer cache types means simpler memory accounting."

Algorithmic complexity: _O(1)_ amortized cache pop with hot CPU-local freelist -> _O(1)_ general-purpose allocator with global locking, slab coloring loss, and worse locality. The asymptote is unchanged; the constant explodes.

What to do:

* Pick one cache. `task_struct_cachep` (`kernel/fork.c`) is a good target; every fork pays.
* Replace `kmem_cache_alloc(task_struct_cachep, ...)` with `kmalloc(sizeof(struct task_struct), ...)` (and similarly for free), gated on `CONFIG_*`.
* Be careful: some caches have constructors that initialize the object. You will need to call them by hand.

How to benchmark: a tight `fork`/`exit` loop (10 000 iterations) measuring spawn rate. Sample `slabtop`: the targeted cache should drop to zero, and `kmalloc-N` (where N is the object size's bucket) should grow by the same amount.

Tools: `perf stat -e cache-misses,LLC-load-misses,instructions`, `slabtop -o -s a`, `/proc/slabinfo`.

Story angle: "We have 200 specialized slab caches. A single allocator is easier to audit and just as fast on modern CPUs."

---

## Inventing Your Own

If none of the five appeals to you, invent your own. The criteria:

* One subsystem, one data-structure swap. Not a bag of unrelated changes.
* Plausibly faster on the surface. The pitch in your pull request should be defensible at a glance.
* Actually slower under a realistic workload. Pick the workload and measure the slowdown.
* A `CONFIG_*` flag so both kernels build from one tree.

Examples of swaps that fit the brief but are not on the list: replace the page cache `xarray` with a linked list; replace the timer wheel with a min-heap; replace the buddy allocator with first-fit linear search; replace per-CPU statistics counters with one global counter under a spinlock.

---

## References

### Projects

* [Project 1](https://github.com/toksaitov/task_struct-list-project), a worked example of a singly-linked list sabotage, with `vng --build`, `perf stat`, and a `fork`/`exit` benchmark.
* [Project 2](https://github.com/toksaitov/ext4-bitmap-project), a worked example of a bitmap-to-list sabotage, with `trace-cmd`, kernel tracepoints, and CDF plots.
* [Project 3](https://github.com/toksaitov/vma-hlist-project), a worked example of a redundant-index sabotage, with `slabtop`, `/proc/slabinfo`, and a three-way kernel comparison.

### Resources

* [Linux Documentation](https://docs.kernel.org), the official kernel documentation.
* [LWN.net Kernel Index](https://lwn.net/Kernel/Index/), a curated set of articles on most subsystems mentioned above.
* [Linux Cross Reference](https://elixir.bootlin.com/linux/v6.8/source), a source browser for fast symbol navigation.

## Deadline

This is a bonus project. The deadline is the last class of the course.
