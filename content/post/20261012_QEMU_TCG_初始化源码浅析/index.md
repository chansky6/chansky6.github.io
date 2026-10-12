---
title: QEMU TCG 代码缓存管理（Code Buffer）分析
description: 分析 QEMU TCG 代码缓存(Code Buffer)的设计与实现：内存布局、初始化流程与分配/清空的完整生命周期
date: 2026-10-12T14:30:00+08:00
image: https://cn.bing.com/th?id=OHR.WhiteHouse_EN-US6161912809_UHD.jpg&rf=LaDigue_UHD.jpg&pid=hp&w=1920&h=1080&rs=1&c=4
math: true
license: 
comments: true
draft: false
categories:
    - 虚拟化
tags:
    - QEMU
    - TCG
    - 源码分析
---
本文基于 *QEMU v11.1.1* 分析，以 *System Mode* 为主线 

## TCG 翻译流程
回顾 TCG 的运行主线，可以概括为：先根据当前 Guest PC 查找 TB 缓存；若命中，则直接执行已有 TB；若未命中，则触发翻译，生成新的 TB，写入缓存，再开始执行。其逻辑如下：

```
                              +---------------+                    
       +----------------------| Do something  |-------------------+
       |                      +---------------+                   |
       v                                                          |
+--------------+       +----------------+ Y       +---------+     |
|   Guest PC   +------>| Check TB Cache +-------->| Exec TB +-----+
+--------------+       +------+---------+         +---------+      
                              | N                      ^           
                              v                        |           
                        +-------------+                |           
                        | translation |                |           
                        +-----+-------+                |           
                              v                        |           
                       +------------------+             |           
                       |*Save TB to Cache +-------------+           
                       +------------------+
```

## Code Buffer的内存布局
从 TCG 翻译流程可知，翻译结果不是一次性使用，而是会被缓存起来。 这块保存翻译结果的区域，就是代码缓存。

可以把代码缓存理解成一块专门分配出来的可执行内存。QEMU 在这里存放两类内容：一类是和当前 Host 相关的公共代码，例如 prologue 和 epilogue；另一类是各个 TB 自身对应的宿主机机器码。当前 TB 的入口地址会记录在相应字段中，之后执行时，QEMU 就可以直接跳到这段机器码上运行，而不必重新翻译。
```
code_buffer = mmap(min(max(physmem/8, 1MiB), 1GiB))                                               
|                                             TCGContext.code_gen_ptr  
v                                              v                   
+-----------+----------+-------------+---------+------------------+
|           |          |             |         |                  |
|  prologue | epilogue |  TB.struct  | TB.code |     ...          |
|           |          |             |         |                  |
+-----------+----------+-------------+---------+------------------+
^           ^                        ^                             
|           |                        |                             
|           tcg_code_gen_epilogue    |                             
|                                    tb.tc.ptr                     
tcg_qemu_tb_exec
```

## 初始化的整体流程
```
main → qemu_init
 └─ configure_accelerators()                    system/vl.c:3805
     └─ accel_init_machine()                    accel/accel-system.c:39
         └─ tcg_init_machine()                  accel/tcg/tcg-all.c:104
             ├─ MTTCG/RR 决策 + page_init() + tb_htable_init()
             ├─ tcg_init()                      tcg/tcg.c:1817
             │   ├─ tcg_context_init()          tcg/tcg.c:1762
             │   └─ tcg_region_init()           tcg/region.c:757  ← code buffer 初始化主逻辑
             │       ├─ 定尺寸: clamp(physmem/8, 1MiB, 后端上限)
             │       ├─ alloc_code_gen_buffer()          region.c:694
             │       │   └─ alloc_code_gen_buffer_anon() region.c:563
             │       │       └─ mmap(NULL, size, PROT_NONE,      ← 整块占位
             │       │              MAP_PRIVATE|MAP_ANONYMOUS)   region.c:568
             │       ├─ 切 region + guard 页 + 逐片 mprotect(RW)  region.c:795/829
             │       └─ region 0 发给 tcg_init_ctx               region.c:868
             └─ tcg_prologue_init()             tcg/tcg.c:1853  ← 序言写入 region 0 头部
```

## Code Buffer 的生命周期
Code Buffer 的生命周期特点：**一次分配、循环重置、永不释放**。
```
   ┌─► bump 分配 TB, code_gen_ptr 前进 (tcg_tb_alloc)
   │      │ 越过 highwater
   │      ▼
   │   tcg_region_alloc 换下一片 (region.c:374)
   │      │ n 片全部耗尽
   │      ▼
   │   tb_flush → tcg_region_reset_all (region.c:427)
   │      清空全部索引 (哈希表/jmp cache/region 树)
   │      current=0, region 重新逐个发给各 context
   │      prologue 之后的内容全部作废, 从 after_prologue 重新 bump
   └──────┘
```