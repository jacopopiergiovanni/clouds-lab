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
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->42.1<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.01<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->3.9<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->5<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
Ranked fastest to slowest: local word count (0.01 s), Spark TF-IDF (3.9 s), cloud MapReduce (42.1 s). Local and cloud are like-for-like, the same word count, so the cloud job is about 4,000 times slower. Spark does more work (two reduceByKeys and a join) on one machine in a tenth of the time. So the cloud job's time is overhead, not compute: function invocations, Cloud Storage round trips, workflow scheduling and retry backoff on a job too small to amortise them.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->
The Spark job had 3 shuffles, giving 4 stages (confirmed in 3a): the reduceByKey in term_freq (stage 0), the reduceByKey in doc_freq (stage 1), and the join (stage 2), followed by the collect stage. The chaos run had 5 failed attempts across 16 tasks, about 31%, which is close to the 30% per-attempt failure rate that `--chaos 0.3` should produce.
<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
flatMap, map, reduceByKey, join and the final map (to_row) are transformations since they only extend the lineage. collect is the only action. Running build_tfidf_rdd computed nothing and just built the lineage, which toDebugString printed. Running collect triggered all stages at once and returned the rows to the driver.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
If the worker holding one output partition of the first reduceByKey died, Spark would rerun only the stage 0 map task that produced the lost shuffle output (stage 0 has 2 tasks and wrote 51 KB), reading the source documents again. The other partition and the outputs of later stages are reused. Recovery starts at the first shuffle boundary.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows (workflow.yaml, the mr-wordcount workflow).<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->the Cloud Run function mr-worker (main.py), one invocation per map or reduce task.<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Cloud Storage, the <project>-eurecomgpt bucket (the input splits, shuffle files and outputs under mr-*/).<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner--> partition() in mapreduce.py, which computes crc32(word) % num_reducers.<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The barrier is the parallel step (map_phase): it only completes when all its for branches finish, and reduce_phase comes after it in the sequence. No separate wait step is needed.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
A retried task rewrites the same files with the same content, because run_map overwrites split-*/part-*.json and its output depends only on its input, so repeating it changes nothing. If it appended, a retry after a partial write or timeout would duplicate pairs in the shuffle files, the counts would be inflated, and the cloud result would no longer match the local reference.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->
With --chaos 0.3 the Steps view showed failed call_map steps, each followed by a succeeded try_retry_except, and Logs Explorer showed 5 mr-worker requests with HTTP 503 at the same time. Every task eventually succeeded, the execution finished SUCCEEDED in 42.1 s, and the result was identical to the local reference (matches_local true).
<!--/answer-->
