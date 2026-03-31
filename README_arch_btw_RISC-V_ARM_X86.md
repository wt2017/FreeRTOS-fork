# x86 vs ARM vs RISC-V: Architectural Comparison

Cross-architecture comparison of memory consistency models and interrupt mechanisms.

---

# Chapter 1: Memory Consistency Models

Hardware memory ordering guarantees, observable reordering behaviors, and the barrier instructions required on each architecture.

## 1.1 x86 TSO (Total Store Order)

x86 provides the strongest memory ordering guarantees among the three architectures. It permits exactly **one** relaxation:

- **Allowed:** Store → Load reordering (a later load may complete before an earlier store to a *different* address becomes globally visible, due to the store buffer).
- **Forbidden:** Load → Load, Store → Store, Load → Store reordering.

ARM and RISC-V use **weak memory models** — they permit all four types of reordering unless the software inserts explicit barrier instructions. This trades programming simplicity for lower power consumption and simpler hardware implementation.

## 1.2 Reordering Behaviors

The table below shows each reordering type, a concrete code scenario that exposes it, the behavior on each architecture, and the barrier fix code for ARM and RISC-V.

| Reordering Type | Code Scenario | x86 TSO | ARM | RISC-V | ARM Barrier Fix | RISC-V Barrier Fix |
|---|---|---|---|---|---|---|
| **Store → Store** | T1: `data = 42;`<br>`flag = 1;`<br>T2: `r1 = flag;`<br>`r2 = data;`<br>Can T2 see `r1==1` but `r2≠42`? | **Forbidden.**<br>Stores become visible in program order. T2 sees `data==42` if it sees `flag==1`. | **Allowed.**<br>Store buffer may drain out of order. T2 may observe `flag==1` while `data` is still stale. | **Allowed.**<br>RWMO permits `W→W` reordering. Same observable effect as ARM. | `data = 42;`<br>`dmb ish;`<br>`flag = 1;`<br><br>`DMB ISH` = Data Memory Barrier, Inner Shareable. Prevents all memory operations before the barrier from being reordered with those after. | `data = 42;`<br>`fence rw,w;`<br>`flag = 1;`<br><br>`fence rw,w` = all reads/writes before the fence are ordered before all writes after the fence. |
| **Load → Load** | T1: `x = 1;`<br>T2: `y = 1;`<br>T3: `r1 = y;`<br>`r2 = x;`<br>Can T3 see `r1==1` but `r2==0`? | **Forbidden.**<br>Loads execute in program order. If T3 sees `y==1`, all writes before `y=1` (including `x=1`) are also visible. | **Allowed.**<br>CPU may satisfy the second load from cache before the first load completes. T3 may see `y==1` but `x==0`. | **Allowed.**<br>RWMO permits `R→R` reordering. Same observable effect as ARM. | `r1 = y;`<br>`dmb ish;`<br>`r2 = x;`<br><br>Ensures the first load completes before the second load issues. | `r1 = y;`<br>`fence r,r;`<br>`r2 = x;`<br><br>`fence r,r` = all reads before the fence are ordered before all reads after. |
| **Load → Store** | T1: `r1 = flag;`<br>`data = 99;`<br>T2: `r2 = data;`<br>`flag = 1;`<br>Can both threads see stale values (r1==0, r2==old)? | **Forbidden.**<br>A store cannot become visible before a preceding load completes. | **Allowed.**<br>The store may reach the write buffer and become globally visible before the load retires. Both threads may read stale values. | **Allowed.**<br>RWMO permits `R→W` reordering. Same observable effect as ARM. | `r1 = flag;`<br>`dmb ish;`<br>`data = 99;`<br><br>Prevents the store from being issued before the load completes. | `r1 = flag;`<br>`fence rw,rw;`<br>`data = 99;`<br><br>`fence rw,rw` = full ordering in both directions. |
| **Store → Load** | T1: `data = 42;`<br>`r1 = flag;`<br>T2: `flag = 1;`<br>`r2 = data;`<br>Can T1 see `r1==1` but T2 sees `r2≠42`? | **Allowed.**<br>The only reordering x86 TSO permits. T1's load may be satisfied before its own prior store (`data=42`) drains from the store buffer to cache. | **Allowed.**<br>Same reordering as x86, plus ARM may reorder additional pairs beyond this. | **Allowed.**<br>RWMO permits `W→R` reordering. Same as x86 TSO for this pair. | `data = 42;`<br>`dmb ish;`<br>`r1 = flag;`<br><br>`DMB` prevents the load from being issued before the store is globally visible. | `data = 42;`<br>`fence rw,r;`<br>`r1 = flag;`<br><br>`fence rw,r` = all reads/writes before the fence are ordered before all reads after. |
| **Multi-copy Atomicity** | CPU 0: `STORE A = 1`<br><br>Can CPU 1 observe `A==1` while CPU 2 still observes `A==0`?<br><br>(A single store visible to different cores at different times.) | **Atomic.**<br>A store becomes visible to all other cores simultaneously. No core can observe a "partial" propagation. | **ARMv7: Not atomic.**<br>A store may reach CPU 1's cache line before CPU 2's. Different cores can observe different values for the same address at the same instant.<br><br>**ARMv8 (AArch64): Atomic.**<br>Multi-copy atomicity is now required. | **Atomic.**<br>RWMO requires multi-copy atomicity. A store is either visible to no core or to all cores. | **No software fix for ARMv7.**<br>This is a hardware property, not a reordering that barriers can prevent. ARMv7 software must use locks or atomic primitives (LDXR/STX) for consensus. | **Not applicable.**<br>RWMO mandates multi-copy atomicity. |

