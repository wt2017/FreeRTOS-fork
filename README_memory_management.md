# heap_4.c 源码详解

> 源文件路径：`FreeRTOS/Source/portable/MemMang/heap_4.c`，共 639 行。
>
> heap_4 是 FreeRTOS **最常用的堆管理方案**，采用首次适应（First-Fit）算法分配、地址排序空闲链表管理、释放时自动合并相邻空闲块以减少碎片。

## 1. 文件头与编译守卫（第 1–52 行）

```c
// heap_4.c:1–27  — MIT 许可证头
// heap_4.c:37–38  — 包含 <stdlib.h> 和 <string.h>
// heap_4.c:50–52  — 编译守卫：若 configSUPPORT_DYNAMIC_ALLOCATION == 0 则报错
#if (configSUPPORT_DYNAMIC_ALLOCATION == 0)
    #error This file must not be used if configSUPPORT_DYNAMIC_ALLOCATION is 0
#endif
```

heap_4 只能在启用动态分配时使用。与 heap_1/2/3/5 互斥——链接时只能选择其中一个，因为它们都定义相同的符号 `pvPortMalloc` / `vPortFree`。

## 2. 宏定义与常量（第 54–84 行）

### 2.1 可选特性开关（第 54–56 行）

```c
// heap_4.c:54–56
#ifndef configHEAP_CLEAR_MEMORY_ON_FREE
    #define configHEAP_CLEAR_MEMORY_ON_FREE  0   // 默认不清零释放的内存
#endif
```

### 2.2 算术溢出/下溢检测宏（第 58–74 行）

| 宏 | 功能 | 代码位置 |
|----|------|----------|
| `heapMINIMUM_BLOCK_SIZE` | `2 × xHeapStructSize`，分割后剩余部分的最小阈值 | 第 59 行 |
| `heapBITS_PER_BYTE` | 固定为 8 | 第 62 行 |
| `heapSIZE_MAX` | `~(size_t)0`，size_t 能表示的最大值 | 第 65 行 |
| `heapMULTIPLY_WILL_OVERFLOW(a, b)` | 乘法溢出检测 | 第 68 行 |
| `heapADD_WILL_OVERFLOW(a, b)` | 加法溢出检测 | 第 71 行 |
| `heapSUBTRACT_WILL_UNDERFLOW(a, b)` | 减法下溢检测 | 第 74 行 |

这些宏在 `pvPortMalloc` 和 `pvPortCalloc` 中用于防御性检查，防止整数溢出导致的安全漏洞。

### 2.3 块分配标志位宏（第 76–84 行）

```c
// heap_4.c:80
#define heapBLOCK_ALLOCATED_BITMASK  ((size_t)1 << (sizeof(size_t) * 8 - 1))
```

`xBlockSize` 的最高位（MSB）被用作分配状态标志：

| 宏 | 操作 | 代码行 |
|----|------|--------|
| `heapBLOCK_SIZE_IS_VALID(xBlockSize)` | 检查 MSB 是否为 0（大小合法） | 81 |
| `heapBLOCK_IS_ALLOCATED(pxBlock)` | 检查 MSB 是否为 1（已分配） | 82 |
| `heapALLOCATE_BLOCK(pxBlock)` | MSB 置 1（标记为已分配） | 83 |
| `heapFREE_BLOCK(pxBlock)` | MSB 清零（恢复真实大小） | 84 |

**设计意义：** 不需要额外的标志字段，复用 `xBlockSize` 的一位即可区分已分配/空闲状态。代价是单个块最大为 `2^(sizeof(size_t)*8 - 1)` 字节（32 位系统上为 2 GB），对嵌入式系统完全足够。

## 3. 堆存储区定义（第 88–96 行）

```c
// heap_4.c:89–96
#if (configAPPLICATION_ALLOCATED_HEAP == 1)
    extern uint8_t ucHeap[configTOTAL_HEAP_SIZE];   // 由应用程序定义（可放入特定内存区域）
#else
    PRIVILEGED_DATA static uint8_t ucHeap[configTOTAL_HEAP_SIZE];  // 编译器静态分配
#endif
```

两种模式：

- **默认**（`configAPPLICATION_ALLOCATED_HEAP == 0`）：编译器在 BSS 段分配 `ucHeap[]`。
- **自定义**（`configAPPLICATION_ALLOCATED_HEAP == 1`）：应用程序自行定义数组，可通过 `__attribute__((section(".dlm")))` 等手段放入特定内存区域（如 RISC-V 的 DLM）。

## 4. 核心数据结构（第 98–137 行）

### 4.1 块头结构体 BlockLink_t（第 100–104 行）

```c
// heap_4.c:100–104
typedef struct A_BLOCK_LINK
{
    struct A_BLOCK_LINK * pxNextFreeBlock;  // 空闲链表中的下一个空闲块
    size_t xBlockSize;                      // 本块总大小（含块头本身），MSB 为分配标志
} BlockLink_t;
```

**内存布局：** 每个内存块（空闲或已分配）开头 8/16 字节（取决于 32/64 位架构）存放此结构体。用户通过 `pvPortMalloc` 拿到的指针跳过了这个头部。

