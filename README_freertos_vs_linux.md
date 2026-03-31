# FreeRTOS vs Linux: Architectural Comparison

A detailed technical comparison across scheduling, memory management, file systems, device drivers, and network packet handling.

## 1. Design Philosophy

| | FreeRTOS | Linux |
|---|---|---|
| **Type** | Real-time operating system (RTOS) | General-purpose operating system (GPOS) |
| **Target** | Microcontrollers, embedded systems (KB–MB RAM) | Servers, desktops, mobile, embedded (MB–TB RAM) |
| **Priority** | Determinism, low latency, small footprint | Fairness, throughput, scalability |
| **Address space** | Single address space (all tasks share memory) | Per-process virtual address spaces (MMU isolation) |
| **Binary size** | ~5–20 KB kernel | ~20–80 MB compressed kernel image |
| **API surface** | ~50 kernel API functions | ~400 syscalls |

## 2. Process / Task Scheduling

### FreeRTOS

**Algorithm:** Fixed-priority preemptive scheduling with optional time slicing.

- **Ready lists:** Array of `List_t` (`pxReadyTasksLists[configMAX_PRIORITIES]`), one per priority level. No dynamic priority adjustment.
- **Task selection:** O(1) via bitmap + CLZ instruction (`uxTopReadyPriority`). Equal-priority tasks share CPU via round-robin (`pxIndex` advances).
- **Priorities:** Fixed at compile time (`configMAX_PRIORITIES`, typically 5–32). No nice values or weight-based fairness.
- **Preemption:** `configUSE_PREEMPTION` enables/disables. Per-task preemption disable supported (`configUSE_TASK_PREEMPTION_DISABLE` enables `xPreemptionDisable` in TCB).
- **Tick:** Fixed tick rate (`configTICK_RATE_HZ`, typically 100–1000 Hz). Tickless idle supported (`portSUPPRESS_TICKS_AND_SLEEP`).
- **SMP:** `configNUMBER_OF_CORES` (added in V11). Core affinity via `uxCoreAffinityMask`. Simple load balancing — no scheduling domain hierarchy.
- **Real-time guarantees:** Worst-case context switch time is deterministic and bounded (typically sub-microsecond on Cortex-M).

### Linux

**Algorithm:** CFS (Completely Fair Scheduler) for SCHED_NORMAL; FIFO/RR/Deadline for real-time.

- **Run queue:** Per-CPU red-black tree (`struct cfs_rq`) ordered by `vruntime` (virtual runtime). Leftmost node = next task. O(log N) insert/delete.
- **Task selection:** Weighted fairness based on nice values (-20 to +19). No fixed time slices — CPU time divided proportionally.
- **Priorities:** 0–139 (0–99 real-time, 100–139 normal). Real-time policies: `SCHED_FIFO` (run until blocked), `SCHED_RR` (round-robin with quantum), `SCHED_DEADLINE` (EDF with runtime/period/deadline).
- **SMP load balancing:** Hierarchical scheduling domains: SMT (hyper-threading siblings) → MC (cores sharing L2/L3) → NUMA. Periodic and on-demand migration of tasks between domains.
- **Tickless:** `CONFIG_NO_HZ_IDLE` (idle tickless) and `CONFIG_NO_HZ_FULL` (full tickless for real-time/HPC).
- **Real-time:** PREEMPT_RT patch set makes most kernel code preemptible. Worst-case latency is ~10–100 µs on x86 (vs sub-µs for FreeRTOS on Cortex-M).

### Key Difference

FreeRTOS uses **static, compile-time priorities** and is always preemptive. Linux uses **dynamic vruntime-based fairness** for normal tasks and supports preemption at multiple levels (voluntary, full, RT). FreeRTOS gives deterministic worst-case latency; Linux optimizes for fairness and throughput.

```
FreeRTOS Ready Lists:                    Linux CFS:
Priority 0 (lowest): [TaskA]            RB-Tree:
Priority 1: [TaskB ↔ TaskC]              ┌─────────┐
Priority 2: [TaskD]                      │ vruntime │
  ...                                     └────┬────┘
Priority N (highest): []                    ┌──┴──┐
                                            │     │
Task selection: bitmap CLZ = O(1)        (lesser) (greater)
                                          ┌─┴─┐   ┌─┴─┐
                                          │   │   │   │
```

## 3. Memory Management

### FreeRTOS

**No MMU. No virtual memory. No per-task address spaces.**

All tasks share a single flat address space. There is no hardware-enforced memory isolation between tasks.

- **Heap schemes** (5 mutually exclusive options in `portable/MemMang/`; exactly one is linked into the binary — all define the same symbols `pvPortMalloc`/`vPortFree`, so linking more than one causes duplicate symbol errors):

