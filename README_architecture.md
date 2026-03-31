# FreeRTOS Architecture

This document describes the architecture of FreeRTOS, including its kernel, supplementary libraries (FreeRTOS-Plus), and all submodule components.

## 1. Overall Structure

```
FreeRTOS (this repo)
├── FreeRTOS/Source/                  # Kernel (submodule: FreeRTOS-Kernel)
├── FreeRTOS/Demo/                    # 200+ kernel demos by arch/board/toolchain
├── FreeRTOS/Test/                    # Kernel tests (CMock, CBMC, VeriFast, Target)
│
├── FreeRTOS-Plus/Source/             # Supplementary libraries
│   ├── FreeRTOS-Plus-TCP/            # TCP/IP stack (submodule)
│   ├── Application-Protocols/        # coreHTTP, coreMQTT, coreMQTT-Agent, coreSNTP
│   │   └── network_transport/        # TLS/plain transport implementations
│   ├── AWS/                          # device-shadow, device-defender, jobs, ota, sigv4, fleet-provisioning
│   ├── FreeRTOS-Cellular-Interface/  # Cellular modem abstraction (submodule)
│   ├── FreeRTOS-Cellular-Modules/    # bg96, hl7802, sara-r4 modem drivers (submodules)
│   ├── FreeRTOS-Plus-CLI/            # Command-line interpreter
│   ├── FreeRTOS-Plus-Trace/          # Percepio trace recorder (submodule)
│   ├── coreJSON/                     # JSON parser (submodule)
│   ├── corePKCS11/                   # PKCS#11 crypto interface (submodule)
│   ├── Reliance-Edge/                # File system
│   └── Utilities/                    # backoff_algorithm, logging
│
├── FreeRTOS-Plus/ThirdParty/         # mbedtls, wolfSSL, glib, libslirp, tinycbor
├── FreeRTOS-Plus/Demo/               # Plus library demos
├── FreeRTOS-Plus/Test/               # Plus library tests
│
└── tools/                            # Uncrustify config, AWS config helpers
```

**29 git submodules** in total, pinned by `manifest.yml`. Clone with `--recurse-submodules`.

## 2. FreeRTOS Kernel

**Version:** V11.1.0+
**License:** MIT
**Location:** `FreeRTOS/Source/` (submodule: FreeRTOS-Kernel)

### 2.1 Core Source Files

| File | Purpose |
|------|---------|
| `tasks.c` | Task management, scheduler, context switching |
| `queue.c` | Queues, semaphores, mutexes (unified implementation) |
| `list.c` | Doubly-linked circular list primitives |
| `timers.c` | Software timers via daemon task |
| `stream_buffer.c` | Stream buffers and message buffers |
| `event_groups.c` | Multi-bit event synchronization |
| `croutine.c` | Co-routines (legacy, disabled by default — `configUSE_CO_ROUTINES` defaults to 0) |

### 2.2 Task Control Block (TCB_t)

Defined in `tasks.c` (typedef chain: `tskTaskControlBlock` → `tskTCB` → `TCB_t`). Key fields:

| Field | Guard Condition | Description |
|---|---|---|
| **`pxTopOfStack`** | Always | Must be first member. Points to task's saved stack context. |
| **`xMPUSettings`** | `portUSING_MPU_WRAPPERS == 1` | MPU region configuration (inserted early for alignment) |
| **`uxCoreAffinityMask`** | `configUSE_CORE_AFFINITY == 1 && configNUMBER_OF_CORES > 1` | SMP core affinity bitmask |
| **`xStateListItem`** | Always | Determines which state list (Ready/Blocked/Suspended/Terminated) |
| **`xEventListItem`** | Always | Used when waiting on kernel objects (queues, event groups) |
| **`uxPriority`** | Always | Current priority |
| **`pxStack`** | Always | Points to start of allocated stack memory |
| **`xTaskRunState`** | `configNUMBER_OF_CORES > 1` | Which core the task runs on; `-1` = not running, `-2` = scheduled to yield |
| **`pcTaskName`** | Always | `char[ configMAX_TASK_NAME_LEN ]` |
| **`uxBasePriority`** | `configUSE_MUTEXES == 1` | Original priority (for mutex priority inheritance) |
| **`ulNotifiedValue[]`** | `configUSE_TASK_NOTIFICATIONS == 1` | Task notification values array of `configTASK_NOTIFICATION_ARRAY_ENTRIES` (default 1) |
| **`ucNotifyState[]`** | `configUSE_TASK_NOTIFICATIONS == 1` | Notification states, same array size as above |
| **`uxCriticalNesting`** | `portCRITICAL_NESTING_IN_TCB == 1` | Per-task critical section nesting counter |
| **`ucStaticallyAllocated`** | `tskSTATIC_AND_DYNAMIC_ALLOCATION_POSSIBLE` | Whether TCB was statically allocated |
| **`uxTCBNumber`** | `configUSE_TRACE_FACILITY == 1` | Unique task ID for tracing |
| **`ulRunTimeCounter`** | `configGENERATE_RUN_TIME_STATS == 1` | Accumulated CPU time |

### 2.3 Scheduler

**Algorithm:** Priority-based preemptive scheduling with optional time slicing.

**Key data structures** (all in `tasks.c`):
- `pxReadyTasksLists[configMAX_PRIORITIES]` — One `List_t` per priority level.
- `pxDelayedTaskList1/2` — Delay lists swapped on tick overflow, sorted by wake time.
- `uxTopReadyPriority` — Highest priority containing ready tasks (bitmap for O(1) lookup).

**Task selection:** `taskSELECT_HIGHEST_PRIORITY_TASK()` has three variants:
  1. **Generic single-core** (`configUSE_PORT_OPTIMISED_TASK_SELECTION == 0`): while-loop from `uxTopReadyPriority` downward.
  2. **Port-optimized** (`configUSE_PORT_OPTIMISED_TASK_SELECTION == 1`): Uses `portGET_HIGHEST_PRIORITY()` — implemented as CLZ (`__clz`, `__builtin_clz`) on ARM Cortex-M, RISC-V, and PIC32 ports for O(1) bitmap lookup.
  3. **SMP** (`configNUMBER_OF_CORES > 1`): Delegates to `prvSelectHighestPriorityTask()`, which handles core affinity, task run state, and `configRUN_MULTIPLE_PRIORITIES`.
  
  Equal-priority tasks share CPU time via round-robin (`pxIndex` advances in the ready list).

**Tick handler** (`xTaskIncrementTick`): Walks delay list, moves expired tasks to ready lists, triggers context switch if unblocked task has higher priority.

**Startup:** `vTaskStartScheduler()` creates idle task(s) (one per SMP core), timer daemon task, disables interrupts, calls `xPortStartScheduler()` (port-specific).

### 2.4 Inter-Task Communication

| Mechanism | Header | Characteristics |
|-----------|--------|----------------|
| **Queues** | `queue.h` | Items copied by value. Deferred ISR processing via `cRxLock`/`cTxLock`. |
| **Binary Semaphore** | `semphr.h` | Queue with length=1, itemSize=0. No priority inheritance. |
| **Counting Semaphore** | `semphr.h` | Queue with length=maxCount, itemSize=0. |
| **Mutex** | `semphr.h` | Priority inheritance: holder's `uxPriority` elevated to waiter's `uxBasePriority`. |
| **Recursive Mutex** | `semphr.h` | Same task can take multiple times; tracks `uxRecursiveCallCount`. |
| **Event Groups** | `event_groups.h` | 24/56 usable bits (depending on tick width). `xEventGroupSync()` for barriers. |
| **Task Notifications** | `task.h` | Lightweight 1:1 IPC via TCB `ulNotifiedValue[]`. Actions: set bits, increment, overwrite. |
| **Stream Buffers** | `stream_buffer.h` | Single-reader, single-writer circular buffer. Variable-length. |
| **Message Buffers** | `message_buffer.h` (header-only, delegates to `stream_buffer.c`) | Built on stream buffers with length-prefixed messages. No separate `message_buffer.c` — all APIs are inline wrappers. |