```
内存块结构：
┌──────────────────────┬────────────────────────────────┐
│ BlockLink_t (块头)    │ 用户数据区                      │
│ pxNextFreeBlock (4B) │                                │
│ xBlockSize    (4B)   │                                │
└──────────────────────┴────────────────────────────────┘
↑                      ↑
pxBlock                pvReturn (用户拿到的指针)
```

### 4.2 堆保护（Heap Protector）（第 106–131 行）

```c
// heap_4.c:110–130
#if (configENABLE_HEAP_PROTECTOR == 1)
    extern void vApplicationGetRandomHeapCanary(portPOINTER_SIZE_TYPE *pxHeapCanary);
    PRIVILEGED_DATA static portPOINTER_SIZE_TYPE xHeapCanary;

    // 将 BlockLink_t 指针与随机 canary 异或后存储
    #define heapPROTECT_BLOCK_POINTER(pxBlock) \
        ((BlockLink_t *)(((portPOINTER_SIZE_TYPE)(pxBlock)) ^ xHeapCanary))
#else
    #define heapPROTECT_BLOCK_POINTER(pxBlock) (pxBlock)
#endif
```

启用后，所有写入 `pxNextFreeBlock` 的指针值都与 `xHeapCanary` 异或，读取时再异或还原。若发生堆缓冲溢出覆盖了指针字段，还原出的地址是随机乱值，会被 `heapVALIDATE_BLOCK_POINTER` 断言捕获。

### 4.3 指针合法性断言（第 133–136 行）

```c
// heap_4.c:134–136
#define heapVALIDATE_BLOCK_POINTER(pxBlock) \
    configASSERT(((uint8_t *)(pxBlock) >= &(ucHeap[0])) && \
                 ((uint8_t *)(pxBlock) <= &(ucHeap[configTOTAL_HEAP_SIZE - 1])))
```

验证块指针落在 `ucHeap[]` 的地址范围内。若堆保护启用后指针被篡改，此处会触发断言失败。

## 5. 前向声明（第 140–153 行）

```c
// heap_4.c:146 — 释放时将块插入空闲链表（含合并）
static void prvInsertBlockIntoFreeList(BlockLink_t *pxBlockToInsert) PRIVILEGED_FUNCTION;

// heap_4.c:152 — 首次 malloc 时自动初始化堆结构
static void prvHeapInit(void) PRIVILEGED_FUNCTION;
```

`PRIVILEGED_FUNCTION` 宏在 MPU 移植中用于标记只能在特权模式下调用的函数。

## 6. 全局状态变量（第 155–169 行）

### 6.1 块头实际大小（第 158 行）

```c
// heap_4.c:158
static const size_t xHeapStructSize = (sizeof(BlockLink_t) + (portBYTE_ALIGNMENT - 1))
                                    & ~((size_t)portBYTE_ALIGNMENT_MASK);
```

将 `sizeof(BlockLink_t)` 向上取整到 `portBYTE_ALIGNMENT`（通常 8 字节）。在 32 位系统上：`BlockLink_t` = 8 字节，对齐到 8 后 `xHeapStructSize` = 8。

### 6.2 空闲链表哨兵与统计变量（第 161–169 行）

```c
// heap_4.c:161–162 — 链表哨兵
PRIVILEGED_DATA static BlockLink_t xStart;          // 头哨兵，不在堆内
PRIVILEGED_DATA static BlockLink_t *pxEnd = NULL;   // 尾哨兵，在堆末尾

// heap_4.c:166–169 — 统计变量
PRIVILEGED_DATA static size_t xFreeBytesRemaining = 0U;
PRIVILEGED_DATA static size_t xMinimumEverFreeBytesRemaining = 0U;
PRIVILEGED_DATA static size_t xNumberOfSuccessfulAllocations = 0U;
PRIVILEGED_DATA static size_t xNumberOfSuccessfulFrees = 0U;
```

| 变量 | 类型 | 含义 |
|------|------|------|
| `xStart` | `BlockLink_t` | 头哨兵，`.xBlockSize = 0`，`.pxNextFreeBlock` 指向第一个真正空闲块 |
| `pxEnd` | `BlockLink_t *` | 尾哨兵，放在堆末尾，`.xBlockSize = 0`，`.pxNextFreeBlock = NULL`。**`NULL` 是堆未初始化的标志** |
| `xFreeBytesRemaining` | `size_t` | 当前空闲字节总数 |
| `xMinimumEverFreeBytesRemaining` | `size_t` | 历史最小空闲字节数（堆耗尽风险的指标） |
| `xNumberOfSuccessfulAllocations` | `size_t` | 累计成功调用 `pvPortMalloc` 的次数 |
| `xNumberOfSuccessfulFrees` | `size_t` | 累计成功调用 `vPortFree` 的次数 |

## 7. `pvPortMalloc()` — 内存分配（第 173–351 行）

这是堆分配的主入口。整体分为三个阶段。

### 7.1 阶段一：请求大小调整（第 182–219 行）

将用户请求的字节数调整为包含块头、满足对齐要求的实际分配大小。

```
用户请求 xWantedSize 字节
    │
    ├─ xWantedSize == 0? → 跳过，pvReturn 保持 NULL
    │
    ├─ xWantedSize + xHeapStructSize 会溢出? → xWantedSize = 0，分配必然失败
    │
    ├─ xWantedSize += xHeapStructSize              （加上块头）
    │
    ├─ 已对齐? → 不变
    │  未对齐? → 向上取整到 portBYTE_ALIGNMENT 边界
    │            若取整后溢出 → xWantedSize = 0
    │
    └─ 最终 xWantedSize = 实际需要分配的总字节数
```