## 1.3 Summary

| Reordering Type | x86 TSO | ARMv7 | ARMv8 / AArch64 | RISC-V (RVWMO) |
|---|---|---|---|---|
| Store → Store | **Forbidden** | Allowed | Allowed | Allowed |
| Load → Load | **Forbidden** | Allowed | Allowed | Allowed |
| Load → Store | **Forbidden** | Allowed | Allowed | Allowed |
| Store → Load | Allowed | Allowed | Allowed | Allowed |
| Multi-copy Atomicity | **Yes** | **No** | Yes | Yes |
| Primary barrier instruction | `MFENCE` / `LOCK` prefix | `DMB` / `DSB` / `ISB` | `DMB` / `DSB` / `ISB` | `FENCE` + acquire/release on atomics |

## 1.4 Why ARM/RISC-V Use Weak Models

x86's strong ordering guarantees are not free — the hardware must maintain a strictly-ordered store buffer and suppress many out-of-order opportunities, consuming more power and limiting achievable clock frequency. ARM and RISC-V target power-constrained embedded and mobile systems, so they:

1. **Push the consistency burden to software.** The programmer or compiler inserts barriers only where needed (typically around lock acquire/release), not everywhere.
2. **Allow simpler hardware.** The CPU can aggressively reorder memory operations for higher throughput and lower power, since it does not need to maintain global ordering by default.
3. **Provide fine-grained barrier primitives.** Instead of a single heavy fence, ARM offers `DMB ISHST` (store-store only) and RISC-V offers `fence w,w` (store-store only), allowing the programmer to pay only for the ordering actually required.

## 1.5 Impact on FreeRTOS

FreeRTOS SMP support (`configNUMBER_OF_CORES > 1`) must insert correct barriers in context switches, queue operations, and inter-core notification paths. Without proper barriers, race conditions on ARM/RISC-V manifest as **intermittent, load-dependent bugs that never reproduce on x86**.

### ARM Cortex-M (portmacro.h)

```c
#define portYIELD()                                     \
    {                                                   \
        portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT;  \
        __asm volatile ( "dsb" ::: "memory" );          \
        __asm volatile ( "isb" );                       \
    }
```

- PendSV is triggered **first** by writing to the NVIC Interrupt Control register. The DSB/ISB barriers **follow** to ensure the write is visible before subsequent memory accesses (e.g., in ISR-context code). This ordering is critical: the DSB guarantees the write has completed, and the ISB flushes the pipeline so the next instruction fetch sees the new state.
- **Cortex-M4F/CM7 note:** These ports additionally save/restore LR (R14) as the EXC_RETURN value on the process stack, allowing the PendSV handler to pop the return mode directly. CM3 reconstructs EXC_RETURN in LR via `orr r14, #0xd`.

### RISC-V (portmacro.h)

```c
#define portYIELD()    __asm volatile ( "ecall" );
```

