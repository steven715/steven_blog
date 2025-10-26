# CPU 知識

## CPU 組成

一個CPU在硬體層面是包含以下組件

1. Core 一個核心有的 Component
   - L1 Cache (per core)
   - L2 Cache (per core)
   - Registry File: 所有CPU執行運算用的暫存器集合(RAX, RBX, RSP, RIP, ...)
   - Excution Unit/ ALU: 作整數與浮點數計算
   - FPU, SIMD Unit: 做 SIMD / AVX / FMA 運算
   - Instructment Pipeline: CPU 解碼、執行、撰寫結果的流水線
   - Branch Predictor: 預測分支跳躍路徑，預先拉指令

2. 多 Core 共享
   - L3 Cache (shared)
   - Memory Controller + DRAM

## Linux Scheduler

在linxu中，**Scheduler**是一個在每個CPU core上持續執行的邏輯機制，負責

1. 挑選下一個要執行的 Task(進程或線程)
2. 保存與恢復 Context (Context Switch)
3. 管理每個core上的runnable thread的`runqueue`

## Thread Context

Thread Context 會含有以下狀態

1. CPU Registers: 儲存邏輯與記憶體位址
2. Stack Pointer (RSP): 支援函式呼叫與區域變數
3. Program Counter (RIP): 決定thread續跑從哪開始
4. Segment Registers: 用來保存 TLS (Thread Local Storage)
5. FPU/SIMD Registers: SSE、AVX 寄存器
6. Page Table Base Register: 指向目前記憶體空間的page table，**換Process時會更新**
7. Kernel Stack/Uesr Stack: 每個thread有一個在kernel模式下專屬的Stack，為了處理中斷與系統呼叫
8. Task_struct/Thread_info: kernel使用的結構體，儲存排程狀態，Linux會根據這些作調度

這些 Context 平時都存在kernel中，當context switch發生時，由kernel負責儲存與恢復。

## 一個Context Switch 的處理過程

Linux中對於Context Switch的發生會有以下幾步
1. CPU發出中斷 => 切進 kernel mode => 觸發 scheduler
2. kernel 儲存目前thread的context到其task_struct
3. kernel 從runqueue中選出下一個thread (CFS)
4. 將新的thread的context載入CPU
5. 切出kernel mode，跳到新的RIP，開始執行新的thread

以上過程大概會花費幾百到上千的cycle (CPU 指令運行一次的時間單位)。

## CPU Scheduler - CFS

預設的排程器是 CFS (Completely Fair Scheduler)，他有一個核心概念是 `vruntime` ，這個vruntime會記錄每個thread運行的時間，CFS每次會挑選 vruntime 最小的線程出來執行。

CFS 內部的 `vruntime`是透過紅黑樹來管理的，key是 vruntime的時間，value則是thread的狀態資訊。