**代码证据：**

```c
// heap_4.c:186–188 — 加上块头大小
if (heapADD_WILL_OVERFLOW(xWantedSize, xHeapStructSize) == 0) {
    xWantedSize += xHeapStructSize;
}

// heap_4.c:192–199 — 字节对齐，向上取整
if ((xWantedSize & portBYTE_ALIGNMENT_MASK) != 0x00) {
    xAdditionalRequiredSize = portBYTE_ALIGNMENT - (xWantedSize & portBYTE_ALIGNMENT_MASK);
    if (heapADD_WILL_OVERFLOW(xWantedSize, xAdditionalRequiredSize) == 0) {
        xWantedSize += xAdditionalRequiredSize;
    } else {
        xWantedSize = 0;   // 溢出，放弃
    }
}
```

### 7.2 阶段二：首次分配时初始化堆（第 221–232 行）

```c
// heap_4.c:221
vTaskSuspendAll();          // 冻结调度器（禁止任务切换，中断仍然开启）
{
    // heap_4.c:225–228
    if (pxEnd == NULL) {    // pxEnd == NULL 说明堆未初始化
        prvHeapInit();
    }
    // ... 后续分配逻辑 ...
}
(void)xTaskResumeAll();     // 恢复调度器
```

`pxEnd == NULL` 是堆未初始化的唯一判据。初始化只执行一次。

### 7.3 阶段三：首次适应搜索 + 分割（第 238–313 行）

**步骤 3a — 合法性检查（第 238–240 行）：**

```c
// heap_4.c:238 — 确认调整后的大小没有触及 MSB 标志位
if (heapBLOCK_SIZE_IS_VALID(xWantedSize) != 0) {
    // heap_4.c:240 — 确认堆内有足够空间
    if ((xWantedSize > 0) && (xWantedSize <= xFreeBytesRemaining)) {
```

**步骤 3b — 首次适应（First-Fit）搜索（第 244–253 行）：**

从 `xStart` 沿空闲链表按**地址升序**遍历，找第一个大小足够的块：

```c
// heap_4.c:244–253
pxPreviousBlock = &xStart;
pxBlock = heapPROTECT_BLOCK_POINTER(xStart.pxNextFreeBlock);
heapVALIDATE_BLOCK_POINTER(pxBlock);

while ((pxBlock->xBlockSize < xWantedSize) &&
       (pxBlock->pxNextFreeBlock != heapPROTECT_BLOCK_POINTER(NULL))) {
    pxPreviousBlock = pxBlock;
    pxBlock = heapPROTECT_BLOCK_POINTER(pxBlock->pxNextFreeBlock);
    heapVALIDATE_BLOCK_POINTER(pxBlock);
}
```

遍历结束后：
- `pxPreviousBlock` = 目标块的前驱
- `pxBlock` = 找到的候选块（或 `pxEnd` 表示没找到）

**步骤 3c — 判断是否找到（第 257 行）：**

```c
if (pxBlock != pxEnd) {   // 没走到末尾哨兵 → 找到了合适块
```

**步骤 3d — 计算返回指针（第 261–262 行）：**

```c
pvReturn = (void *)(((uint8_t *)heapPROTECT_BLOCK_POINTER(pxPreviousBlock->pxNextFreeBlock))
                     + xHeapStructSize);
```

从块起始地址跳过 `BlockLink_t` 块头，即为用户数据区起始地址。

**步骤 3e — 从空闲链表中摘除（第 266 行）：**

```c
pxPreviousBlock->pxNextFreeBlock = pxBlock->pxNextFreeBlock;
```

将目标块从空闲链表中移除。

**步骤 3f — 块分割（Split）（第 272–289 行）：**

如果找到的块比需要的大，且剩余部分 > `heapMINIMUM_BLOCK_SIZE`（`2 × xHeapStructSize`），则一分为二：

```c
// heap_4.c:278 — 在当前块内偏移 xWantedSize 处创建新块头
pxNewBlockLink = (void *)(((uint8_t *)pxBlock) + xWantedSize);

// heap_4.c:283–284 — 重新划分大小
pxNewBlockLink->xBlockSize = pxBlock->xBlockSize - xWantedSize;  // 后半段（新空闲块）
pxBlock->xBlockSize = xWantedSize;                                // 前半段（将分配出去）

// heap_4.c:287–288 — 新空闲块替代原块在链表中的位置
pxNewBlockLink->pxNextFreeBlock = pxPreviousBlock->pxNextFreeBlock;
pxPreviousBlock->pxNextFreeBlock = heapPROTECT_BLOCK_POINTER(pxNewBlockLink);
```

**为什么新空闲块不需要遍历插入？** 因为链表按地址排序，原块被一分为二后，后半段的地址一定紧接在前半段之后，所以直接接在前驱后面就保持了地址有序。

若剩余部分 ≤ `heapMINIMUM_BLOCK_SIZE`，则不分割——剩余空间太小不值得维护一个新块头，直接将整块分配出去（内部碎片）。

**步骤 3g — 更新统计（第 295–306 行）：**