- `ecall` triggers a synchronous environment-call exception (mcause == 11). The trap handler (`freertos_risc_v_trap_handler` in `portASM.S`) detects this, saves context, and calls `vTaskSwitchContext()`.
- **No `fence` instruction is used** — the port relies on `ecall`'s implicit ordering guarantee (the instruction is globally visible). No CLINT MSIP memory-mapped register access is needed.
- **Contrast with SMP RISC-V ports:** Some multi-hart implementations do use CLINT `msip[]` for inter-processor interrupts (IPI), but the standard FreeRTOS single-hart RISC-V port uses `ecall` exclusively.

### Key insight

On x86, most of these barriers compile to no-ops or single `mfence` instructions. On ARM/RISC-V, they are **mandatory for correctness** — omitting them causes silent data corruption in SMP configurations. A FreeRTOS application that works correctly on a single-core Cortex-M may break immediately when ported to a dual-core Cortex-M33 or RISC-V with multiple harts, if the port layer does not include the appropriate fences.

---

# Chapter 2: Interrupt Mechanisms

Comparison of interrupt handling across x86-64 (APIC), ARM Cortex-M (NVIC), and RISC-V (CLINT + PLIC).

## 2.1 Overview

| Dimension | x86-64 (APIC) | ARM Cortex-M (NVIC) | RISC-V (CLINT + PLIC) |
|---|---|---|---|
| **Design target** | General-purpose high performance | Real-time, low latency, deterministic | Modular, minimal, extensible |
| **Interrupt controller** | Local APIC (per-core) + I/O APIC (external), MSI for PCIe | NVIC tightly coupled inside CPU core | CLINT (local timer/IPI) + PLIC (external devices), separate IP blocks |
| **Interrupt vector count** | 256 (0–31 CPU-reserved exceptions, 32–255 available) | 16 system exceptions + 1–240 external IRQs (vendor-configured) | 16 local traps + 1–1023 PLIC external sources (platform-defined) |
| **Interrupt latency** | Non-deterministic (µs-scale, depends on pipeline/APIC state) | **Deterministic** (12 cycles, Cortex-M4) | Non-deterministic (depends on software register save time) |
| **Hardware auto-saved context** | RFLAGS, CS, RIP, SS (+ optional error code) | **8 registers**: xPSR, PC, LR, R12, R3, R2, R1, R0 | PC → `mepc`, MIE → `mstatus.MPIE` only; general-purpose registers saved by software |
| **Stack switching on interrupt** | Ring 3 → Ring 0 via TSS (hardware loads kernel RSP from TSS/IST) | Thread mode (PSP) → Handler mode (MSP) automatic | **No hardware stack switch**; software uses `mscratch` to load a dedicated ISR stack |
| **Interrupt return** | `IRETQ` (restores RIP, CS, RFLAGS, RSP, SS) | `BX LR` where LR contains a magic `EXC_RETURN` value (e.g., `0xFFFFFFFD`) | `MRET` (M-mode) / `SRET` (S-mode); restores PC ← `mepc`, MIE ← `MPIE` |

## 2.2 Vector Table

| | x86-64 | ARM Cortex-M | RISC-V |
|---|---|---|---|
| **Table structure** | IDT (Interrupt Descriptor Table): 256 × 16-byte gate descriptors (segment selector + offset + gate type + DPL) | Function pointer array: one address per exception/IRQ | Two modes: **Direct** (all interrupts jump to `mtvec` BASE) or **Vectored** (asynchronous interrupts jump to `mtvec + 4 × cause`) |
| **Table location** | Linear address stored in `IDTR` register | `VTOR` register (default `0x00000000`, relocatable at runtime) | `mtvec` CSR (BASE field, 4-byte aligned) |
| **Entry content** | Gate descriptor: code segment selector + instruction pointer + privilege level + gate type | Raw function pointer (32-bit address) | Raw address (4-byte aligned) or single common entry |
| **Indexing** | CPU multiplies vector number by 16 to index IDT | CPU uses exception number × 4 to index table | Hardware uses `mcause` code to compute `mtvec + 4 × cause` (vectored mode) |

## 2.3 Interrupt Entry Flow

