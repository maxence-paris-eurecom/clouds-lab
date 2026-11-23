# Phase 2 — Reflection: where does parallelism stop helping?

Copy this file to `submission/phase2_reflection.md`, fill in every answer slot, and commit it:

```bash
cp phase-2-tiny-gpt/reflection_template.md submission/phase2_reflection.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or your threading plot freely.

Numbers in section 1 must **match `submission/phase2_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Your measurements

| | Value |
|---|---|
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->422.204<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->2.559<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->428293.4<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->13824.0<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4801<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2164.544 ms; 2 threads: 2114.237 ms; 4 threads: 2805.643 ms; 8 threads: 2189.57 ms. The fastest was 2 threads (2114.237 ms), only about 2% faster than 1 thread.
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
The naive version spends its time in the Python interpreter: every dot product is a separate loop iteration with interpreter overhead, doing one multiply-add at a time. The vectorized version hands whole matrices to NumPy, which calls a BLAS library compiled with SIMD (AVX) instructions that apply one instruction to several floats per cycle, with cache-friendly blocking and no per-element Python cost. Result: 422.2 ms against 2.56 ms, about 165 times faster for the same maths.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
The curve never really goes down: 2165 ms at 1 thread, 2114 ms at 2 (the best, only 2% better), then 2806 ms at 4 (30% slower than 1 thread) and 2190 ms at 8. The Colab runtime has very few vCPUs (torch's default thread count printed 1), so extra threads cannot run in parallel; they time-share the same core. They also have to synchronise at the end of every matrix operation and share memory bandwidth and caches, so beyond the cores available the coordination costs more than it saves.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
A transformer training step is dominated by large dense matrix multiplications (attention, the MLP, the projections) in which thousands of independent multiply-adds can run at once, which is what a GPU's thousands of arithmetic units are built for. 100 steps took 13824 ms on the T4 against 428293 ms on the CPU, about 31 times faster. A workload that would not benefit: one that is sequential or branch-heavy, such as a pointer-chasing graph traversal, or small jobs where copying data to the GPU costs more than the compute.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Three different limits bound the three experiments. Vectorization (SIMD) was limited by nothing but the interpreter: the same work ran about 165x faster, 422.2 ms to 2.56 ms. The thread sweep was bound by hardware I do not have: with about one usable core, 4 threads (2806 ms) were slower than 1 (2165 ms) because synchronisation and shared memory bandwidth cost more than the extra threads gave, the best being 2 threads at 2114 ms. The GPU gave about 31x (428293 ms to 13824 ms) because the matrix work divides into thousands of independent pieces; its limit is then the part that cannot be divided (the 2000 steps run one after another, and sampling is sequential) and the cost of moving data to the device. So parallelism stops helping when the work cannot be split, when there are no more execution units, or when coordination and data movement cost more than they save.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would train with several batch sizes (16, 64, 256) on the GPU and plot time per step. If time per step stays flat, the GPU was underused and the limit was the work available; if it grows linearly, the GPU is saturated. That would show where the 31x stops growing.
<!--/answer-->