```c
// heap_4.c:295 — 减少空闲计数
xFreeBytesRemaining -= pxBlock->xBlockSize;

// heap_4.c:297–300 — 更新高水位线
if (xFreeBytesRemaining < xMinimumEverFreeBytesRemaining) {
    xMinimumEverFreeBytesRemaining = xFreeBytesRemaining;
}

// heap_4.c:306 — 记录分配块大小（传给 traceMALLOC）
xAllocatedBlockSize = pxBlock->xBlockSize;
```

**步骤 3h — 标记已分配（第 310–312 行）：**

```c
heapALLOCATE_BLOCK(pxBlock);                                // xBlockSize 的 MSB 置 1
pxBlock->pxNextFreeBlock = heapPROTECT_BLOCK_POINTER(NULL); // 已分配块不在链表中，next 设 NULL
xNumberOfSuccessfulAllocations++;
```

### 7.4 阶段四：收尾（第 329–351 行）

```c
// heap_4.c:329 — 调用用户可覆写的 trace 宏
traceMALLOC(pvReturn, xAllocatedBlockSize);

// heap_4.c:334 — 恢复调度器
(void)xTaskResumeAll();

// heap_4.c:336–347 — 分配失败回调
#if (configUSE_MALLOC_FAILED_HOOK == 1)
    if (pvReturn == NULL) {
        vApplicationMallocFailedHook();
    }
#endif

// heap_4.c:349 — 最终对齐检查
configASSERT((((size_t)pvReturn) & (size_t)portBYTE_ALIGNMENT_MASK) == 0);
return pvReturn;
```

### 7.5 分配流程总结图

```
pvPortMalloc(xWantedSize)
│
├─ 调整大小: +块头, 对齐, 溢出检查
│
├─ vTaskSuspendAll()
│   ├─ pxEnd == NULL? → prvHeapInit()
│   ├─ 大小合法 且 不超过 xFreeBytesRemaining?
│   │   ├─ First-Fit 遍历空闲链表
│   │   ├─ 找到 (pxBlock != pxEnd)?
│   │   │   ├─ 计算用户指针 pvReturn
│   │   │   ├─ 从链表摘除
│   │   │   ├─ 剩余 > heapMINIMUM_BLOCK_SIZE? → 分割
│   │   │   ├─ 更新 xFreeBytesRemaining, 高水位线
│   │   │   ├─ heapALLOCATE_BLOCK (MSB=1)
│   │   │   ├─ pxBlock->pxNextFreeBlock = NULL
│   │   │   └─ xNumberOfSuccessfulAllocations++
│   │   └─ 未找到 → pvReturn 保持 NULL
│   └─ traceMALLOC(pvReturn, size)
│
├─ xTaskResumeAll()
├─ pvReturn == NULL 且 configUSE_MALLOC_FAILED_HOOK? → vApplicationMallocFailedHook()
└─ return pvReturn
```

## 8. `vPortFree()` — 内存释放（第 354–410 行）

### 8.1 定位块头（第 356–366 行）

```c
// heap_4.c:356
uint8_t *puc = (uint8_t *)pv;    // 用户指针

// heap_4.c:363 — 往回退一个块头大小，得到 BlockLink_t 指针
puc -= xHeapStructSize;
pxLink = (void *)puc;
```

### 8.2 合法性校验（第 368–374 行）

```c
// heap_4.c:369 — 必须标记为已分配（MSB = 1）
configASSERT(heapBLOCK_IS_ALLOCATED(pxLink) != 0);

// heap_4.c:370 — 已分配块的 pxNextFreeBlock 必须为 NULL
configASSERT(pxLink->pxNextFreeBlock == heapPROTECT_BLOCK_POINTER(NULL));
```

这两项检查防止：
- **释放野指针**：不是通过 `pvPortMalloc` 获得的指针，MSB 不为 1。
- **重复释放**：已经被释放的块 MSB 被清零，再次释放时第一个断言失败。
- **释放空闲链表中的块**：空闲块的 `pxNextFreeBlock` 不为 NULL，第二个断言失败。

### 8.3 条件分支：已分配确认（第 372–408 行）

```c
if (heapBLOCK_IS_ALLOCATED(pxLink) != 0) {              // 再次确认（非 assert 级别）
    if (pxLink->pxNextFreeBlock == heapPROTECT_BLOCK_POINTER(NULL)) {  // 再次确认
```

在 assert 被禁用的生产构建中，这两层 if 充当运行时防护。

### 8.4 清除分配标志（第 378 行）

```c
heapFREE_BLOCK(pxLink);   // xBlockSize 的 MSB 清零，恢复真实大小
```

**必须在后续操作之前执行**，因为合并时需要用 `xBlockSize` 计算相邻地址，真实大小不能包含 MSB 标志位。

### 8.5 可选：清零用户数据（第 379–388 行）

```c
#if (configHEAP_CLEAR_MEMORY_ON_FREE == 1)
{
    if (heapSUBTRACT_WILL_UNDERFLOW(pxLink->xBlockSize, xHeapStructSize) == 0) {
        (void)memset(puc + xHeapStructSize, 0, pxLink->xBlockSize - xHeapStructSize);
    }
}
#endif
```

