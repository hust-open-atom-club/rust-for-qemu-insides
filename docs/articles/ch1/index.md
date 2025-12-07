# 一、QEMU 中的 Rust 实现

Rust for QEMU 利用 FFI 思想，通过 bingen 将关键 C 接口导出并封装成符合 Rust 安全标准的接口，用于 Rust 建模硬件设备。

本章节我们将介绍一些关键组件的 Rust 实现，比如 QOM、MemoryRegion、SysBus 等。