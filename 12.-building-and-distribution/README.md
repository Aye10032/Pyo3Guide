# 构建与分发

本章指南详细介绍了如何使用 PyO3 构建和分发项目。实现这一目标的方法因项目是用 Rust 实现的 Python 模块还是嵌入 Python 的 Rust 二进制文件而异。对于这两种类型的项目，还有一些共同的问题，例如要构建的 Python 版本和使用的 [linker](https://en.wikipedia.org/wiki/Linker_(computing)) 参数。

本章的内容旨在为已经阅读过 PyO3 [README](./index.md) 的用户提供指导。它依次涵盖了 Python 模块和 Rust 二进制文件的选择。最后还有一节关于使用 PyO3 进行交叉编译项目的内容。

还有一个附加的小节专门讨论 [支持多个 Python 版本](./building-and-distribution/multiple-python-versions.md)。

## 配置 Python 版本

PyO3 使用构建脚本（由 [`pyo3-build-config`] crate 支持）来确定 Python 版本并设置正确的 linker 参数。默认情况下，它将按以下顺序尝试使用：
 - 任何活动的 Python 虚拟环境。
 - `python` 可执行文件（如果是 Python 3 解释器）。
 - `python3` 可执行文件。

您可以通过设置 `PYO3_PYTHON` 环境变量来覆盖 Python 解释器，例如 `PYO3_PYTHON=python3.7`、`PYO3_PYTHON=/usr/bin/python3.9`，甚至是 PyPy 解释器 `PYO3_PYTHON=pypy3`。

一旦找到 Python 解释器，`pyo3-build-config` 将执行它以查询 `sysconfig` 模块中的信息，这些信息对于配置其余的编译是必需的。

要验证 PyO3 将使用的配置，您可以运行一个设置了环境变量 `PYO3_PRINT_CONFIG=1` 的编译。下面是执行此操作的示例输出：

```console
$ PYO3_PRINT_CONFIG=1 cargo build
   Compiling pyo3 v0.14.1 (/home/david/dev/pyo3)
error: failed to run custom build command for `pyo3 v0.14.1 (/home/david/dev/pyo3)`

Caused by:
  process didn't exit successfully: `/home/david/dev/pyo3/target/debug/build/pyo3-7a8cf4fe22e959b7/build-script-build` (exit status: 101)
  --- stdout
  cargo:rerun-if-env-changed=PYO3_CROSS
  cargo:rerun-if-env-changed=PYO3_CROSS_LIB_DIR
  cargo:rerun-if-env-changed=PYO3_CROSS_PYTHON_VERSION
  cargo:rerun-if-env-changed=PYO3_PRINT_CONFIG

  -- PYO3_PRINT_CONFIG=1 is set, printing configuration and halting compile --
  implementation=CPython
  version=3.8
  shared=true
  abi3=false
  lib_name=python3.8
  lib_dir=/usr/lib
  executable=/usr/bin/python
  pointer_width=64
  build_flags=
  suppress_build_script_link_lines=false
```

`PYO3_ENVIRONMENT_SIGNATURE` 环境变量可用于在其值更改时触发重建，它没有其他效果。

### 高级：配置文件

如果您将上述 `PYO3_PRINT_CONFIG` 的输出配置保存到文件中，可以手动覆盖内容并通过 `PYO3_CONFIG_FILE` 环境变量将其反馈给 PyO3。

如果您的构建环境异常到 PyO3 的常规配置检测无法正常工作，使用这样的配置文件将为您提供灵活性，使 PyO3 能够为您工作。要查看支持的完整选项集，请参阅 [`InterpreterConfig` struct](https://docs.rs/pyo3-build-config/{{#PYO3_DOCS_VERSION}}/pyo3_build_config/struct.InterpreterConfig.html) 的文档。

## 构建 Python 扩展模块

Python 扩展模块的编译方式取决于它们所编译的操作系统（和架构）。除了多个操作系统（和架构）外，还有许多不同的 Python 版本被积极支持。上传到 [PyPI](https://pypi.org/) 的包通常希望上传覆盖多个操作系统/架构/版本组合的预构建 "wheels"，以便所有这些不同平台上的用户无需自己编译包。包供应商可以选择使用 "abi3" 限制的 Python API，这允许他们的 wheels 在多个 Python 版本上使用，从而减少需要编译的 wheels 数量，但限制了他们可以使用的功能。

有多种方法可以实现这一点：可以使用 `cargo` 来构建扩展模块（以及一些手动工作，具体取决于操作系统）。PyO3 生态系统有两个打包工具，[`maturin`] 和 [`setuptools-rust`]，它们抽象了操作系统之间的差异，并支持为 PyPI 上传构建 wheels。

PyO3 有一些 Cargo 特性来配置项目以构建 Python 扩展模块：
 - `extension-module` 特性，构建 Python 扩展模块时必须启用。
 - `abi3` 特性及其版本特定的 `abi3-pyXY` 伴随特性，用于选择使用有限的 Python API，以支持单个 wheel 中的多个 Python 版本。

本节描述了每个打包工具，然后描述如何在没有它们的情况下手动构建。接下来将解释 `extension-module` 特性。最后，有一节描述 PyO3 的 `abi3` 特性。

### 打包工具

PyO3 生态系统有两个主要选择来抽象开发 Python 扩展模块的过程：
- [`maturin`] 是一个命令行工具，用于构建、打包和上传 Python 模块。它对项目布局做出了明确的选择，因此需要很少的配置。这使得它成为从头开始构建 Python 扩展的用户的绝佳选择，而不需要灵活性。
- [`setuptools-rust`] 是 `setuptools` 的一个附加组件，它向 `setup.py` 配置文件添加额外的关键字参数。它比 `maturin` 需要更多的配置，但这为将 Rust 添加到现有 Python 包的用户提供了额外的灵活性，这些包无法满足 `maturin` 的约束。

请查阅每个项目的文档以获取有关如何开始使用它们以及如何将 wheels 上传到 PyPI 的完整细节。需要注意的是，虽然 `maturin` 能够开箱即用地构建 [manylinux](https://github.com/pypa/manylinux)-兼容的 wheels，但 `setuptools-rust` 需要更多的努力，[依赖于 Docker](https://setuptools-rust.readthedocs.io/en/latest/building_wheels.html) 来实现这一目的。

在 PyO3 仓库中还有 [`maturin-starter`] 和 [`setuptools-rust-starter`] 示例。

### 手动构建

要手动构建基于 PyO3 的 Python 扩展，首先在使用 PyO3 的 `extension-module` 特性并具有 [`cdylib` crate 类型](https://doc.rust-lang.org/cargo/reference/cargo-targets.html#the-crate-type-field) 的库项目中正常运行 `cargo build`。

构建完成后，将 Cargo 的 `target/` 目录中的共享库符号链接（或复制）并重命名到您希望的输出目录：
- 在 macOS 上，将 `libyour_module.dylib` 重命名为 `your_module.so`。
- 在 Windows 上，将 `libyour_module.dll` 重命名为 `your_module.pyd`。
- 在 Linux 上，将 `libyour_module.so` 重命名为 `your_module.so`。

然后，您可以在输出目录中打开 Python shell，并能够运行 `import your_module`。

如果您正在打包库以供重新分发，您应该通过在其名称中包含 [platform tag](#platform-tags) 来指示您的库是为哪个 Python 解释器编译的。这可以防止不兼容的解释器尝试导入您的库。如果您为 PyPy 编译，您 *必须* 包含平台标签，否则 PyPy 将忽略该模块。

#### Bazel 构建

要在 bazel 中使用 PyO3，需要手动配置 PyO3、PyO3-ffi 和 PyO3-macros。特别是，需要确保它是使用您打算使用的版本的正确 Python 标志编译的。
例如，请参见：
1. https://github.com/OliverFM/pytorch_with_gazelle -- 一个可以使用 PyO3、PyTorch 和 Gazelle 生成 Python 构建文件的最小示例的仓库。
2. https://github.com/TheButlah/rules_pyo3 -- 具有更广泛支持，但已过时。

#### 平台标签

与上述建议的仅使用 `.so` 或 `.pyd` 扩展名（取决于操作系统）不同，您可以在共享库扩展名前加上平台标签，以指示它与哪个解释器兼容。您可以从 `sysconfig` 模块查询解释器的平台标签。以下是一些示例输出：

```bash
# macOS 上的 CPython 3.10
.cpython-310-darwin.so

# Linux 上的 PyPy 7.3（Python 3.8）
$ python -c 'import sysconfig; print(sysconfig.get_config_var("EXT_SUFFIX"))'
.pypy38-pp73-x86_64-linux-gnu.so
```

因此，例如，在 macOS 上 CPython 3.10 的有效模块库名称是 `your_module.cpython-310-darwin.so`，而在 Linux 上为 PyPy 7.3 编译时的等效名称将是 `your_module.pypy38-pp73-x86_64-linux-gnu.so`。

有关平台标签的更多背景信息，请参见 [PEP 3149](https://peps.python.org/pep-3149/)。

#### macOS

在 macOS 上，由于 `extension-module` 特性禁用了对 `libpython` 的链接（[见下一节](#the-extension-module-feature)），需要设置一些额外的 linker 参数。`maturin` 和 `setuptools-rust` 都会自动为 PyO3 传递这些参数，但使用手动构建的项目需要直接设置这些参数以支持 macOS。

设置正确的 linker 参数的最简单方法是添加一个 [`build.rs`](https://doc.rust-lang.org/cargo/reference/build-scripts.html)，内容如下：

```rust,ignore
fn main() {
    pyo3_build_config::add_extension_module_link_args();
}
```

请记得在 `Cargo.toml` 的 `build-dependencies` 部分中添加 `pyo3-build-config`。

使用 `pyo3-build-config` 的替代方法是在 cargo 配置文件（例如 `.cargo/config.toml`）中添加以下内容：

```toml
[target.x86_64-apple-darwin]
rustflags = [
  "-C", "link-arg=-undefined",
  "-C", "link-arg=dynamic_lookup",
]

[target.aarch64-apple-darwin]
rustflags = [
  "-C", "link-arg=-undefined",
  "-C", "link-arg=dynamic_lookup",
]
```

使用 macOS 系统的 python3（`/usr/bin/python3`，与通过 homebrew、pyenv、nix 等安装的 python 相对）可能会导致运行时错误，例如 `Library not loaded: @rpath/Python3.framework/Versions/3.8/Python3`。这些可以通过在 `.cargo/config.toml` 中添加另一个配置来解决：

```toml
[build]
rustflags = [
  "-C", "link-args=-Wl,-rpath,/Library/Developer/CommandLineTools/Library/Frameworks",
]
```

或者，可以在 `build.rs` 中包含：

```rust
fn main() {
    println!(
        "cargo:rustc-link-arg=-Wl,-rpath,/Library/Developer/CommandLineTools/Library/Frameworks"
    );
}
```

有关 macOS 链接问题的更多讨论和解决方法，请参见 [此问题](https://github.com/PyO3/pyo3/issues/1800#issuecomment-906786649)。

最后，请记住，在 macOS 上，`extension-module` 特性将导致 `cargo test` 失败，除非使用 `--no-default-features` 标志（请参见 [常见问题](https://pyo3.rs/main/faq.html#i-cant-run-cargo-test-or-i-cant-build-in-a-cargo-workspace-im-having-linker-issues-like-symbol-not-found-or-undefined-reference-to-_pyexc_systemerror)）。

### `extension-module` 特性

PyO3 的 `extension-module` 特性用于禁用对 Unix 目标的 `libpython` 的链接。

这是必要的，因为默认情况下 PyO3 会链接到 `libpython`。这使得二进制文件、测试和示例 "正常工作"。然而，Unix 上的 Python 扩展必须不链接到 libpython，以符合 [manylinux](https://www.python.org/dev/peps/pep-0513/) 规范。

不链接到 `libpython` 的缺点是，二进制文件、测试和示例（通常嵌入 Python）将无法构建。如果您在单个项目中有扩展模块以及其他输出，则需要使用可选的 Cargo 特性在不构建扩展模块时禁用 `extension-module`。有关示例解决方法，请参见 [常见问题](faq.md#i-cant-run-cargo-test-or-i-cant-build-in-a-cargo-workspace-im-having-linker-issues-like-symbol-not-found-or-undefined-reference-to-_pyexc_systemerror)。

### `Py_LIMITED_API`/`abi3`

默认情况下，Python 扩展模块只能与其编译时所用的相同 Python 版本一起使用。例如，为 Python 3.5 构建的扩展模块无法在 Python 3.8 中导入。[PEP 384](https://www.python.org/dev/peps/pep-0384/) 引入了有限 Python API 的概念，它将具有稳定的 ABI，使得使用它构建的扩展模块可以在多个 Python 版本中使用。这也被称为 `abi3`。

使用有限 Python API 构建扩展模块的优点是，包供应商只需构建和分发单个副本（针对每个操作系统/架构），用户可以在所有从 [最低版本](#minimum-python-version-for-abi3) 及以上的 Python 版本上安装它。缺点是 PyO3 无法使用依赖于已知确切 Python 版本编译的优化。您需要决定这对您的扩展模块是否重要。还可以设计您的扩展模块，以便您可以分发 `abi3` wheels，但允许从源代码编译的用户受益于额外的优化 - 请参见本指南的 [支持多个 Python 版本](./building-and-distribution/multiple-python-versions.md) 部分，特别是 `#[cfg(Py_LIMITED_API)]` 标志。

在构建 Python 包作为 wheels 时，使用 `abi3` 涉及三个步骤：

1. 在 `pyo3` 中启用 `abi3` 特性。这确保 `pyo3` 仅调用属于稳定 API 的 Python C-API 函数，并且在 Windows 上还确保项目链接到正确的共享对象（在其他平台上不需要特殊行为）：

```toml
[dependencies]
pyo3 = { {{#PYO3_CRATE_VERSION}}, features = ["abi3"] }
```

2. 确保构建的共享对象正确标记为 `abi3`。这通过告诉构建系统您正在使用有限 API 来实现。[`maturin`] >= 0.9.0 和 [`setuptools-rust`] >= 0.11.4 支持 `abi3` wheels。
请参见 [相应的](https://github.com/PyO3/maturin/pull/353) [PRs](https://github.com/PyO3/setuptools-rust/pull/82) 以获取更多信息。

3. 确保 `.whl` 正确标记为 `abi3`。对于使用 `setuptools` 的项目，通过将 `--py-limited-api=cp3x`（其中 `x` 是 wheel 支持的最低 Python 版本，例如 `--py-limited-api=cp35` 对于 Python 3.5）传递给 `setup.py bdist_wheel` 来实现。

#### `abi3` 的最低 Python 版本

由于单个 `abi3` wheel 可以与许多不同的 Python 版本一起使用，PyO3 有特性标志 `abi3-py37`、`abi3-py38`、`abi3-py39` 等，以设置 `abi3` wheel 的最低所需 Python 版本。
例如，如果您设置 `abi3-py37` 特性，则您的扩展 wheel 可以在所有 Python 3 版本中使用，从 Python 3.7 开始。`maturin` 和 `setuptools-rust` 将为 wheel 赋予类似 `my-extension-1.0-cp37-abi3-manylinux2020_x86_64.whl` 的名称。

由于您的扩展模块可能会在多个不同的 Python 版本上运行，您可能会发现需要在运行时检查 Python 版本以自定义行为。请参见本指南的 [相关部分](./building-and-distribution/multiple-python-versions.md#checking-the-python-version-at-runtime) 以支持运行时的多个 Python 版本。

PyO3 只能将您的扩展模块链接到不超过您主机 Python 版本的 abi3 版本。例如，如果您设置 `abi3-py38` 并尝试使用 Python 3.7 的主机编译 crate，则构建将失败。

> 注意：如果您设置了多个 `abi3` 版本特性标志，则最低版本始终优先。例如，如果同时设置了 `abi3-py37` 和 `abi3-py38`，则 PyO3 将构建一个支持 Python 3.7 及以上的 wheel。

#### 在没有 Python 解释器的情况下构建 `abi3` 扩展

作为一项高级特性，您可以在设置环境变量 `PYO3_NO_PYTHON` 的情况下构建 PyO3 wheel，而无需调用 Python 解释器。
此外，如果构建主机的 Python 解释器未找到或过旧或其他不可用，PyO3 仍会在显示警告消息后尝试编译 `abi3` 扩展模块。
在类 Unix 系统上，这无条件有效；在 Windows 上，您还必须设置 `RUSTFLAGS` 环境变量，使其包含 `-L native=/path/to/python/libs`，以便链接器可以找到 `python3.lib`。

如果 `python3.dll` 导入库不可用，可以启用实验性的 `generate-import-lib` crate 特性，所需的库将由 PyO3 自动创建和使用。

*注意*：MSVC 目标需要在 `PATH` 中可用的 LLVM binutils (`llvm-dlltool`)，以使自动导入库生成特性正常工作。

#### 缺失特性

由于 Python API 的限制，有一些 `pyo3` 特性在为 `abi3` 编译时无法使用。这些特性包括：

- `#[pyo3(text_signature = "...")]` 在 Python 3.10 或更高版本之前不适用于类。
- 类上的 `dict` 和 `weakref` 选项在 Python 3.9 或更高版本之前不受支持。
- 缓冲区 API 在 Python 3.11 或更高版本之前不受支持。
- 依赖于确切 Python 版本编译的优化。

## 在 Rust 中嵌入 Python

如果您想在 Rust 程序中嵌入 Python 解释器，可以通过两种模式实现：动态和静态。我们将在以下部分中介绍这两种模式。每种模式都会影响您如何分发程序。与其自己学习如何做到这一点，您可能想考虑使用像 [PyOxidizer] 这样的项目，将您的应用程序及其所有依赖项打包成一个文件。

PyO3 会根据您配置的 Python 发行版（[见上文](#configuring-the-python-version)）是否包含共享库或静态库，自动在这两种链接模式之间切换。静态库通常在未使用 `--enable-shared` 配置选项从源代码编译的 Python 发行版中看到。

### 动态嵌入 Python 解释器

动态嵌入 Python 解释器比静态嵌入要简单得多。这是通过将您的程序链接到 Python 共享库（例如 UNIX 上的 `libpython.3.9.so`，或 Windows 上的 `python39.dll`）来实现的。Python 解释器的实现位于共享库中。这意味着当操作系统运行您的 Rust 程序时，它还需要能够找到 Python 共享库。

这种嵌入模式非常适合需要访问 Python 解释器的 Rust 测试。它也非常适合安装在 Python 虚拟环境中的 Rust 软件，因为虚拟环境设置了适当的环境变量以定位正确的 Python 共享库。

要将程序分发给非技术用户，您需要考虑在分发中包含 Python 共享库，并设置包装脚本以设置正确的环境变量（例如 UNIX 上的 `LD_LIBRARY_PATH`，或 Windows 上的 `PATH`）。

请注意，PyPy 不能嵌入 Rust（或任何其他软件）。对此的支持在 [PyPy 问题跟踪器](https://github.com/pypy/pypy/issues/3836) 上进行跟踪。

### 静态嵌入 Python 解释器

静态嵌入 Python 解释器意味着将 Python 静态库的内容直接包含在您的 Rust 二进制文件中。这意味着要分发您的程序，您只需发送二进制文件：它在二进制文件中包含 Python 解释器！

在 Windows 上，几乎从不进行静态链接，因此 Python 发行版通常不包括静态库。以下信息仅适用于 UNIX。

Python 静态库通常称为 `libpython.a`。

静态链接有很多复杂性，列举如下。由于这些原因，PyO3 目前尚未对这种嵌入模式提供一流支持。有关更多信息和讨论您遇到的任何问题，请参见 [PyO3 GitHub 上的第 416 号问题](https://github.com/PyO3/pyo3/issues/416)。

在静态嵌入解释器时，[`auto-initialize`](features.md#auto-initialize) 特性故意被禁用，因为这通常是新用户在运行测试程序时无意中完成的。使用动态嵌入测试 PyO3 要容易得多。

已知的复杂性包括：
  - 要导入编译的扩展模块（例如其他 Rust 扩展模块或用 C 编写的模块），您的二进制文件在编译时必须设置正确的链接器标志，以导出 `libpython.a` 的原始内容，以便扩展可以使用它们（例如 `-Wl,--export-dynamic`）。
  - 创建 `libpython.a` 时使用的 C 编译器和标志必须与您的 Rust 编译器和标志兼容，否则您将遇到编译失败。

    显著不同的编译器版本可能会出现如下错误：

    ```text
    lto1: fatal error: bytecode stream in file 'rust-numpy/target/release/deps/libpyo3-6a7fb2ed970dbf26.rlib' generated with LTO version 6.0 instead of the expected 6.2
    ```

    不匹配的标志可能导致如下错误：

    ```text
    /usr/bin/ld: /usr/lib/gcc/x86_64-linux-gnu/9/../../../x86_64-linux-gnu/libpython3.9.a(zlibmodule.o): relocation R_X86_64_32 against `.data' can not be used when making a PIE object; recompile with -fPIE
    ```

如果您在静态链接解释器时遇到这些或其他复杂性，请在 [PyO3 GitHub 上的第 416 号问题](https://github.com/PyO3/pyo3/issues/416) 上讨论。希望最终讨论将包含足够的信息和解决方案，以便 PyO3 能够为静态嵌入提供一流支持。

### 嵌入 Python 解释器时导入模块

当您运行带有嵌入解释器的 Rust 二进制文件时，任何创建的 `#[pymodule]` 模块在嵌入的解释器初始化之前都不会被添加到名为 `PyImport_Inittab` 的表中，因此无法导入。这将导致您在嵌入的解释器中执行的 Python 语句，如 `import your_new_module` 失败。您可以在初始化 Python 解释器之前调用宏 [`append_to_inittab`]({{#PYO3_DOCS_URL}}/pyo3/macro.append_to_inittab.html) 来将模块函数添加到该表中。（Python 解释器将通过调用 `prepare_freethreaded_python`、`with_embedded_python_interpreter` 或 `Python::with_gil` 进行初始化，并启用 [`auto-initialize`](features.md#auto-initialize) 特性。）

## 交叉编译

得益于 Rust 出色的交叉编译支持，使用 PyO3 进行交叉编译相对简单。要开始，您需要一些软件：

* 目标的工具链。
* 针对您要针对的平台和使用的工具链的 Cargo `.config` 中的适当选项。
* 已为您的目标编译的 Python 解释器（在构建 "abi3" 扩展模块时可选）。
* 为您的主机构建的 Python 解释器，并通过 `PATH` 或设置 [`PYO3_PYTHON`](#configuring-the-python-version) 变量可用（在构建 "abi3" 扩展模块时可选）。

在获得上述内容后，您可以使用 Cargo 的 `--target` 标志构建交叉编译的 PyO3 模块。PyO3 的构建脚本将根据您的主机机器和所需目标检测到您正在尝试进行交叉编译。

在交叉编译时，PyO3 的构建脚本无法执行目标 Python 解释器以查询配置，因此您可能需要设置一些额外的环境变量：

* `PYO3_CROSS`：如果存在，此变量强制 PyO3 配置为交叉编译。
* `PYO3_CROSS_LIB_DIR`：此变量可以设置为包含目标的 libpython DSO 和与之相关的 `_sysconfigdata*.py` 文件的目录（对于类 Unix 目标），或 Windows 目标的 Python DLL 导入库。此变量仅在输出二进制文件必须显式链接到 libpython 时需要（例如，当目标为 Windows 和 Android 或嵌入 Python 解释器时），或者在绝对需要从 `_sysconfigdata*.py` 获取解释器配置时需要。
* `PYO3_CROSS_PYTHON_VERSION`：目标 Python 安装的主版本和次版本（例如 3.9）。如果 PyO3 无法从 `abi3-py3*` 特性确定要针对的版本，或者如果未设置 `PYO3_CROSS_LIB_DIR`，或者如果 `PYO3_CROSS_LIB_DIR` 中存在多个 Python 版本，则仅在此变量需要。
* `PYO3_CROSS_PYTHON_IMPLEMENTATION`：目标 Python 安装的 Python 实现名称（"CPython" 或 "PyPy"）。默认情况下假定为 CPython，除非为类 Unix 目标设置了 `PYO3_CROSS_LIB_DIR`，并且 PyO3 可以从 `_sysconfigdata*.py` 获取解释器配置。

一个实验性的 `pyo3` crate 特性 `generate-import-lib` 使用户能够在不设置 `PYO3_CROSS_LIB_DIR` 环境变量或提供任何 Windows Python 库文件的情况下交叉编译扩展模块。它使用外部 [`python3-dll-a`] crate 为 MinGW-w64 和 MSVC 编译目标生成 Python DLL 的导入库。
`python3-dll-a` 使用 binutils `dlltool` 程序为 MinGW-w64 目标生成 DLL 导入库。
可以通过设置 `PYO3_MINGW_DLLTOOL` 环境变量来覆盖交叉目标的默认 `dlltool` 命令名称。
*注意*：MSVC 目标需要在主机系统上可用的 LLVM binutils 或 MSVC 构建工具。
更具体地说，`python3-dll-a` 要求在针对 `*-pc-windows-msvc` 时，`llvm-dlltool` 或 `lib.exe` 可执行文件必须存在于 `PATH` 中。当设置 `ZIG_COMMAND` 环境变量为已安装的 Zig 程序名称（`"zig"` 或 `"python -m ziglang"`）时，可以使用 Zig 编译器可执行文件替代 `llvm-dlltool`。

一个示例可能如下所示（假设您的目标 sysroot 位于 `/home/pyo3/cross/sysroot`，并且您的目标是 `armv7`）：

```sh
export PYO3_CROSS_LIB_DIR="/home/pyo3/cross/sysroot/usr/lib"

cargo build --target armv7-unknown-linux-gnueabihf
```

如果交叉库目录中有多个 Python 版本，并且您无法设置更精确的位置以包含 libpython DSO 和 `_sysconfigdata*.py` 文件，则可以设置所需版本：
```sh
export PYO3_CROSS_PYTHON_VERSION=3.8
export PYO3_CROSS_LIB_DIR="/home/pyo3/cross/sysroot/usr/lib"

cargo build --target armv7-unknown-linux-gnueabihf
```

或者另一个示例，使用相同的 sysroot 但为 Windows 构建：
```sh
export PYO3_CROSS_PYTHON_VERSION=3.9
export PYO3_CROSS_LIB_DIR="/home/pyo3/cross/sysroot/usr/lib"

cargo build --target x86_64-pc-windows-gnu
```

在上述示例中，可以启用任何 `abi3-py3*` 特性，而不是设置 `PYO3_CROSS_PYTHON_VERSION`。

在为 Unix 和 macOS 目标交叉编译扩展模块时，通常可以省略 `PYO3_CROSS_LIB_DIR`，或者在为 Windows 交叉编译扩展模块并启用实验性的 `generate-import-lib` crate 特性时。

以下资源也可能对交叉编译有用：
 - [github.com/japaric/rust-cross](https://github.com/japaric/rust-cross) 是关于 Rust 交叉编译的入门指南。
 - [github.com/rust-embedded/cross](https://github.com/rust-embedded/cross) 使用 Docker 使 Rust 交叉编译更容易。

[`pyo3-build-config`]: https://github.com/PyO3/pyo3/tree/main/pyo3-build-config
[`maturin-starter`]: https://github.com/PyO3/pyo3/tree/main/examples/maturin-starter
[`setuptools-rust-starter`]: https://github.com/PyO3/pyo3/tree/main/examples/setuptools-rust-starter
[`maturin`]: https://github.com/PyO3/maturin
[`setuptools-rust`]: https://github.com/PyO3/setuptools-rust
[PyOxidizer]: https://github.com/indygreg/PyOxidizer
[`python3-dll-a`]: https://docs.rs/python3-dll-a/latest/python3_dll_a/