清零范围为：块头之后、整个用户数据区。防止释放后数据残留导致信息泄露（安全敏感场景）。

### 8.6 插入空闲链表并合并（第 390–397 行）

```c
vTaskSuspendAll();
{
    xFreeBytesRemaining += pxLink->xBlockSize;   // 恢复空闲计数
    traceFREE(pv, pxLink->xBlockSize);           // 调用 trace 宏
    prvInsertBlockIntoFreeList(pxLink);           // 插入并合并（核心逻辑，见第 9 节）
    xNumberOfSuccessfulFrees++;
}
(void)xTaskResumeAll();
```

### 8.7 释放流程总结图

```
vPortFree(pv)
│
├─ pv == NULL? → 直接返回
│
├─ 定位块头: puc = pv - xHeapStructSize
├─ 校验: MSB==1 且 next==NULL
│
├─ heapFREE_BLOCK (MSB 清零)
├─ configHEAP_CLEAR_MEMORY_ON_FREE? → memset 清零用户数据
│
├─ vTaskSuspendAll()
│   ├─ xFreeBytesRemaining += xBlockSize
│   ├─ traceFREE(pv, size)
│   ├─ prvInsertBlockIntoFreeList(pxLink)   ← 核心合并逻辑
│   └─ xNumberOfSuccessfulFrees++
└─ xTaskResumeAll()
```

## 9. `prvInsertBlockIntoFreeList()` — 插入空闲链表并合并（第 504–569 行）

**这是 heap_4 与 heap_2 的核心区别**。heap_2 释放时只插入链表不合并，heap_4 会自动合并地址相邻的空闲块。

### 9.1 按地址找到插入位置（第 511–514 行）

```c
// heap_4.c:511
for (pxIterator = &xStart;
     heapPROTECT_BLOCK_POINTER(pxIterator->pxNextFreeBlock) < pxBlockToInsert;
     pxIterator = heapPROTECT_BLOCK_POINTER(pxIterator->pxNextFreeBlock))
{
    // 空循环 — 纯粹遍历找到地址插入点
}
```

空闲链表按地址升序排列。遍历结束后，`pxBlockToInsert` 应插入在 `pxIterator` 和 `pxIterator->pxNextFreeBlock` 之间：

```
pxIterator → [pxIterator->pxNextFreeBlock]
                     ↑
              pxBlockToInsert 应插入此处
              (pxBlockToInsert 地址 > pxIterator 地址)
              (pxBlockToInsert 地址 < pxIterator->pxNextFreeBlock 地址)
```

### 9.2 指针合法性检查（第 516–519 行）

```c
if (pxIterator != &xStart) {
    heapVALIDATE_BLOCK_POINTER(pxIterator);
}
```

跳过 `xStart`（它不在堆内，不在 `ucHeap[]` 范围内）。

### 9.3 向前合并：与前一相邻空闲块合并（第 523–529 行）

```c
// heap_4.c:523–529
puc = (uint8_t *)pxIterator;

if ((puc + pxIterator->xBlockSize) == (uint8_t *)pxBlockToInsert) {
    // 前一块末尾 == 当前块起始 → 地址相邻，可以合并
    pxIterator->xBlockSize += pxBlockToInsert->xBlockSize;  // 吞并当前块
    pxBlockToInsert = pxIterator;  // 操作指针回退到合并后的大块
}
```

判断条件：**`前一空闲块地址 + 前一空闲块大小 == 释放块地址`**。

合并后 `pxBlockToInsert` 指向了合并后的大块（起始地址是 `pxIterator`），后续的向后合并检查将在大块上进行。

### 9.4 向后合并：与后一相邻空闲块合并（第 537–555 行）

```c
// heap_4.c:537–555
puc = (uint8_t *)pxBlockToInsert;

if ((puc + pxBlockToInsert->xBlockSize) == (uint8_t *)heapPROTECT_BLOCK_POINTER(pxIterator->pxNextFreeBlock)) {
    // 当前块末尾 == 后一空闲块起始 → 地址相邻，可以合并
    if (heapPROTECT_BLOCK_POINTER(pxIterator->pxNextFreeBlock) != pxEnd) {
        // 后一空闲块不是尾哨兵 → 正常合并
        pxBlockToInsert->xBlockSize += heapPROTECT_BLOCK_POINTER(pxIterator->pxNextFreeBlock)->xBlockSize;
        pxBlockToInsert->pxNextFreeBlock = heapPROTECT_BLOCK_POINTER(pxIterator->pxNextFreeBlock)->pxNextFreeBlock;
    } else {
        // 后一空闲块是尾哨兵 pxEnd → 不吞并哨兵，只接链表
        pxBlockToInsert->pxNextFreeBlock = heapPROTECT_BLOCK_POINTER(pxEnd);
    }
}
else {
    // 不相邻 → 不合并，只接链表指针
    pxBlockToInsert->pxNextFreeBlock = pxIterator->pxNextFreeBlock;
}
```

判断条件：**`释放块地址 + 释放块大小 == 后一空闲块地址`**。

特殊处理：`pxEnd` 是哨兵，`xBlockSize = 0`，不应被吞并。遇到 `pxEnd` 时只接链表不合并。

### 9.5 更新前驱的 next 指针（第 561–564 行）

