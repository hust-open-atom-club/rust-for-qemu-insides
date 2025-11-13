# 2025 年 5 月 QEMU 中的 Rust 更新

!!! note "主要作者"
    
    - 原文链接：[Rust in QEMU update, April 2025](https://lore.kernel.org/qemu-rust/d3d1944e-2482-4aa7-b621-596246a08107@gnu.org/)
    - 翻译：[@zevorn](https://github.com/zevorn)


目前 QEMU 对 Rust 的支持仍处于试验阶段，过去三个月的大部分时间，上游开发者都在努力清理多余的 bindings，并让安全的 Rust 能够使用更多功能。

和之前一样，本篇文章主要涵盖 maintainer Paolo 所关注的内容，也就是让使用安全的 Rust 编写设备成为可能。正因如此，像 QAPI 和异步（块设备）等主题并未包含在内。

总的来说，Paolo 认为进展是不错的：上一次更新中提到的大部分缺失功能都已修复，或者至少在未来几个月有了相应的计划。

## QEMU 10.0 中的状态

使用 `--enable-rust` 编译的 QEMU 可在所有受支持的构建平台上完成编译。它能通过持续集成（CI）检查，且 `make check-unit` 会运行针对 rust/qemu-api 的测试。`make check-qtests` 涵盖了 Rust 版的 pl011 和 HPET 设备模型，包括前者的迁移功能。pl011 完全使用安全代码实现（迁移和 qdev 属性部分除外）。HPET 仅在一些小型且范围相当有限的情况下使用不安全代码（见下文）。

自上次更新以来，早期 binding 代码中的一些问题逐渐显现；特别是孤儿规则（orphan rules）导致在 qemu_api crate 外部实现类变得异常困难，且总体上难以将 qemu_api crate 拆分为多个部分 —— 例如，拆分为工具感兴趣的部分和仅系统模拟器使用的部分。另一项重要变更则是将 bindgen 生成的类型与 Rust 代码实际使用的结构体区分开来。这使得 Send、Sync 或 Zeroable 等 trait 能够针对 C 和 Rust 结构体独立指定。

得益于 Kevin Wolf 在 block layer 方面的工作，出现了一个新模块，用于在 C 语言的成功 / 错误码（-errno）约定与 `io::Result` 之间进行转换。该模块也用于字符设备 binding。

## 构建系统

开发者可以使用 ninja 轻松访问 clippy、rustfmt 和 rustdoc。Meson 1.8 原生支持 clippy 和 rustdoc（包括文档测试），但由于 1.8.0 版本中存在一些回归问题，这一点将不得不等到下一个稳定版本才能实现。此次 Meson 的更新还将使得可以同时使用 `--enable-modules` 和 `--enable-rust`。

Rust 目前仍未启用，并且默认情况下不会检查它是否存在。主要原因是 Rust 静态库还会静态链接到 Rust 标准库，从而导致生成的可执行文件体积增大（这也会让发行版对我们不满）。一个待处理的 [Meson 拉取请求][1] 将解决这个问题，前提是 system/main.c 被重写或以 Rust 包装。


## 设备方面的改进

HPET 实时迁移支持已准备好合并。与之前一样，Rust 版本中缺少一些近期的 pl011 提交。

日志记录（Logging）和追踪（tracing）已被提议作为谷歌代码之夏的一个项目。

## 剩余的不安全代码


qdev  binding 涵盖了基本类和接口，包括 GPIO 引脚、定时器、时钟和 MemoryRegionOps。VMState 仍需要用于 pre_save/post_load 的不安全回调，其最终版本要等到最低支持的 Rust 版本提升至 1.83.0 才行。

除了 VMState，pl011 和 HPET 代码中其余的 unsafe 块都可以在不提升语言版本的情况下移除。

HPET 会进行一些非常简单的内存访问；一个不错的安全解决方案可能是 vm-memory crate。虽然 Paolo 还没研究过如何使用它，但 vm-memory 和 vm-virtio 的编写考虑到了 QEMU 的使用场景。

`instance_init` 方法正在使用不安全的代码。有多种解决方案可以解决这个问题：Paolo 计划采用的一种是使用诸如 [pin_init][2] 或 [pinned_init][3] 的方法，但 Paolo 也在研究并基于 `std::mem::MaybeUninit` 字段投影开发了一个更简单的版本。这个版本仅从实现中移除了 unsafe，而没有从 instance_init 方法本身移除，但它的侵入性更小，短期内可能是一个可行的方案。

安全的 Rust 所提供的功能足以支持新设备的加入，即便对于 QEMU 中一些尚未有 binding 的部分，它们可能需要使用一些不安全的代码。添加到 QEMU 中的大多数设备都很简单，不会进行任何复杂的 DMA 操作；虽然用 Rust 重写这类简单设备几乎没有什么好处，但一旦支持跟踪和日志功能，用 Rust 编写新设备将带来显著的收益。尽管迁移和 `instance_init` 中的不安全代码会被视为每一个添加到 QEMU 中的 Rust 设备的技术债务，但 Paolo 预计未来几个月不会有大量的 Rust 设备涌现，因此这不会成为一个问题。

QEMU 的 C 数据结构与 Rust 对应结构之间仍然没有互操作性，这方面也没有新的进展。和之前一样，一旦我们需要一个可能失败的 `realize()` 实现，或者在处理 QAPI 时，我们会解决这个问题。

## Rust 版本要求

补丁已列入计划（且大部分已审核通过），旨在将最低支持的 Rust 版本提升至 1.77.0。不过，为了支持常量中对静态变量的引用，可能还需要至少再升级一次版本。这一特性在 1.83.0 版本中已稳定，并且对于安全 Rust 中的迁移支持至关重要。

这将意味着在使用发行版提供的编译器的 Debian bookworm 系统上，需要放弃对 `--enable-rust`”` 的支持。如果有任何仅用 Rust 编写且没有对应的 C 版本的设备被贡献进来，那么将“启用 Rust”和“启用所有用 Rust 编写的设备”这两个选项分开可能是值得的。这样一来，pl011 和 HPET 设备的 C 版本在 bookworm 系统上仍可使用。

## 设备的编码风格

pl011 和 HPET 是独立开发的，它们在一些用法上存在差异，这些差异本可以统一。Peter Maydell 有几点观察发现：

 Paolo 确实注意到，这两个设备在结构设计上存在一些不一致之处，例如：

* pl011 的主源文件是 device.rs，而 hpet 的主源文件是 hpet.rs

* 在某些地方，我们使用寄存器中位的实际名称（例如 Interrupt 的 OE、BE 等常量），而在另一些地方，我们似乎对这些名称进行了重命名（例如 pl011 的 Flags 中使用的是 clear_to_send，而不是 CTS 等）

* pl011 为其寄存器定义了命名字段，但 hpet 却是像这样处理的：

    ```
    self.config.get() & (1 << HPET_CFG_LEG_RT_SHIFT) != 0
    ```

* pl011 将 PL011State 和 PL011Registers 进行了分离，但 HPET 没有。就像 Paolo 不久前在一封邮件中提到的，Paolo 认为这种 State/Registers 的分离方式，我们要么应该将其明确地、有意地、正式地确立为设备模型设计的推荐规范之一


Paolo 认为，我们应该明确对于这类代码编写而言，什么才是正确且最佳的风格，并保持一致性。在 C 语言设备模型中，长期以来存在着多种不同的编写风格，这是个问题，所以我们在 Rust 这边应该力求更统一的风格。

Paolo 注意到一点，在 Rust QEMU 代码中，Paolo 倾向于大量使用 const 和 static，但有几个 crate 并不支持这种风格，包括我们用于命名字段的 bilge crate 以及 bitflags 等其他 crate。这两种情况都与 Rust 没有 const traits 有关，例如 from()/into() 或运算符重载方面就没有。 

Paolo 已经有了一个类似 bitflags 的宏原型，它对 const 更友好，而且我们还需要决定是继续使用 bilge、对其进行分支开发、重写它还是采取其他方式。

## 下一步计划

关于缺失的功能，跟踪点和日志记录仍然是优先级最高的缺失特性，或许还有 DMA，这也是鼓励使用 Rust 实现新设备之前的主要障碍。希望这个缺口能在今年夏天填补上。

在实验方面，如果有人想尝试将 vm-memory crate 用于 DMA，那会非常有意思。不过，Paolo 建议接下来的步骤主要是清理现有内容，确保我们为 Rust 在 QEMU 中更广泛的使用做好准备。

如果有人喜欢做些琐碎的工作，现在拆分 qemu_api crate 是可行的，而且是件好事。

如果有人有良好的品味，他们可以结合 Peter 上面的意见仔细检查代码，进行清理，使 pl011 和 HPET 都能成为 QEMU 中 Rust 代码的良好示例。

Paolo 还认为，是时候重新考虑使用过程宏来简化 QOM/qdev 类的声明了。例如：

```rust
     #[derive(qemu_api_macros::Object(class_name="pl011", class=PL011Class))]
     #[derive(qemu_api_macros::Device(vmsd=VMSTATE_HPET))
     pub struct PL011State {
         pub parent_obj: ParentField<SysBusDevice>,
         pub iomem: MemoryRegion,
         pub regs: BqlRefCell<PL011Registers>,
         pub interrupts: [InterruptSource; IRQMASK.len()],
         pub clock: Owned<Clock>,

         #[qemu_api_macros::property(name="chr")]
         pub char_backend: CharBackend,

         #[qemu_api_macros::property(name="migrate-clk", default=true)]
         pub migrate_clock: bool,
     }
```

与此相关，Paolo 最近发现了 [attrs crate][4]，它提供了一种在过程宏中解析属性内容的简便方法。

[1]: https://github.com/mesonbuild/meson/pull/14224
[2]: https://docs.rs/pin_init/
[3]: https://docs.rs/pinned_init/
[4]: https://docs.rs/attrs/