# Phase 3 — Comparison Writeup

Copy this file to `submission/phase3_comparison.md`, fill in every answer slot, and commit it:

```bash
cp phase-3-mapreduce-spark/comparison_template.md submission/phase3_comparison.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers in section 1 must **match `submission/phase3_report.json`** — copy them across, do not
retype from memory; they are checked against it.

---

## 1. Facts from your runs

Numbers only (e.g. `12`, `20.3`, `0.041`).

| | Value |
|---|---|
| Map tasks in the cloud job (`mapreduce.num_splits`) | <!--answer:num_map_tasks-->12<!--/answer--> |
| Reduce tasks in the cloud job (`mapreduce.num_reducers`) | <!--answer:num_reduce_tasks-->4<!--/answer--> |
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->26.9<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.003<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->2.1<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->3<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
Ranking, fastest first: local single process (0.003 s), Spark collect (2.1 s), cloud MapReduce (26.9 s). The like-for-like pair is local against cloud, the same word count: the local run only does the computation, while the cloud job pays for 16 function invocations, container cold starts, Cloud Storage reads and writes for every shuffle file, and Workflows scheduling. Spark did a heavier job (two reduceByKeys and a join) on one machine in 2.1 s, so the cloud job's 26.9 s is almost all overhead, not computation: on 60 documents the cluster is slower.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
The Spark job had 4 stages, hence 3 shuffles: the first reduceByKey (term frequency, spark_tfidf.py line 32), the second reduceByKey (document frequency, line 45) and the join (line 58). In my chaos run, 3 of the 19 attempts (16 tasks plus 3 retries) got a simulated 503, about 16%, or 3 of 16 tasks (19%). That is below the 30% expected from --chaos 0.3, but plausible variance on so few attempts.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
flatMap, map, reduceByKey, join and parallelize are lazy: flatMap, map, reduceByKey and join are transformations, parallelize only creates the source RDD. collect is the only action. Running build_tfidf_rdd computed nothing; it only recorded the lineage that toDebugString prints, with its shuffle boundaries. Running collect then triggered the whole pipeline at once: 4 stages and 96 tasks, returning 5975 rows to the driver.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
If a worker holding an output partition of the first reduceByKey died, Spark would recompute only that lost partition from its lineage: re-run the stage-0 map tasks that feed it, starting at the first shuffle boundary (the term-frequency reduceByKey, stage 0). Everything already computed and kept in the other partitions, and the later stages' finished shuffle files, can be reused; nothing needs to be replicated, and the job is not restarted from scratch.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows: the mr-wordcount workflow (workflow.yaml), with its parallel/for fan-out and try/retry.<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->The mr-worker Cloud Run function (main.py, entry point mr_worker), one invocation per map or reduce task, running as the mr-runner service account.<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Cloud Storage: the bucket clouds-maxence-eurecom-515618-eurecomgpt, holding the input splits, the shuffle files and the output partitions under the mr-<timestamp>/ folder.<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->The partition() function in mapreduce.py (zlib.crc32 of the word modulo the number of reducers), which every map task uses so a word always goes to the same reducer.<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The barrier is the map_phase `parallel` step itself: a parallel step only completes when all its branches have finished, and the reduce_phase step is the next one in the sequence, so it cannot start before the last map task is done.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
Retrying is safe because each task is idempotent: given the same input split it writes the same deterministic output file at a fixed path, overwriting any earlier or partial attempt, so running it twice leaves the same state. If run_map appended instead of overwriting, a retried task would add its pairs a second time and the word counts would be duplicated (wrong answer), with the error depending on how many retries happened.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
With --chaos 0.3 the execution still ended SUCCEEDED (26.9 s) and the result was identical to the local reference (matches_local true, 2221 words). Cloud Run's request log for mr-worker shows 3 requests answered 503 among 35 POSTs in the two runs, and the workflow retried them automatically with the default retry predicate until each task returned 200, so the retries cost time but not correctness.
<!--/answer-->