```c
// heap_4.c:561–564
if (pxIterator != pxBlockToInsert) {
    // 没有发生向前合并 → pxIterator 和 pxBlockToInsert 是两个独立块
    // 更新前驱的 next 指向新插入的块
    pxIterator->pxNextFreeBlock = heapPROTECT_BLOCK_POINTER(pxBlockToInsert);
}
// 若发生了向前合并 → pxIterator == pxBlockToInsert（指针已在 9.3 中回退）
// 前驱就是合并块自身，next 已在 9.4 中正确设置，不需要再次赋值
```

### 9.6 四种合并场景示意

```
场景1: 前后都相邻（三块合一）
  释放前: ... → [Free A] → [Allocated X(释放中)] → [Free B] → ...
  释放后: ... → [    Free A + X + B    ] → ...

场景2: 仅前相邻（前两块合一）
  释放前: ... → [Free A] → [Allocated X(释放中)] → [Allocated Y] → ...
  释放后: ... → [  Free A + X  ] → [Allocated Y] → ...

场景3: 仅后相邻（后两块合一）
  释放前: ... → [Allocated Y] → [Allocated X(释放中)] → [Free B] → ...
  释放后: ... → [Allocated Y] → [  Free X + B  ] → ...

场景4: 前后都不相邻（不合并）
  释放前: ... → [Allocated Y] → [Allocated X(释放中)] → [Allocated Z] → ...
  释放后: ... → [Allocated Y] → [Free X] → [Allocated Z] → ...
```

## 10. `prvHeapInit()` — 堆初始化（第 456–501 行）

在 `pxEnd == NULL` 时由 `pvPortMalloc` 自动调用（第 225–228 行），只执行一次。

### 10.1 起始地址对齐（第 463–470 行）

```c
uxStartAddress = (portPOINTER_SIZE_TYPE)ucHeap;

if ((uxStartAddress & portBYTE_ALIGNMENT_MASK) != 0) {
    // ucHeap 地址不是 portBYTE_ALIGNMENT 的整数倍 → 向上取整
    uxStartAddress += (portBYTE_ALIGNMENT - 1);
    uxStartAddress &= ~((portPOINTER_SIZE_TYPE)portBYTE_ALIGNMENT_MASK);
    // 对齐损失的空间从总大小中扣除
    xTotalHeapSize -= (size_t)(uxStartAddress - (portPOINTER_SIZE_TYPE)ucHeap);
}
```

例如 `ucHeap` 在地址 `0x20000003`，`portBYTE_ALIGNMENT = 8`，则对齐到 `0x20000008`，损失 5 字节。

### 10.2 初始化堆保护 canary（第 472–476 行）

```c
#if (configENABLE_HEAP_PROTECTOR == 1)
{
    vApplicationGetRandomHeapCanary(&(xHeapCanary));
}
#endif
```

### 10.3 设置头哨兵 xStart（第 480–481 行）

```c
xStart.pxNextFreeBlock = (void *)heapPROTECT_BLOCK_POINTER(uxStartAddress);
xStart.xBlockSize = (size_t)0;
```

`xStart` 是一个全局变量（不在堆内），其 `pxNextFreeBlock` 指向堆的起始地址。

### 10.4 放置尾哨兵 pxEnd（第 485–490 行）

```c
// 计算堆末尾地址，预留一个块头大小的空间给 pxEnd
uxEndAddress = uxStartAddress + (portPOINTER_SIZE_TYPE)xTotalHeapSize;
uxEndAddress -= (portPOINTER_SIZE_TYPE)xHeapStructSize;          // 预留 pxEnd 占用空间
uxEndAddress &= ~((portPOINTER_SIZE_TYPE)portBYTE_ALIGNMENT_MASK); // 对齐
pxEnd = (BlockLink_t *)uxEndAddress;
pxEnd->xBlockSize = 0;
pxEnd->pxNextFreeBlock = heapPROTECT_BLOCK_POINTER(NULL);
```

`pxEnd` 放在堆空间的**最末尾**，占用 `xHeapStructSize` 字节。其 `xBlockSize = 0`，`pxNextFreeBlock = NULL`。

### 10.5 创建初始空闲块（第 494–500 行）

```c
pxFirstFreeBlock = (BlockLink_t *)uxStartAddress;
pxFirstFreeBlock->xBlockSize = (size_t)(uxEndAddress - (portPOINTER_SIZE_TYPE)pxFirstFreeBlock);
pxFirstFreeBlock->pxNextFreeBlock = heapPROTECT_BLOCK_POINTER(pxEnd);

xMinimumEverFreeBytesRemaining = pxFirstFreeBlock->xBlockSize;
xFreeBytesRemaining = pxFirstFreeBlock->xBlockSize;
```

从堆起始到 `pxEnd` 之间是一个完整的空闲块，大小 = `pxEnd 地址 - 堆起始地址`。

### 10.6 初始化后的内存布局

```
xStart (全局变量)
  .pxNextFreeBlock ──→
  .xBlockSize = 0
                       ucHeap[]
                  ┌───────────────────────────────────────────────────┬──────────────┐
                  │ BlockLink_t          用户数据区（全部空闲）          │   pxEnd      │
                  │ .xBlockSize = 全部   可分配空间                     │ .xBlockSize=0│
                  │ .pxNextFreeBlock ──────────────────────────────────→ .next = NULL │
                  └───────────────────────────────────────────────────┴──────────────┘
                  ↑ uxStartAddress (对齐后)                             ↑ uxEndAddress

空闲链表: xStart → [初始空闲块] → pxEnd → NULL
```

