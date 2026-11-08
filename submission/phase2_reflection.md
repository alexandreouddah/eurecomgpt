# Phase 2 — Reflection

## 1. Measurements

**Naive attention (ms):** <!--answer:attn_naive_ms-->641.925<!--/answer-->

**Vectorized attention (ms):** <!--answer:attn_vectorized_ms-->2.945<!--/answer-->

**CPU training (100 steps, ms):** <!--answer:cpu_ms-->414208.0<!--/answer-->

**GPU training (100 steps, ms):** <!--answer:gpu_ms-->13701.2<!--/answer-->

**Final training loss (2,000 steps):** <!--answer:final_loss-->1.4728<!--/answer-->

**CPU threading sweep:**

<!--answer:thread_sweep-->
1 thread(s): 2706.418 ms/step
2 thread(s): 1980.125 ms/step
4 thread(s): 2159.1 ms/step
8 thread(s): 2385.439 ms/step
Fastest: 2 threads (1980.125 ms/step; 1.37x faster than one thread).
<!--/answer-->

## 2. Vectorization and SIMD

<!--answer:vectorization_why-->
Naive attention took 641.925 ms, while the vectorized version took 2.945 ms (217.97x faster). Both calculate the same attention operation. The naive implementation runs many small operations through Python loops; the vectorized implementation delegates large matrix operations to optimized native numerical libraries. These kernels can exploit SIMD, cache locality and much lower interpreter overhead. The large measured speedup is therefore not attributable to SIMD alone: removing Python-level loops and improving memory access patterns are also essential.
<!--/answer-->

## 3. Why CPU threading stops scaling

<!--answer:threading_plateau-->
With 1, 2, 4 and 8 intra-op CPU threads, the measured times were 2706.418, 1980.125, 2159.1 and 2385.439 ms/step. Two threads were fastest (1.37x faster than one); four and eight threads were 9.0% and 20.5% slower than two, respectively. More threads do not imply more physical CPU cores. Once available parallel execution resources are saturated, scheduling, synchronization, contention for shared caches and memory bandwidth, and the serial part of each step outweigh any benefit. This short sweep identifies an empirical optimum for this Colab session, not a universal optimal thread count.
<!--/answer-->

## 4. GPU acceleration

<!--answer:gpu_why-->
For the same 100-step training workload, CPU execution took 414208.0 ms and the T4 GPU took 13701.2 ms (30.23x faster). Transformer training involves large tensor multiplications and parallel gradient calculations, which map well to GPU compute units and memory bandwidth. This advantage is workload-dependent: kernel-launch overhead, data transfers and insufficient work per kernel can dominate small tasks. The comparison uses fresh models and is a measured end-to-end result, not a hardware-independent guarantee.
<!--/answer-->

## 5. Limits of intra-node parallelism

<!--answer:parallelism_limit-->
The experiments reveal three different limits. Vectorization accelerated attention by 217.97x, mainly by replacing Python loops with optimized matrix kernels that can exploit SIMD. CPU intra-op threading improved from 2706.418 to 1980.125 ms/step at 2 threads, but degraded to 2385.439 ms at eight threads: available cores, synchronization, shared bandwidth and serial work limit scaling. The T4 completed 100 training steps 30.23x faster than the CPU by exploiting data-parallel tensor operations, yet a GPU still incurs launch and transfer overhead. The full 2,000-step training reached loss 1.4728 with 4935680 parameters; the Shakespeare-like sample has dialogue structure but remains linguistically imperfect. The lesson is to match the parallelism strategy to workload size and bottlenecks rather than maximize threads or device throughput blindly.
<!--/answer-->

## 6. Next experiment

<!--answer:next_experiment-->
I would benchmark the same training step on CPU and T4 for increasing batch sizes and sequence lengths, with identical inputs, warm-up iterations, repeated trials and GPU synchronization before timing. I would record median step time and peak memory alongside throughput. My hypothesis is that GPU acceleration increases as tensor operations grow, while CPU threading saturates once contention dominates; small workloads may be limited by GPU launch and transfer overhead.
<!--/answer-->
