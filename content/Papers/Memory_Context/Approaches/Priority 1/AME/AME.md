---
Date: 11 2025
Authors: Xinkui Zhao, Qingyu Ma, Yifan Zhang, Hengxuan Lou, Guanjie Cheng, Shuiguang Deng, Jianwei Yin
Venue: arxiv
Paper: "AME: An Efficient Heterogeneous Agentic Memory Engine for Smartphones"
Memory type:
  - Token-level
Agent env: Multi-agent
Record format: Text
Memory architecture:
  - 3-tier
  - graph-a
Tackle Module: Memory design
Need offline initialization: false
Fine-tuning?: false
Other tags:
---
## 1. Terminology

| Term                                       | Definition                                                                                                                                                                                                                                                                                                                                                                                        |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| AME (Agentic Memory Engine)                | The proposed on-device CPU–GPU–NPU hybrid vector database/index runtime, co-designed with smartphone SoCs to serve continuously updated agentic memory.                                                                                                                                                                                                                                           |
| SoC (System-on-Chip)                       | A mobile chip integrating a CPU (performance + efficiency clusters), GPU (compute-unit clusters with private GMEM), and NPU (scalar cores + vector unit + matrix engine), all sharing a System-Level Cache.                                                                                                                                                                                       |
| IVF (Inverted File Index)                  | A vector index that partitions the embedding space into clusters via centroids and restricts search to the nearest clusters' posting lists, reducing distance computations versus exhaustive scan.                                                                                                                                                                                                |
| HNSW (Hierarchical Navigable Small World)  | A graph-based approximate nearest-neighbor index that layers proximity graphs, relying on pointer-chasing traversal and large, cache-resident neighbor lists.                                                                                                                                                                                                                                     |
| HMX (Hexagon Matrix eXtension)             | The NPU's dedicated matrix engine that executes GEMM operations on FP16 tile-major operands with a minimum kernel shape of 32×64×64 (M×N×K).                                                                                                                                                                                                                                                      |
| HVX (Hexagon Vector eXtension)             | The NPU's vector-processing cluster used for FP32↔FP16 type conversion (`vcvt`), tile packing/unpacking (`vdeal`), and in-place transpose (`vshuff`) without spilling to DDR.                                                                                                                                                                                                                     |
| TCM (Tightly Coupled Memory)               | An 8 MiB on-chip SRAM buffer local to the NPU subsystem used to stage matrix tiles for HMX/HVX, avoiding repeated DDR round-trips.                                                                                                                                                                                                                                                                |
| SLC (System-Level Cache)                   | An 8 MiB cache shared by CPU, GPU, and NPU that anchors the SoC's unified memory architecture.                                                                                                                                                                                                                                                                                                    |
| GEMM (General Matrix Multiply)             | The dense matrix-multiplication primitive to which AME reformulates vector similarity, insertion, and centroid-update operations.                                                                                                                                                                                                                                                                 |
| FastRPC                                    | This is Qualcomm's system for devices like phones. It offloads tasks (like image processing) from the main CPU to the Digital Signal Processor (DSP).<br><br>- **How it works:** A file defines the task. It splits the task into a "stub" (for the CPU) and a "skeleton" (for the DSP). The CPU tells the DSP to do the work. <br><br>Each call costs 200–700 μs, motivating batched invocation. |
| SMT (Simultaneous Multi-Threading runtime) | AME's two-thread scheme that overlaps GEMM execution on one thread with asynchronous DMA prefetch of the next tile on the other, using double-buffered TCM.                                                                                                                                                                                                                                       |
| Data Adaptation Layer                      | The NPU-resident component performing FP32↔FP16 conversion, in-place layout transpose, and batched/shared-memory invocation, eliminating CPU-side preprocessing.                                                                                                                                                                                                                                  |
| Windowed Batch Submission                  | The scheduler's strategy of submitting only a bounded window of tasks to a global queue at a time, capping peak memory while avoiding per-task pipeline stalls.                                                                                                                                                                                                                                   |
| Template-driven execution                  | AME's mapping of four recurring access patterns (query, update, index rebuild, query–update hybrid) to fixed profiling-guided CPU/GPU/NPU routing templates.                                                                                                                                                                                                                                      |
| G1 (SoC–Database Mismatch)                 | The gap between server-oriented vector-database assumptions (wide bandwidth, large DRAM, mature accelerator ecosystems) and mobile SoC constraints (tight bandwidth, limited on-chip memory, no unified matrix runtime).                                                                                                                                                                          |
| G2 (Workload Mismatch)                     | The gap between servers' offline-built, relatively static indexes and on-device usage, which requires indexes to absorb continuous inserts/deletes concurrently with queries.                                                                                                                                                                                                                     |
| DDR Memory                                 | the standard type of random-access memory (RAM) used in modern computers, servers, and smartphones. It transfers data on both the rising and falling edges of the clock signal, effectively doubling the data transfer rate without increasing the clock frequency                                                                                                                                |
| default variable pass-through interface    | when the CPU wants to hand data to the NPU (via FastRPC), the arguments (e.g., the matrix data) get serialized, copied into the RPC transport buffer, sent across the CPU–NPU boundary, and deserialized/copied again on the receiving side. Same thing happens in reverse for the output.                                                                                                        |
| ION                                        | Linux kernel memory allocator lets you share the _same physical memory_ across processors. ION buffers are converted to file descriptors and mapped into the NPU's address space via `fastrpc_mmap`/`HAP_mmap` --> NPU use the same memory slot with CPU.                                                                                                                                         |
| DMA (Direct Memory Access)                 | is a hardware mechanism that lets data move between memory and a device (or between two memory locations) without the CPU having to shepherd every byte.                                                                                                                                                                                                                                          |