## 11. 统计查询 API（第 413–434 行）

### 11.1 `xPortGetFreeHeapSize()`（第 413–416 行）

```c
size_t xPortGetFreeHeapSize(void) {
    return xFreeBytesRemaining;
}
```

直接返回全局变量，无锁。返回的是**所有空闲块大小之和**（包含块头开销），不是最大连续空闲块。

### 11.2 `xPortGetMinimumEverFreeHeapSize()`（第 419–422 行）

```c
size_t xPortGetMinimumEverFreeHeapSize(void) {
    return xMinimumEverFreeBytesRemaining;
}
```

返回历史最小空闲值。如果此值接近 0，说明堆曾接近耗尽。

### 11.3 `xPortResetHeapMinimumEverFreeHeapSize()`（第 425–428 行）

```c
void xPortResetHeapMinimumEverFreeHeapSize(void) {
    xMinimumEverFreeBytesRemaining = xFreeBytesRemaining;
}
```

重置高水位线为当前值，用于分阶段测量（例如：启动阶段峰值 vs 稳态运行峰值）。

### 11.4 `vPortInitialiseBlocks()`（第 431–434 行）

```c
void vPortInitialiseBlocks(void) {
    /* This just exists to keep the linker quiet. */
}
```

空函数，仅为了兼容旧的链接要求。heap_4 使用 `pxEnd == NULL` 判断初始化状态，不需要此函数。

## 12. `pvPortCalloc()` — 分配并清零（第 437–453 行）

```c
void *pvPortCalloc(size_t xNum, size_t xSize) {
    void *pv = NULL;

    if (heapMULTIPLY_WILL_OVERFLOW(xNum, xSize) == 0) {  // 乘法溢出检查
        pv = pvPortMalloc(xNum * xSize);                  // 调用 malloc
        if (pv != NULL) {
            (void)memset(pv, 0, xNum * xSize);            // 清零
        }
    }
    return pv;
}
```

等价于标准 C 的 `calloc`：分配 `xNum × xSize` 字节并全部置零。先做乘法溢出检查，再委托给 `pvPortMalloc`。

## 13. `vPortGetHeapStats()` — 详细堆统计（第 572–621 行）

```c
void vPortGetHeapStats(HeapStats_t *pxHeapStats) {
    BlockLink_t *pxBlock;
    size_t xBlocks = 0, xMaxSize = 0, xMinSize = SIZE_MAX;

    // ── 第一阶段：冻结调度器，遍历空闲链表 ──
    vTaskSuspendAll();                                    // heap_4.c:577
    {
        pxBlock = heapPROTECT_BLOCK_POINTER(xStart.pxNextFreeBlock);
        if (pxBlock != NULL) {                            // 堆未初始化时为 NULL
            while (pxBlock != pxEnd) {                    // 遍历直到尾哨兵
                xBlocks++;                                 // 空闲块计数
                if (pxBlock->xBlockSize > xMaxSize)        // 最大块
                    xMaxSize = pxBlock->xBlockSize;
                if (pxBlock->xBlockSize < xMinSize)        // 最小块
                    xMinSize = pxBlock->xBlockSize;
                pxBlock = heapPROTECT_BLOCK_POINTER(pxBlock->pxNextFreeBlock);
            }
        }
    }
    (void)xTaskResumeAll();                               // heap_4.c:607

    // 遍历结果写入（不需要锁，这些是局部变量）
    pxHeapStats->xSizeOfLargestFreeBlockInBytes = xMaxSize;
    pxHeapStats->xSizeOfSmallestFreeBlockInBytes = xMinSize;
    pxHeapStats->xNumberOfFreeBlocks = xBlocks;

    // ── 第二阶段：关中断，原子读取全局计数器 ──
    taskENTER_CRITICAL();                                 // heap_4.c:613
    {
        pxHeapStats->xAvailableHeapSpaceInBytes = xFreeBytesRemaining;
        pxHeapStats->xNumberOfSuccessfulAllocations = xNumberOfSuccessfulAllocations;
        pxHeapStats->xNumberOfSuccessfulFrees = xNumberOfSuccessfulFrees;
        pxHeapStats->xMinimumEverFreeBytesRemaining = xMinimumEverFreeBytesRemaining;
    }
    taskEXIT_CRITICAL();                                  // heap_4.c:620
}
```

**两级锁设计：**
- `vTaskSuspendAll`（冻结调度器）用于遍历链表——耗时较长但中断仍开启。
- `taskENTER_CRITICAL`（关中断）用于读取几个整型变量——极短临界区，保证快照一致性。

分两级的原因是：链表遍历可能耗时较长，长时间关中断会影响实时性；而全局计数器可能被 ISR 之外的上下文修改（虽然实际上 heap_4 的计数器只在任务上下文更新），用关中断保证原子读取。

## 14. `vPortHeapResetState()` — 状态重置（第 629–637 行）