| | heap_1 | heap_2 | heap_3 | heap_4 | heap_5 |
|---|---|---|---|---|---|
| **One-line summary** | Bump/arena — allocate only, never free | Best-fit free list — free but no coalescing | Thin wrapper around libc `malloc`/`free`. **POSIX:** wraps glibc ptmalloc2. **Bare-metal:** wraps newlib/picolibc malloc (you must provide `_sbrk_r()`). | First-fit free list with adjacent block coalescing | Same as heap_4, across non-contiguous memory regions |
| **Heap storage** | Static `ucHeap[configTOTAL_HEAP_SIZE]` | Static `ucHeap[configTOTAL_HEAP_SIZE]` | **POSIX (glibc):** main arena via `brk()` (initial ~132 KB); thread arenas via `mmap()` (≥1 MB each); large allocs (≥128 KB) via individual `mmap()`. **Bare-metal (newlib):** no `ucHeap[]`, no `configTOTAL_HEAP_SIZE`. Memory obtained via `_sbrk_r()` — a bump pointer from linker symbol `_end` (end of `.bss`) toward stack. No `brk()`, no `mmap()`, no virtual memory. | Static `ucHeap[configTOTAL_HEAP_SIZE]` | User-defined via `vPortDefineHeapRegions()` |
| **Block metadata** | None — no per-block header, just a global index `xNextFreeByte` | `BlockLink_t {pxNextFreeBlock, xBlockSize}` — ~8–16 byte header per block | **POSIX (glibc):** `malloc_chunk {prev_size, size, fd, bk, fd_nextsize, bk_nextsize}` — 16–56 byte overhead. 3 flag bits in `size`: `PREV_INUSE`, `IS_MMAPPED`, `NON_MAIN_ARENA`. Allocated chunks store only `prev_size` + `size`. **Bare-metal (newlib):** simpler `{size, fd, bk}` header (~12 bytes on 32-bit). No `prev_size`, no `fd_nextsize`/`bk_nextsize`, no flag bits. | `BlockLink_t {pxNextFreeBlock, xBlockSize}` — ~8–16 byte header per block; MSB of `xBlockSize` = allocated flag | Same as heap_4 |
| **Free list structure** | No free list — just a bump index | Singly-linked list, sorted by **block size** (ascending) | **POSIX (glibc):** 5 bin types: **tcache** (per-thread, 64 bins, max 7 each, LIFO, no coalesce); **fast bins** (10 bins ≤ 176 B, LIFO, no coalesce); **unsorted bin** (1 bin, FIFO cache); **small bins** (62 bins, exact-size FIFO); **large bins** (63 bins, size-range sorted). **Bare-metal (newlib):** single free list, singly-linked by address — essentially identical to FreeRTOS heap_4's approach. No bins, no tcache. | Singly-linked list, sorted by **memory address** (ascending) | Same as heap_4 |
| **Sentinel nodes** | None | `xStart` (head, size=0) + `xEnd` (tail, size=`configADJUSTED_HEAP_SIZE`) | **POSIX (glibc):** `top chunk` (wilderness at heap high-water mark) + `bin_head[]` array. **Bare-metal (newlib):** `__malloc_free_list` head pointer. No top chunk concept — raw memory obtained on demand from `_sbrk_r()`. | `xStart` (head, size=0) + `pxEnd` (tail sentinel, size=0) | Same as heap_4; `pxEnd` moves to end of last region |
| **pvPortMalloc algorithm** | 1. Align requested size up to `portBYTE_ALIGNMENT`. 2. Check `xNextFreeByte + size < configADJUSTED_HEAP_SIZE`. 3. Return `pucAlignedHeap + xNextFreeByte`; increment `xNextFreeByte`. O(1). | 1. Add `xHeapStructSize` header + align. 2. Walk size-sorted free list for **best-fit** (smallest block ≥ requested). 3. If block is ≥ 2× `xHeapStructSize` larger, split into allocated + new free block; insert remainder back into size-sorted list. 4. Mark block allocated (MSB of `xBlockSize`). O(n). | 1. `vTaskSuspendAll()`. 2. Call `malloc()`. **POSIX (glibc):** tcache O(1) → fast bin O(1) → small bin O(1) → unsorted bin → large bins → split top chunk → `brk()`/`mmap()`. **Bare-metal (newlib):** walk single free list **first-fit** → if found, split if remainder ≥ minimum → if not found, call `_sbrk_r()` to get more memory from `_end` bump pointer. 3. `xTaskResumeAll()`. | 1. Add `xHeapStructSize` header + align. 2. Walk address-sorted free list for **first-fit** (first block ≥ requested). 3. If remainder ≥ 2× `xHeapStructSize`, split: new free block inserted at same position (still address-sorted). 4. Mark allocated (MSB), update `xFreeBytesRemaining`, track `xMinimumEverFreeBytesRemaining`. O(n). | Same as heap_4. `configASSERT(pxEnd)` on entry — crashes if `vPortDefineHeapRegions()` not called first. |
| **vPortFree algorithm** | **No-op that asserts `pv == NULL`** — memory cannot be freed. | 1. Subtract header offset to get `BlockLink_t *`. 2. Validate allocated flag (MSB) and `pxNextFreeBlock == NULL`. 3. Clear allocated flag. 4. Insert back into **size-sorted** free list. 5. Update `xFreeBytesRemaining`. O(n). No merge of adjacent blocks. | 1. `vTaskSuspendAll()`. 2. Call `free()`. **POSIX (glibc):** tcache push O(1) → fast bin push O(1) → else coalesce with neighbors (`unlink()` + merge), place in unsorted bin → if borders top, merge into top + possible `brk()`-release. **Bare-metal (newlib):** coalesce with adjacent free blocks, insert into single free list (address-ordered). Memory is never returned to `_sbrk_r()` — the bump pointer only grows. 3. `xTaskResumeAll()`. | 1. Subtract header offset, validate allocated flag + null next pointer. 2. Clear allocated flag. 3. `prvInsertBlockIntoFreeList()`: walk address-sorted list to find insertion point. **Coalesce with previous block** if `(prev_addr + prev_size) == freed_addr`. **Coalesce with next block** if `(freed_addr + freed_size) == next_addr`. 4. Update `xFreeBytesRemaining`. O(n). | Same algorithm as heap_4. Coalescing also merges across region boundaries (regions are linked by sentinel blocks in the address-ordered list). |
| **Block splitting** | N/A — no blocks, no splitting | Yes — if `(blockSize - wantedSize) > 2 × xHeapStructSize`, split. Remainder re-inserted into size-sorted list. | **POSIX (glibc):** Yes — if remainder ≥ `MINSIZE` (32 B on 64-bit), split; remainder goes to appropriate bin or unsorted bin. **Bare-metal (newlib):** Yes — similar minimum-size threshold, remainder stays in free list. | Yes — same threshold as heap_2. Remainder stays in place (address order preserved automatically). | Same as heap_4 |
| **Coalescing** | N/A | **No** — freed blocks are inserted but never merged with neighbors. Causes fragmentation over time. | **POSIX (glibc):** Aggressive for normal chunks — coalesces with adjacent free neighbors on free (`unlink()` + merge). Fast bins and tcache defer coalescing until `malloc_consolidate()`. **Bare-metal (newlib):** Yes — always coalesces with address-adjacent free neighbors on free. Essentially identical to heap_4's `prvInsertBlockIntoFreeList()`. | **Yes** — on free, merges with address-adjacent predecessor and/or successor. Up to 3 blocks merged into 1. | Same as heap_4 |
| **Allocation status tracking** | N/A | MSB of `BlockLink_t.xBlockSize` set = allocated, cleared = free | **POSIX (glibc):** `PREV_INUSE` bit in next chunk's `size` field. Free chunks have valid `fd`/`bk`; allocated chunks reuse that space for user data. **Bare-metal (newlib):** No explicit allocated/free flag bit. A block is "allocated" if it is not in the free list — status is implicit by list membership. | MSB of `BlockLink_t.xBlockSize` set = allocated; `pxNextFreeBlock` set to `NULL` when allocated | Same as heap_4 |
| **Heap initialization** | Automatic: on first `pvPortMalloc`, align `ucHeap` start address. Set `pucAlignedHeap` once. | Automatic: on first `pvPortMalloc`, `prvHeapInit()` creates one large free block spanning entire heap, linked between `xStart` and `xEnd`. | **POSIX (glibc):** Lazy — on first `malloc()`, glibc initializes main arena, calls `brk()` (~132 KB), sets up top chunk, bins, tcache. Thread arenas created on first `malloc()` from new threads. **Bare-metal (newlib):** No initialization needed — free list starts empty. On first `malloc()`, free list is empty so `_sbrk_r()` is called immediately to get raw memory from `_end` bump pointer. BSP must ensure linker script defines `_end` correctly. | Automatic: on first `pvPortMalloc`, `prvHeapInit()` creates one free block from aligned start to `pxEnd` (placed at heap tail). | **Manual**: application MUST call `vPortDefineHeapRegions(&regions)` before any `pvPortMalloc`. Iterates `HeapRegion_t[]` array (terminated by `{NULL, 0}`), aligns each region, creates free blocks, links regions via sentinel blocks. Regions must be in ascending address order. |
| **Statistics** | `xPortGetFreeHeapSize()` = `configADJUSTED_HEAP_SIZE - xNextFreeByte` | `xPortGetFreeHeapSize()` = `xFreeBytesRemaining` | **POSIX (glibc):** `mallinfo()`/`mallinfo2()` returns arena size, free blocks, total allocated, top chunk size. `malloc_stats()` prints to stderr. **Bare-metal (newlib):** `mallinfo()` available but minimal — returns total allocated and free space. No FreeRTOS-level stats exposed in either case. | `xPortGetFreeHeapSize()`, `xPortGetMinimumEverFreeHeapSize()`, `vPortGetHeapStats()` (free block count, largest/smallest block, alloc/free counters) | Same as heap_4 |
| **Heap protector** | N/A | N/A | **POSIX (glibc):** `MALLOC_CHECK_` env var and compiled-in heap corruption checks. Chunk boundary tags validated on free. No canary-based pointer protection. **Bare-metal (newlib):** No heap protection. No canaries, no bounds checking. Corruption silently corrupts the free list. | `configENABLE_HEAP_PROTECTOR`: XORs block pointers with random canary; validates pointer bounds within `ucHeap[]`. | Same, validates against `pucHeapLowAddress`/`pucHeapHighAddress` across all regions. |
| **Config clear on free** | N/A | `configHEAP_CLEAR_MEMORY_ON_FREE`: `memset(puc + xHeapStructSize, 0, blockSize - xHeapStructSize)` | **POSIX (glibc):** No built-in zeroing. `MALLOC_PERTURB_` env var fills with pattern byte on alloc/free for debugging. **Bare-metal (newlib):** No zeroing, no debug perturb. Freed memory retains its contents until overwritten. | Same as heap_2 | Same as heap_4 |
| **Thread safety** | `vTaskSuspendAll()` / `xTaskResumeAll()` around allocation | `vTaskSuspendAll()` / `xTaskResumeAll()` around both alloc and free | `vTaskSuspendAll()` / `xTaskResumeAll()` wrapping `malloc`/`free`. **POSIX (glibc):** internally uses per-arena mutex + lock-free tcache. **Bare-metal (newlib):** newlib's `malloc()` is **not thread-safe** by itself — `vTaskSuspendAll()` is the only protection. No internal locking. | `vTaskSuspendAll()` / `xTaskResumeAll()` around alloc and free; `taskENTER_CRITICAL()` inside `vPortGetHeapStats()` for atomic counter snapshot | Same as heap_4 |
| **Fragmentation risk** | None (no free) | **High** — no coalescing means freed holes accumulate | **POSIX (glibc):** Low–Medium — aggressive coalescing reduces external fragmentation; fast bins and tcache create temporary fragmentation that resolves under pressure. **Bare-metal (newlib):** Low — coalescing merges adjacent holes, same as heap_4. Internal fragmentation from per-chunk overhead (~12 B on 32-bit) and minimum alignment. | **Low** — coalescing merges adjacent holes | Same as heap_4 |
| **Time complexity** | Alloc O(1), Free N/A | Alloc O(n), Free O(n) | **POSIX (glibc):** Alloc O(1) amortized (tcache/fast bin) to O(n) (large bins/brk); Free O(1) (tcache/fast bin) to O(n) (coalesce + unsorted bin). **Bare-metal (newlib):** Alloc O(n) (first-fit walk); Free O(n) (coalesce + insert). | Alloc O(n), Free O(n) | Alloc O(n), Free O(n) |
| **Typical use case** | Tiny systems that never free (create all tasks/queues at boot, run forever) | Legacy code; real-time deterministic free (constant-time insert) but accepts fragmentation | **POSIX:** hosted ports where glibc `malloc` is already thread-safe and performant. **Bare-metal:** rarely used — heap_4 is preferred because it gives deterministic behavior without depending on toolchain libc quality or requiring `_sbrk_r()` implementation. | General-purpose embedded — **most commonly used** (vast majority of modern demos) | Systems with scattered memory (e.g., internal SRAM + external SDRAM, memory-mapped peripherals) |