| Step | x86-64 | ARM Cortex-M | RISC-V |
|---|---|---|---|
| **1. Identify source** | APIC delivers vector number to CPU | NVIC determines highest-priority pending exception | Hardware sets `mcause` and `mip` bits |
| **2. Save context** | Hardware pushes SS, RSP, RFLAGS, CS, RIP to kernel stack | Hardware pushes xPSR, PC, LR, R12, R3–R0 to PSP or MSP (8 words) | Hardware saves PC → `mepc`, MIE → `mstatus.MPIE` only |
| **3. Disable interrupts** | Clears `RFLAGS.IF` (interrupt flag) | Sets `LR = EXC_RETURN` magic value; enters Handler mode | Clears `mstatus.MIE` (disables same-privilege interrupts) |
| **4. Switch stack** | Loads kernel RSP from TSS (or IST for specific vectors) | Switches from PSP (Thread) to MSP (Handler) | **No hardware switch**; ISR software must load stack from `mscratch` |
| **5. Jump to handler** | Loads handler address from IDT[vector] | Loads handler address from VTOR[exception_number] | Loads `mtvec` (direct) or `mtvec + 4 × cause` (vectored) |
| **6. Return** | `IRETQ`: pops RIP, CS, RFLAGS, RSP, SS | `BX EXC_RETURN`: hardware pops 8 registers from stack | `MRET`: restores PC ← `mepc`, MIE ← `MPIE` |

### ARM EXC_RETURN Values

When an exception is taken on Cortex-M, the hardware writes a magic constant into LR. When the handler executes `BX LR` with this value, the CPU recognizes it as an exception return rather than a normal branch:

| EXC_RETURN | Return Mode | Stack Used |
|---|---|---|
| `0xFFFFFFF1` | Handler mode | MSP |
| `0xFFFFFFF9` | Thread mode | MSP |
| `0xFFFFFFFD` | Thread mode | PSP (used by RTOS: return to a task) |
| `0xFFFFFFE1` | Handler mode, FPU extended frame | MSP |
| `0xFFFFFFE9` | Thread mode, FPU extended frame | MSP |
| `0xFFFFFFED` | Thread mode, FPU extended frame | PSP |

**FPU-related values** (`0xFFFFFFEx`): On Cortex-M4F and Cortex-M7 with FPU enabled, bit [4] of EXC_RETURN indicates the stack frame type — 0 = basic frame (no FPU save), 1 = extended frame (includes S0–S15, FPSCR). The hardware determines which frame type to push based on whether the current task was using the FPU. The port initialization sets `portINITIAL_EXC_RETURN = 0xFFFFFFFD`.

**Note:** `0xFFFFFFBC` is **not a valid EXC_RETURN value** on any Cortex-M processor. Bit [0] must be 1 (Thumb state) for all valid returns.

### RISC-V Interrupt Entry (Software Responsibilities)

Since RISC-V hardware saves minimal state, the ISR software must handle register saving:

```
Hardware (automatic):
  mepc     ← PC              // save interrupted PC
  MPIE     ← MIE             // save interrupt-enable state
  MIE      ← 0               // disable interrupts
  mcause   ← exception code  // identify source
  PC       ← mtvec(BASE)     // jump to handler (direct mode)

Software (manual, per FreeRTOS portcontextSAVE_CONTEXT_INTERNAL):
  1. Decrement sp by portCONTEXT_SIZE (31 words for RV32I, 15 for RV32E)
  2. Save x1(ra), x5–x15 (and x16–x31 for non-RV32E) to stack
     Note: x2(sp), x3(gp), x4(tp) are NOT saved
  3. Conditionally save FPU registers (lazy: only if mstatus.FS == DIRTY)
  4. Conditionally save VPU registers (lazy: only if mstatus.VS == DIRTY)
  5. Save mstatus CSR to stack[1]
  6. Read mcause → a0, mepc → a1
  7. Save mepc (the return address) at stack[0]
     - For synchronous exceptions: a1+4 (skip faulting instruction)
     - For asynchronous interrupts: a1 unmodified
  8. Switch to ISR stack via mscratch
  9. Dispatch by mcause: timer interrupt → xTaskIncrementTick;
     ecall → vTaskSwitchContext; other → application handler
  10. Execute handler
  11. Restore: mstatus ← stack[1], x1 ← stack[2], x5–x31
  12. Conditionally restore FPU/VPU (if FS/VS == DIRTY in restored mstatus)
  13. sp += portCONTEXT_SIZE
  14. Execute mret (hardware restores PC ← mepc, MIE ← mstatus.MPIE)
```

## 2.4 Interrupt Controller Architecture

### x86-64: APIC

