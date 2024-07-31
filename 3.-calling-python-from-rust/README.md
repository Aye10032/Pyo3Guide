# 在Rust代码中调用Python

本指南的这一章节记录了一些从Rust与Python代码交互的方法。

下面是对`'py`生命周期的介绍，以及关于PyO3的API如何处理Python代码的一些一般性说明。

子章节还涵盖以下主题：
 - PyO3的API中可用的Python对象类型
 - 如何处理Python异常
 - 如何调用Python函数
 - 如何执行现有的Python代码

## `'py`生命周期

为了安全地与Python解释器交互，Rust线程必须具有相应的Python线程状态，并持有[全局解释器锁（GIL）](#the-global-interpreter-lock)。PyO3有一个`Python<'py>`令牌，用于证明这些条件得到了满足。其生命周期`'py`是PyO3 API的核心部分。

`Python<'py>`令牌有三个用途：

* 它提供了Python解释器的全局API，例如[`py.eval_bound()`][eval]和[`py.import_bound()`][import]。
* 它可以传递给需要证明持有GIL的函数，例如[`Py::clone_ref`][clone_ref]。
* 它的生命周期`'py`用于将PyO3的许多类型绑定到Python解释器，例如[`Bound<'py, T>`][Bound]。

绑定到`'py`生命周期的PyO3类型，例如`Bound<'py, T>`，都包含一个`Python<'py>`令牌。这意味着它们可以完全访问Python解释器，并提供与Python对象交互的完整API。

请查阅[PyO3的API文档][obtaining-py]以了解如何获取这些令牌之一。

### 全局解释器锁

Python中的并发编程得益于全局解释器锁（GIL），它确保只有一个Python线程可以同时使用Python解释器及其API。这使得它可以用于同步代码。有关基于GIL保证的同步工具，请参见[`pyo3::sync`]模块。

非Python操作（系统调用和本地Rust代码）可以解锁GIL。有关如何使用PyO3的API做到这一点，请参见[并行性部分](parallelism.md)。

## Python的内存模型

Python的内存模型与Rust的内存模型在两个关键方面有所不同：
- 没有所有权的概念；所有Python对象都是共享的，通常通过引用计数实现
- 没有独占（`&mut`）引用的概念；任何引用都可以修改Python对象

PyO3的API通过提供[智能指针][smart-pointers]类型`Py<T>`、`Bound<'py, T>`和（非常少用的）`Borrowed<'a, 'py, T>`来反映这一点。这些智能指针都使用Python的引用计数。有关这些类型的更多细节，请参见[类型子章节](./types.md)。

由于缺乏独占的`&mut`引用，PyO3对Python对象的API，例如[`PyListMethods::append`]，使用共享引用。这是安全的，因为Python对象具有内部机制来防止数据竞争（截至撰写时，Python GIL）。

[smart-pointers]: https://doc.rust-lang.org/book/ch15-00-smart-pointers.html
[obtaining-py]: {{#PYO3_DOCS_URL}}/pyo3/marker/struct.Python.html#obtaining-a-python-token
[`pyo3::sync`]: {{#PYO3_DOCS_URL}}/pyo3/sync/index.html