**Source code evidence index** (all paths relative to `FreeRTOS/Source/portable/MemMang/`):

| Aspect | heap_1 | heap_2 | heap_3 | heap_4 | heap_5 |
|---|---|---|---|---|---|
| **Heap storage** | `heap_1.c:71` `ucHeap[]`, `heap_1.c:75` `xNextFreeByte` | `heap_2.c:93` `ucHeap[]` | N/A — libc heap (glibc: `brk()`/`mmap()`; newlib: `_sbrk_r()`) | `heap_4.c:95` `ucHeap[]` | N/A — user-defined via `HeapRegion_t` |
| **Block metadata** | N/A — no per-block header, only global `xNextFreeByte` | `heap_2.c:99-103` `BlockLink_t`, `heap_2.c:78-82` MSB macros | N/A — libc internal (glibc: `malloc_chunk`; newlib: `{size,fd,bk}`) | `heap_4.c:100-104` `BlockLink_t`, `heap_4.c:80-84` MSB macros | `heap_5.c:158-162` same as heap_4 |
| **Free list structure** | N/A — no free list (bump allocator) | `heap_2.c:135-153` size-sorted insert macro | N/A — libc internal (glibc: 5 bin types; newlib: single free list) | `heap_4.c:504-569` address-sorted `prvInsertBlockIntoFreeList` | `heap_5.c:485-550` same as heap_4 |
| **Sentinel nodes** | N/A — no list structure | `heap_2.c:110` `xStart,xEnd`; `heap_2.c:385-386` init | N/A — libc internal | `heap_4.c:161-162` `xStart,pxEnd`; `heap_4.c:489-490` init | `heap_5.c:192-193` same as heap_4 |
| **pvPortMalloc** | `heap_1.c:116-123` bump alloc | `heap_2.c:224-231` best-fit walk; `heap_2.c:245-261` split | `heap_3.c:63-65` `malloc()` wrapper | `heap_4.c:244-253` first-fit walk; `heap_4.c:273-289` split; `heap_4.c:310-311` mark | `heap_5.c:226` `configASSERT(pxEnd)`; rest same as heap_4 |
| **vPortFree** | `heap_1.c:148-151` asserts `pv==NULL` | `heap_2.c:304-330` validate + size-sorted insert (no coalesce) | `heap_3.c:87-92` `free()` wrapper | `heap_4.c:363-398` validate + coalesce insert | `heap_5.c:399-433` same as heap_4 + cross-region coalesce |
| **Coalescing** | N/A — no free operation | N/A — size-sorted insert never checks address adjacency | N/A — libc internal (glibc: yes except fast/tcache; newlib: always) | `heap_4.c:523-529` prev merge; `heap_4.c:537-551` next merge | `heap_5.c:485-550` same as heap_4 + cross-region |
| **Heap init** | `heap_1.c:108-112` lazy align | `heap_2.c:371-393` `prvHeapInit` | N/A — `heap_3.c:102-105` no-op; libc handles init | `heap_4.c:456-501` `prvHeapInit` | `heap_5.c:553-669` `vPortDefineHeapRegions` (manual) |
| **pvPortCalloc** | N/A — no `pvPortCalloc` | `heap_2.c:352-368` multiplication-overflow check + `pvPortMalloc` + `memset` | N/A — no `pvPortCalloc` (use libc `calloc`) | `heap_4.c:437-453` same pattern as heap_2 | `heap_5.c:466-482` same as heap_4 |
| **Heap protector** | N/A — no block pointers to protect | N/A — no `configENABLE_HEAP_PROTECTOR` support | N/A — libc internal (glibc: `MALLOC_CHECK_`; newlib: none) | `heap_4.c:126` canary XOR; `heap_4.c:134-136` bounds check | `heap_5.c:130` canary XOR; `heap_5.c:137-140` multi-region bounds |
| **Statistics** | `heap_1.c:162-165` `xPortGetFreeHeapSize` only | `heap_2.c:340-343` `xPortGetFreeHeapSize` only | N/A — no FreeRTOS stats (glibc: `mallinfo()`; newlib: `mallinfo()`) | `heap_4.c:413-424` size APIs; `heap_4.c:572-621` `vPortGetHeapStats` | `heap_5.c:448-463` size APIs; `heap_5.c:672-727` `vPortGetHeapStats` |
| **Clear on free** | N/A — no free operation | `heap_2.c:320-324` `configHEAP_CLEAR_MEMORY_ON_FREE` | N/A — libc internal (glibc: `MALLOC_PERTURB_`; newlib: none) | `heap_4.c:379-388` `configHEAP_CLEAR_MEMORY_ON_FREE` | `heap_5.c:414-423` same as heap_4 |
| **Thread safety** | `heap_1.c:106,128` `vTaskSuspendAll`/`xTaskResumeAll` | `heap_2.c:204,280` `vTaskSuspendAll`/`xTaskResumeAll` | `heap_3.c:63,68` alloc; `heap_3.c:87,92` free | `heap_4.c:221,334` alloc; `heap_4.c:390,398` free; `heap_4.c:613,620` stats `taskENTER_CRITICAL` | `heap_5.c:267,369` alloc; `heap_5.c:425,433` free |
| **State reset** | `heap_1.c:174-177` `vPortHeapResetState` | `heap_2.c:401-406` `vPortHeapResetState` | `heap_3.c:102-105` `vPortHeapResetState` (no-op) | `heap_4.c:629-637` `vPortHeapResetState` | `heap_5.c:735-748` `vPortHeapResetState` |