```
┌──────────────┐     ┌──────────────┐
│  Local APIC   │     │  Local APIC   │     ← One per CPU core
│   (CPU 0)     │     │   (CPU 1)     │        Manages local interrupts, IPI, timer
└──────┬───────┘     └──────┬───────┘
       │                     │
       └─────────┬───────────┘
                 │  System Bus
       ┌─────────┴───────────┐
       │     I/O APIC        │           ← Routes external device IRQs
       │   (24 IRQ lines)    │              to specific CPU cores
       └─────────┬───────────┘
                 │
       ┌─────────┴───────────┐
       │   PCIe Devices      │           ← MSI/MSI-X: write interrupt
       └─────────────────────┘              messages directly to Local APIC
```

- **Local APIC**: per-core; handles APIC Timer, performance counter interrupts, inter-processor interrupts (IPI: `RESCHEDULE`, `CALL_FUNCTION`, etc.)
- **I/O APIC**: routes external device IRQ lines to designated CPU cores; 24 IRQ inputs
- **MSI (Message Signaled Interrupts)**: PCIe devices write directly to Local APIC via memory-mapped transaction; bypasses I/O APIC; supports up to 2048 vectors
- **Interrupt affinity**: each vector's destination CPU is programmable via APIC registers

### ARM Cortex-M: NVIC

```
┌─────────────────────────────────────────────┐
│              Cortex-M Core                   │
│  ┌─────────┐  ┌──────────────────────────┐  │
│  │  CPU     │←→│  NVIC                    │  │
│  │  Core    │  │  - 1–240 external IRQs   │  │
│  │          │  │  - 16 system exceptions   │  │
│  │          │  │  - Per-IRQ priority       │  │
│  │          │  │  - Auto context save      │  │
│  │          │  │  - Tail-chaining          │  │
│  │          │  │  - Late arrival           │  │
│  └─────────┘  └──────────────────────────┘  │
└─────────────────────────────────────────────┘
```

- **NVIC is integrated into the CPU core**: enables deterministic 12-cycle interrupt latency (Cortex-M4)
- **External interrupts**: 1–240 configurable (e.g., STM32F4 implements 82)
- **System exceptions**: Reset, NMI, HardFault, MemManage, BusFault, UsageFault, SVCall, PendSV, SysTick, etc.
- **Priority grouping**: AIRCR.PRIGROUP splits 8-bit priority into preemption priority + subpriority

### RISC-V: CLINT + PLIC

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│  Hart 0  │  │  Hart 1  │  │  Hart 2  │       Each hart has independent
│ mstatus  │  │ mstatus  │  │ mstatus  │          CSRs: mie/mip/mtvec/mepc
│ mie/mip  │  │ mie/mip  │  │ mie/mip  │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │              │              │
     └──────────────┼──────────────┘
                    │
        ┌───────────┴───────────┐
        │  CLINT                │             Core-local interrupts
        │  - mtime (global)     │                per-hart timer comparator
        │  - mtimecmp[0..N]     │                per-hart software interrupt bit
        │  - msip[0..N]         │
        └───────────────────────┘

        ┌───────────────────────┐
        │  PLIC                 │             Platform-Level Interrupt Controller
        │  - 1–1023 sources     │                per-source priority (typically 0–7)
        │  - Per-source priority │               per-hart threshold and enable
        │  - Per-hart threshold  │               routing to specific harts
        │  - Interrupt routing   │
        └───────────────────────┘
