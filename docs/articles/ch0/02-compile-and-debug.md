# 开发环境搭建与调试

## 背景与路线

QEMU 社区已将引入 Rust 视为既定路线：在 9.2 起 `--enable-rust` 面向开发者开放，10.x 期望默认启用，11.x 以后 Rust 成为强制依赖。

目前 QEMU 的 Rust 支持栈是通过在 `meson/configure` 中新增 `--enable-rust` 来开启 Rust 相关的源码编译，`scripts/cargo_wrapper.py` 负责在 Meson 里驱动 Cargo 组织 Rust 的源码编译，`rust/qemu-api`、`rust/util` 等 crate 提供 FFI 绑定。

当前的测试策略，要求在所有测试进程里设置 `RUST_BACKTRACE=1` 以保留 panic 踪迹。

## 平台与依赖规划

| 组件 | 建议版本 | 说明 |
| --- | --- | --- |
| 主机操作系统 | 64 位 Linux（x86_64 / aarch64） | `rust/meson.build` 会拒绝未列入 `supported_oses/supported_cpus` 的组合，避免出现不受支持的平台。 |
| Rust toolchain | 最新 stable（至少 1.76+，建议使用随 rustup 更新的 stable） | 邮件中提到的宏实现依赖未来的 1.83 特性，因此务必保持工具链最新。 |
| Meson / Ninja | Meson ≥ 1.5.0，Ninja ≥ 1.11 | `configure` 已显式要求 Meson 1.5.0 |
| LLVM / Clang / libclang | 与 Rust toolchain 对应版本 | `bindgen` 需要匹配的 clang/libclang，若自动检测失败需通过 `CLANG_PATH`、`LIBCLANG_PATH` 指定 |
| Python / pip | Python 3.10+，pip 可用 | Meson、`scripts/cargo_wrapper.py` 依赖 Python。 |

建议把 `rustup`, `meson`, `ninja`, `clang`, `pkg-config`, `glib2-devel`, `capstone`, `pixman` 等依赖交由系统包管理器安装，再使用 `rustup` 维护 Rust。

Rust toolchain 安装示例

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup self update
rustup toolchain install stable
rustup default stable
rustup component add rust-src rustfmt clippy
```

如需仅为 QEMU 使用特定工具链，可在仓库根目录创建 `.cargo/config.toml` 指向 `rust-toolchain`，但推荐直接使用 upstream 的 `rust-toolchain.toml` 约束以避开偏差。

## 获取源码

```bash
git clone https://gitlab.com/qemu-project/qemu.git qemu-rust
cd qemu-rust
git submodule update --init --recursive
```

若本地需要长期跟踪主线，可在 `origin/master` 基础上 `rebase`，但记得同步更新 `meson`、`rust` 子模块。如果进行 rust for qemu 开发，建议直接使用 master 主分支，随时同步上游最新进展。

## 构建流程

推荐使用 out-of-tree 构建，以下命令示例涵盖 KVM、aarch64/x86_64 系统仿真、Rust 组件与严格 lint：

```bash
mkdir build-rust && cd build-rust
../configure \
  --enable-kvm \
  --target-list="aarch64-softmmu,x86_64-softmmu" \
  --with-coroutine=sigaltstack \
  --enable-rust \
  --enable-strict-rust-lints
```

`configure` 会检查 `rustc`, `cargo`, `bindgen`，并把 `-Drust=enabled` 传递给 Meson 生成最终的编译配置文件。

如果遇到 rust package 相关的报错，需要把 `qemu/subprojects` 下 rust 相关的文件夹删除后重新 configure。

之后编译 QEMU 源码，下面提供两种编译方式：

```bash
cd build-rust
ninja -j$(nproc)
# or
./pyvenv/bin/meson compile -C .
```

编译构建流程会通过 Meson：

1. 调用 `scripts/cargo_wrapper.py`，把 `config-host.h`、构建 / 源路径、`rs_build_type` 传给 Cargo，以保持 C / Rust 侧配置一致；

2. 生成 `rust/qemu-api`, `rust/util` 等 crate 的 FFI 绑定，并链接到最终的 QEMU 可执行文件。

## 常用 make 命令

```bash
# 调用 Rust 的静态分析器
make clippy

# 相当于 cargo fmt --check
make rustfmt

# 构建 rust 文档
make rustdoc
```

## 运行与验证

1. **单元 / 集成测试**

```bash
cd build-rust
RUST_BACKTRACE=1 ./pyvenv/bin/meson test -C . --suite rust
```

设置 `RUST_BACKTRACE=1` 可对齐邮件中对 CI 的要求，在 panic 时提供完整堆栈（见 2024-01-23-292.txt:46500-46545）。

2. **系统仿真验证**（以 Rust 版 PL011 为例）：

```bash
./qemu-system-aarch64 \
  -machine virt \
  -cpu cortex-a57
  -m 1G \
  -device pl011-rust \
  -kernel arm64-baremetal-demo/workload.bin \
  -nographic
```

Rust 设备目前通过 `-device <name>` 与传统驱动共存；未来 9.2+ 会陆续加入 `-rust-device`/`-x-device` 短选项。

下一章节，我们介绍如何使用 Rust 编写设备模型。