```c
void vPortHeapResetState(void) {
    pxEnd = NULL;                                       // 触发下次 malloc 重新初始化
    xFreeBytesRemaining = (size_t)0U;
    xMinimumEverFreeBytesRemaining = (size_t)0U;
    xNumberOfSuccessfulAllocations = (size_t)0U;
    xNumberOfSuccessfulFrees = (size_t)0U;
}
```

将所有全局状态变量重置为零。下次调用 `pvPortMalloc` 时，因 `pxEnd == NULL` 会重新执行 `prvHeapInit`。

**注意：** 不会清零堆内存内容，也不会释放已分配的块。必须在重启调度器之前调用，否则已分配的指针全部失效。

## 15. 线程安全总结

| 操作 | 锁机制 | 代码位置 |
|------|--------|----------|
| `pvPortMalloc` 全过程 | `vTaskSuspendAll()` / `xTaskResumeAll()` | 第 221, 334 行 |
| `vPortFree` 插入链表 | `vTaskSuspendAll()` / `xTaskResumeAll()` | 第 390, 398 行 |
| `vPortGetHeapStats` 遍历链表 | `vTaskSuspendAll()` / `xTaskResumeAll()` | 第 577, 607 行 |
| `vPortGetHeapStats` 读计数器 | `taskENTER_CRITICAL()` / `taskEXIT_CRITICAL()` | 第 613, 620 行 |

`vTaskSuspendAll` 禁止任务切换但不禁中断，开销较小，适合链表遍历等耗时操作。`taskENTER_CRITICAL` 关中断，适合极短的临界区。

## 16. 完整生命周期示例

假设 `configTOTAL_HEAP_SIZE = 200`，`xHeapStructSize = 8`，`portBYTE_ALIGNMENT = 8`。

```
初始状态（prvHeapInit 后）:
  xStart ──→ [BlockLink_t: size=184, next→pxEnd] → [pxEnd: size=0, next=NULL]
              ↑ 地址 0                              ↑ 地址 184
              ├──────── 184 字节空闲 ────────┤

pvPortMalloc(64):
  调整大小: 64 + 8(块头) = 72, 已对齐, xWantedSize = 72
  First-Fit: 空闲块 184 >= 72 → 命中
  分割: 72 < 184 - 72 = 112 > 16(heapMINIMUM) → 分割
  结果:
  xStart ──→ [Free: size=112, next→pxEnd] → [pxEnd]
                ↓ 偏移 0 处（已分配，不在链表中）
                [Allocated: size=72(MSB=1), next=NULL]
                ├── BlockLink_t(8B) ──→ pvReturn 指向此处
                └── 用户数据(64B)

pvPortMalloc(32):
  调整大小: 32 + 8 = 40, xWantedSize = 40
  First-Fit: 空闲块 112 >= 40 → 命中
  分割: 40 < 112 - 40 = 72 > 16 → 分割
  结果:
  xStart → [Free:72] → [pxEnd]
              ↓ 偏移 72
              [Alloc:40]
              ↓ 偏移 112
              [Alloc:72]

vPortFree(第一个 64B 块, pv 指向偏移 8):
  定位块头: pv - 8 = 偏移 0
  MSB 清零, 插入空闲链表
  prvInsertBlockIntoFreeList:
    找插入位置: 在 xStart 之后
    向前合并: xStart.xBlockSize=0, 0+0 != 偏移0 → 不合并
    向后合并: 偏移0+40=偏移40 != 偏移72(下一个空闲块) → 不合并
  结果:
  xStart → [Free:40] → [Free:72] → [pxEnd]
              ↓ 偏移 72
              [Alloc:72]

vPortFree(第二个 32B 块, pv 指向偏移 80):
  定位块头: pv - 8 = 偏移 72
  MSB 清零, 插入空闲链表
  prvInsertBlockIntoFreeList:
    找插入位置: 在 Free:40(偏移0) 之后, Free:72(偏移72) 之前
    向前合并: 偏移0+40=40 != 偏移72 → 不合并
    向后合并: 偏移72+40=112 == 偏移112(Free:72) → 合并! → [Free:112]
    最终: Free:40(偏移0) → Free:112(偏移72) → pxEnd
  结果:
  xStart → [Free:40] → [Free:112] → [pxEnd]

（此时偏移 0 的 40 字节和偏移 72 的 112 字节之间还有偏移 40-72 的区域
  是已分配但已释放的状态... 实际上此处只有两个已分配块都释放了，演示的是
  释放第二个块时向后合并的场景）

最终完全释放后:
  xStart → [Free:184] → [pxEnd]
  （与初始状态完全一致）
```

## 17. 算法复杂度

| 操作 | 时间复杂度 | 说明 |
|------|------------|------|
| `pvPortMalloc` | O(n) | n = 空闲块数量，需遍历链表找 First-Fit |
| `vPortFree` | O(n) | 需遍历链表找插入位置 + 检查合并 |
| `prvInsertBlockIntoFreeList` | O(n) | 遍历找地址插入点 |
| `vPortGetHeapStats` | O(n) | 遍历整个空闲链表统计 |
| `xPortGetFreeHeapSize` | O(1) | 直接返回全局变量 |
| `xPortGetMinimumEverFreeHeapSize` | O(1) | 直接返回全局变量 |

n = 空闲块数量（最坏情况 = 碎片化严重的堆）。对嵌入式系统（堆通常 1–64 KB，碎片块通常几十个），O(n) 完全可接受。