```

- **CLINT (Core Local Interruptor)**: handles per-hart timer interrupts (MTIP) and software interrupts (MSIP, used for IPI)
- **PLIC (Platform-Level Interrupt Controller)**: handles all external device interrupts; each source has configurable priority and can be routed to specific harts
- **No standardized controller IP**: CLINT/PLIC are de facto standards but vendors may substitute custom designs

## 2.5 Priority, Nesting, and Special Features

| Feature | x86-64 (APIC) | ARM Cortex-M (NVIC) | RISC-V (CLINT + PLIC) |
|---|---|---|---|
| **Priority levels** | 256 (vector number high 4 bits → 16 classes × 16 vectors each) | Vendor-defined (typically 4 bits = 16 levels); 8-bit register, upper bits implemented | Local: fixed order (MEI > MSI > MTI); PLIC external: per-source priority (typically 0–7, 8 levels) |
| **Priority semantics** | Lower vector number = higher priority (via APIC TPR) | Lower numeric value = higher priority (0 = highest) | PLIC: higher numeric value = higher priority; local interrupts: fixed hardware order |
| **Interrupt nesting** | Yes; APIC automatically preempts lower-priority handlers | Yes; higher preemption-priority interrupts preempt lower-priority handlers | Yes; re-enable MIE=1 inside handler to allow nesting |
| **Tail-chaining** | No | **Yes**: when a same-priority interrupt is pending at handler exit, CPU skips the save/restore cycle and jumps directly to the next handler (~6 cycles overhead) | No |
| **Late arrival** | No | **Yes**: if a higher-priority interrupt arrives during the stacking phase, the hardware switches to the higher-priority handler immediately, avoiding double stacking | No |
| **Priority grouping** | No grouping (flat 256 levels) | **Configurable**: AIRCR.PRIGROUP splits priority byte into preemption priority (determines nesting) + subpriority (determines order within same preemption level) | No grouping; local interrupts have fixed priority; PLIC has per-source priority + per-hart threshold |

## 2.6 Interrupt Masking

| Mechanism | x86-64 | ARM Cortex-M | RISC-V |
|---|---|---|---|
| **Global interrupt disable** | `CLI` → clears `RFLAGS.IF`<br>`STI` → sets `RFLAGS.IF` | `CPSID I` → sets `PRIMASK=1`<br>`CPSIE I` → clears `PRIMASK=0` | `csrci mstatus, 8` → clears `MIE=0`<br>`csrsi mstatus, 8` → sets `MIE=1` |
| **Selective masking** | APIC TPR (Task Priority Register): sets a priority threshold; interrupts at or below the threshold are blocked | `BASEPRI`: masks all interrupts with priority value ≥ threshold; higher-priority interrupts still fire | `mie` CSR: per-type enable bits (MEIE for external, MTIE for timer, MSIE for software) |
| **NMI behavior** | NMI pin cannot be masked by software | NMI cannot be masked | No standard NMI; implementations may define non-maskable interrupts |
| **RTOS critical section** | `CLI`/`STI` (all interrupts blocked) | `BASEPRI = configMAX_SYSCALL_INTERRUPT_PRIORITY`: masks only interrupts at or below the syscall priority threshold; zero-latency interrupts above threshold remain active | Clear `mie` bits for specific interrupt types |

### ARM BASEPRI — FreeRTOS Zero-Latency Interrupts

ARM's `BASEPRI` register is the most flexible selective masking mechanism among the three architectures. FreeRTOS leverages it to implement critical sections without blocking high-priority interrupts:

```c
/* FreeRTOS portMACRO.h — ARM Cortex-M */
#define portSET_INTERRUPT_MASK_FROM_ISR()  \
    __set_BASEPRI( configMAX_SYSCALL_INTERRUPT_PRIORITY << ( 8 - configPRIO_BITS ) )

#define portCLEAR_INTERRUPT_MASK_FROM_ISR( uxSaved )  \
    __set_BASEPRI( uxSaved )
```

Interrupts with priority numerically **lower** than `configMAX_SYSCALL_INTERRUPT_PRIORITY` (higher urgency) can still preempt — these are "zero-latency" interrupts that never call FreeRTOS API functions. Only interrupts at or below the threshold are blocked during critical sections.

## 2.7 Software Interrupts and Inter-Processor Interrupts (IPI)

| | x86-64 | ARM Cortex-M | RISC-V |
|---|---|---|---|
| **Software interrupt instruction** | `INT n` (triggers vector n); `SYSCALL`/`SYSENTER` for fast system calls | `SVC n` (triggers SVCall exception, immediate operand as argument) | `ECALL` (triggers Environment Call; routes to M-mode or S-mode based on current privilege) |
| **Inter-processor interrupt** | Local APIC ICR (Interrupt Command Register): write destination CPU + vector number; supports broadcast, self-IPI | Not applicable (Cortex-M is single-core by design; Cortex-A uses GIC SGI) | Single-core: `ecall` (triggers mcause==11 synchronous exception). Multi-hart: CLINT `msip[hart_id]` for inter-hart IPI |
| **Typical uses** | System calls (`SYSCALL`), scheduler IPI (`RESCHEDULE_VECTOR`), TLB shootdown (`INVALIDATE_TLB_VECTOR`) | System calls (`SVC`), FreeRTOS context switch trigger (`PendSV`) | System calls (`ECALL`), inter-hart notification (MSIP), scheduler tick (CLINT timer) |

### ARM PendSV — RTOS Context Switch Mechanism

FreeRTOS on ARM Cortex-M uses **PendSV** (Pendable Service Call) for context switching — a design unique to ARM:

```
SysTick interrupt → xPortSysTickHandler()
                      │
                      ├─ Increment tick count
                      └─ If context switch needed:
                            SCB->ICSR = portNVIC_PENDSVSET   ← pends PendSV
                               │
                               ▼
                   PendSV_Handler()              ← lowest-priority interrupt
                     │
                     ├─ Save current task registers to stack
                     ├─ vTaskSwitchContext()      ← select next task
                     └─ Restore next task registers from stack