---

## 2. Method Summary (What)

AME is an **on-device vector database engine for smartphone agentic memory** that reformulates core vector-index operations (similarity search, insertion, centroid update, index rebuild) as accelerator-native GEMM computations, then **routes these GEMMs across a smartphone SoC's CPU, GPU, and NPU through a shared-memory pipeline.** It pairs this hardware-aware matrix pipeline with a workload-aware IVF index and scheduler that route latency-critical queries, frequent small updates, and large batch rebuilds to whichever compute unit profiling shows is best suited, using a fixed set of execution templates and a memory-bounded task scheduler.

---

## 3. What it solves (Why)

- Mobile platform's heterogenous resources (CPU, NPU, GPU) is underutilized in retrieval-augmented workloads
- Lack mobile-oriented vector database design: Server-oriented vector databases (Flat, HNSW, IVF, Faiss IVF-PQ, CAGRA) are ineffective when ported directly to mobile SoCs 
	- **IVF (CPU/GPU):** relies on wide, fast memory to probe many lists in parallel and ample DRAM for centroids/postings — unavailable under mobile bandwidth and memory budgets.
	- **HNSW (CPU, cache-rich):** assumes large memory for graph neighbors and thread-rich, cache-friendly pointer chasing, leading to irregular access patterns and out-of-memory failure at high recall targets on smartphones.
	- **Faiss IVF-PQ (CPU):** is heavy-compute and DRAM-bound, with no path to exploit the NPU/GPU.
	- **Faiss IVF-PQ (GPU) / CAGRA (GPU):** are CUDA-only and either high-power or VRAM-hungry, incompatible with mobile GPU/NPU ecosystems entirely.
- lack a unified matrix runtime for distributing work across CPU/GPU/NPU, and on-device usage requires indexes to serve queries while continuously absorbing inserts/deletes/rebuilds, a pattern server-side static-index designs do not target.

---

## 4. Methodology (How)

> **Running example:** A 100k-vector agentic memory corpus, 1024-dimensional embeddings (BGE-Large-en-v1.5), indexed with an IVF of 128 clusters (a multiple of 64) on a Snapdragon 8 Elite Gen 5.