**ISR interaction:** All objects have `...FromISR()` variants that never block and return `xHigherPriorityTaskWoken`. Caller must call `portYIELD_FROM_ISR()` to defer context switch.

### 2.5 Memory Management

Five heap implementations in `portable/MemMang/`:

| Scheme | Alloc | Free | Coalesce | Multi-region | Use case |
|--------|-------|------|----------|-------------|----------|
| **heap_1** | Bump pointer | No | N/A | No | No free needed |
| **heap_2** | Best fit | Yes | No | No | Fixed-size alloc/free |
| **heap_3** | Wraps `malloc`/`free` | Yes | C library | No | C library available |
| **heap_4** | First fit by address | Yes | Adjacent merge | No | General purpose |
| **heap_5** | First fit by address | Yes | Adjacent merge | **Yes** | Non-contiguous memory |

All use `vTaskSuspendAll()`/`xTaskResumeAll()` for thread safety. heap_5 requires `vPortDefineHeapRegions()` before any allocation.

**Additional APIs (heap_2/4/5):** `pvPortCalloc()` — allocate and zero-fill (multiplication-overflow safe). `vPortHeapResetState()` — reset heap state for reinitialization (all 5 schemes). `xPortResetHeapMinimumEverFreeHeapSize()` — reset historical low-water mark (heap_4/5). `vPortGetHeapStats()` — detailed fragmentation statistics with two-stage locking (heap_4/5).

### 2.6 Portable Layer

Each architecture port provides `portmacro.h` and `port.c` defining:
- `pxPortInitialiseStack()` — Initialize task stack frame
- `xPortStartScheduler()` — Set up tick timer, start first task
- `portYIELD()` — Trigger context switch (PendSV on ARM Cortex-M)
- `portENTER_CRITICAL()`/`portEXIT_CRITICAL()` — Nestable critical sections
- `portSET_INTERRUPT_MASK_FROM_ISR()` — ISR-safe critical sections (BASEPRI on ARM)
- `portSUPPRESS_TICKS_AND_SLEEP()` — Tickless idle

**Supported architectures** (in `portable/GCC/`): ARM Cortex-M0/M3/M4/M4F/M7/M23/M33/M55/M85, Cortex-A CA9/CA53, AArch64, RISC-V (RV32I, RV32E, RV64), AVR, MicroBlaze, TriCore, RX, and more. Community and partner ports in separate submodules.

**RISC-V port specifics:** The RISC-V port (`portable/GCC/RISC-V/`) uses `ecall` for `portYIELD()`, `mret` for interrupt return, and implements **lazy FPU/VPU context save**: registers are saved/restored only when `mstatus.FS` (FPU status) or `mstatus.VS` (vector status) indicates the previous task dirtied them, detected by checking bits [14:13] and [10:9] == 3 (DIRTY). The trap handler is 256-byte aligned for `mtvec` in direct mode, supports both vectored and direct dispatch via `mcause`.

### 2.7 SMP Support

Controlled by `configNUMBER_OF_CORES` (default 1). Key mechanisms:
- `pxCurrentTCBs[configNUMBER_OF_CORES]` replaces single `pxCurrentTCB`
- Spinlocks (`portGET_TASK_LOCK`/`portGET_ISR_LOCK`) protect scheduler data
- Core affinity (`uxCoreAffinityMask`) controls which cores a task can run on
- `configRUN_MULTIPLE_PRIORITIES` — When 0, only one priority level runs simultaneously

### 2.8 MPU Support

