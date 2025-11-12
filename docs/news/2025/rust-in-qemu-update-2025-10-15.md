# 2025 年 10 月 QEMU 中的 Rust 更新

!!! note "主要作者"
    
    - 原文链接：[Rust in QEMU update, October 2025](https://lore.kernel.org/qemu-rust/20251015112139.1197033-1-pbonzini@redhat.com/)
    - 翻译：[@zevorn](https://github.com/zevorn)

10 月份上游的开发进展依然可喜，主要成果有：带来更清晰的代码结构、升级 MSRV（最低支持的 Rust 版本升级到 1.83）、合并了关键功能、扩展 CI 支持、开发 QAPI/QMP 初始绑定等，以及计划在 QEMU 10.2 版本中整合 Meson 1.9.2 bugfixes、推进 rust-vmm 的 IOMMU 支持的进行中工作，还提及未来深化 QAPI/QMP 集成、默认启用 Rust、整合 Meson 与 Cargo 的规划。

## QEMU 10.2 中的状态

虽然该项目仍处于“孵化中”状态，但许多独立部分的实验性已降低，且 QEMU 中 Rust 代码的整体结构更加清晰。

现在支持的最低 Rust 版本为 1.83，计划不再额外提升版本。唯一不支持的平台是 Debian bookworm mips64el（bookworm 上的其他架构仍可构建）和 Ubuntu LTS 版本，这些平台计划使用更新版本的 Rust，但目前尚未可用。

Rust for QEMU 已在几乎所有 CI 中启用了 Rust。一旦某些 Meson 漏洞被修复（目前这些漏洞阻碍了在 Windows 和 macOS 上启用 Rust），所有目标平台都将支持 Rust。Marc-André Lureau 和 Martin Kletzander 都已发现并修复了 Meson 中的一些漏洞，他们的补丁将包含在 Meson 1.9.2 中。Meson 1.9.2 版本还将能够使用 rustc 链接模拟器二进制文件，这会使生成的可执行文件更小，构建系统也更简单一些。

今年夏天，Tanish Desai 编写了大部分跟踪支持代码；该代码现已合并，填补了设备的 Rust 版本与 C 版本之间最大的功能差距。虽然仍缺少对 dtrace 和 ust 后端的支持，但很快将通过现有的 probe crate 添加 dtrace 支持。对于 ust，计划将其弃用。

有了这些改进，用 Rust 编写新设备已经具有实际优势，特别是当开发者不需要除已绑定的 API 之外的其他 API 时。目前构建系统的样板代码仍然较多，但这是暂时的问题，现有的 pl011 和 HPET 设备提供了易于遵循的范例。

如果有任何用 Rust 编写且没有 C 对应版本的设备被贡献进来，或许值得将“启用 Rust”与“启用所有用 Rust 编写的设备”分开。这样，在所有平台都拥有足够新的编译器版本且不存在构建系统问题之前，pl011 和 HPET 设备的 C 版本可以保持可用。

## 改进和清理

随着 `VMStateDescriptionBuilder` 的近期合并，除了 QOM 的 `instance_init` 方法外，设备中不再需要不安全的代码。

迁移支持也得到了改进，以支持线程安全（非 BQL）设备。计划将这些改进应用于 HPET，该设备最近在 C 语言版本中实现了无 BQL。线程安全性或许也是将其他简单设备从 C 重写为 Rust 的一个很好的理由，例如中断控制器。

Rust 与 C 的互操作性现在分布在多个 crate 中，每个 crate 都链接到相应的 C 代码。这应该会简化块设备方面的工作，因为不再需要为工具和模拟器两次构建支持代码，同时也使文档没那么晦涩难懂。

此外，GLib 绑定使用 glib_sys，这使得仅能使用 Rust for QEMU 支持的最低版本 glib（2.66）中存在的功能。现在 meson.build 文件中的遗留问题少了很多。这些文件仍然相当大，但它们具有一致性且应该易于阅读。

Zhao Liu 正在研究清理 HPET 以分离寄存器结构（PL011 已经实现了这一点），同时启用无锁 MMIO。

## 其他进行中的工作

**vm-memory 集成**

Zhao Liu 还研究了在 QEMU 中使用 vm-memory crate。由于 QEMU 支持 IOMMU，这需要对该 crate 进行修改。Hanna Czenczek 在 rust-vmm 中对 IOMMU 支持的工作即将合并，Rust for QEMU 将等待该工作完成后再继续 QEMU 中的 vm-memory 集成。尽管如此，Zhao Liu 的工作为未来的 API 可能是什么样子提供了一些见解 —— 例如，安全的 Rust 中的内存存储可能如下所示：

```rust
ADDRESS_SPACE_MEMORY.store::<Le32>(addr, 42);
```

**Meson 对 Cargo 的支持**

如[更早期的文章][earlier updates]中所述，在进行 QEMU 工作的同时，Paolo 也在研究改进 Meson 对 Cargo 和 Rust 的支持。这些改进的主要目的是让 QEMU 能够将 Cargo 子项目用于其依赖的 crate，这样只需将依赖项添加到 Cargo.lock（该文件目前位于 rust/子目录中）即可添加依赖。Hanna 特别强烈要求简化添加新的 Rust 依赖项的流程。:)

Meson 1.10 将已经能够读取超级项目的 Cargo.lock，但之后还需要支持读取 QEMU 的 Cargo.toml。这有两个目标：一是解析依赖项所需的功能集，二是能够用一两行 Meson 代码添加新的简单构建目标（例如设备和测试）。这项工作的第一部分位于：

  - [Proposal: rust.workspace() Cargo integration](https://github.com/mesonbuild/meson/issues/14639)

  - [Use toplevel Cargo.toml for feature resolution](https://github.com/mesonbuild/meson/pull/15069)

这是一项较长期的工作。Paolo 计划从 Meson 1.10 开始，在未来几个 Meson 版本中完成合并，但这不会阻碍 QEMU 的进一步工作。目前 Rust for QEMU 大约有 20 个依赖项，平均每月增加约 1 个。

## 后续计划

除以下几项外，C 语言和 Rust 语言设备在功能上已几乎对等：

- dtrace 支持 [Stefan 负责]

- HPET 中的无锁 MMIO [Zhao 负责]

一旦上述问题得到解决，且 Ubuntu 系统配备 Rust 1.83，QEMU 就可以默认启用 Rust 了。

关于 QObject 和 QAPI 与 serde 的集成，相关讨论已在邮件列表中进行。这一集成将实现 QMP 集成，也有助于支持 QOM 属性，但由于目前还没有用户使用，所以不计划在 QEMU 10.2 中实现。

一种可能性是重新开展更多 Marc-André 在 2022 年关于 QMP 命令的工作，当时他使用了 qemu-ga。

对构建系统的清理工作，足以重启 block layer 的相关工作。欢迎更多人参与进来，为更多后端和总线类型添加绑定，例如（分别对应的）块设备和 I2C。

[earlier updates]: https://lore.kernel.org/qemu-rust/c2342e56-b6d8-4115-8318-d8047a46f1ad@redhat.com/