![[AME.pdf#page=3&rect=317,539,560,728|AME, p.3]]
### A. System / Memory Structure — Hardware-Aware Heterogeneous Vector Computation Pipeline

1. **Data Adaptation Layer (on the NPU):**
- **_Type conversion:_** 
	- FP32 → FP16 conversion (going _into_ the NPU)
		- The FP32 embedding values are loaded into **HVX vector registers** (not touched by the CPU).
		- `vcvt` (vector convert) runs across the register, converting each FP32 element to FP16, halving the storage size in-register.
		- `vdeal` then reshuffles those converted values into HMX's expected **tile-major packed format**

		- All of this happens **inside HVX registers** — nothing is written back to DDR mid-process, so there's no extra memory traffic or peak-memory doubling.
	- FP16 → FP32 conversion (coming _out_ of the NPU)
		- The GEMM's raw output `Z` is in FP16, packed tile-major.
		- HVX runs the **mirror process**: `vshuff` unpacks the tiles back into flat row-major order, then `vcvt` expands FP16 values back up to FP32.
		- The result `Z(FP32)` is now in a format the CPU can consume normally (e.g., for top-k aggregation).

> rearranging a chunk of the 1024 values into 64×64 tiles matching HMX's minimum kernel shape, rather than leaving them in flat row-major order, ready to feed directly into HMX's GEMM units.

- ***In-place transpose (also inside HVX, if needed)***
	- Vector similarity needs an `AB^T` computation, but HMX is optimized for a plain `AX` pattern.
	- So before/after conversion, HVX performs an **on-chip transpose** using a sequence of `vshuff` (vector shuffle) sub-block swaps, reordering elements register-to-register into the transposed tile-major form HMX expects.
	- This avoids a naïve transpose, which would require writing the whole matrix back to DDR, transposing it there, and re-reading it
> Original (row-major, as loaded from memory):
> 
> Row0: \[ b00  b01  b02  b03 ]
> Row1: \[ b10  b11  b12  b13 ]
> Row2: \[ b20  b21  b22  b23 ]
> Row3: \[ b30  b31  b32  b33 ]
> 
> Step 1a: swap b01 ↔ b10, and b23 ↔ b32, etc. (element-pair level)
> 
> Row0: \[ b00  b10  b02  b12 ]
> Row1: \[ b01  b11  b03  b13 ]
> Row2: \[ b20  b30  b22  b32 ]
> Row3: \[ b21  b31  b23  b33 ]
> 
> Stage 2 (Target) :swap the larger 2×2 quadrant blocks (interleave at the block level)
> 
> Row0: \[ b00  b10  b20  b30 ]
> Row1: \[ b01  b11  b21  b31 ]
> Row2: \[ b02  b12  b22  b32 ]
> Row3: \[ b03  b13  b23  b33 ]
> 


- **_Amortizing invocation overhead:_** 
	- since each FastRPC call costs 200–700 μs, multiple GEMM tasks are batched into a single invocation
	- ION-based shared-memory mapping avoids user-space/driver copies.

2. **Execution-Transfer Overlapping:** 
- the NPU's TCM is only 8 MiB (too small for full matrix tiles), an SMT runtime runs two cooperative threads :
	- **Thread A (compute thread):** runs HMX/HVX operations on the tile currently sitting in TCM.
	- **Thread B (transfer thread):** issues an **asynchronous DMA request** to pull the _next_ tile from DDR into TCM, while Thread A is still busy computing.
- TCM is split into (at least) two buffer regions:
	- One holds the tile currently being computed on.
	- The other is being filled by DMA with the next tile.
    
> Time →
> 
> Buffer A: \[ DMA loads tile 1 \] → \[ HMX computes tile 1 \] → (idle, refilling next)
> Buffer B:                                       \[ DMA loads tile 2 ]     → \[ HMX computes tile 2 ]
>                                                         ↑
>                                                   starts the instant tile 1's compute finishes
    
2. **Data Sharing Across Units:** 
- a unified, file-descriptor-based memory framework maps a single host buffer into each device's address space (OpenCL for GPU; ION + `fastrpc_mmap` + `HAP_mmap` for NPU), avoiding replication. 
	- **Allocate on the host (CPU):** the CPU allocates a buffer through the ION allocator, this buffer lives in physical DRAM (or on-chip memory, depending on the ION heap chosen).
	- **Convert to a file descriptor:** ION buffers are represented as file descriptors (this is a standard Linux DMA-BUF pattern) 
	- **Register with the NPU via `fastrpc_mmap`:** that file descriptor is handed to the NPU driver, which maps the _same physical memory_ into the NPU's own RTOS virtual address space.
	- **`HAP_mmap` on the NPU side:** inside the NPU's runtime, `HAP_mmap` manages that mapping and keeps a consistent translation between the DDR-backed buffer and the NPU's on-chip OS address space.
- Because Snapdragon uses one-way cache coherence, AME explicitly flushes CPU cache lines before the NPU polls shared buffers to guarantee data freshness.

### B. Retrieval Flow — Hardware-Aware Vector Index Design

Take **centroid assignment during index build**:
- a matrix of database vectors: **M vectors × K dimensions**
- a matrix of centroids: **N clusters × K dimensions**
Computing the (squared Euclidean, or cosine-related) distance between _every_ vector and _every_ centroid reduces to computing:

```
Distances = Embeddings (M × K)  ×  Centroids^T (K × N)
          = a single M × N GEMM
```

So, instead of looping vector-by-vector, centroid-by-centroid, you do it as **one dense GEMM**: `M × N × K`. This is precisely the `M × N × K` shape that HMX's kernel is built around.

Why insertion and rebuild are GEMMs too

- **Insertion:** assigning a batch of new vectors to clusters = same distance-to-centroid computation, just with a smaller M (the batch size), still an M×N×K GEMM.
- **Index rebuild:** recomputing centroids and reassigning the _entire_ corpus = the same GEMM, just at full corpus scale (large M).
- **Query:** computing distance from a query (or batch of queries) to centroids, then to in-cluster vectors, is again the same dot-product-matrix pattern, just with M = number of queries.
1. The NPU's minimum GEMM kernel is 32×64×64 (M×N×K), so the IVF index is redesigned around this shape:
    - Cluster count (the **N** dimension for insertion/rebuild GEMMs) is rounded to a multiple of 64.
    - Embedding dimension (**K**) is typically already a multiple of 64 (e.g., 1024 in the running example).
    - Database vector count (**M**) is rounded up to the nearest multiple of 32.
    
    > With 128 clusters and 1024-dim embeddings, centroid-update GEMMs exactly fill NPU tiles (128 = 2×64), avoiding fragmented, partially-filled kernels.
    
2. **Query template:** for latency-critical RAG lookups, LLM prefilling/decoding is assigned to the NPU while vector search (probing the relevant IVF clusters) runs on the CPU.

### C. Updating Flow — Workload-Aware Vector Index Design and Scheduling

1. AME profiles CPU, GPU, and NPU GEMM throughput across matrix sizes (Figure 4) to determine which unit suits which access pattern, then defines four templates:
    
    - **Update template:** small-matrix inserts/replaces use CPU (metadata/index coherence) + GPU (batched insertion); the NPU is skipped since matrices are too small to justify its overhead.
    
    > A new interaction vector is inserted into the 100k-vector index: the CPU updates cluster metadata while the GPU batches the vector into its assigned posting list.
    
    - **Index template:** full index rebuilds, which expose large matrices across the whole corpus, jointly use CPU, GPU, and NPU for high-throughput GEMM.
    - **Query–update hybrid template:** mixed workloads keep NPU prefilling/decoding on the query side while CPU/GPU share search and insertion based on queue depth and system load.
    - Across all templates, top-k aggregation post-processing runs on the CPU.
1. **Memory-efficient (worker-pulled) scheduler:**
	- each logical operation is decomposed into fine-grained tasks. 
	- Instead of submitting all tasks at once (memory spike) or one task per idle worker (pipeline bubbles), a **Windowed Batch Submission** strategy releases only a bounded window of tasks to a global queue
	- backend-bound worker threads (CPU/GPU/NPU) pull tasks when idle, giving implicit load balancing without a central dispatcher.

---

## 5. Benchmarks

### 5.1 Other Baselines

|Baseline|Type|
|---|---|
|Flat|Exact search, brute-force|
|HNSW|Graph-based ANN index|
|IVF-HNSW|Hybrid inverted-file + graph index|
|Single-backend variants of AME|AME restricted to CPU-only or GPU-only execution, to isolate the benefit of heterogeneous scheduling|

### 5.2 Benchmarks

| Item               | Detail                                                                               |
| ------------------ | ------------------------------------------------------------------------------------ |
| Dataset            | [[HotpotQA]] — 113k Wikipedia question–answer pairs                                  |
| Corpus sizes       | 10k, 100k, and 1M embedding vectors (three constructed corpora)                      |
| Embedding model    | BGE-Large-en-v1.5                                                                    |
| LLM backbone       | Llama 3-3B (prefilling/decoding via Genie SDK)                                       |
| Hardware platforms | Snapdragon 8 Gen 4 (Qualcomm Cloud Phone) and Snapdragon 8 Gen 5 (Redmi K90 Pro Max) |
| Metrics            | Latency (ms), QPS, IPS, Recall@K, GFLOPS                                             |

### 5.3 Notable Results

|Result|Value|
|---|---|
|Query throughput vs. baselines at matched recall|up to 1.4×|
|Index construction speed vs. HNSW (same recall target)|up to 7× faster|
|Index construction speed vs. AME's own single-backend variant|up to 2.5× faster|
|Insertion throughput under concurrent query workload vs. HNSW|up to 6×|
|Concurrent-insertion speedup vs. HNSW|up to 2.1×|
|Concurrent-insertion speedup vs. own single-backend variant|up to 1.5×|
|NPU GEMM throughput, ablation (HVX-only → full AME)|195 → 637 GFLOPS|
|Effect of IVF cluster count not being a multiple of 64|markedly higher (fragmented) build latency, confirmed empirically (Fig. 9)|
|HNSW at very high recall on large corpus|fails to build at all (memory exhaustion on smartphone)|

---

## 6. Strengths

- **Full-stack hardware–software co-design:** AME jointly redesigns the vector-index algorithm (IVF cluster/tile alignment) and the low-level execution pipeline (data adaptation, DMA overlap, shared memory)
- **Real-device, real-workload evaluation:** all results are measured on physical Snapdragon 8 Gen 4/Gen 5 devices across three corpus scales
- **Fine-grained ablation of NPU optimizations:** the five-configuration ablation (Figure 8) isolates the individual and combined contribution of SMT, TCM staging, DMA transfers, and execution-transfer overlap, showing each is necessary.
- **Empirically validated design hypothesis:** the claim that cluster count should be a multiple of 64 (to match the NPU's minimum GEMM kernel) is directly tested via a latency sweep (Figure 9), not just asserted.
- **Workload-differentiated scheduling:** the template-driven design is grounded in measured per-device GEMM throughput profiles rather than a fixed heuristic, so routing decisions reflect actual hardware capability at each matrix size.

---

## 7. Gaps

- **Single-vendor hardware scope:** all evaluation is on Qualcomm Snapdragon 8-series SoCs (Hexagon NPU/HMX/HVX); there is no evidence the approach generalizes to other mobile NPU architectures (e.g., Apple Neural Engine, MediaTek APU, Samsung Exynos NPU) 
- **No deletion evaluation:** G2 explicitly motivates support for "frequent inserts, deletions," but the reported hybrid workload experiments (Figure 7) measure only insertion throughput (IPS) and query throughput (QPS) 
- **Single embedding/LLM configuration:**
- **No energy or thermal measurements:** mobile power and thermal budgets are cited as a core motivating constraint (Section 3), but the evaluation reports only throughput, latency, and recall 
- **Scalability beyond 1M vectors untested:** the largest evaluated corpus is 1M vectors; a genuinely long-lived, continuously accumulating agentic memory (weeks/months of interaction history) could exceed this scale, and behavior of the hardware-aware IVF design under further growth or memory pressure is not characterized.
- **Index staleness/recall drift under continuous updates not measured:** while concurrent insert/query throughput is reported, the paper does not assess whether sustained insertion during serving degrades retrieval recall over time (e.g., before periodic rebuilds restore quality).
- **backend-skewed queue contention:** the memory-efficient scheduler may schedule all NPU-bound task in the queue, making other backends waiting endlessly

---

## 8. Highlights

> [!PDF|255, 208, 0] [[AME.pdf#page=1&annotation=767R|AME, p.1]]
> > Keeping the agent’s vector memory on-device ensures low latency and privacy by avoiding network access to sensitive data

