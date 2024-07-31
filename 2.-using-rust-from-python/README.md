# 2. Using Rust from Python

## 从 Python 使用 Rust

本章将解释如何将 Rust 代码封装成 Python 对象。

PyO3 使用 Rust 的“过程宏”提供一个强大而简单的 API，以指示 Rust 代码应如何映射到 Python 对象。

PyO3 可以生成的三种类型的 Python 对象是：

* Python 模块，通过 `#[pymodule]` 宏
* Python 函数，通过 `#[pyfunction]` 宏
* Python 类，通过 `#[pyclass]` 宏（加上 `#[pymethods]` 来定义这些类的方法）

接下来的子章节将依次介绍这些内容。
