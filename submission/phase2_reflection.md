[phase2_reflection_final.md](https://github.com/user-attachments/files/34556455/phase2_reflection_final.md)
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
| Naive attention, ms (`attention.naive_ms`) | <!--answer:attn_naive_ms-->641.925<!--/answer--> |
| Vectorized attention, ms (`attention.vectorized_ms`) | <!--answer:attn_vectorized_ms-->2.945<!--/answer--> |
| CPU training, ms for 100 steps (`devices.cpu_ms`) | <!--answer:cpu_ms-->414208.0<!--/answer--> |
| GPU training, ms for 100 steps (`devices.gpu_ms`) | <!--answer:gpu_ms-->13701.2<!--/answer--> |
| Final training loss (`training.final_loss`) | <!--answer:final_loss-->1.4728<!--/answer--> |

**The thread sweep.** Give your four timings from `threads` — 1, 2, 4 and 8 threads — and say
which count was fastest. (verbatim: the numbers, in order)

<!--answer:thread_sweep-->
1 thread: 2706.418 ms/step  
2 threads: 1980.125 ms/step  
4 threads: 2159.1 ms/step  
8 threads: 2385.439 ms/step  
Fastest: 2 threads (1.37× faster than 1 thread).
<!--/answer-->

---

## 2. Intra-node parallelism, three ways

**Why vectorizing wins.** Both attention implementations do the same arithmetic and produce the
same result (the unit test checks that). Explain where the naive version's time actually goes,
and what the vectorized one does differently — in terms of the machine, not the Python.
(~70 words)

<!--answer:vectorization_why-->
Naive attention took 641.925 ms, versus 2.945 ms for vectorized attention (217.97× faster). The naive version issues many small, sequential operations, repeatedly paying interpreter and memory-access overhead. Vectorization batches the work into large matrix operations executed by optimized BLAS kernels. These exploit SIMD instructions, cache locality and efficient memory access, with far less per-operation dispatch. The 217.97× gain therefore reflects both hardware-level vectorization and removal of Python-loop overhead; it is not attributable to SIMD alone.
<!--/answer-->

**Where the thread curve flattens, and why.** Using your sweep: more threads stopped helping —
or started hurting — at some point. Say where, and give the reason. Your Colab runtime's vCPU
count is part of the answer; so is what the threads have to coordinate on. (~80 words)

<!--answer:threading_plateau-->
Step time fell from 2706.418 ms with one thread to 1980.125 ms with two (1.37× speedup), then rose to 2159.1 ms with four and 2385.439 ms with eight. The notebook describes a 2-vCPU Colab environment, so adding workers beyond two cannot create additional CPU capacity. Workers compete for cores, caches and memory bandwidth while coordinating the execution of shared tensor operations through scheduling, synchronization and aggregation. These costs, together with serial work, explain why performance worsened after two threads rather than scaling indefinitely.
<!--/answer-->

**What made the GPU faster.** The model, the data and the step count were identical; only the
device changed. Say what about a transformer training step suits a GPU, and name one workload
that would *not* see this speed-up. (~70 words)

<!--answer:gpu_why-->
Training 100 steps took 414208.0 ms on CPU and 13701.2 ms on the T4 GPU, a 30.23× speedup. Transformer forward and backward passes contain dense matrix multiplications and gradient calculations with many independent arithmetic operations. A GPU executes these concurrently across numerous compute units and benefits from high memory bandwidth. Tiny matrix operations or very small batches may not see this gain because kernel launches and CPU–GPU data transfers can outweigh the useful parallel work.
<!--/answer-->

---

## 3. The question

**Where does parallelism stop helping, and why?** Pull the three experiments together. Your
answer should distinguish the *kinds* of limit you met — work that cannot be divided, hardware
you do not have, coordination that costs more than it saves, data that has to move — and say
which one bound each experiment. One concrete number from section 1 per claim. (~150 words)

<!--answer:parallelism_limit-->
Parallelism helps only while the useful work exceeds the cost of dividing, coordinating and moving it. For attention, vectorization cut runtime from 641.925 to 2.945 ms (217.97×): large native matrix kernels exploited SIMD and efficient data reuse, while the naive implementation was dominated by small operations and dispatch. This gain is not pure SIMD acceleration, and dependencies within attention still limit what can run independently.

CPU threading hit a hardware and coordination limit: with roughly two Colab vCPUs, two threads reached 1980.125 ms/step, but eight took 2385.439 ms/step. Extra workers competed for execution resources, caches and memory bandwidth while synchronizing shared operations; the serial fraction could not be parallelized.

On the T4, 100 training steps fell from 414208.0 ms (CPU) to 13701.2 ms (GPU), because matrix-heavy computation offered enough parallel work. Yet transfers, launches, memory capacity and unavoidable sequential dependencies still impose limits. The 30.23× comparison uses the notebook's restored CPU thread setting, not an optimally tuned multicore CPU.
<!--/answer-->

**What you would measure next.** One experiment this phase did not run that would sharpen the
answer, and what result would change your mind. (~50 words)

<!--answer:next_experiment-->
I would repeat 100-step CPU/T4 benchmarks at batch sizes 1, 4, 16 and 64, holding sequence length fixed, warming up both devices and synchronizing CUDA before timing. I would record median step time, throughput and peak memory. If GPU speedup did not increase with batch size before memory saturation, my workload-amortization hypothesis would be weakened.
<!--/answer-->