When `portUSING_MPU_WRAPPERS == 1`:
- API functions are redirected to `MPU_` prefixed versions
- V2 wrappers use opaque handles and `xPortIsAuthorizedToAccessKernelObject()`
- `TaskParameters_t.xRegions[]` defines per-task memory regions with R/W/X permissions

## 3. FreeRTOS-Plus-TCP (TCP/IP Stack)

**Version:** V4.4.0
**Location:** `FreeRTOS-Plus/Source/FreeRTOS-Plus-TCP/`
**Submodule from:** FreeRTOS/FreeRTOS-Plus-TCP

A complete TCP/IP stack providing IPv4/IPv6 dual-stack support with a Berkeley-sockets API. Runs as a dedicated FreeRTOS task (the "IP-task").

### 3.1 Architecture

The stack is event-driven. A single IP-task processes events from a queue (`xNetworkEventQueue`):

```
                    ┌─────────────────────────────────────────┐
                    │           IP-Task (prvIPTask)            │
                    │  ┌──────────────────────────────────┐   │
                    │  │ prvProcessIPEventsAndTimers()     │   │
                    │  │   xQueueReceive(xNetworkEventQueue)│   │
                    │  │   switch(event):                  │   │
                    │  │     eNetworkRxEvent → prvHandleEthernetPacket()  │
                    │  │     eNetworkTxEvent → prvForwardTxPacket()      │
                    │  │     eStackTxEvent → vProcessGeneratedUDPPacket()│
                    │  │     eDHCPEvent → DHCP state machine │
                    │  │     eTCPTimerEvent → TCP timers     │
                    │  │     eARPTimerEvent → ARP cache age  │
                    │  │     ...                            │   │
                    │  └──────────────────────────────────┘   │
                    └─────────────────────────────────────────┘
```

### 3.2 Key Types

- **`NetworkInterface_t`** (`FreeRTOS_Routing.h`) — Physical network interface with function pointers for init, output, PHY status, MAC filtering. Linked list supports multi-homing.
- **`NetworkEndPoint_t`** (`FreeRTOS_Routing.h`) — IP address configuration on an interface. Holds IPv4/IPv6 settings, DHCP/RA state machines.
- **`NetworkBufferDescriptor_t`** (`FreeRTOS_IP.h`) — Packet buffer descriptor tracking Ethernet buffer pointer, data length, associated interface/endpoint, and optional `pxNextBuffer` for driver-linked-list optimization. IP address is a union of `ul_IPAddress` (IPv4) and `x_IPv6Address` (IPv6).

### 3.3 Key Source Files

| File | Purpose |
|------|---------|
| `FreeRTOS_IP.c` | IP-task main loop, event dispatch, initialization |
| `FreeRTOS_Sockets.c` | Berkeley sockets API |
| `FreeRTOS_Routing.c` | Interface/endpoint management |
| `FreeRTOS_ARP.c` | IPv4 ARP resolution and cache |
| `FreeRTOS_ND.c` | IPv6 Neighbor Discovery |
| `FreeRTOS_DHCP.c` / `FreeRTOS_DHCPv6.c` | DHCP clients |
| `FreeRTOS_DNS.c` | DNS resolver (IPv4/IPv6), LLMNR, mDNS |
| `FreeRTOS_TCP_IP.c` | TCP state machine |
| `FreeRTOS_TCP_Reception.c` / `FreeRTOS_TCP_Transmission.c` | TCP data handling |
| `FreeRTOS_UDP_IP.c` | UDP processing |
| `FreeRTOS_IPv4.c` / `FreeRTOS_IPv6.c` | IP protocol processing |
| `FreeRTOS_IP_Timers.c` | Software timers for ARP, DHCP, DNS, TCP |
| `FreeRTOS_Stream_Buffer.c` | Circular buffer for TCP streams |

### 3.4 Buffer Management

Two allocation schemes in `source/portable/BufferManagement/`:

