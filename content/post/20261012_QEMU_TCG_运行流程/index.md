---
title: QEMU TCG 二进制动态翻译原理和运行流程
description: 深入分析 QEMU TCG 二进制动态翻译（Binary Translation）的核心原理与完整运行流程，涵盖 Guest 指令解码、中间代码（TCG IR）生成、优化与 Host 机器码翻译，以及翻译块（TB）的执行与缓存机制
date: 2026-10-12T19:30:00+08:00
math: true
license: 
comments: true
categories:
    - 虚拟化
tags:
    - QEMU
    - TCG
    - 源码分析
---

QEMU 支持多种加速器（accel），分为两种：指令模拟技术（如 TCG）、虚拟化技术（如 KVM、HVF 等）。QEMU有两种运行模式：System Mode 模拟整个机器（CPU、内存和虚拟设备）以运行客户机操作系统；而 User Mode 只模拟 CPU 并翻译系统调用，以支持进程级别的用户态程序。

> 注意术语：Target：在 TCG 子系统中，Target 指的是 Host 主机，代指翻译的目标架构；而在 QEMU 的其他部分中，Target 为 Guest 的同义词，即指被模拟的架构。

```
+---------------+      +----------------+      +---------------+
|               |      |                |      |               |
| Source binary | ---> |    QEMU IR     | ---> | Target binary |
|     code      |      |                |      |     code      |
|               |      |                |      |               |
+---------------+      +----------------+      +---------------+
      Guest                                          Host
```