**heap_3 libc internals** — source code is external (not in this repository):
- **POSIX (glibc ptmalloc2):** Source at [`malloc/malloc.c`](https://sourceware.org/git/?p=glibc.git;a=blob;f=malloc/malloc.c) in the glibc repository. Key functions: `__libc_malloc()` → `_int_malloc()` (tcache → fastbin → smallbin → unsorted → largebin → top → sysmalloc), `__libc_free()` → `_int_free()` (tcache → fastbin → coalesce → unsorted bin). Data structure: `struct malloc_chunk` with `prev_size`, `size` (3 flag bits), `fd`, `bk`, `fd_nextsize`, `bk_nextsize`.
- **Bare-metal (newlib):** Source at [`newlib/libc/stdlib/malloc.c`](https://sourceware.org/git/?p=newlib-cygwin.git;a=blob;f=newlib/libc/stdlib/malloc.c) in the newlib repository. `_malloc_r()` walks a single free list (first-fit), calls `_sbrk_r()` to obtain raw memory from `_end`. BSP must provide `_sbrk_r()` implementation — typically a bump pointer from linker symbol `_end` toward stack. Memory is never returned (bump pointer only grows).
- **Thread safety:** `vTaskSuspendAll()`/`xTaskResumeAll()` (disables scheduler, not interrupts).
- **Protection:** Optional MPU support restricts per-task memory regions (read/write/execute). Not isolation — just fault detection.
- **Typical heap size:** 1–64 KB total.

#### Memory Usage Analysis

##### Static Memory (Compile-Time)

Static memory includes global/static variables and objects created via `xTaskCreateStatic()` and related static-creation APIs. Analysis is done entirely through toolchain utilities — FreeRTOS provides no runtime API for this.

| Method | Command / Mechanism | Output |
|--------|---------------------|--------|
| Section size summary | `riscv-none-elf-size output.elf` | `.text` (Flash code), `.data` (Flash+RAM initialized), `.bss` (RAM zero-init) |
| Per-symbol breakdown | `riscv-none-elf-nm --size-sort --print-size output.elf` | Each symbol's address and size, sorted by size |
| Section headers | `riscv-none-elf-objdump -h output.elf` | All sections with start address, size, and flags |
| Linker map file | `-Wl,-Map=output.map` at link time | Full symbol-to-section mapping, memory region usage |
| Static-only mode | `configSUPPORT_STATIC_ALLOCATION=1`, `configSUPPORT_DYNAMIC_ALLOCATION=0` | All kernel objects use application-provided buffers (`xTaskCreateStatic`, `xQueueCreateStatic`, etc.) — total RAM fully predictable at compile time |

Total RAM = `.bss` + `.data` + (sum of all task stack sizes) + `configTOTAL_HEAP_SIZE`. Total Flash = `.text` + `.data`.

Static-only mode demo: `FreeRTOS/Demo/WIN32-MSVC-Static-Allocation-Only/`

##### Dynamic / Runtime Memory

FreeRTOS provides runtime APIs depending on the heap implementation and configuration flags.

| Goal | API / Tool | Requires | Available in |
|------|-----------|----------|--------------|
| Current free heap | `xPortGetFreeHeapSize()` | — | heap_1/2/3/4/5 |
| Heap historical minimum | `xPortGetMinimumEverFreeHeapSize()` | — | heap_4/5 |
| Reset heap minimum | `xPortResetHeapMinimumEverFreeHeapSize()` | — | heap_4/5 |
| Detailed heap stats | `vPortGetHeapStats(&HeapStats_t)` | `configUSE_TRACE_FACILITY=1` | heap_4/5 |
| Per-task stack high watermark | `uxTaskGetStackHighWaterMark(handle)` | `INCLUDE_uxTaskGetStackHighWaterMark=1` | All (reads TCB) |
| Per-task stack HWM (large stacks) | `uxTaskGetStackHighWaterMark2(handle)` | `INCLUDE_uxTaskGetStackHighWaterMark2=1` | All (returns `configSTACK_DEPTH_TYPE`) |
| All-tasks text summary | `vTaskList(buf)` | `configUSE_TRACE_FACILITY=1` | All |
| All-tasks structured data | `uxTaskGetSystemState(TaskStatus_t*, count, &totalRuntime)` | `configUSE_TRACE_FACILITY=1` | All |
| Stack overflow detection | `configCHECK_FOR_STACK_OVERFLOW=2` + `vApplicationStackOverflowHook()` | Application hook | All |
| Allocation/free logging | Override `traceMALLOC(pv, size)` / `traceFREE(pv, size)` macros | — (empty by default, override in `FreeRTOSConfig.h`) | heap_1/2/3/4/5 |
| Allocation failure hook | `configUSE_MALLOC_FAILED_HOOK=1` + `vApplicationMallocFailedHook()` | Application hook | All |
| Heap canary protection | `configENABLE_HEAP_PROTECTOR=1` + `vApplicationGetRandomHeapCanary()` | Application hook | heap_4/5 |

**`HeapStats_t` fields (heap_4/5):**

| Field | Meaning |
|-------|---------|
| `xAvailableHeapSpaceInBytes` | Current free heap |
| `xSizeOfLargestFreeBlockInBytes` | Largest contiguous free block |
| `xSizeOfSmallestFreeBlockInBytes` | Smallest free block |
| `xNumberOfFreeBlocks` | Free block count (fragmentation indicator) |
| `xMinimumEverFreeBytesRemaining` | Historical minimum free heap |
| `xNumberOfSuccessfulAllocations` | Cumulative successful `pvPortMalloc` calls |
| `xNumberOfSuccessfulFrees` | Cumulative successful `vPortFree` calls |

**`TaskStatus_t` memory-related fields (from `uxTaskGetSystemState`):**

| Field | Meaning |
|-------|---------|
| `pcTaskName` | Task name string |
| `uxCurrentPriority` | Current priority |
| `usStackHighWaterMark` | Minimum-ever remaining stack (words) — same value as `uxTaskGetStackHighWaterMark()` |
| `xHandle` | Task handle (can be passed to `uxTaskGetStackHighWaterMark` later) |

**How stack high watermark works:** At task creation, the entire stack is filled with `0xA5` (`tskSTACK_FILL_BYTE`, `tasks.c:121`). At query time, `prvTaskCheckFreeStackSpace()` (`tasks.c:6375`) scans from stack bottom counting untouched `0xA5` bytes. This is a pure software pattern match — it does **not** distinguish physical memory types (ILM/DLM/SRAM on RISC-V, ITCM/DTCM on ARM). It works on any memory region accessible through the CPU's unified address space.

**RISC-V multi-memory-region systems:** To separately track usage across ILM, DLM, and SRAM, use the linker script to define region boundary symbols and compute usage at runtime from those symbols. For dynamic allocation across regions, use heap_5 with `vPortDefineHeapRegions()` — but note that `vPortGetHeapStats()` returns **aggregated** statistics across all regions, not per-region breakdowns. Per-region tracking requires custom instrumentation.

#### FreeRTOS Mechanisms for Placing Kernel Objects in Specific Memory Regions

FreeRTOS does not manage ILM/DLM (RISC-V) or ITCM/DTCM (ARM) directly, but provides four mechanisms for the application developer to control which kernel objects are placed in fast tightly-coupled memory vs. slow bulk memory:

| | Linker Script + `__section__` | `pvPortMallocStack` | `HeapRegion_t` (heap_5) | MPU Regions |
|---|---|---|---|---|
| **Placement timing** | Compile/link time | Runtime (task creation) | Runtime (`pvPortMalloc` call) | Runtime (context switch) |
| **Placement granularity** | Individual variable | Task stack only | Any heap-allocated object | Address range |
| **Requires linker script** | Yes | Yes (separate heap array placed in DLM) | Yes (region addresses from linker) | No |
| **Dynamic / Static** | Purely static | Dynamic allocation | Dynamic allocation | Runtime-configurable |
| **Distinguishes fast/slow memory** | Precise per-variable control | Distinguishes stack vs. heap heap | No (first-fit is latency-unaware) | Does not allocate; only controls access permissions |
| **Typical use case** | Critical task TCB, stack, queues in DLM | All real-time task stacks in DLM | General heap spanning multiple memory regions | Isolate task access to DLM |
| **Code example** | `__attribute__((section(".dlm"))) static uint8_t ucDlmHeap[0x8000];`<br>`#define configAPPLICATION_ALLOCATED_HEAP 1`<br>Then place TCB/stack via `xTaskCreateStatic()` using a buffer from `.dlm`. | `uint8_t ucDlmStacks[4096];`<br>`void *pvPortMallocStack(size_t sz) {`<br>&nbsp;&nbsp;`static size_t off = 0;`<br>&nbsp;&nbsp;`if (off + sz > sizeof(ucDlmStacks)) return NULL;`<br>&nbsp;&nbsp;`void *p = &ucDlmStacks[off];`<br>&nbsp;&nbsp;`off += sz;`<br>&nbsp;&nbsp;`return p;`<br>`}` | `HeapRegion_t regions[] = {`<br>&nbsp;&nbsp;`{ (uint8_t *)0x20000000, 0x10000 },`<br>&nbsp;&nbsp;`{ (uint8_t *)0x90000000, 0x80000 },`<br>&nbsp;&nbsp;`{ NULL, 0 }`<br>`};`<br>`vPortDefineHeapRegions(regions);` | `MemoryRegion_t regions[] = {`<br>&nbsp;&nbsp;`{ (void *)0x20000000, 0x10000,`<br>&nbsp;&nbsp;&nbsp;&nbsp;`tskMPU_REGION_READ_WRITE },`<br>&nbsp;&nbsp;`{ 0 }`<br>`};`<br>`TaskParameters_t params = {`<br>&nbsp;&nbsp;`.puxStackBuffer = stackBuf,`<br>&nbsp;&nbsp;`.xRegions = regions, ... };`<br>`xTaskCreateRestricted(&params, &handle);` |

#### FreeRTOS vs Linux Memory Management

| Feature | FreeRTOS | Linux |
|---|---|---|
| **Address space** | Single flat, all tasks share | Per-process virtual (MMU) |
| **Base allocator** | Bump or free-list (byte granularity) | Buddy system (4 KiB page granularity, power-of-2 orders) |
| **Object allocator** | None | SLUB (per-type slab caches for `task_struct`, `dentry`, etc.) |
| **Per-CPU caches** | None | Yes (fastpath alloc/free without locks) |
| **Coalescing** | heap_4/5 only, address-adjacent merge | Buddy merging (power-of-2 coalescing) |
| **Internal fragmentation** | `BlockLink_t` header per block (~8–16 B) | Per-page (buddy) + per-object (slab) wastage |
| **Typical heap size** | 1–64 KB | MB–TB |
| **Thread safety** | `vTaskSuspendAll` (scheduler lock) | Per-area spinlocks + RCU |
| **Demand paging** | No | Yes (page faults) |
| **Swap / OOM** | No | Swap to disk + OOM killer |
| **Memory protection** | Optional MPU (fault detection only) | Full MMU (per-process isolation) |

### Linux

**Full MMU-based virtual memory with per-process address spaces.**

- **Page tables:** 4-level on x86_64: PGD → PUD → PMD → PTE → physical page (4 KiB pages, 48-bit VA).
- **Physical allocation:** Buddy allocator (power-of-two page blocks, coalescing on free).
- **Kernel object cache:** SLUB allocator (default) on top of buddy allocator. Dedicated caches for `task_struct`, `dentry`, `inode`, `sk_buff`, etc.
- **Virtual memory:** Per-process `mm_struct` with VMA linked list. `mmap()` for file-backed and anonymous mappings. Copy-on-write for `fork()`. Demand paging via page faults.
- **OOM killer:** When memory exhausted, selects victim process by `oom_score` and kills it.
- **Page cache:** Caches file data pages, shared across processes. Under pressure, evicted before anonymous pages.

### Key Difference

FreeRTOS runs on bare-metal MPU or no-MMU MCUs — all tasks trust each other, sharing one address space. Linux uses MMU hardware to provide **per-process isolation**, virtual memory, demand paging, and swap. FreeRTOS allocates from a small static heap; Linux manages GB+ of dynamic memory with multiple allocators.

```
FreeRTOS (single address space):       Linux (per-process virtual memory):
┌─────────────────────────────┐        Process A         Process B
│ Task A  │ Task B  │ Task C  │        ┌──────────┐      ┌──────────┐
│ (shares all memory)          │        │ VA 0-4GB │      │ VA 0-4GB │
│                             │        └─────┬────┘      └─────┬────┘
│ ┌─────────────────────────┐ │              │                  │
│ │ Kernel heap (heap_4/5)  │ │         ┌────┴──────────────────┴────┐
│ │ 1–64 KB                 │ │         │    Page Tables (MMU)        │
│ └─────────────────────────┘ │         │    VA → PA translation      │
└─────────────────────────────┘         └─────────┬──────────────────┘
                                                  ▼
                                          ┌──────────────┐
                                          │ Physical RAM │
                                          │ Buddy + SLUB │
                                          └──────────────┘
```

## 4. File System

### FreeRTOS

**No built-in file system.** FreeRTOS is a kernel only — file systems are optional add-ons:

- **Reliance-Edge** (`FreeRTOS-Plus/Source/Reliance-Edge/`): Third-party power-fail-safe file system for embedded. Optional, not compiled by default.
- **FatFS** (not in this repo): Popular FAT12/16/32 implementation by Elm-Chan, commonly used with FreeRTOS but not included.
- **FreeRTOS-Plus-CLI**: Provides a command-line interface for UART consoles but is not a file system.
- **No VFS, no mount points, no unified I/O.** Each storage driver is ad-hoc.
- Applications access storage through direct driver APIs, not through a file descriptor abstraction.

### Linux

**Full VFS (Virtual File System) with multiple file system support.**

- **VFS abstraction:** 4 key objects — superblock, inode, dentry, file. All file systems implement the same VFS operations.
- **Caches:** Page cache (file data), inode cache (file metadata), dentry cache (path → inode resolution).
- **ext4** (default): Journaling file system, extents-based block mapping, 16+ TB file/filesystem support. Journaling modes: `ordered` (default), `journal`, `writeback`.
- **Block layer:** I/O scheduling (mq-deadline, bfq, none), request queues, bio structures.
- **Everything is a file:** Devices, pipes, sockets, `/proc`, `/sys` all accessed through file descriptors.
- **mount namespaking:** Per-container/mount namespace file system views.

### Key Difference

Linux provides a rich, unified VFS layer with caching, journaling, and a "everything is a file" abstraction. FreeRTOS has **no file system at all** in the base kernel — it's purely a task scheduler with IPC primitives. Storage access is entirely application-specific.

## 5. Device Drivers

### FreeRTOS

**No driver framework. No driver model. No device tree. No auto-loading.**

Each driver is a standalone C module with no standardized registration, probing, or enumeration mechanism.

- **Network drivers:** Implement `NetworkInterface_t` function pointers (`pfInitialise`, `pfOutput`, `pfGetPhyLinkStatus`). Registered via `FreeRTOS_AddNetworkInterface()`. ~28 vendor implementations in `FreeRTOS-Plus-TCP/source/portable/NetworkInterface/`.
- **Interrupt handling:** Direct hardware ISR registration via port-specific `xPortInstallInterrupt()` or vendor HAL. No top-half/bottom-half split in the kernel — drivers handle this ad-hoc using `...FromISR()` APIs and deferred task notifications.
- **Configuration:** Compile-time via `FreeRTOSConfig.h` and vendor-specific headers. No runtime device discovery.
- **Loading:** All drivers statically linked. No loadable modules.
- **User-facing interface:** No `/dev`, no sysfs, no udev. Applications call driver functions directly.

### Linux

**Comprehensive device driver model with dynamic loading.**

- **Device model:** `struct device`, `struct driver`, `struct bus_type`. Unified hierarchy via `kobject`. `sysfs` (`/sys`) exposes all devices and attributes.
- **Device tree / ACPI:** Hardware description at boot. Drivers match via `of_match_table` compatible strings (ARM/RISC-V) or ACPI IDs (x86).
- **Loadable kernel modules (LKM):** `insmod`/`modprobe`/`rmmod` for runtime loading/unloading.
- **Interrupt handling:** Structured top-half/bottom-half:
  - **Top-half (hardirq):** Minimal work, acknowledges interrupt, schedules deferred work. Cannot sleep.
  - **Bottom-half:** Softirqs (NET_RX_SOFTIRQ for networking), tasklets, workqueues (can sleep), threaded IRQs.
- **udev:** Userspace device manager. Creates `/dev` nodes on hotplug events.
- **Device classes:** char, block, network, USB, PCI, platform, I2C, SPI — each with subsystem-specific APIs.

### Key Difference

FreeRTOS drivers are **flat C modules** with no framework — each vendor writes standalone code. Linux has a **unified driver model** with device trees, auto-probing, sysfs, loadable modules, and structured interrupt handling. FreeRTOS has no concept of device enumeration or hotplug.

```
FreeRTOS driver model:             Linux driver model:

(NetworkInterface_t)               struct device_driver
  .pfInitialise = my_init            .probe = my_probe
  .pfOutput     = my_output          .remove = my_remove
  .pfGetPhyLinkStatus = my_link    struct device
                                     .of_node → Device Tree
Application calls:                  struct bus_type
  my_init() directly                 .match, .probe, .remove
                                    sysfs: /sys/bus/pci/devices/...
No auto-detection                   udev: /dev/mydev0 (auto-created)
No hotplug                          Module: modprobe mydriver
```

## 6. Network Packet Handling

### FreeRTOS-Plus-TCP

**Single-task event-driven architecture.** The entire TCP/IP stack runs in one FreeRTOS task (the "IP-task").

**Buffer management:**
- `NetworkBufferDescriptor_t`: Fixed-size descriptor pool (`ipconfigNUM_NETWORK_BUFFER_DESCRIPTORS`, typically 8–60).
- Two schemes: BufferAllocation_1 (fixed-size pre-allocated, deterministic) or BufferAllocation_2 (variable-size dynamic, more memory-efficient).
- Buffer contains the complete Ethernet frame (header + payload). Headers are prepended/adjusted in-place.
- No scatter-gather. No zero-copy receive. Single contiguous buffer per packet.

**RX flow:**
```
1. Hardware ISR → driver copies frame into NetworkBufferDescriptor_t
2. Driver enqueues {eNetworkRxEvent, buffer} to IP-task queue
3. IP-task: prvProcessEthernetPacket() dispatches by EtherType
   - ARP → eARPProcessPacket()
   - IPv4/IPv6 → prvProcessIPPacket() → dispatches by protocol
     - UDP → prvProcessUDPPacket() → xProcessReceivedUDPPacket()
     - TCP → xProcessReceivedTCPPacket()
4. Protocol handler finds matching socket, copies data to socket's rxStream
5. Waiting application task is woken via event group or task notification
6. Application calls FreeRTOS_recv() to read from socket buffer
```

**TX flow:**
```
1. Application calls FreeRTOS_send() / FreeRTOS_sendto()
2. Data copied into socket's txStream (TCP) or into NetworkBufferDescriptor (UDP)
3. eTCPTimerEvent or eStackTxEvent sent to IP-task
4. IP-task builds TCP/UDP/IP/Ethernet headers in the buffer
5. ARP resolution if needed (buffer held in pxARPWaitingNetworkBuffer)
6. prvForwardTxPacket() calls pxInterface->pfOutput()
7. Driver transmits frame (esp_wifi_internal_tx, EMAC DMA, etc.)
8. Buffer returned to free pool
```

**Characteristics:**
- All processing in single IP-task context (no concurrency concerns within the stack)
- No NAPI equivalent — pure interrupt-driven from driver
- No netfilter/firewall hooks
- No scatter-gather or zero-copy
- No support for packet forwarding/routing between interfaces

### Linux Network Stack

**Kernel-space stack with NAPI polling, sk_buff, and netfilter hooks.**

**Buffer management:**
- `sk_buff`: The fundamental packet descriptor. Contains `head`/`data`/`tail`/`end` pointers allowing header prepend/append without copying payload. Reference-counted, can be cloned.
- Allocated from SLUB slab caches. Per-CPU recycle caches for hot-path performance.
- Supports scatter-gather (frags), zero-copy (MSG_ZEROCOPY), and multi-buffer (skb_shared_info).

**RX flow:**
```
1. NIC receives frame → DMA into ring buffer
2. NIC raises hardware interrupt
3. Hardirq handler: acknowledges IRQ, calls napi_schedule()
4. NAPI softirq (NET_RX_SOFTIRQ) polls NIC ring buffer up to budget
5. For each frame:
   a. Allocate sk_buff
   b. DMA-unmap, parse headers
   c. netif_receive_skb() → __netif_receive_skb_core()
   d. Deliver to protocol handler (ip_rcv for IPv4)
   e. Netfilter NF_INET_PRE_ROUTING hook (iptables/nftables)
   f. Routing decision (local or forward)
   g. If local: NF_INET_LOCAL_IN → ip_local_deliver()
   h. TCP: tcp_v4_rcv() → find socket → add to receive queue
   i. Wake blocked process (epoll/select/read)
6. Application calls recv() → copy from sk_buff to user buffer
```

**TX flow:**
```
1. Application calls send() / sendmsg()
2. VFS → socket layer → tcp_sendmsg()
3. TCP creates sk_buff with payload, adds to socket write queue
4. tcp_transmit_skb() builds TCP header
5. ip_queue_xmit() → ip_output() builds IP header
6. NF_INET_LOCAL_OUT hook → routing → NF_INET_POST_ROUTING hook
7. Neighbor subsystem resolves MAC (ARP)
8. dev_queue_xmit() → qdisc (traffic control) → NIC driver xmit
9. NIC DMA from sk_buff data
10. Completion interrupt frees sk_buff
```

**Characteristics:**
- NAPI: hybrid interrupt + polling eliminates receive livelock under load
- Netfilter: 5 hook points enable firewall, NAT, connection tracking, QoS
- sk_buff: zero-copy header manipulation, cloning, scatter-gather
- Multi-queue: multiple RX/TX queues mapped to multiple CPU cores (RSS/RPS)
- Traffic control: full qdisc layer (fq_codel, HTB, etc.)
- Supports packet forwarding, bridging, tunneling, VLAN

### Key Difference

FreeRTOS processes all packets in a **single task** with a simple event queue — deterministic but limited throughput (thousands of pps). Linux uses **NAPI polling + softirq** in kernel space with per-CPU processing, handling millions of pps. FreeRTOS has no firewall, no routing, no traffic control. Linux has netfilter, full routing, and a complete qdisc layer.

```
FreeRTOS packet processing:         Linux packet processing:

[NIC ISR]                            [NIC ISR → napi_schedule()]
   │                                        │
   ▼                                        ▼
[Driver: copy to buf]               [NAPI poll (softirq context)]
   │                                        │
   ▼                                        ▼
[xQueueSend to IP-task]             [netif_receive_skb()]
   │                                        │
   ▼                                        ▼
[IP-task: single-threaded]          [ip_rcv → tcp_v4_rcv]
   │                                        │
   ▼                                        ▼
[Socket buffer → wake task]          [sk_buff → socket queue]
   │                                        │
   ▼                                        ▼
[App task: FreeRTOS_recv()]          [App: recv() → copy to user]

No firewall hooks                    Netfilter at 5 points
No routing/forwarding                Full routing/forwarding
Single-threaded                      Per-CPU processing
No traffic shaping                   Full qdisc (tc) layer
```

## 7. Inter-Process / Inter-Task Communication

| Feature | FreeRTOS | Linux |
|---------|----------|-------|
| **Queues** | `QueueHandle_t` — fixed-size items, copied by value | `pipe()`, `mq_send()` (POSIX MQ), `msgsnd()` (System V) |
| **Shared memory** | All memory is shared (single address space) | `shmget()`, `mmap(MAP_SHARED)` with explicit setup |
| **Semaphores** | Binary, counting, recursive (all from `semphr.h`) | `sem_open()` (POSIX), `semctl()` (System V) |
| **Mutexes** | Priority inheritance built-in | `pthread_mutex_t` with PI attribute (`PTHREAD_PRIO_INHERIT`) |
| **Event flags** | `EventGroup_t` — multi-bit wait/set | `eventfd()`, `poll()`/`epoll()` on multiple FDs |
| **Notifications** | Task notifications — lightweight 1:1 | `signal()` (async), `eventfd()` (synchronizable) |
| **Message passing** | Message buffers (length-prefixed stream) | Unix domain sockets, D-Bus |
| **Sockets** | FreeRTOS+TCP Berkeley sockets | POSIX sockets (kernel TCP/IP stack) |

**Key difference:** FreeRTOS IPC is **deterministic** (no unbounded blocking from OS scheduling), but everything shares memory. Linux IPC has **isolation overhead** (copy between kernel/user space) but provides security.

## 8. Boot and Initialization

| | FreeRTOS | Linux |
|---|---|---|
| **Boot loader** | Vendor-specific (or none — runs from reset) | U-Boot, GRUB, systemd-boot |
| **Kernel init** | `main()` → hardware init → `xTaskCreate()` → `vTaskStartScheduler()` | Kernel decompresses → `start_kernel()` → driver probing → `init` (PID 1) |
| **Device discovery** | Compile-time configuration | Device tree / ACPI at boot, udev for hotplug |
| **First user task** | Application-defined (typically `vApplicationTask()`) | `/sbin/init` → systemd/sysvinit |
| **Boot time** | Sub-millisecond to a few ms | 200 ms to several seconds |

## 9. Summary Table

| Dimension | FreeRTOS | Linux |
|-----------|----------|-------|
| **Scheduling** | Fixed-priority preemptive, O(1), deterministic | CFS (fairness), O(log N), + RT policies |
| **Memory** | Single address space, no MMU, 1–64 KB heap | Per-process VA, MMU, buddy+SLUB, GB+ |
| **File system** | None in kernel (optional add-ons) | VFS + ext4/XFS/Btrfs + page cache |
| **Device drivers** | Flat C modules, no framework, static | Unified driver model, LKM, sysfs, udev |
| **Network stack** | Single-task event-driven, ~Kpps | Kernel NAPI + softirq, ~Mpps, netfilter |
| **Interrupts** | Direct ISR → FromISR APIs | Top-half (hardirq) + bottom-half (softirq/workqueue) |
| **Isolation** | None (all tasks share memory) | Full (per-process VA, user/kernel split) |
| **Boot time** | < 1 ms | 200 ms – seconds |
| **Worst-case latency** | Sub-microsecond | 10–100 µs (with PREEMPT_RT) |
| **Code size** | ~5–20 KB | ~20–80 MB (compressed) |
| **API count** | ~50 kernel functions | ~400 syscalls |
| **Use case** | Real-time control, sensors, motor drives | Servers, desktops, phones, complex embedded |