- **BufferAllocation_1** (`xBufferAllocFixedSize = pdTRUE`): Fixed-size pre-allocated pool. All buffers are `ipTOTAL_ETHERNET_FRAME_SIZE`. RAM allocated by `uxNetworkInterfaceAllocateRAMToBuffers()`. Deterministic, no fragmentation.
- **BufferAllocation_2** (`xBufferAllocFixedSize = pdFALSE`): Variable-size, dynamically allocated using `pvPortMalloc()`. Can resize buffers. Requires heap_4 or heap_5. Better memory efficiency.

Both use a pool of `ipconfigNUM_NETWORK_BUFFER_DESCRIPTORS` descriptors tracked via `xFreeBuffersList`.

### 3.5 Network Interface Driver Contract

Each driver populates a `NetworkInterface_t` struct:

```c
typedef struct xNetworkInterface {
    NetworkInterfaceInitialiseFunction_t  pfInitialise;      // Initialize hardware
    NetworkInterfaceOutputFunction_t      pfOutput;          // Send packet
    GetPhyLinkStatusFunction_t            pfGetPhyLinkStatus;// Check PHY link
    NetworkInterfaceMACFilterFunction_t   pfAddAllowedMAC;   // Add MAC filter
    NetworkInterfaceMACFilterFunction_t   pfRemoveAllowedMAC;// Remove MAC filter
    // ... linked list, endpoint list, state flags
} NetworkInterface_t;
```

**Driver implementations** exist in `source/portable/NetworkInterface/` for: ESP32 (WiFi), STM32, Zynq, LPC, ATSAM, PIC32MZ (Ethernet + WiFi), NXP1060, M487, KSZ8851SNL, Linux (pcap), WinPCap, and more.

## 4. Application Protocol Libraries

All protocol libraries are transport-agnostic via `TransportInterface_t` (recv/send/writev function pointers + NetworkContext).

### 4.1 coreHTTP (v3.0.0)

HTTP/1.1 client (RFC 7230/7231). Supports GET/PUT/POST/HEAD, range requests, chunked transfer encoding. Uses llhttp for response parsing. Key API: `HTTPClient_InitializeRequestHeaders()`, `HTTPClient_Send()`, `HTTPClient_ReadHeader()`.

### 4.2 coreMQTT (v2.3.1+)

MQTT 3.1.1 client. Manages QoS 1/2 publish state machines, keep-alive, session management. Key API: `MQTT_Init()`, `MQTT_Connect()`, `MQTT_Publish()`, `MQTT_Subscribe()`, `MQTT_ProcessLoop()`.

### 4.3 coreMQTT-Agent

Manages a shared MQTT connection for multiple FreeRTOS tasks. Runs in a dedicated agent task using `MQTTAgentMessageInterface_t` (pluggable message queue). Allows safe multi-task publish/subscribe through a single connection.

### 4.4 coreSNTP (v1.2.0)

SNTPv4 client (RFC 4330). Supports authenticated and unauthenticated modes. Application provides UDP transport and time callbacks.

### 4.5 network_transport

Bridges protocol libraries to network stacks:

| Transport | TLS Library | Socket Layer |
|-----------|------------|--------------|
| `transport_mbedtls` | mbedTLS | TCP sockets wrapper (FreeRTOS+TCP or cellular) |
| `transport_mbedtls_pkcs11` | mbedTLS + PKCS#11 | TCP sockets wrapper |
| `transport_wolfSSL` | wolfSSL | FreeRTOS+TCP directly |
| `transport_plaintext` | None | FreeRTOS+TCP directly |

`tcp_sockets_wrapper/` abstracts TCP operations with ports for FreeRTOS+TCP and cellular interface.

## 5. AWS IoT Libraries

All under `FreeRTOS-Plus/Source/AWS/`. Use coreMQTT for communication, coreJSON for parsing.

