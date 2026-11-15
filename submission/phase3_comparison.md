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
| Cloud MapReduce wall time, seconds (`mapreduce.cloud_elapsed_s`) | <!--answer:cloud_elapsed_s-->57.7<!--/answer--> |
| Local single-process wall time, seconds (`mapreduce.local_elapsed_s`) | <!--answer:local_elapsed_s-->0.011<!--/answer--> |
| Spark wall time for the `collect`, seconds (`tfidf.spark_elapsed_s`) | <!--answer:spark_elapsed_s-->4.5<!--/answer--> |
| Number of **stages** in the Spark job (read off notebook 2b / 3a) | <!--answer:num_stages-->4<!--/answer--> |
| Task **retries** you counted in your `--chaos` run (0 if none happened) | <!--answer:chaos_retries-->10<!--/answer--> |

**Explain the timings.** The three wall times above cover the same 60 documents, but not the
same job: the single-process run and the cloud MapReduce are *the same word count*; Spark's
time is for *TF-IDF*, which is more work (two `reduceByKey`s and a `join`). Rank the three and
explain what each is spending its time on. Say which comparison is like-for-like, and what the
Spark number — a heavier job, on one machine — tells you about where the cloud job's time goes.
(~80 words)

<!--answer:timing_explanation-->
The local word count took 0.011 s, Spark TF-IDF took 4.5 s, and cloud MapReduce took 57.7 s, from fastest to slowest. Only the local and cloud word-count jobs are directly comparable: both count words in the same 60 documents. The cloud job adds Workflows scheduling, authenticated function calls, network latency and intermediate Cloud Storage I/O. Spark performs heavier TF-IDF processing (two `reduceByKey` aggregations and a `join`) on one machine yet finishes much sooner, illustrating how cloud orchestration dominates this small workload.
<!--/answer-->

**What the counts tell you.** From your stage count: how many shuffles did the Spark job have,
and which operations in `spark_tfidf.py` caused them? From your retry count: roughly what
fraction of the 16 task attempts was retried, and is that what `--chaos 0.3` should produce?
(~50 words)

<!--answer:counts_interpretation-->Four Spark stages indicate three shuffles: the two `reduceByKey` operations (TF and DF) and the `join`. Logs Explorer showed ten HTTP 503 failures during the chaos run, consistent with ten retries for 16 initial tasks (62.5% extra attempts). Thus 10/26 attempts failed (38.5%), versus the configured 30% failure probability per attempt; random variation is expected.<!--/answer-->

---

## 2. Spark: transformations, actions, lineage

**Transformations vs actions.** Go through `spark_tfidf.py` and the notebook's Spark cells.
Name each RDD operation and say whether it is a transformation or an action. What happened
when you ran `build_tfidf_rdd` — and what happened when you ran `collect`? (~60 words)

<!--answer:transformations_vs_actions-->
`parallelize(docs)` creates the input RDD. In `spark_tfidf.py`, `flatMap`, `map`, both `reduceByKey` calls, and `join` are transformations; the final `map(to_row)` is also a transformation. `toDebugString()` inspects the lineage without launching the job. Calling `build_tfidf_rdd()` constructs only this lazy computation graph. Calling `collect()` is an action: it schedules the stages, computes TF-IDF rows, and returns them to the driver.
<!--/answer-->

**Lineage and recovery.** Using the lineage you printed in 2b and the timeline in 3a: if the
worker holding one output partition of the *first* `reduceByKey` died, what would Spark have
to recompute, and what could it reuse? Name the stage or shuffle boundary where the recovery
starts. (~50 words)

<!--answer:lineage_recovery-->
Recovery starts at the shuffle boundary of the first `reduceByKey` (the term-frequency aggregation). Spark would reconstruct the lost reduce-side partition using available map-side shuffle blocks and rerun the required aggregation. It could reuse unaffected partitions and intact shuffle outputs. Only if upstream shuffle blocks were also unavailable would it need to recompute the relevant input-side `flatMap` tasks. Downstream dependent partitions would then be recalculated as necessary.
<!--/answer-->

---

## 3. Your cloud MapReduce, mapped onto Hadoop

Name the GCP service or file that played each role (one line each):

- **JobTracker** (schedules tasks, waits for the map phase, retries failures):
  <!--answer:role_jobtracker-->Cloud Workflows (`mr-wordcount`), configured in `workflow.yaml`.<!--/answer-->
- **Workers** (run your map and reduce code):
  <!--answer:role_workers-->The gen2 Cloud Function `mr-worker` (backed by Cloud Run), executing the map and reduce handlers in `main.py`.<!--/answer-->
- **HDFS** (holds inputs, shuffle files, outputs):
  <!--answer:role_hdfs-->Google Cloud Storage (`<job>/input/`, `<job>/shuffle/`, and `<job>/output/`).<!--/answer-->
- **Partitioner** (decides which reducer owns a key):
  <!--answer:role_partitioner-->The `partition()` function in `mapreduce.py`: `zlib.crc32(word.encode("utf-8")) % num_reducers`.<!--/answer-->

**The barrier.** There is no explicit "wait for all mappers" step in `workflow.yaml`. Where,
exactly, is the barrier between the map and reduce phases — which construct provides it?
(~30 words)

<!--answer:barrier_location-->
The `map_phase` uses a Cloud Workflows `parallel` step with a `for` loop. This step completes only after all map branches have finished; only then does execution proceed to `reduce_phase`.
<!--/answer-->

**Idempotence.** The workflow retries a task that failed. Why is that safe with the worker as
written, and what would go wrong if `run_map` *appended* its pairs to the shuffle file instead
of overwriting it? (~60 words)

<!--answer:idempotence-->
`run_map` and `run_reduce` write to deterministic Cloud Storage object names using an upload that overwrites existing contents. Repeating a task therefore replaces its output rather than accumulating records; simulated HTTP 503 failures also occur before any work. If `run_map` appended pairs on a retry, repeated `(word, 1)` entries could be counted twice, inflating the final frequencies.
<!--/answer-->

**The chaos run.** What did you see in the execution log and in Logs Explorer when you ran
with `--chaos 0.3`? Did the job still succeed, and was the final answer still identical to the
local reference? (~50 words)

<!--answer:chaos_observations-->During the `--chaos 0.3` run, Logs Explorer showed ten HTTP 503 responses from `mr-worker` between 01:41:19 and 01:41:43 UTC. Cloud Workflows was configured to retry these transient errors. The execution nevertheless finished `SUCCEEDED` in 57.7 s and returned 2,221 distinct words; the result matched the local reference exactly. The supplied logs do not identify individual retrying branches.<!--/answer-->
