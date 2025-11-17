# 2025 年 1 月 QEMU 中的 Rust 更新

!!! note "主要贡献者"
    
    - 原文链接：[Rust in QEMU update, January 2025](https://lore.kernel.org/qemu-rust/17ad81c3-98fc-44c2-8f65-f5e2cc07030b@gnu.org/)
    - 翻译：[@zevorn](https://github.com/zevorn)

尽管该项目仍处于实验阶段，但安全的 Rust 所具备的功能已经足以支持开发新设备。和之前一样，本文主要涵盖  Paolo 所关注的内容，即如何使用安全的 Rust 编写设备。“⚠️”图标用来标记最重要的缺失功能。

## QEMU 10.0 中的状态

使用 `--enable-rust` 构建的 QEMU 可在所有受支持的构建平台上编译。它通过了持续集成（CI），且 `make check-unit` 会运行针对 rust/qemu-api 的测试。`make check-qtests` 涵盖了 Rust 版本的 pl011 设备模型，包括迁移功能；HPET 设备的 Rust 转换版本已在邮件列表中发布，可能会在未来几周内合并。

自 9.2 版本以来，定义类、定义回调以及调用 QEMU 核心功能所需的不安全代码量已大幅减少，特别是如果算上一些已发布且等待审核的代码片段的话。QOM 对象可以使用完全安全的代码创建并管理其生命周期；与 C 代码不同，Rust 能够区分嵌入式对象（没有独立的引用计数）和那些在自己的堆块中分配的对象。

现有的绑定包括系统总线（sysbus）设备、通用输入输出（GPIO）引脚、时钟、内存区域操作（MemoryRegionOps）和定时器；Kevin Wolf 正在研究 block layer 以及在工具中使用 Rust。此外，还有一些核心构建块，使得扩展 C 与 Rust 之间的绑定变得更加容易。

去年 12 月，Paolo 发布了 [pl011 的 C 实现与 Rust 代码“预期”形态之间的对比][1]；在发布时，代码虽然可以编译，但远未达到可用状态。目前，pl011 在代码库中的 Rust 实现与预期形态非常相似，主要区别在于字符设备和安全的 instance_init。

## 构建系统

开发者可以使用 ninja 轻松访问 clippy、rustfmt 和 rustdoc。

Meson 1.7 原生支持 clippy，针对 rustdoc 的拉取请求正在进行中。这一点很重要，因为从长远来看，最好避免 src / 目录的泛滥，毕竟 QEMU 有数百个设备。

一个 Meson 的拉取请求也在推进中，该请求将添加对文档测试（doctests）的支持，把它们整合到“make check”中，并且允许它们（与“cargo test --doc”不同）与 [C 代码链接][2]。

⚠️ 由于 rust/qemu-api/meson.build 存在限制，模块目前还无法与 Rust 代码协同工作。针对此问题的修复方案也已提交至 Meson，相关的拉取请求同样处于[开放状态][3]。

Rust 目前仍未启用，并且默认情况下不会检查它是否存在。如果有任何用 Rust 编写且没有对应的 C 版本的设备被贡献进来，或许值得将“启用 Rust”和“启用所有用 Rust 编写的设备”这两个选项分开。

## 设备方面的改进

⚠️ Rust 版本中缺少了一些最近的 pl011 提交。Philippe 主动提出要将它们移植过来，以此自学 Rust。移植设备中的一些问题现已修复，并且 Rust 版本和 C 版本之间的迁移流是兼容的。

⚠️ Philippe 还有一系列工作要实现流量控制以及更好地利用 PL011 FIFO（如果  Paolo 理解正确的话）。目前将其移植到 Rust 会很困难，因此其 Rust 实现因缺乏字符设备绑定而受阻。

⚠️ HPET 缺乏对实时迁移的支持。

⚠️ 与之前一样，日志记录和跟踪功能也仍然缺失。

## 避免未定义行为

QEMU 特有的内部可变性类型 —— 支持 big QEMU lock 的 Cell 和 RefCell 的自研变体，因此可在多线程环境中使用 —— 已包含在代码树中，并被 pl011 和 HPET 设备模型所采用。这意味着 Rust 代码现在遵循了这样一个不变式：可变引用要么是唯一的，要么是从 UnsafeCell<> 获取的。

在某些情况下，Rust 代码仍然会在引用初始化之前创建指向无效数据的引用；这个问题将通过使用 Linux 的 pinned-init crate 来实现 QOM 的 instance_init 方法来解决。pinned-init 在 crates.io 上可用于用户空间，它消除了向安全的 Rust 代码暴露部分初始化对象的需求。

John Baublitz 指出了更多应该包含在 cell 中的字段情况，因为这些字段会被 C 代码修改。其中包括 qdev 属性的字段以及 QOM 类的 C 结构体。与其他问题相比，这里的问题在很大程度上更偏向理论层面，而且 Linux 也提供了处理这些问题的示例。

## 与 C 代码的绑定

尽管 QOM 的面向对象、重度依赖继承的设计对 Rust 来说是一项挑战，但 QEMU 10.0 中的 qdev 绑定将会相对完善，既涵盖类，也涵盖接口。得益于全新的回调功能，设备能够访问 GPIO 引脚、定时器、时钟和 MemoryRegionOps。这应该足以让设备的实现仅需极少使用不安全代码。

上一份路线图中未提及的一个开发领域是 VMState。虽然 9.2 版本中包含了对 C 宏的直接移植，但这需要大量的宏，尤其是它并非类型安全的。后者实际上使得 Rust 版本比 C 代码更糟糕，尽管可以修复，但重新编写以利用 Rust 的类型系统和常量求值是一个更好的方案。VMState API 尚未最终确定（例如，它仍然需要用于 pre_save/post_load 的不安全回调），但新版本已经易用得多。

pl011 和 HPET 代码中剩余的 unsafe 块情况如下：

- pl011 需要字符设备绑定；相关代码已基本就绪，但不一定会包含在 10.0 版本中 HPET 会进行一些非常简单的内存访问；

- 一个不错的安全解决方案可能是 vm-memory crate。Paolo 尚未研究如何使用它，但 vm-memory 和 vm-virtio 的编写考虑到了 QEMU 的使用场景；

- 如前所述，VMState 定义的绑定也在开发中，且仍部分存在不安全情况；

- 在引入 pinned-init crate 之前，instance_init 方法会使用不安全代码。pinned-init 也将有助于创建循环数据结构。

虽然这看起来数量不少，但我们能够列举 unsafe 的使用情况并计划将其移除，这本身就是一项重大成就！

安全的 Rust 所提供的功能已足够支持新增设备，即便对于 QEMU 中那些尚未有绑定的部分，这些设备可能需要一些不安全代码。

添加到 QEMU 中的大多数设备都很简单，不会进行任何复杂的 DMA 操作。在上一份路线图中，Paolo 提到这类简单设备通过用 Rust 重写获益甚微。虽然对于已有的 C 代码设备来说可能确实如此，但在  Paolo 看来，用 Rust 编写新设备已经有其优势。代码通常更清晰，语义的某些方面（例如哪些字段是可变的）也更明确；主要的限制是缺乏对跟踪和日志记录的支持。

## 新的实用 utils 代码

qemu-api 和 qemu-api-macros 这两个 crate 现在包含了更多实用代码，可用于绑定和设备，其中包括：

- 整数的位操作扩展；

- 在枚举和整数之间进行转换的过程宏；

- 回调支持。

回调是自上一篇路线图帖子以来最棒且最出人意料的消息。qemu-api crate 中实现的解决方案能够将 Rust 函数和引用转换为函数指针和不透明值；泛型用于为每个 Rust 函数创建单独的跳板（即单独的函数指针）。时钟、定时器、内存区域和字符设备都在使用这种机制，因此这部分可以认为已经解决。

qemu-api-macros crate 还增加了一小套实用函数，未来可被更多过程宏复用。

Paolo 目前缺少的一个工具是位标志枚举。Rust 生态系统中最常用的 crate 是 [bitflags][4]。

## Rust 版本要求

与之前的路线图相比，主要变化是有三个功能变得更为紧迫：

- 虽然 pinned-init 的代码只需稍作修改即可支持 Rust 1.63.0，但它严重依赖 impl Trait 返回类型；而自 Rust 1.75.0 起，trait 函数才能返回 impl Trait。由于 instance_init 本身就处于一个 trait 中，这是一个重要的限制，可能会推迟 pinned-init 的纳入；唯一支持的、使用较旧 rustc 的发行版是 Debian bookworm。

- 常量中对静态变量的引用（在 1.83.0 版本中稳定）；需要使用蹩脚的解决方法的主要地方是 VMState。

虽然这些都不是启用 Rust 的阻碍，但前两个问题会给每个添加到 QEMU 中的 Rust 设备增加少量技术债务。在 Debian bookworm 上使用 rustc-web 包可能是个不错的选择，截至撰写本文时，该包提供的是 1.78.0 版本的 rustc。

## 下一步计划

在早期讨论中提到的两个领域目前尚未有任何进展：

- 跟踪点和日志记录是最优先需要补充的功能，或许还包括 DMA（直接内存访问）；虽然当前的设备集仅包含有限的跟踪点，但可观测性作为一种调试工具显然十分重要。这是鼓励用 Rust 实现新设备之前的主要障碍。

- QEMU 的 C 数据结构与 Rust 对应结构之间的互操作性也没有新消息。一旦我们需要一个可能失败的 `realize()` 实现，或者着手处理 QAPI 时，就会解决这个问题。

除此之外，或许值得再次研究使用过程宏来简化 QOM/qdev 类的声明，例如针对 qdev 属性的声明。

[1]: https://lists.nongnu.org/archive/html/qemu-rust/2024-12/msg00006.html
[2]: https://github.com/mesonbuild/meson/pull/13933
[3]: https://github.com/mesonbuild/meson/pull/14031
[4]: https://docs.rs/bitflags/2.6.0/bitflags/