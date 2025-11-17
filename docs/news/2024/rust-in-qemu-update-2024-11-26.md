# 2024 年 11 月 QEMU 中的 Rust 更新

!!! note "主要贡献者"
    
    - 原文链接：[Rust in QEMU roadmap](https://lore.kernel.org/qemu-rust/cc40943e-dec1-4890-a1d9-579350ce296f@pbonzini.local/)
    - 翻译：[@zevorn](https://github.com/zevorn)

根据社区会议上呈现的内容，以下是未来 QEMU 中 Rust 子项目的一些想法。

这绝不是详尽无遗的，例如 QAPI 和异步（块设备）就未被包含在内，因为 Paolo 尚未对它们进行研究。

## QEMU 9.2 中的状态

使用 `--enable-rust` 构建的 QEMU 可在所有受支持的构建平台上编译。它通过了持续集成（CI），且 `make check-unit` 会运行针对 rust/qemu-api 的测试。`make check-qtests` 涵盖了 Rust 编写的 pl011 设备模型，包括迁移功能。不过，Rust 并未默认启用，其存在性也不会被默认检查。

定义类、定义回调以及调用任何核心功能都需要大量不安全的代码。尽管如此，当前的 pl011 代码为 Rust 代码更迫切需要哪些功能提供了很好的路线图。

Zhao Liu 正在致力于将 HPET 设备转换为 Rust 实现。它存在相同的局限性，但涵盖了略有不同的功能集（例如计时器），并且提供了第二个设备来验证正在构建的抽象概念。不过，（在 Paolo 看来）在 FFI 抽象准备就绪且不安全代码的使用受到限制之前（尤其是初始化之后），不应纳入这两个设备之外的任何其他设备。

目前，我们关注的是简单设备。这些设备不进行任何 DMA 或内存分配；用 Rust 重写它们几乎没有什么好处。然而，它们体积小且自成一体，能让我们在尝试跑之前先学会走。

对于任何想从事 FFI 相关工作的人，强烈建议咨询子系统维护者，并且要“尽早且经常”发布内容。同时，将邮件列表线程的链接包含在 [QEMU Wiki][5] 中。


## 构建系统

具体讨论详见 [[RFC PATCH 00/11] rust: improved integration with cargo][1]，这为开发者提供了便捷访问 clippy、rustfmt 和 rustdoc 的途径；总体而言，Cargo 是集成除 rustc 之外的工具的最简单方式。该系列还使得可以使用 Cargo.toml 提供的熟悉语法来配置警告。

后续步骤：

- Meson 可以实现对 clippy 和 rustfmt 的访问，类似于其现有的“ninja clang-tidy”和“ninja clang-format”目标。这将移除 QEMU 中的代码，更重要的是，避免每个 Rust/C 混合项目都重复造轮子。

- rustdoc 测试应集成到“make check”中。运行“cargo test --doc”存在一个局限，即这些测试无法与 C 代码链接。Paolo 已经制作了原型，但主要是为了了解 rustdoc 测试的运行方式，并在 Meson 中实现它们。

从长远来看，单独的 rust / 目录可能不会存在，而且由于 QEMU 有数百个设备，最好避免 src / 目录的激增。届时是否仍然需要 Cargo，或者在达到那一步之前，所有必要的功能是否都会包含在 Meson 中，这将是一件值得关注的事情。

小结：

该系列主要是 Python 和 Meson 代码，因此 Paolo 愿意将其合并到 10.0 版本中。

从长远来看，Meson 应该提供类似 Cargo 的功能。在此期间，可以手动实现这些功能，但如果 Meson 1.7.0 版本发布并原生支持 clippy 及相关工具，我们应该升级到该版本。

## 设备方面的改进

Rust 版本中缺少了一些近期的 pl011 提交。Philippe 主动提出要将这些提交移植过来，以此自学 Rust。

Philippe 还有一系列工作，旨在实现流量控制并更好地利用 PL011 FIFO（如果 Paolo 理解正确的话）。目前将其移植到 Rust 会很困难，因此其 Rust 实现因缺乏字符设备绑定而受阻。

此外，日志记录和追踪功能也尚未实现，但这部分内容应该不会阻碍默认启用 `--enable-rust`。

小结：

Philippe 在 FIFO 方面的工作可能会影响在 10.0 版本尽快默认启用 `--enable-rust` 的计划。

## 避免未定义行为

请参考邮件 [[PATCH 0/2] rust: safe wrappers for interrupt sources][2]。目前，QEMU 代码中的 Rust 部分没有遵循这样一个不变式：可变引用要么必须是唯一的，要么必须是从 UnsafeCell<> 获取的。在某些情况下，它还会创建指向无效数据的引用（无论是共享引用还是可变引用），例如空引用。为了能在代码库中纳入一些 Rust 代码并开发构建系统集成，这是一种必要之恶，但显然必须予以修复。

第一部分问题通过对代码进行注释以使用适当的内部可变性类型来解决。这些类型包括支持“大 QEMU 锁”的自定义 Cell 和 RefCell 变体，因此可在多线程环境中使用。特别是，QEMU 的 RefCell 变体还会检查在存在未完成的借用时，绝不能调用 bql_unlock ()。

第二部分问题通过使用 Linux 的 pinned-init crate 来解决，该 crate 可在 crates.io 上用于用户空间。pinned-init crate 避免向安全的 Rust 代码暴露部分初始化的对象。目前，它仅支持 nightly 版本的 Rust，但 Paolo 已提交了一个拉取请求来解决此问题；该拉取请求支持 Debian 的 rustc 1.63.0 版本不成问题。

小结：

* 根据需要引入 cell 类型。BqlCell 已发布，BqlRefCell 已编写但尚未投入使用；

* 计划使用 pinned-init，但会在后续进行。

## 安全的 QOM 类定义

请参考邮件 [[PATCH 0/8] rust: qom: move bridge for TypeInfo and DeviceClass functions to common code][3]。

QOM 的面向对象、重度依赖继承的设计对 Rust 来说是一项挑战，但幸运的是，我们可以借鉴 glib-rs 开发者的经验。要让 QOM 能在安全、符合 Rust 风格的代码中使用，有两个相互独立的部分：即定义 QOM 类和调用 QOM 方法。

关于定义 QOM 类，预计 Rust 代码大多会定义叶子类，这会让任务稍微简单一些。或者至少，任何由 Rust 代码定义的超类也只会在 Rust 中被继承。
Manos 曾发布过一次初步尝试，但在评审中 Paolo 指出，他的过程宏方法仅限于 DeviceState 的子类。能有更易于适配其他类的方法会好得多。

目前，Paolo 发布了一个系列，它减少了代码重复，但仍主要聚焦于 DeviceClass 的四个元素（realize、传统重置、属性、vmstate）。这部分很简单。
第二步是在类型系统中编码 QOM 层次结构。一个新的 ClassInitImpl trait 会初始化特定类（例如 DeviceClass）的方法，然后“递归”到超类：

```rust
    trait ClassInitImpl<ClassType> {
        fn class_init(class: &mut ClassType);
    }

    impl ClassInitImpl<DeviceClass> for T where T: DeviceImpl {
        fn class_init(class: &mut DeviceClass) {
            dc.realize = rust_realize_fn::<T>;
            ...
            <Self as ClassInitImpl::<ObjectClass>>::class_init(class.parent_class);
        }
    }

    impl ClassInitImpl<ObjectClass> for T where T: ObjectImpl { ... }
```

这对已发布部分中的 class_init 实现进行了泛化。相同的递归模式也可用于支持 QOM 接口。

小结：

- 欢迎对第一部分进行评审。希望当 Paolo 发布第二部分（12 月中旬左右？）时，第一部分至少已部分完成评审；

- 过程宏仍然是一个很有价值的工具。将它们用于 qdev 属性应该很容易从 Manos 的提交中提取出来。在评审中，Paolo 还提议构建一个小型“工具包”，供未来更多的过程宏复用。

## QOM 方法调用

这是 Paolo 去年 7 月开始的 QOM 工作的一个独立分支，当时 Paolo 研究了 glib-rs 对 GObject 的实现。其理念是能够编写出看起来像原生 Rust 但调用 QOM 方法的代码。例如：

```rust
    let dev = PL011State::new();
    dev.prop_set_chr("chardev", chr)
    dev.realize().unwrap_fatal();		// operate on Error**
    dev.mmio_map(0, addr);
    dev.connect_irq(0, irq);
    Object::into_raw(dev)
```

请注意，方法可以来自多个类（DeviceState、SysBusDevice，甚至在“new”的情况下还可以来自 Object）。这使得将设备实现为类似 SysBusDevice<PL011Properties, PL011Registers > 这样的形式变得更加困难；例如，你仍然必须在 SysBusDevice 的超类中提供诸如“prop_set_chr”或上述示例中的“realize”等方法。

glib-rs 的实现基于一个必须手动编写的 IsA trait（例如，PL011State 实现了 `IsA<Object>、IsA<DeviceState>、IsA<SysBusDevice>`）。然后，Rust 的自动解引用行为可以扩展到所有超类；方法的包装器在 trait 中实现：

```rust
    trait DeviceMethods {
        fn realize(&self) -> Result<(), qemu_api::Error> {
        ...
        device_realize((self as *const _).cast_mut().cast::<DeviceState>, ...);
        ...
        }
    }
```

并且该 trait 被应用于所有能解引用为 DeviceState 子类的事物：

```rust
    impl<R: Deref> DeviceMethods for R where R::Target: IsA<DeviceState> {}
```

与此相关，IsA 特性支持类型安全的编译时检查转换。与 C 代码不同，转换为超类的写法可以确保：如果目标类型不是超类，编译器会报错，而且这种转换没有运行时成本。

## 回调函数

这是“紧急”变更中成熟度最低的一项，或许也正因如此，一个良好的设计显得更为重要。PL011 有针对字符设备和内存区域的回调函数，但其他使用场景还包括计时器、下半部和中断接收器（也称为 GPIO 输入）。

到目前为止，Paolo 只做了一个略显简陋的纯 Rust 示例，其中使用的回调函数就像在 C 语言中常见的那样（即使用函数指针和 void* 不透明指针）；该示例还实现了“绑定”功能，你只需编写一个 Rust 特性，函数指针就会自动填充。你可以在 [bonzini/callbacks.rs][4] 找到它。

尽管这段代码看起来完全不符合 Rust 的惯用风格，但用 Rust 来做这个实验，是为了确保 Miri（Rust 的实验性“未定义行为检测器”）能够认可它。

Zhao 正在开发的 HPET 特别有意思，因为它还包含计时器数组。

小结：

- 这需要有勇于尝试的人来研究。这并非不可能，但既具有一定难度，又属于高优先级任务；

- Linux 有一些相关示例，但实在难以理解。如果能以保留一些易于验证的“不安全”代码块为代价（同时仍为计时器、内存区域等提供安全的回调函数），找到更简单的方法，那可能是一个更好的短期方案。

## 跟踪/日志记录

跟踪点和日志记录目前尚未得到支持，也没有人开始着手相关工作。

关于跟踪功能，Paolo 尚不清楚有多少 C 代码可以复用，以及有多少 Rust 代码可以自动生成。这会是一个合适的 Outreachy 或编程之夏项目，因为它对不安全代码的需求有限，且范围定义明确。

纯 Rust 实现会很有意思，但要注意的是，跟踪子系统中普遍使用 printf 风格的基于 % 的字段格式化方式。

日志记录是 QEMU 中的一个小组件，这是一个定义从 C 代码转换到 Rust 代码的编码风格的好机会，例如枚举和函数的命名方式。Paolo 唯一的要求是允许日志代码使用与 format!() 相同的语法！

小结：

- 简单，优先级低；

- 有人想通过一个小项目来学习 Rust FFI 吗？

## 数据结构互操作性

这方面的典型例子是“Error”，即提供一种简便的方法，实现从 QEMU 的 Error 对象到 Rust 的 Error trait 的相互转换。

Paolo 和 Marc-André都曾致力于此，且采用了两种不同的方法。Marc-André的方法以 glib-rs 为基础，其核心在于关注期望的所有权语义（即生成的数据归 Rust 所有还是归 C 所有），这是因为 glib-rs 本身受到了 GIR（GObject 跨语言自省接口）的启发。而 Paolo 的方法则尝试扩展 Rust 中 From<>/Into<>/Clone 这些 trait 背后的理念，以实现 Rust 对象与 C 对象之间的相互转换和克隆。

两者的基本操作类似：

```
    from_glib_full          from_foreign
    from_glib_none          cloned_from_foreign
    to_glib_full            clone_to_foreign_ptr
    to_glib_none            clone_to_foreign
```

不过，设计和代码却大相径庭。

小结：

- 显然，Paolo 对此带有偏见，所以这个设计需要更多人来出谋划策。此事虽不紧急，但必须在我们着手处理 QAPI 之前解决。或许，在 users.rust-lang.org 上提问也是个办法；

- 目前 Paolo 不打算在这方面开展任何工作，但 Paolo 希望无论我们选择哪种方法，最终都能形成一个外部 crate，在 QEMU 之外进行开发，仅供 QEMU 使用。

## 近期对 Rust 所期望的特性

QEMU 支持 rustc 1.63.0 及更新版本。值得注意的是，以下特性目前缺失：

- core::ffi（1.64.0 版本新增）。可改用 std::os::raw 和 std::ffi；

- “let ... else”语法（1.65.0 版本新增）。可改用 if let。目前 QEMU 已在其 vendored 的 bilge crate 副本中对此进行了补丁修复；

- 作为 const 函数的 CStr::from_bytes_with_nul()（1.72.0 版本新增）；

- 作为 const 函数的 MaybeUninit::zeroed ()（1.75.0 版本新增）。QEMU 使用了 Zeroable trait，你可以通过手动实现该 trait 来提供 Zeroable::ZERO 关联常量；

预计一旦 QEMU 停止支持 Debian bookworm，其支持的最低 rustc 版本就会升级到 1.75.0。不过，届时仍会缺失以下特性：

- c"" 字面量（在 1.77.0 版本稳定）。QEMU 提供了 c_str!() 宏，可轻松定义 CStr 常量，因此影响不大；

- offset_of!（在 1.77.0 版本稳定）。QEMU 大量使用 offset_of!()，并在 qemu_api crate 中提供了一个支持 const 的替代方案。另外需要注意的是，嵌套的 offset_of! 仅在 Rust 1.82.0 版本中才稳定下来，一旦我们开始更广泛地使用 BqlCell 和 BqlRefCell，它可能会派上用场。因此，我们在一段时间内仍会依赖自己的替代方案；

- &raw（在 1.82.0 版本稳定）。目前改用 addr_of! 和 addr_of_mut!，不过希望对原始指针的需求会随着时间的推移而减少；

- new_uninit（在 1.82.0 版本稳定）。pinned_init crate 内部会用到这个特性，但如果 QEMU vendored 了该 crate 的副本，那么可以很容易地对其进行补丁修复。
在常量中引用静态变量（在 1.83.0 版本稳定）。目前，唯一的替代方案是使用 const 函数。

QEMU 还支持 bindgen 0.60.x 版本，该版本缺失--generate-cstr 选项。此选项需要 0.66.x 版本，一旦不再需要支持这些旧版本，QEMU 就会采用该选项。这仅会影响少数几个地方，一旦重新添加该选项，这些地方就会因无法编译而很容易被找到。

[1]: https://lore.kernel.org/qemu-devel/20241108180139.117112-1-pbonzini@redhat.com/
[2]: https://lore.kernel.org/qemu-devel/20241122074756.282142-1-pbonzini@redhat.com/
[3]: https://lore.kernel.org/qemu-devel/20241125080507.115450-1-pbonzini@redhat.com/
[4]: https://gist.github.com/bonzini/2c63a905851f4f78573d022b1f196f4f
[5]: https://wiki.qemu.org/RustInQemu