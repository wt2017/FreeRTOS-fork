# 链接器脚本（.ld 文件）分析报告

## 概述

本文档基于对 FreeRTOS 项目中多个 .ld 文件的深入分析，总结了链接器脚本在编译可执行文件中的作用及编译过程中的使用细节。

## .ld 文件的作用

.ld 文件是**链接器脚本**（Linker Script），在编译可执行文件时起着至关重要的作用：

### 1. **内存布局定义**
- 定义目标硬件的内存映射（Memory Map）
- 指定不同内存区域的起始地址和大小
- 例如：FLASH、RAM、SRAM、DDR 等内存区域

### 2. **段（Section）分配**
- 控制编译器生成的各个段（.text、.data、.bss、.rodata 等）在内存中的位置
- 指定哪些代码/数据应该放在哪个内存区域

### 3. **入口点指定**
- 定义程序的入口点（通常是 Reset_Handler）
- 确保系统复位后能正确跳转到启动代码

### 4. **堆栈配置**
- 设置堆（heap）和栈（stack）的大小和位置
- 确保有足够的内存供动态分配和函数调用使用

### 5. **符号定义**
- 定义全局符号供启动代码和应用程序使用
- 例如：_estack（栈顶）、_etext（代码结束）、_sdata（数据开始）等

## 编译过程中的使用细节

### 1. **编译阶段**
- 在编译的最后阶段（链接阶段）使用
- 通过 GCC 的 `-T` 选项指定链接器脚本
- 例如：`arm-none-eabi-gcc -T ./mps2_m3.ld ...`

### 2. **Makefile 中的配置**
```makefile
LDFLAGS = -T ./mps2_m3.ld
LDFLAGS += -Xlinker -Map=$(OUTPUT_DIR)/RTOSDemo.map
$(LD) $(CFLAGS) $(LDFLAGS) $(OBJS_OUTPUT) -o $(IMAGE)
```

### 3. **链接器的工作流程**
1. **收集输入文件**：所有目标文件（.o）和库文件
2. **解析 .ld 文件**：读取内存布局和段分配规则
3. **符号解析**：解决所有未定义的符号引用
4. **段合并**：将相同类型的段合并到一起
5. **地址分配**：根据 .ld 文件为每个段分配具体地址
6. **重定位**：修正所有地址相关的引用
7. **生成输出**：生成最终的可执行文件

### 4. **关键语法元素**
- **MEMORY**：定义内存区域（名称、属性、起始地址、大小）
- **SECTIONS**：定义段的分配规则
- **ENTRY**：指定程序入口点
- **PROVIDE**：定义符号供外部使用
- **KEEP**：防止链接器优化掉特定段
- **ALIGN**：对齐要求
- **> region**：指定段应该放在哪个内存区域
- **AT> lma**：指定加载地址（Load Memory Address）

### 5. **不同架构的差异**
- **ARM Cortex-M**：通常有 FLASH 和 RAM 区域，需要处理 .isr_vector（中断向量表）
- **RISC-V**：可能有多个 ITIM（指令紧耦合内存）、DTIM（数据紧耦合内存）区域
- **多核系统**：需要为每个核心分配独立的栈和内存区域

## 实际应用示例

### 示例 1：STM32L010RB（ARM Cortex-M0+）
```ld
/* Memories definition */
MEMORY
{
  RAM    (xrw)    : ORIGIN = 0x20000000,   LENGTH = 20K
  FLASH  (rx)     : ORIGIN = 0x8000000,    LENGTH = 128K
}

/* Entry Point */
ENTRY(Reset_Handler)

/* Stack and heap sizes */
_Min_Heap_Size = 0x200;
_Min_Stack_Size = 0x400;
```

### 示例 2：RISC-V 多核系统（PolarFire SoC）
```ld
MEMORY
{
    envm (rx)      : ORIGIN = 0x20220100, LENGTH = 128k - 0x100
    dtim (rwx)     : ORIGIN = 0x01000000, LENGTH = 7k
    e51_itim (rwx) : ORIGIN = 0x01800000, LENGTH = 28k
    u54_1_itim (rwx) : ORIGIN = 0x01808000, LENGTH = 28k
    /* ... 其他核心的 ITIM ... */
}

/* 每个核心的栈大小 */
STACK_SIZE_E51_APPLICATION = 8k;
STACK_SIZE_U54_1_APPLICATION = 8k;
```

### 示例 3：CSKY 架构
```ld
MEMORY
{
    I-SRAM : ORIGIN = 0x0,        LENGTH = 0x40000   /* 指令 SRAM 256KB */
    D-SRAM : ORIGIN = 0x20000000, LENGTH = 0xc0000   /* 数据 SRAM 768KB */
    O-SRAM : ORIGIN = 0x50000000, LENGTH = 0x800000  /* 片外 SRAM 8MB */
}

REGION_ALIAS("REGION_TEXT",    I-SRAM);
REGION_ALIAS("REGION_RODATA",  I-SRAM);
REGION_ALIAS("REGION_DATA",    D-SRAM);
REGION_ALIAS("REGION_BSS",     D-SRAM);
```

## 链接器脚本的关键部分详解

### 1. **MEMORY 命令**
定义目标系统的内存区域，每个区域有：
- **名称**：如 FLASH、RAM、SRAM 等
- **属性**：r（可读）、w（可写）、x（可执行）
- **起始地址**（ORIGIN）：内存区域的开始地址
- **长度**（LENGTH）：内存区域的大小