```

PendSV is set to the **lowest interrupt priority**, ensuring the context switch never preempts any other ISR. This is the standard pattern for ARM Cortex-M RTOS implementations.

## 2.8 Privilege Modes and Interrupt Isolation

| | x86-64 | ARM Cortex-M | RISC-V |
|---|---|---|---|
| **Privilege modes** | Ring 0 (kernel) / Ring 3 (user); interrupts always handled in Ring 0 | Handler mode (interrupts) / Thread mode (privileged or unprivileged) | M-mode (highest) / S-mode (OS) / U-mode (user); interrupts can be delegated |
| **Stack pointer model** | TSS provides Ring 0 RSP; IST (Interrupt Stack Table) provides up to 7 dedicated interrupt stacks | MSP (handler + privileged thread) / PSP (unprivileged thread) | No hardware stack switching; software uses `mscratch` to store a dedicated interrupt-context stack pointer |
| **Interrupt delegation** | Not supported — all interrupts are handled in Ring 0 | Not supported — all exceptions are handled in Handler mode (highest privilege) | **Supported** via `mideleg` / `medeleg` CSRs: M-mode can delegate specific interrupts/exceptions to S-mode |

### RISC-V Interrupt Delegation

RISC-V is unique among the three architectures in supporting interrupt delegation from M-mode to S-mode:

```
mideleg register bits:
  Bit [3]  (MSIE)  → delegate software interrupts to S-mode
  Bit [5]  (MTIE)  → delegate timer interrupts to S-mode
  Bit [9]  (MEIE)  → delegate external interrupts to S-mode
  Bit [x]          → delegate specific PLIC IRQ to S-mode
```

When delegated, interrupts are taken in S-mode using `stvec`/`sepc`/`scause` instead of `mtvec`/`mepc`/`mcause`. This allows an OS (e.g., Linux) running in S-mode to handle device interrupts directly without trapping to M-mode first.

## 2.9 Summary

| Dimension | x86-64 (APIC) | ARM Cortex-M (NVIC) | RISC-V (CLINT + PLIC) |
|---|---|---|---|
| **Interrupt controller** | Distributed (Local APIC + I/O APIC + MSI) | Integrated (NVIC inside core) | Separate IP (CLINT + PLIC) |
| **Vector table** | IDT gate descriptors (segment + offset) | Function pointer array | Address array or single entry |
| **Hardware context save** | RFLAGS, CS, RIP, SS | **8 general registers** (most automated) | PC and MIE only (least automated) |
| **Interrupt return** | `IRETQ` | `BX LR` (EXC_RETURN magic) | `MRET` / `SRET` |
| **Priority levels** | 256 | 8–256 (typically 16) | PLIC: typically 8; local: fixed |
| **Tail-chaining / Late arrival** | No | **Yes** | No |
| **Selective masking** | APIC TPR | `BASEPRI` (flexible threshold) | `mie` CSR (per-type) |
| **FPU/VPU context save** | Software always saves (XRSTOR/FXSAVE) | Hardware always saves (extended frame) | **Lazy save**: FPU/VPU registers saved only when `mstatus.FS/VS == DIRTY`; mstatus.FS/VS marked CLEAN after save |
| **IPI** | APIC ICR (target CPU + vector) | N/A (single-core) | Single-core: `ecall`; Multi-hart: CLINT `msip` |
| **Interrupt delegation** | No | No | **Yes** (`mideleg`/`medeleg`) |
| **Typical entry latency** | Tens of cycles (non-deterministic) | **12 cycles** (deterministic) | Tens of cycles (depends on software save) |
| **Target domain** | Servers, desktops, laptops | Real-time microcontrollers | Scalable (MCU to HPC) |