| Library | Purpose |
|---------|---------|
| **device-shadow** | AWS IoT Device Shadow service — sync device state with cloud |
| **device-defender** | Report security metrics to Device Defender service |
| **jobs** | AWS IoT Jobs — OTA job lifecycle management |
| **fleet-provisioning** | Device provisioning (Claim certificates, RegisterThing) |
| **ota** | Over-the-air firmware updates via MQTT or HTTP, CBOR-encoded |
| **sigv4** | AWS Signature V4 signing for HTTP requests (e.g., S3 downloads) |

**OTA** is the most complex: state machine manages check→download→verify→apply lifecycle. Portable interfaces: `OtaPal_t` (platform: flash write, reset), `OtaOSInterface_t` (timers, events).

## 6. Cellular Interface

**Location:** `FreeRTOS-Plus/Source/FreeRTOS-Cellular-Interface/` (v1.3.0)

Three-layer architecture:
1. **Public API** (`cellular_api.h`) — Socket operations, SIM management, network registration
2. **Common layer** (`cellular_common_api.c`) — Shared implementation
3. **Modem port** — AT command handlers for specific modems (bg96, hl7802, sara-r4 in `FreeRTOS-Cellular-Modules/`)

Uses `CellularCommInterface_t` (HAL for modem UART). Can serve as transport for `tcp_sockets_wrapper`.

## 7. Utility Libraries

| Library | Purpose |
|---------|---------|
| **coreJSON** (v3.2.0) | Zero-allocation JSON parser. `JSON_Search()` with dot-path queries. |
| **corePKCS11** (v3.6.2) | PKCS#11 wrapper. PAL interface for NVM object storage. PKI utils for signature conversion. |
| **backoff_algorithm** (v1.3.0) | Full Jitter exponential backoff for network retries. |
| **logging** | Unified `LogError()`/`LogWarn()`/`LogInfo()`/`LogDebug()` macros. |
| **FreeRTOS-Plus-CLI** | UART command registration and parsing. |

## 8. Third-Party Libraries

| Library | Purpose |
|---------|---------|
| **mbedTLS** (v3.5.1) | TLS 1.2/1.3, X.509, crypto primitives. Primary TLS library. |
| **wolfSSL** (v5.6.4) | Alternative TLS library. |
| **libslirp** | User-mode network stack for POSIX simulation (NAT, DHCP forwarding). |
| **glib** | GNOME utility library (dependency of libslirp). |
| **tinycbor** | CBOR codec (used by OTA library). |
| **CMock** | Mock/frame generation for unit tests (ThrowTheSwitch). Two instances. |

## 9. Testing

| Framework | Location | Purpose |
|-----------|----------|---------|
| **CMock** | `FreeRTOS/Test/CMock/` | Kernel unit tests (tasks, queue, list, timers, stream_buffer, message_buffer, event_groups, smp) |
| **CBMC** | `FreeRTOS/Test/CBMC/` | Bounded model checking proofs for memory safety |
| **VeriFast** | `FreeRTOS/Test/VeriFast/` | Formal verification of linked list and queue operations |
| **Target** | `FreeRTOS/Test/Target/` | Hardware integration tests (Raspberry Pi Pico SMP) |

## 10. Layer Dependency Diagram

```
Application
    │
    ▼
AWS Libraries (OTA, Shadow, Defender, Jobs, Fleet Provisioning, SigV4)
    │                │
    ▼                ▼
coreMQTT-Agent   coreHTTP   coreSNTP
    │                │           │
    ▼                ▼           ▼
coreMQTT     (TransportInterface_t)
    │
    ▼
network_transport (mbedTLS / wolfSSL / PKCS#11 / plaintext)
    │              │
    ▼              ▼
TCP Sockets Wrapper
    │
    ▼
FreeRTOS-Plus-TCP (IPv4/IPv6, DHCP, DNS, ARP/ND, TCP, UDP)
    │
    ▼
Network Interface Driver (Ethernet MAC / WiFi / Cellular)
    │
    ▼
FreeRTOS Kernel (Tasks, Queues, Timers, Semaphores, Event Groups)

Cross-cutting: coreJSON, corePKCS11, backoff_algorithm, logging
```