### 2. **SECTIONS 命令**
定义如何将输入段映射到输出段：
- **.text**：程序代码段
- **.data**：已初始化的全局/静态变量
- **.bss**：未初始化的全局/静态变量
- **.rodata**：只读数据（常量、字符串等）
- **.isr_vector**：中断向量表（ARM 特有）

### 3. **特殊处理**
- **KEEP()**：确保特定段不被链接器优化掉（如启动代码）
- **ALIGN()**：对齐要求，确保地址符合硬件要求
- **LOADADDR()**：获取段的加载地址
- **PROVIDE()**：定义符号，可在 C 代码中引用

## 编译命令示例

```bash
# 使用链接器脚本编译
arm-none-eabi-gcc \
  -mcpu=cortex-m3 \
  -T mps2_m3.ld \
  -Xlinker -Map=output.map \
  -Xlinker --gc-sections \
  -nostartfiles \
  -specs=nano.specs \
  source1.o source2.o \
  -o firmware.elf

# 生成二进制文件
arm-none-eabi-objcopy -O binary firmware.elf firmware.bin
```

## 调试和验证

### 1. **生成映射文件**
使用 `-Xlinker -Map=output.map` 选项生成内存映射文件，包含：
- 所有符号的地址
- 各段的大小和位置
- 内存使用统计

### 2. **查看段信息**
```bash
arm-none-eabi-size firmware.elf
arm-none-eabi-objdump -h firmware.elf
```

### 3. **常见问题排查**
- **内存不足**：检查段大小是否超过内存区域长度
- **对齐错误**：确保关键数据（如向量表）正确对齐
- **符号未定义**：检查 PROVIDE 定义的符号是否正确引用

## FreeRTOS 特殊考虑

### 1. 堆数组的放置
FreeRTOS 支持通过 `configAPPLICATION_ALLOCATED_HEAP 1` 让应用程序自定义 `ucHeap[]` 数组的位置：

```ld
/* 将 FreeRTOS 堆放入外部 SDRAM */
__attribute__((section(".sdram_heap")))
static uint8_t ucHeap[configTOTAL_HEAP_SIZE];

/* 链接器脚本中 */
.sdram_heap (NOLOAD) : {
    *(.sdram_heap)
} > SDRAM
```

### 2. 堆栈方向
FreeRTOS 通过 `portSTACK_GROWTH` 宏定义栈增长方向：
- **ARM Cortex-M:** `portSTACK_GROWTH = -1`（栈向下增长）
- **RISC-V / ARM Cortex-A:** `portSTACK_GROWTH = 0`（栈向上增长）
- 链接器脚本必须据此在内存区域适当位置设置栈顶（如 ARM 将 `_estack` 定义在 RAM 最高地址）

### 3. MPU 对齐要求
使用 MPU wrappers 时，任务的内存区域必须满足 MPU 对齐要求：
- Cortex-M3/M4 MPU：区域必须是 32 字节对齐
- Cortex-M33/M55 MPU：区域需按 32 字节对齐，大小需为 2 的幂
- 链接器脚本需确保 `.text`、`.data`、`.bss` 段按 MPU 粒度对齐

### 4. ARM 中断向量表对齐
Cortex-M 的中断向量表必须 **256 字节对齐**（VTOR 寄存器的低 9 位为 0）：
```ld
.isr_vector :
{
    KEEP(*(.isr_vector))
} > FLASH AT > FLASH
/* 确保 .isr_vector 大小不超过 256 字节边界 */
```

### 5. heap_5 多区域内存布局
使用 heap_5 时，需要在链接器脚本中为非连续内存区域定义符号，供 `vPortDefineHeapRegions()` 使用：
```ld
_sram_start = ORIGIN(SRAM);
_sram_size   = LENGTH(SRAM);
_sdram_start = ORIGIN(SDRAM);
_sdram_size  = LENGTH(SDRAM);
```
应用程序通过 `HeapRegion_t regions[] = {{&_sram_start, _sram_size}, {&_sdram_start, _sdram_size}, {NULL, 0}};` 在运行时注册。

## 重要性总结

.ld 文件是嵌入式系统开发中的关键文件，它：

1. **确保代码正确加载**：将启动代码放在正确的复位向量位置
2. **优化内存使用**：合理分配有限的内存资源
3. **支持硬件特性**：利用不同内存区域的特性（如快速 SRAM、非易失 FLASH）
4. **实现多核支持**：为多核系统分配独立的内存空间
5. **提供调试信息**：通过生成的 .map 文件了解内存布局
6. **跨平台兼容**：适应不同架构和内存配置的需求

## 最佳实践

1. **模块化设计**：将通用配置放在基础 .ld 文件中，特定配置通过 INCLUDE 引入
2. **版本控制**：将 .ld 文件与硬件设计文档同步更新
3. **文档注释**：在 .ld 文件中详细注释每个区域和段的用途
4. **验证测试**：编译后验证内存使用是否在硬件限制内
5. **向后兼容**：保持旧项目的 .ld 文件结构，便于维护和升级

---

*本文档基于对 FreeRTOS 项目中 66 个不同架构的 .ld 文件分析编写，涵盖了 ARM Cortex-M、RISC-V、CSKY 等多种架构的链接器脚本实现。*