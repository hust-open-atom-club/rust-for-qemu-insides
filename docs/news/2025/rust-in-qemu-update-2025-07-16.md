# 2025 年 7 月 QEMU 中的 Rust 更新

!!! note "主要贡献者"
    
    - 原文链接：[Rust in QEMU update, July 2025](https://lore.kernel.org/qemu-rust/c2342e56-b6d8-4115-8318-d8047a46f1ad@redhat.com/)
    - 翻译：[@zevorn](https://github.com/zevorn)

上游发布 Rust for QEMU 开发进展更新的邮件，一般是每三个月一次。但这次更新稍早一些，主要是因为 QEMU 即将到来的版本冻结期，而且许多子项目已经到了需要更新的阶段。

7 月份的进展很大，目前用 Rust 编写的设备源码几乎完全安全了；日志记录和 tracing 功能已经有了初步开发成果；上游开发者开始着手研究 C 与 Rust 数据结构的互操作性。

当前的障碍主要在如何更好的处理 Rust 标准库的静态链接；以及需要正式确定集成 QAPI/QMP 的设计；同时还需要为关键安全特性升级到 Rust 1.83。

下一步计划，需要完成对 tracing 和 DMA 的支持；另外会探索用 Rust 编写的新设备，并报告 bindings 中可能存在的的漏洞。

## QEMU 10.1 中的状态

关于平台可用性，从 10.0 到 10.1 的唯一变化是，最低支持的 Rust 版本现在是 1.77，而不是 1.63。这意味着，仅在 mips64el 架构上，无法在启用 Rust 的情况下使用 Debian bookworm 构建 QEMU。其他架构在 bookworm 上仍然可以构建。

随着 Meson 更新到 1.8 版本，对 clippy 和 rustdoc 的支持从 QEMU 的 meson.build 文件转移到了 Meson 自身。此次更新还使得可以同时使用 `--enable-modules` 和 `--enable-rust`。

新增了两个依赖项：anyhow，一个常用的 crate，提供通用的错误实现；以及 foreign，Paolo 编写的一个 crate，用于提供 C 和 Rust 数据类型之间的转换工具。两者都用于从 Rust 代码中访问 Error*。

一旦 Rust 版本从 1.77 再次升级到 1.83，Rust 设备基本上将不再需要不安全的 Rust 代码。这意味着用 Rust 编写新设备将带来实际好处。如果有任何用 Rust 编写且没有对应的 C 版本的设备被贡献出来，那么将“启用 Rust”和“启用所有用 Rust 编写的设备”分开可能是值得的。这样，在编译器版本不够新的平台上，pl011 和 HPET 设备的 C 版本仍然可以使用。

## 构建系统

Rust 仍未默认启用。主要原因是 Rust 的静态库也会静态链接到 Rust 标准库，这会导致生成的可执行文件体积膨胀（也会让发行版对 QEMU 不满）。一个待处理的 [Meson 拉取请求][1] 将解决这个问题，前提是 system/main.c 被重写或以 Rust 包装。

正如在早期更新中提到的，在进行 QEMU 相关工作的同时，Paolo 也在研究如何改进 Meson 对 Cargo 和 Rust 的支持。过去几个月虽然没有太多新消息，但计划是在其 Cargo 支持中添加对交叉编译和 [lints] 部分的支持。因此，Paolo 提出了一项关于 meson.build 文件与 Cargo 包更紧密集成的[提案][2]。

Stefano Garzarella 正考虑将 buildigvm 工具合并到 QEMU 的 contrib/ 目录中。buildigvm 是用 Rust 编写的，但它有相当多的依赖项，这使得（目前）通过 Meson 来驱动其构建并不现实。虽然 Stefano 目前打算调用 Cargo 来构建 buildigvm，但这种情况预计不会一直持续下去；buildigvm 将为 Meson 的更改提供进一步的测试平台。

## 设备方面的改进

10.1 版本的新特性包括日志支持以及 Error* 绑定，这样 `realize()` 实现就可能失败。

随着最近用于属性支持的过程宏的合并，除了 QOM 的 instance_init 方法外，设备中基本上不再有不安全的代码。不过，合并对 qdev 和 VMState 的改进将需要把最低支持的 Rust 版本提升到 1.83.0。

pl011 和 HPET 剩下的缺失功能是跟踪。一名学生正在从事这方面的工作，他已经实现了 simpletrace 后端，接下来会处理 syslog 和 log。dtrace 后端的计划是使用 [probe crate][3]，可能还会对其进行扩展以支持 ust。

Zhao 正在研究在 QEMU 中使用 vm-memory crate。这使得设备能够执行 DMA（直接内存访问），包括 HPET（高精度事件定时器）。

## 清理工作

随着 Rust bindings 以及首批几个设备初具雏形，是时候清理一些在过去一年中变得有些杂乱无章的东西了。

一项重要的变更（这将让 Kevin 能更轻松地继续他在 block layer bindings 方面的工作）是将 qemu-api crate 拆分为多个部分，大致与 C 代码中使用的 `declare_dependency()` 语句相匹配。

此外，pl011 和 HPET 是独立开发的，在一些可以统一的用法上有时存在差异。主要有两点：

- 文件命名（PL011 的主要源文件是 `device.rs`，而 HPET 的是 `hpet.rs`）以及设备代码与寄存器模块的拆分方式

- 调整 HPET 以使用 bilge crate 和 / 或 bits 宏

这些对于正在学习 Rust 的人来说可能是不错的任务。

## 下一步计划

目前最优先的缺失功能 —— 跟踪（tracing）正在开发中；DMA（直接内存访问）也是如此。

下一步可能是更好地集成 QAPI/QMP（Marc-André Lureau 在 2022 年就已有相关原型）以及支持 QOM 属性。

上个月，QEMU 讨论了是继续采用 Marc-André 的 C 与 Rust 表示间的转换方法，为 Rust 数据结构实现 QEMU 的 Visitor 接口；还是专注于 QObject 与 Rust 之间的转换，并使用 serde 进行代码生成。Markus 倾向于后者（尽管必须说明的是，目前还没有相关代码，而且可能存在未预见的障碍）。

最后，Paolo 将在一段时间内不再积极为 QEMU 编写 Rust 代码，但 Paolo 会参与代码审查，并继续关注上游 Meson 对该语言的支持情况。

[1]: https://github.com/mesonbuild/meson/pull/14224
[2]: https://github.com/mesonbuild/meson/issues/14639
[3]: https://github.com/cuviper/probe-rs