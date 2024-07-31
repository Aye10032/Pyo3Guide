# 安装

要开始使用 PyO3，您需要三样东西：Rust 工具链、Python 环境和构建方式。我们将在下面逐一介绍这些内容。

> 如果您想与 PyO3 的维护者和其他 PyO3 用户交流，可以考虑加入 [PyO3 Discord 服务器](https://discord.gg/33kcChzH7f)。我们很想听听您开始使用的经验，以便让 PyO3 对每个人都尽可能易于访问！

## Rust

首先，确保您的系统上已安装 Rust。如果您还没有安装，请尝试按照 [这里](https://www.rust-lang.org/tools/install) 的说明进行操作。PyO3 可以在 `stable` 和 `nightly` 版本上运行，因此您可以选择最适合您的版本。所需的最低 Rust 版本为 1.63。

如果您可以运行 `rustc --version` 并且版本足够新，那么您就可以开始了！

## Python

要使用 PyO3，您至少需要 Python 3.7。虽然您可以直接使用系统上的默认 Python 解释器，但建议使用虚拟环境。

## 虚拟环境

虽然您可以使用任何您喜欢的虚拟环境管理器，但如果您想为多个不同的 Python 版本进行开发或测试，我们特别推荐使用 `pyenv`，本书中的示例将使用它。有关 `pyenv` 的安装说明可以在 [这里](https://github.com/pyenv/pyenv#getting-pyenv) 找到。（注意：要获取 `pyenv activate` 和 `pyenv virtualenv` 命令，您还需要安装 [`pyenv-virtualenv`](https://github.com/pyenv/pyenv-virtualenv) 插件。[pyenv 安装程序](https://github.com/pyenv/pyenv-installer#installation--update--uninstallation) 将同时安装这两个。）

在使用 `pyenv` 安装时保留源代码是有用的，这样将来调试时可以查看原始源文件。这可以通过在 `pyenv install` 命令中传递 `--keep` 标志来实现。

例如：

```bash
pyenv install 3.12 --keep
```

### 构建

有许多构建和 Python 包管理系统，例如 [`setuptools-rust`](https://github.com/PyO3/setuptools-rust) 或 [手动构建](./building-and-distribution.md#manual-builds)。我们推荐使用 `maturin`，您可以在 [这里](https://maturin.rs/installation.html) 安装。它是为与 PyO3 一起使用而开发的，提供了最“开箱即用”的体验，特别是如果您打算发布到 PyPI。`maturin` 只是一个 Python 包，因此您可以像安装其他 Python 包一样添加它。

系统 Python:
```bash
pip install maturin --user
```

pipx:
```bash
pipx install maturin
```

pyenv:
```bash
pyenv activate pyo3
pip install maturin
```

poetry:
```bash
poetry add -G dev maturin
```

安装后，您可以运行 `maturin --version` 来检查是否正确安装。

# 开始一个新项目

首先，您应该创建一个文件夹和虚拟环境来容纳您的新项目。这里我们将使用推荐的 `pyenv`：

```bash
mkdir pyo3-example
cd pyo3-example
pyenv virtualenv pyo3
pyenv local pyo3
```

之后，您应该安装构建管理器。在这个例子中，我们将使用 `maturin`。激活虚拟环境后，将 `maturin` 添加到其中：

```bash
pip install maturin
```

现在您可以初始化新项目：

```bash
maturin init
```

如果 `maturin` 已经安装，您也可以直接使用它创建新项目：

```bash
maturin new -b pyo3 pyo3-example
cd pyo3-example
pyenv virtualenv pyo3
pyenv local pyo3
```

# 向现有项目添加

遗憾的是，`maturin` 目前无法在现有项目中运行，因此如果您想在现有项目中使用 Python，基本上有两个选择：

1. 按照上述方法创建一个新项目，并将现有代码移动到该项目中
2. 根据需要手动编辑项目配置

如果您选择第二个选项，请注意以下事项：

## Cargo.toml

确保您希望从 Python 访问的 Rust crate 被编译为库。您也可以有二进制输出，但您希望从 Python 访问的代码必须在库部分。此外，确保 crate 类型为 `cdylib`，并将 PyO3 添加为依赖项，如下所示：

```toml
# 如果您在 `Cargo.toml` 中已经有 [package] 信息，可以忽略
# 这一部分！
[package]
# 这里的 `name` 是包的名称。
name = "pyo3_start"
# 这些是良好的默认值：
version = "0.1.0"
edition = "2021"

[lib]
# 本地库的名称。这是将在 Python 中用于导入库的名称
# （即 `import string_sum`）。如果您更改此名称，您还必须更改
# `src/lib.rs` 中的 `#[pymodule]` 的名称。
name = "pyo3_example"

# "cdylib" 是生成供 Python 导入的共享库所必需的。
crate-type = ["cdylib"]

[dependencies]
pyo3 = { {{#PYO3_CRATE_VERSION}}, features = ["extension-module"] }
```

## pyproject.toml

您还应该创建一个 `pyproject.toml`，内容如下：

```toml
[build-system]
requires = ["maturin>=1,<2"]
build-backend = "maturin"

[project]
name = "pyo3_example"
requires-python = ">=3.7"
classifiers = [
    "Programming Language :: Rust",
    "Programming Language :: Python :: Implementation :: CPython",
    "Programming Language :: Python :: Implementation :: PyPy",
]
```

## 运行代码

之后，您可以设置 Rust 代码以便在 Python 中可用，如下所示；例如，您可以将此代码放在 `src/lib.rs` 中：

```rust
use pyo3::prelude::*;

/// 将两个数字的和格式化为字符串。
#[pyfunction]
fn sum_as_string(a: usize, b: usize) -> PyResult<String> {
    Ok((a + b).to_string())
}

/// 用 Rust 实现的 Python 模块。此函数的名称必须与
/// `Cargo.toml` 中的 `lib.name` 设置匹配，否则 Python 将无法
/// 导入该模块。
#[pymodule]
fn pyo3_example(m: &Bound<'_, PyModule>) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(sum_as_string, m)?)
}
```

现在您可以运行 `maturin develop` 来准备 Python 包，之后您可以像这样使用它：

```bash
$ maturin develop
# 在 maturin 运行编译时会有大量进度输出...
$ python
>>> import pyo3_example
>>> pyo3_example.sum_as_string(5, 20)
'25'
```

有关如何从 Rust 使用 Python 代码的更多说明，请参见 [Python from Rust](python-from-rust.md) 页面。

## Maturin 导入钩子

在开发过程中，代码中的任何更改都需要在测试之前运行 `maturin develop`。为了简化开发过程，您可能希望安装 [Maturin Import Hook](https://github.com/PyO3/maturin-import-hook)，它将在导入代码更改的库时自动运行 `maturin develop`。