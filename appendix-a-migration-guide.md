# 从旧版 PyO3 迁移

本指南可以帮助您通过破坏性更改将代码从一个 PyO3 版本升级到下一个版本。
有关所有更改的详细列表，请参见 [CHANGELOG](changelog.md)。

## 从 0.22.* 升级到 0.23

### 移除 `gil-refs` 特性
<details open>
<summary><small>点击展开</small></summary>

PyO3 0.23 完成了对 "GIL Refs" API 的移除，取而代之的是在 PyO3 0.21 中引入的新 "Bound" API。

随着旧 API 的移除，许多带有 `_bound` 后缀的 "Bound" API 函数不再需要这些后缀，因为这些名称已经被释放。例如，`PyTuple::new_bound` 现在仅为 `PyTuple::new`（现有名称仍然存在，但已被弃用）。

之前：

```rust
# #![allow(deprecated)]
# use pyo3::prelude::*;
# use pyo3::types::PyTuple;
# fn main() {
# Python::with_gil(|py| {
// 例如，对于 PyTuple。许多这样的 API 已被更改。
let tup = PyTuple::new_bound(py, [1, 2, 3]);
# })
# }
```

之后：

```rust
# use pyo3::prelude::*;
# use pyo3::types::PyTuple;
# fn main() {
# Python::with_gil(|py| {
// 例如，对于 PyTuple。许多这样的 API 已被更改。
let tup = PyTuple::new(py, [1, 2, 3]);
# })
# }
```
</details>

## 从 0.21.* 升级到 0.22

### 继续弃用 `gil-refs` 特性
<details open>
<summary><small>点击展开</small></summary>

在 PyO3 0.21 中引入 "Bound" API 之后，计划移除 "GIL Refs" API，所有与 GIL Refs 相关的功能现在都被限制在 `gil-refs` 特性下，并在使用时发出弃用警告。

请参见 <a href="#from-021-to-022">0.21 迁移条目</a> 以获取升级帮助。
</details>

### 弃用尾随可选参数的隐式默认值
<details open>
<summary><small>点击展开</small></summary>

在 `pyo3` 0.22 中，尾随 `Option<T>` 类型参数的隐式 `None` 默认值已被弃用。要迁移，请在受影响的函数或方法上放置 `#[pyo3(signature = (...))]` 属性，并指定所需的行为。
迁移警告指定了相应的签名以保持当前行为。在 0.23 中，任何包含 `Option<T>` 类型参数的函数都将要求提供签名，以防止意外和未注意到的行为变化。在 0.24 中，这一限制将再次被解除，`Option<T>` 类型参数将被视为其他参数 _没有_ 特殊处理。

之前：

```rust
# #![allow(deprecated, dead_code)]
# use pyo3::prelude::*;
#[pyfunction]
fn increment(x: u64, amount: Option<u64>) -> u64 {
    x + amount.unwrap_or(1)
}
```

之后：

```rust
# #![allow(dead_code)]
# use pyo3::prelude::*;
#[pyfunction]
#[pyo3(signature = (x, amount=None))]
fn increment(x: u64, amount: Option<u64>) -> u64 {
    x + amount.unwrap_or(1)
}
```
</details>

### `Py::clone` 现在被限制在 `py-clone` 特性下
<details open>
<summary><small>点击展开</small></summary>
如果您依赖 `impl<T> Clone for Py<T>` 来满足现有 Rust 代码的特征要求，而这些代码并未考虑基于 PyO3 的代码，则必须启用新引入的 `py-clone` 特性。

但是，请注意，行为与以前的版本不同。如果在未持有 GIL 的情况下调用 `Clone`，我们尝试延迟这些引用计数增量的应用，直到 PyO3 基于的代码重新获取它。这被证明无法以安全的方式实现，因此被移除。现在，如果在未持有 GIL 的情况下调用 `Clone`，我们将引发 panic，而调用代码可能没有准备好。

与此相关，我们还添加了一个 `pyo3_disable_reference_pool` 条件编译标志，该标志移除了应用延迟引用计数减少所需的基础设施，这些减少是由 `impl<T> Drop for Py<T>` 隐含的。它们似乎不是安全性隐患，因为在最坏的情况下会导致内存泄漏。然而，全球同步会显著增加跨越 Python-Rust 边界的开销。启用此特性将消除这些成本，并使 `Drop` 实现如果在未持有 GIL 的情况下被调用则中止进程。
</details>

### 对于简单枚举，要求显式选择比较
<details open>
<summary><small>点击展开</small></summary>

在 `pyo3` 0.22 中，新的 `#[pyo3(eq)]` 选项允许使用 Rust 的 `PartialEq` 自动实现 Python 相等性。之前，简单枚举会根据其判别式自动实现相等性。为了使 PyO3 更加一致，这种自动相等性实现被弃用，取而代之的是对所有 `#[pyclass]` 类型进行选择性实现。同样，简单枚举支持与整数的比较，这不在 Rust 的 `PartialEq` 派生中，因此被拆分到 `#[pyo3(eq_int)]` 属性中。

要迁移，请在简单枚举类上放置 `#[pyo3(eq, eq_int)]` 属性。

之前：

```rust
# #![allow(deprecated, dead_code)]
# use pyo3::prelude::*;
#[pyclass]
enum SimpleEnum {
    VariantA,
    VariantB = 42,
}
```

之后：

```rust
# #![allow(dead_code)]
# use pyo3::prelude::*;
#[pyclass(eq, eq_int)]
#[derive(PartialEq)]
enum SimpleEnum {
    VariantA,
    VariantB = 42,
}
```
</details>

### `PyType::name` 重新设计以更好地匹配 Python 的 `__name__`
<details open>
<summary><small>点击展开</small></summary>

此函数之前会尝试直接从 Python 类型对象的 C API 字段 (`tp_name`) 读取，在这种情况下，它会返回 `Cow::Borrowed`。然而，`tp_name` 的内容没有明确定义的语义。

相反，`PyType::name()` 现在返回相当于 Python 的 `__name__` 的内容，并返回 `PyResult<Bound<'py, PyString>>`。

与 PyO3 0.21 版本的 `PyType::name()` 最接近的等价物已作为新函数 `PyType::fully_qualified_name()` 引入，该函数相当于 `__module__` 和 `__qualname__` 连接为 `module.qualname`。

之前：

```rust,ignore
# #![allow(deprecated, dead_code)]
# use pyo3::prelude::*;
# use pyo3::types::{PyBool};
# fn main() -> PyResult<()> {
Python::with_gil(|py| {
    let bool_type = py.get_type_bound::<PyBool>();
    let name = bool_type.name()?.into_owned();
    println!("Hello, {}", name);

    let mut name_upper = bool_type.name()?;
    name_upper.to_mut().make_ascii_uppercase();
    println!("Hello, {}", name_upper);

    Ok(())
})
# }
```

之后：

```rust
# #![allow(dead_code)]
# use pyo3::prelude::*;
# use pyo3::types::{PyBool};
# fn main() -> PyResult<()> {
Python::with_gil(|py| {
    let bool_type = py.get_type_bound::<PyBool>();
    let name = bool_type.name()?;
    println!("Hello, {}", name);

    // （如果需要完整的点路径，请将 `name()` 切换为 `fully_qualified_name()`）
    let mut name_upper = bool_type.fully_qualified_name()?.to_string();
    name_upper.make_ascii_uppercase();
    println!("Hello, {}", name_upper);

    Ok(())
})
# }
```
</details>

## 从 0.20.* 升级到 0.21
<details>
<summary><small>点击展开</small></summary>

PyO3 0.21 引入了新的 `Bound<'py, T>` 智能指针，替代了现有的 "GIL Refs" API 来与 Python 对象交互。例如，在 PyO3 0.20 中，引用 `&'py PyAny` 将用于与 Python 对象交互。在 PyO3 0.21 中，更新后的类型为 `Bound<'py, PyAny>`。进行此更改将 Rust 所有权语义从 PyO3 的内部移出并转移到用户代码中。此更改修复了与 gevent 交互的已知安全性边缘情况，并改善了 CPU 和内存性能。有关讨论的完整历史，请参见 https://github.com/PyO3/pyo3/issues/3382。

"GIL Ref" `&'py PyAny` 和类似类型（如 `&'py PyDict`）继续作为弃用 API 可用。由于新 API 的优势，建议所有用户尽快进行升级。

除了主要的 API 类型重构外，PyO3 还需要对其他 API 进行一些小的破坏性调整，以弥补正确性和安全性差距。

更新到 PyO3 0.21 的推荐步骤如下：
  1. 启用 `gil-refs` 特性以消除与 API 更改相关的弃用警告
  2. 修复所有其他 PyO3 0.21 迁移步骤
  3. 禁用 `gil-refs` 特性并迁移离弃用的 API

以下部分按此顺序排列。
</details>

### 启用 `gil-refs` 特性
<details>
<summary><small>点击展开</small></summary>

为了使 PyO3 生态系统顺利过渡到 GIL Refs API，PyO3 0.21 中未更改任何消耗或生成 GIL Refs 的 API。相反，引入了使用 `Bound<T>` 智能指针的变体，例如返回 `Bound<PyTuple>` 的 `PyTuple::new_bound` 是 `PyTuple::new` 的替代形式。GIL Ref API 已被弃用，但为了简化迁移，可以通过启用 `gil-refs` 特性来禁用这些弃用警告。

> 唯一一个在原地更改的现有 API 是 `pyo3::intern!` 宏。几乎所有使用此宏的地方都不需要更新代码以考虑其更改为立即返回 `&Bound<PyString>`，并且添加 `intern_bound!` 替代被认为会给用户增加更多工作。

建议用户将此作为更新到 PyO3 0.21 的第一步，以便弃用警告不会妨碍解决其余迁移步骤。

之前：

```toml
# Cargo.toml
[dependencies]
pyo3 = "0.20"
```

之后：

```toml
# Cargo.toml
[dependencies]
pyo3 = { version = "0.21", features = ["gil-refs"] }
```
</details>

### `PyTypeInfo` 和 `PyTryFrom` 已被调整
<details>
<summary><small>点击展开</small></summary>

`PyTryFrom` 特性已经过时，其 `try_from` 方法现在与 2021 版前导中的 `TryFrom::try_from` 冲突。它的许多功能也与 `PyTypeInfo` 重复。

为了在弃用 GIL Refs API 的过程中收紧 PyO3 特性，`PyTypeInfo` 特性有了一个更简单的伴随 `PyTypeCheck`。`PyAny::downcast` 和 `PyAny::downcast_exact` 方法不再使用 `PyTryFrom` 作为约束，而是分别使用 `PyTypeCheck` 和 `PyTypeInfo`。

要迁移，请将所有类型转换切换为使用 `obj.downcast()` 而不是 `try_from(obj)`（对于 `downcast_exact` 也是如此）。

之前：

```rust,ignore
# #![allow(deprecated)]
# use pyo3::prelude::*;
# use pyo3::types::{PyInt, PyList};
# fn main() -> PyResult<()> {
Python::with_gil(|py| {
    let list = PyList::new(py, 0..5);
    let b = <PyInt as PyTryFrom>::try_from(list.get_item(0).unwrap())?;
    Ok(())
})
# }
```

之后：

```rust,ignore
# use pyo3::prelude::*;
# use pyo3::types::{PyInt, PyList};
# fn main() -> PyResult<()> {
Python::with_gil(|py| {
    // 请注意，PyList::new 在 GIL Refs API 移除过程中已被弃用，
    // 请参见下面关于迁移到 Bound<T> 的部分。
    #[allow(deprecated)]
    let list = PyList::new(py, 0..5);
    let b = list.get_item(0).unwrap().downcast::<PyInt>()?;
    Ok(())
})
# }
```
</details>

### `Iter(A)NextOutput` 已被弃用
<details>
<summary><small>点击展开</small></summary>

`__next__` 和 `__anext__` 魔法方法现在可以返回任何可以直接转换为 Python 对象的类型，就像所有其他 `#[pymethods]` 一样。`IterNextOutput` 用于 `__next__`，`IterANextOutput` 用于 `__anext__`，因此被弃用。最重要的是，这一变化允许从 `__anext__` 返回一个可等待对象，而无需非理性地将其包装在 `Yield` 或 `Some` 中。只有返回类型 `Option<T>` 和 `Result<Option<T>, E>` 仍以特殊方式处理，其中 `Some(val)` 产生 `val`，而 `None` 停止迭代。

以使用 `IterNextOutput` 实现 Python 迭代器为例，例如：

```rust,ignore
use pyo3::prelude::*;
use pyo3::iter::IterNextOutput;

#[pyclass]
struct PyClassIter {
    count: usize,
}

#[pymethods]
impl PyClassIter {
    fn __next__(&mut self) -> IterNextOutput<usize, &'static str> {
        if self.count < 5 {
            self.count += 1;
            IterNextOutput::Yield(self.count)
        } else {
            IterNextOutput::Return("done")
        }
    }
}
```

如果不需要通过 `StopIteration` 返回 `"done"`，则应将其写为：

```rust
use pyo3::prelude::*;

#[pyclass]
struct PyClassIter {
    count: usize,
}

#[pymethods]
impl PyClassIter {
    fn __next__(&mut self) -> Option<usize> {
        if self.count < 5 {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}
```

这种形式还有额外的好处：它在以前的 PyO3 版本中已经有效，它与 Rust 的 [`Iterator` trait](https://doc.rust-lang.org/stable/std/iter/trait.Iterator.html) 的签名相匹配，并且允许在 CPython 中使用快速路径，完全避免引发 `StopIteration` 异常的成本。请注意，使用 [`Option::transpose`](https://doc.rust-lang.org/stable/std/option/enum.Option.html#method.transpose) 和 `Result<Option<T>, E>` 变体时，这种形式也可以用于包装可失败的迭代器。

另外，实施也可以像在 Python 中一样进行，即通过“引发” `StopIteration` 异常：

```rust
use pyo3::prelude::*;
use pyo3::exceptions::PyStopIteration;

#[pyclass]
struct PyClassIter {
    count: usize,
}

#[pymethods]
impl PyClassIter {
    fn __next__(&mut self) -> PyResult<usize> {
        if self.count < 5 {
            self.count += 1;
            Ok(self.count)
        } else {
            Err(PyStopIteration::new_err("done"))
        }
    }
}
```

最后，异步迭代器可以直接返回一个可等待对象，而无需混淆地包装：

```rust
use pyo3::prelude::*;

#[pyclass]
struct PyClassAwaitable {
    number: usize,
}

#[pymethods]
impl PyClassAwaitable {
    fn __next__(&self) -> usize {
        self.number
    }

    fn __await__(slf: Py<Self>) -> Py<Self> {
        slf
    }
}

#[pyclass]
struct PyClassAsyncIter {
    number: usize,
}

#[pymethods]
impl PyClassAsyncIter {
    fn __anext__(&mut self) -> PyClassAwaitable {
        self.number += 1;
        PyClassAwaitable {
            number: self.number,
        }
    }

    fn __aiter__(slf: Py<Self>) -> Py<Self> {
        slf
    }
}
```
</details>

### `PyType::name` 已重命名为 `PyType::qualname`
<details>
<summary><small>点击展开</small></summary>

`PyType::name` 已重命名为 `PyType::qualname`，以表明它确实返回 [qualified name](https://docs.python.org/3/glossary.html#term-qualified-name)，与 `__qualname__` 属性相匹配。新添加的 `PyType::name` 现在返回包括模块名称的完整名称，这对应于 `__module__.__name__` 的属性级别。
</details>

### `PyCell` 已被弃用
<details>
<summary><small>点击展开</small></summary>

在 Rust 中实现的与 Python 对象的交互不再需要通过 `PyCell<T>`。相反，与 Python 对象的交互现在始终通过 `Bound<T>` 或 `Py<T>` 进行，无论 `T` 是原生 Python 对象还是在 Rust 中实现的 `#[pyclass]`。分别使用 `Bound::new` 或 `Py::new` 创建，并使用 `Bound::borrow(_mut)` / `Py::borrow(_mut)` 借用 Rust 对象。
</details>

### 从 GIL Refs API 迁移到 `Bound<T>`
<details>
<summary><small>点击展开</small></summary>

为了最小化使用 GIL Refs API 的代码的破坏，已通过为所有接受或返回 GIL Refs 的函数添加补充来引入 `Bound<T>` 智能指针。这允许通过用新 API 替换弃用的 API 来迁移代码。

要识别要迁移的内容，请暂时关闭 `gil-refs` 特性，以查看几乎所有使用接受和生成 GIL Refs 的 API 的弃用警告。通过一个或多个 PR，应该可以根据弃用提示更新代码。根据您的开发环境，关闭 `gil-refs` 特性可能会引入 [一些非常有针对性的破坏](#deactivating-the-gil-refs-feature)，因此您可能需要先修复这些问题。

例如，以下 API 已获得更新的变体：
- `PyList::new`、`PyTuple::new` 和类似构造函数已被替换为 `PyList::new_bound`、`PyTuple::new_bound` 等。
- `FromPyObject::extract` 有了新的 `FromPyObject::extract_bound`（请参见下面的部分）
- `PyTypeInfo` 特性已添加新的 `_bound` 方法以接受/返回 `Bound<T>`。

由于新的 `Bound<T>` API 将所有权从 PyO3 框架转移到用户代码，因此在切换到新 API 时，用户代码预计需要在几个地方进行调整：
- 代码需要偶尔添加 `&` 来借用新智能指针作为 `&Bound<T>` 以传递这些类型（或使用 `.clone()`，代价是增加 Python 引用计数）。
- `Bound<PyList>` 和 `Bound<PyTuple>` 不支持使用 `list[0]` 进行索引，您应该使用 `list.get_item(0)`。
- `Bound<PyTuple>::iter_borrowed` 比 `Bound<PyTuple>::iter` 更高效。`Bound<PyTuple>` 的默认迭代不能返回借用引用，因为 Rust 还没有“借用迭代器”。同样，`Bound<PyTuple>::get_borrowed_item` 比 `Bound<PyTuple>::get_item` 更高效，原因相同。
- `&Bound<T>` 不实现 `FromPyObject`（尽管在 GIL Refs API 完全移除后可能可以做到这一点）。使用 `bound_any.downcast::<T>()` 而不是 `bound_any.extract::<&Bound<T>>()`。
- `Bound<PyString>::to_str` 现在从 `Bound<PyString>` 借用，而不是从 `'py` 生命周期借用，因此在某些情况下，代码需要将智能指针存储为值，而不是之前仅用作临时的 `&PyString`。（有关此的更多详细信息，请参见 [下面的部分](#deactivating-the-gil-refs-feature)）。
- `.extract::<&str>()` 现在从源 Python 对象借用。更新的最简单方法是更改为 `.extract::<PyBackedStr>()`，这保留了 Python 引用的所有权。有关更多信息，请参见 [关于禁用 `gil-refs` 特性的部分](#deactivating-the-gil-refs-feature)。

要在 `&PyAny` 和 `&Bound<PyAny>` 之间进行转换，请使用 `as_borrowed()` 方法：

```rust,ignore
let gil_ref: &PyAny = ...;
let bound: &Bound<PyAny> = &gil_ref.as_borrowed();
```

要在 `Py<T>` 和 `Bound<T>` 之间进行转换，请使用 `bind()` / `into_bound()` 方法，使用 `as_unbound()` / `unbind()` 从 `Bound<T>` 返回到 `Py<T>`。

```rust,ignore
let obj: Py<PyList> = ...;
let bound: &Bound<'py, PyList> = obj.bind(py);
let bound: Bound<'py, PyList> = obj.into_bound(py);

let obj: &Py<PyList> = bound.as_unbound();
let obj: Py<PyList> = bound.unbind();
```

<div class="warning">

⚠️ 警告：悬空指针陷阱 💣

> 由于所有权的变化，使用 `.as_ptr()` 将 `&PyAny` 和其他 GIL Refs 转换为 `*mut pyo3_ffi::PyObject` 的代码应注意避免创建悬空指针，因为 `Bound<PyAny>` 现在携带所有权。
>
> 例如，以下使用 `Option<&PyAny>` 的模式在迁移到 `Bound<PyAny>` 智能指针时很容易创建悬空指针：
>
> ```rust,ignore
> let opt: Option<&PyAny> = ...;
> let p: *mut ffi::PyObject = opt.map_or(std::ptr::null_mut(), |any| any.as_ptr());
> ```
>
> 迁移此代码的正确方法是使用 `.as_ref()` 以避免在 `map_or` 闭包中丢弃 `Bound<PyAny>`：
>
> ```rust,ignore
> let opt: Option<Bound<PyAny>> = ...;
> let p: *mut ffi::PyObject = opt.as_ref().map_or(std::ptr::null_mut(), Bound::as_ptr);
> ```
<div>

#### 迁移 `FromPyObject` 实现

`FromPyObject` 有了一个新方法 `extract_bound`，该方法接受 `&Bound<'py, PyAny>` 作为参数，而不是 `&PyAny`。`extract` 和 `extract_bound` 都已根据对方提供默认实现，以避免在更新到 0.21 时立即破坏代码。

所有 `FromPyObject` 的实现应从 `extract` 切换到 `extract_bound`。

之前：

```rust,ignore
impl<'py> FromPyObject<'py> for MyType {
    fn extract(obj: &'py PyAny) -> PyResult<Self> {
        /* ... */
    }
}
```

之后：

```rust,ignore
impl<'py> FromPyObject<'py> for MyType {
    fn extract_bound(obj: &Bound<'py, PyAny>) -> PyResult<Self> {
        /* ... */
    }
}
```

预计在 0.22 中将删除 `extract_bound` 的默认实现，而在 0.23 中将删除 `extract`。

#### PyO3 无法发出 GIL Ref 弃用警告的情况

尽管 PyO3 生成了大量的弃用警告以帮助从 GIL Refs 迁移到 Bound API，但仍有一些情况 PyO3 无法自动警告使用 GIL Refs。值得在处理完所有弃用警告后手动检查这些情况：

- `FromPyObject` 特性的单个实现无法被弃用，因此 PyO3 无法警告使用像 `.extract<&PyAny>()` 这样的代码模式，这会生成 GIL Ref。
- `#[pyfunction]` 参数中的 GIL Refs 会发出警告，但如果 GIL Ref 被包装在另一个容器中，例如 `Vec<&PyAny>`，则 PyO3 无法对此发出警告。
- `wrap_pyfunction!(function)(py)` 延迟参数形式的 `wrap_pyfunction` 宏，接受 `py: Python<'py>`，会生成 GIL Ref，由于类型推断的限制，PyO3 无法对此特定情况发出警告。

</details>

### 禁用 `gil-refs` 特性
<details>
<summary><small>点击展开</small></summary>

作为迁移的最后一步，禁用 `gil-refs` 特性将为最佳性能设置代码，并旨在为 PyO3 0.22 设置向前兼容的 API。

此时，需要管理 GIL Ref 内存的代码可以安全地移除对 `GILPool` 的使用（这些使用是通过调用 `Python::new_pool` 和 `Python::with_pool` 构造的）。弃用警告将突出显示这些情况。

只有一段代码在禁用这些特性时会发生变化：直接从输入数据借用的类型的 `FromPyObject` 特性实现无法在没有 GIL Refs 的情况下由 PyO3 实现（在 GIL Refs API 正在被移除的过程中）。主要受影响的类型是 `&str`、`Cow<'_, str>`、`&[u8]`、`Cow<'_, u8>`。

为了使 PyO3 的核心功能在 GIL Refs API 被移除的过程中继续工作，禁用 `gil-refs` 特性将 `FromPyObject` 的实现从 `&str`、`Cow<'_, str>`、`&[u8]`、`Cow<'_, u8>` 移动到一个新的临时特性 `FromPyObjectBound`。该特性是 `FromPyObject` 的预期未来形式，并具有额外的生命周期 `'a`，以使这些类型能够从 Python 对象借用数据。

PyO3 0.21 引入了 [`PyBackedStr`]({{#PYO3_DOCS_URL}}/pyo3/pybacked/struct.PyBackedStr.html) 和 [`PyBackedBytes`]({{#PYO3_DOCS_URL}}/pyo3/pybacked/struct.PyBackedBytes.html) 类型以帮助处理这种情况。避免从 `&str` 提取的最简单方法是使用这些类型。对于像 `Vec<&str>` 这样的复杂类型，现在无法直接从 Python 对象提取，推荐的升级路径是使用 `Vec<PyBackedStr>`。

需要注意的是，由于提取到这些类型现在将它们与输入生命周期绑定，因此一些非常常见的模式可能需要拆分为多行 Rust 代码。例如，以下直接在 `.getattr()` 的结果上调用 `.extract::<&str>()` 的代码在禁用 `gil-refs` 特性时需要调整。

之前：

```rust,ignore
# #[cfg(feature = "gil-refs")] {
# use pyo3::prelude::*;
# use pyo3::types::{PyList, PyType};
# fn example<'py>(py: Python<'py>) -> PyResult<()> {
#[allow(deprecated)] // GIL Ref API
let obj: &'py PyType = py.get_type::<PyList>();
let name: &'py str = obj.getattr("__name__")?.extract()?;
assert_eq!(name, "list");
# Ok(())
# }
# Python::with_gil(example).unwrap();
# }
```

之后：

```rust
# #[cfg(any(not(Py_LIMITED_API), Py_3_10))] {
# use pyo3::prelude::*;
# use pyo3::types::{PyList, PyType};
# fn example<'py>(py: Python<'py>) -> PyResult<()> {
let obj: Bound<'py, PyType> = py.get_type_bound::<PyList>();
let name_obj: Bound<'py, PyAny> = obj.getattr("__name__")?;
// 数据的生命周期不再是 `'py`，而是上面 `name_obj` 智能指针的生命周期
let name: &'_ str = name_obj.extract()?;
assert_eq!(name, "list");
# Ok(())
# }
# Python::with_gil(example).unwrap();
# }
```

为了完全避免担心生命周期问题，还可以使用新的 `PyBackedStr` 类型，该类型存储对 Python `str` 的引用，而不附加生命周期。特别是，`PyBackedStr` 有助于 `abi3` 构建，适用于 Python 3.10 之前的版本。由于对这些旧版本的 `abi3` CPython API 的限制，PyO3 无法在这些版本上提供 `FromPyObjectBound` 的实现。对于较旧的 `abi3` 构建，迁移的最简单方法是将任何 `.extract::<&str>()` 的情况替换为 `.extract::<PyBackedStr>()`。或者，使用 `.extract::<Cow<str>>()`、`.extract::<String>()` 将数据复制到 Rust 中。

以下示例使用与上面相同的代码片段，但这次提取的最终类型是 `PyBackedStr`：

```rust
# use pyo3::prelude::*;
# use pyo3::types::{PyList, PyType};
# fn example<'py>(py: Python<'py>) -> PyResult<()> {
use pyo3::pybacked::PyBackedStr;
let obj: Bound<'py, PyType> = py.get_type_bound::<PyList>();
let name: PyBackedStr = obj.getattr("__name__")?.extract()?;
assert_eq!(&*name, "list");
# Ok(())
# }
# Python::with_gil(example).unwrap();
```
</details>

## 从 0.19.* 升级到 0.20

### 删除对旧技术的支持
<details>
<summary><small>点击展开</small></summary>

PyO3 0.20 将最低 Rust 版本提高到 1.56。这使得可以使用更新的语言特性，并简化了项目的维护。
</details>

### `PyDict::get_item` 现在返回 `Result`
<details>
<summary><small>点击展开</small></summary>

在 PyO3 0.19 及更早版本中，`PyDict::get_item` 是使用一个 Python API 实现的，该 API 会抑制所有异常并在这些情况下返回 `None`。这包括在查找的键的 `__hash__` 和 `__eq__` 实现中的错误。

Python 核心开发者的新建议反对使用这些抑制异常的 API，而是允许异常向上冒泡。`PyDict::get_item_with_error` 已经实现了这种推荐的行为，因此该 API 已被重命名为 `PyDict::get_item`。

之前：

```rust,ignore
use pyo3::prelude::*;
use pyo3::exceptions::PyTypeError;
use pyo3::types::{PyDict, IntoPyDict};

# fn main() {
# let _ =
Python::with_gil(|py| {
    let dict: &PyDict = [("a", 1)].into_py_dict(py);
    // `a` 在字典中，值为 1
    assert!(dict.get_item("a").map_or(Ok(false), |x| x.eq(1))?);
    // `b` 不在字典中
    assert!(dict.get_item("b").is_none());
    // `dict` 不是可哈希的，因此这会失败并引发 `TypeError`
    assert!(dict
        .get_item_with_error(dict)
        .unwrap_err()
        .is_instance_of::<PyTypeError>(py));
});
# }
```

之后：

```rust,ignore
use pyo3::prelude::*;
use pyo3::exceptions::PyTypeError;
use pyo3::types::{PyDict, IntoPyDict};

# fn main() {
# let _ =
Python::with_gil(|py| -> PyResult<()> {
    let dict: &PyDict = [("a", 1)].into_py_dict(py);
    // `a` 在字典中，值为 1
    assert!(dict.get_item("a")?.map_or(Ok(false), |x| x.eq(1))?);
    // `b` 不在字典中
    assert!(dict.get_item("b")?.is_none());
    // `dict` 不是可哈希的，因此这会失败并引发 `TypeError`
    assert!(dict
        .get_item(dict)
        .unwrap_err()
        .is_instance_of::<PyTypeError>(py));

    Ok(())
});
# }
```
</details>

### 必需参数不再接受可选参数之后
<details>
<summary><small>点击展开</small></summary>

[尾随 `Option<T>` 参数](./function/signature.md#trailing-optional-arguments) 具有自动默认值 `None`。为了避免在修改函数签名时出现不必要的更改，在 PyO3 0.18 中，弃用在 `Option<T>` 参数之后有必需参数而不使用 `#[pyo3(signature = (...))]` 来指定预期的默认值。在 PyO3 0.20 中，这成为了一个硬错误。

之前：

```rust,ignore
#[pyfunction]
fn x_or_y(x: Option<u64>, y: u64) -> u64 {
    x.unwrap_or(y)
}
```

之后：

```rust
# #![allow(dead_code)]
# use pyo3::prelude::*;

#[pyfunction]
#[pyo3(signature = (x, y))] // x 和 y 都没有默认值且是必需的
fn x_or_y(x: Option<u64>, y: u64) -> u64 {
    x.unwrap_or(y)
}
```
</details>

### 删除弃用的函数形式
<details>
<summary><small>点击展开</small></summary>

在 PyO3 0.18 中，`#[args]` 属性用于 `#[pymethods]`，以及在 `#[pyfunction]` 中直接指定函数签名，已被弃用。此功能在 PyO3 0.20 中已被移除。

之前：

```rust,ignore
#[pyfunction]
#[pyo3(a, b = "0", "/")]
fn add(a: u64, b: u64) -> u64 {
    a + b
}
```

之后：

```rust
# #![allow(dead_code)]
# use pyo3::prelude::*;

#[pyfunction]
#[pyo3(signature = (a, b=0, /))]
fn add(a: u64, b: u64) -> u64 {
    a + b
}
```

</details>

### 移除 `IntoPyPointer` 特性
<details>
<summary><small>点击展开</small></summary>

特性 `IntoPyPointer`，提供了许多类型上的 `into_ptr` 方法，已被移除。`into_ptr` 现在作为所有以前实现此特性的类型的固有方法可用。
</details>

### `AsPyPointer` 现在是 `unsafe` 特性
<details>
<summary><small>点击展开</small></summary>

特性 `AsPyPointer` 现在是 `unsafe trait`，这意味着任何外部实现都必须标记为 `unsafe impl`，并确保它们遵循返回有效指针的约束。
</details>

## 从 0.18.* 升级到 0.19

### 在 `__traverse__` 实现中禁止访问 `Python`
<details>
<summary><small>点击展开</small></summary>

在 Python 的垃圾回收的 `__traverse__` 实现中，禁止执行除访问正在遍历的 `#[pyclass]` 的成员之外的任何操作。这意味着禁止进行 Python 函数调用或其他 API 调用。

以前的 PyO3 版本允许访问 `Python`（例如通过 `Python::with_gil`），这可能导致 Python 解释器崩溃或以其他方式混淆垃圾回收算法。

尝试获取 GIL 现在将引发 panic。有关更多详细信息，请参见 [#3165](https://github.com/PyO3/pyo3/issues/3165)。

```rust,ignore
# use pyo3::prelude::*;

#[pyclass]
struct SomeClass {}

impl SomeClass {
    fn __traverse__(&self, pyo3::class::gc::PyVisit<'_>) -> Result<(), pyo3::class::gc::PyTraverseError> {
        Python::with_gil(|| { /*...*/ })  // 错误：这将引发 panic
    }
}
```
</details>

### 更智能的 `anyhow::Error` / `eyre::Report` 转换，当内部错误是“简单” `PyErr` 时
<details>
<summary><small>点击展开</small></summary>

当从 `anyhow::Error` 或 `eyre::Report` 转换为 `PyErr` 时，如果内部错误是“简单” `PyErr`（没有源错误），则将直接使用内部错误作为 `PyErr`，而不是将其包装在带有原始信息转换为字符串的新 `PyRuntimeError` 中。

```rust,ignore
# #[cfg(feature = "anyhow")]
# #[allow(dead_code)]
# mod anyhow_only {
# use pyo3::prelude::*;
# use pyo3::exceptions::PyValueError;
#[pyfunction]
fn raise_err() -> anyhow::Result<()> {
    Err(PyValueError::new_err("original error message").into())
}

fn main() {
    Python::with_gil(|py| {
        let rs_func = wrap_pyfunction!(raise_err, py).unwrap();
        pyo3::py_run!(
            py,
            rs_func,
            r"
        try:
            rs_func()
        except Exception as e:
            print(repr(e))
        "
        );
    })
}
# }
```

之前，上述代码将打印 `RuntimeError('ValueError: original error message')`，这可能会令人困惑。

之后，相同的代码将打印 `ValueError: original error message`，这更直接。

但是，如果 `anyhow::Error` 或 `eyre::Report` 有源，则原始异常仍将被包装在 `PyRuntimeError` 中。
</details>

### 已删除弃用的 `Python::acquire_gil`，必须使用 `Python::with_gil`
<details>
<summary><small>点击展开</small></summary>

虽然 [`Python::acquire_gil`](https://docs.rs/pyo3/0.18.3/pyo3/marker/struct.Python.html#method.acquire_gil) 提供的 API 看似方便，但它有些脆弱，因为 GIL 令牌 [`Python`](https://docs.rs/pyo3/0.18.3/pyo3/marker/struct.Python.html) 的设计依赖于正确的嵌套，并在未正确使用时引发 panic，例如：

```rust,ignore
# #![allow(dead_code, deprecated)]
# use pyo3::prelude::*;

#[pyclass]
struct SomeClass {}

struct ObjectAndGuard {
    object: Py<SomeClass>,
    guard: GILGuard,
}

impl ObjectAndGuard {
    fn new() -> Self {
        let guard = Python::acquire_gil();
        let object = Py::new(guard.python(), SomeClass {}).unwrap();

        Self { object, guard }
    }
}

let first = ObjectAndGuard::new();
let second = ObjectAndGuard::new();
// 引发 panic，因为 `second` 中的 guard 仍然存在。
drop(first);
drop(second);
```

替代方案是 [`Python::with_gil`](https://docs.rs/pyo3/0.18.3/pyo3/marker/struct.Python.html#method.with_gil)，它虽然更麻烦，但通过设计强制执行正确的嵌套，例如：

```rust,ignore
# #![allow(dead_code)]
# use pyo3::prelude::*;

#[pyclass]
struct SomeClass {}

struct Object {
    object: Py<SomeClass>,
}

impl Object {
    fn new(py: Python<'_>) -> Self {
        let object = Py::new(py, SomeClass {}).unwrap();

        Self { object }
    }
}

// 它要么强制我们在再次获取之前释放 GIL。
let first = Python::with_gil(|py| Object::new(py));
let second = Python::with_gil(|py| Object::new(py));
drop(first);
drop(second);

// 或者确保在外部锁释放之前释放内部锁。
Python::with_gil(|py| {
    let first = Object::new(py);
    let second = Python::with_gil(|py| Object::new(py));
    drop(first);
    drop(second);
});
```

此外，`Python::acquire_gil` 提供了 `GILGuard` 的所有权，可以自由存储和传递。这通常没有帮助，因为它可能会长时间保持锁定，从而阻碍程序其他部分的进展。由于与 GIL 令牌相关的生成生命周期，使用 `Python::with_gil` 可以避免此问题，因为 GIL 令牌只能沿着调用链传递。通常，这个问题也可以完全避免，因为任何 GIL 绑定引用 `&'py PyAny` 都意味着通过 [`PyAny::py`](https://docs.rs/pyo3/latest/pyo3/types/struct.PyAny.html#method.py) 方法访问 GIL 令牌 `Python<'py>`。
</details>

## 从 0.17.* 升级到 0.18

### 在 `Option<_>` 参数之后的必需参数将不再自动推断
<details>
<summary><small>点击展开</small></summary>

在 `#[pyfunction]` 和 `#[pymethods]` 中，如果像 `i32` 这样的“必需”函数输入位于 `Option<_>` 输入之后，则 `Option<_>` 将被隐式视为必需的。（所有尾随 `Option<_>` 参数都被视为可选的，默认值为 `None`）。

从 PyO3 0.18 开始，这被弃用，未来的 PyO3 版本将要求使用 [`#[pyo3(signature = (...))]` 选项](./function/signature.md) 明确声明程序员的意图。

之前，下面示例中的 x 需要从 Python 代码中传递：

```rust,compile_fail
# #![allow(dead_code)]
# use pyo3::prelude::*;

#[pyfunction]
fn required_argument_after_option(x: Option<i32>, y: i32) {}
```

之后，明确指定预期的 Python 签名：

```rust
# #![allow(dead_code)]
# use pyo3::prelude::*;

// 如果 x 确实是必需的
#[pyfunction(signature = (x, y))]
fn required_argument_after_option_a(x: Option<i32>, y: i32) {}

// 如果 x 是可选的，y 也需要有默认值
#[pyfunction(signature = (x=None, y=0))]
fn required_argument_after_option_b(x: Option<i32>, y: i32) {}
```
</details>

### `__text_signature__` 现在自动生成用于 `#[pyfunction]` 和 `#[pymethods]`
<details>
<summary><small>点击展开</small></summary>

[`#[pyo3(text_signature = "...")]` 选项](./function/signature.md#making-the-function-signature-available-to-python) 之前是设置生成的 Python 函数的 `__text_signature__` 属性的唯一支持方式。

PyO3 现在能够根据其 Rust 签名（或新的 `#[pyo3(signature = (...))]` 选项）自动填充 `__text_signature__`。这些自动生成的 `__text_signature__` 值当前仅在所有默认值上呈现 `...`。许多 `#[pyo3(text_signature = "...")]` 选项在更新到 PyO3 0.18 时可以被移除，但在具有默认值的情况下，手动实现可能仍然更可取。

例如：

```rust
# use pyo3::prelude::*;

// 这里的 `text_signature` 选项不再必要，因为 PyO3 将自动生成完全相同的值。
#[pyfunction(text_signature = "(a, b, c)")]
fn simple_function(a: i32, b: i32, c: i32) {}

// 由于自动生成的签名将是 "(a, b=..., c=...)"，因此 `text_signature` 在这里仍然提供价值。
#[pyfunction(signature = (a, b = 1, c = 2), text_signature = "(a, b=1, c=2)")]
fn function_with_defaults(a: i32, b: i32, c: i32) {}

# fn main() {
#     Python::with_gil(|py| {
#         let simple = wrap_pyfunction!(simple_function, py).unwrap();
#         assert_eq!(simple.getattr("__text_signature__").unwrap().to_string(), "(a, b, c)");
#         let defaulted = wrap_pyfunction!(function_with_defaults, py).unwrap();
#         assert_eq!(defaulted.getattr("__text_signature__").unwrap().to_string(), "(a, b=1, c=2)");
#     })
# }
```
</details>

## 从 0.16.* 升级到 0.17

### 对 `PyMapping` 和 `PySequence` 类型的类型检查已更改
<details>
<summary><small>点击展开</small></summary>

之前，`PyMapping` 和 `PySequence`（在 `PyTryFrom` 中实现）的类型检查使用 Python C-API 函数 `PyMapping_Check` 和 `PySequence_Check`。
不幸的是，这些函数不足以区分此类类型，导致不一致的行为（见 [pyo3/pyo3#2072](https://github.com/PyO3/pyo3/issues/2072)）。

PyO3 0.17 将这些向下转换检查更改为显式测试类型是否是相应抽象基类 `collections.abc.Mapping` 或 `collections.abc.Sequence` 的子类。请注意，这需要调用 Python，这可能会对性能产生影响。如果这种性能损失是一个问题，您可以执行自己的检查并使用 `try_from_unchecked`（不安全）。

另一个副作用是，用 PyO3 定义的 pyclass 需要与相应的 Python 抽象基类进行 _注册_，以使向下转换成功。已添加 `PySequence::register` 和 `PyMapping:register` 以便于从 Rust 代码中执行此操作。这相当于从 Python 中调用 `collections.abc.Mapping.register(MappingPyClass)` 或 `collections.abc.Sequence.register(SequencePyClass)`。

例如，对于在 Rust 中定义的映射类：
```rust,compile_fail
use pyo3::prelude::*;
use std::collections::HashMap;

#[pyclass(mapping)]
struct Mapping {
    index: HashMap<String, usize>,
}

#[pymethods]
impl Mapping {
    #[new]
    fn new(elements: Option<&PyList>) -> PyResult<Self> {
    // ...
    // 省略此映射 pyclass 的实现 - 基本上是对 HashMap 的包装
}
```

您必须在向下转换之前将类注册到 `collections.abc.Mapping`：
```rust,compile_fail
let m = Py::new(py, Mapping { index }).unwrap();
assert!(m.as_ref(py).downcast::<PyMapping>().is_err());
PyMapping::register::<Mapping>(py).unwrap();
assert!(m.as_ref(py).downcast::<PyMapping>().is_ok());
```

请注意，这一要求在未来可能会消失，当 pyclass 能够直接从抽象基类继承时（见 [pyo3/pyo3#991](https://github.com/PyO3/pyo3/issues/991)）。
</details>

### `multiple-pymethods` 特性现在需要 Rust 1.62
<details>
<summary><small>点击展开</small></summary>

由于 `multiple-pymethods` 特性所依赖的 `inventory` crate 的限制，该特性现在需要 Rust 1.62。有关更多信息，请参见 [dtolnay/inventory#32](https://github.com/dtolnay/inventory/issues/32)。
</details>

### 为 `&str` 添加 `impl IntoPy<Py<PyString>>`
<details>
<summary><small>点击展开</small></summary>

这可能会导致推断错误。

之前：
```rust,compile_fail
# use pyo3::prelude::*;
#
# fn main() {
Python::with_gil(|py| {
    // 无法推断 `Py<PyAny>` 或 `Py<PyString>`
    let _test = "test".into_py(py);
});
# }
```

之后，可能需要一些类型注释：

```rust
# use pyo3::prelude::*;
#
# fn main() {
Python::with_gil(|py| {
    let _test: Py<PyAny> = "test".into_py(py);
});
# }
```
</details>

### `pyproto` 特性现在默认禁用
<details>
<summary><small>点击展开</small></summary>

为准备在未来的 PyO3 版本中移除弃用的 `#[pyproto]` 属性宏，该特性现在被限制在选择性启用的特性标志下。这也为不使用弃用宏的代码节省了编译时间。
</details>

### `PyTypeObject` 特性已被弃用
<details>
<summary><small>点击展开</small></summary>

`PyTypeObject` 特性已经几乎无用；几乎所有功能都已经在 `PyTypeInfo` 特性上，`PyTypeObject` 具有基于此的通用实现。在 PyO3 0.17 中，最后一个方法 `PyTypeObject::type_object` 被移至 `PyTypeInfo::type_object`。

要迁移，请更新特性约束和导入，从 `PyTypeObject` 更改为 `PyTypeInfo`。

之前：

```rust,ignore
use pyo3::Python;
use pyo3::type_object::PyTypeObject;
use pyo3::types::PyType;

fn get_type_object<T: PyTypeObject>(py: Python<'_>) -> &PyType {
    T::type_object(py)
}
```

之后：

```rust,ignore
use pyo3::{Python, PyTypeInfo};
use pyo3::types::PyType;

fn get_type_object<T: PyTypeInfo>(py: Python<'_>) -> &PyType {
    T::type_object(py)
}

# Python::with_gil(|py| { get_type_object::<pyo3::types::PyList>(py); });
```
</details>

### `impl<T, const N: usize> IntoPy<PyObject> for [T; N]` 现在要求 `T: IntoPy` 而不是 `T: ToPyObject`
<details>
<summary><small>点击展开</small></summary>

如果这导致错误，只需实现 `IntoPy`。因为 pyclasses 已经实现了 `IntoPy`，您可能不需要担心这个问题。
</details>

### 每个 `#[pymodule]` 现在只能在每个进程中初始化一次
<details>
<summary><small>点击展开</small></summary>

为了使 PyO3 模块在 Python 子解释器中存在时是安全的，目前必须显式禁用在同一进程中多次初始化 `#[pymodule]` 的能力。尝试这样做现在将引发 `ImportError`。
</details>

## 从 0.15.* 升级到 0.16

### 删除对旧技术的支持
<details>
<summary><small>点击展开</small></summary>

PyO3 0.16 将最低 Rust 版本提高到 1.48，最低 Python 版本提高到 3.7。这使得可以使用更新的语言特性（启用 0.16 中的其他一些添加）并简化了项目的维护。
</details>

### `#[pyproto]` 已被弃用
<details>
<summary><small>点击展开</small></summary>

在 PyO3 0.15 中，`#[pymethods]` 属性宏获得了实现 "魔法方法"（如 `__str__`，即 "dunder" 方法）的支持。此实现当时尚未完全定稿，还有一些边缘情况待决定。现有的 `#[pyproto]` 属性宏未被触及，因为它涵盖了这些边缘情况。

在 PyO3 0.16 中，`#[pymethods]` 的实现已完成，现在是实现魔法方法的首选方式。为了使 PyO3 项目向前发展，`#[pyproto]` 已被弃用（预计在 PyO3 0.18 中移除）。

从 `#[pyproto]` 迁移到 `#[pymethods]` 是直接的；在大多数情况下，只需直接从 `#[pyproto]` 特性实现中复制现有方法即可。

之前：

```rust,compile_fail
use pyo3::prelude::*;
use pyo3::class::{PyObjectProtocol, PyIterProtocol};
use pyo3::types::PyString;

#[pyclass]
struct MyClass {}

#[pyproto]
impl PyObjectProtocol for MyClass {
    fn __str__(&self) -> &'static [u8] {
        b"hello, world"
    }
}

#[pyproto]
impl PyIterProtocol for MyClass {
    fn __iter__(slf: PyRef<self>) -> PyResult<&PyAny> {
        PyString::new(slf.py(), "hello, world").iter()
    }
}
```

之后：

```rust,compile_fail
use pyo3::prelude::*;
use pyo3::types::PyString;

#[pyclass]
struct MyClass {}

#[pymethods]
impl MyClass {
    fn __str__(&self) -> &'static [u8] {
        b"hello, world"
    }

    fn __iter__(slf: PyRef<self>) -> PyResult<&PyAny> {
        PyString::new(slf.py(), "hello, world").iter()
    }
}
```
</details>

### 移除 `PartialEq` 对对象包装器的支持
<details>
<summary><small>点击展开</small></summary>

Python 对象包装器 `Py` 和 `PyAny` 具有 `PartialEq` 的实现，以便 `object_a == object_b` 将比较 Python 对象的指针相等性，这对应于 Python 中的 `is` 操作符，而不是 `==` 操作符。这已被移除，取而代之的是新方法：使用 `object_a.is(object_b)`。这也有一个好处，即不需要 `object_a` 和 `object_b` 具有相同的包装类型；现在可以直接比较 `Py<T>` 和 `&PyAny`，而无需进行转换。

要检查 Python 对象相等性（Python 的 `==` 操作符），请使用新方法 `eq()`。
</details>

### 容器魔法方法现在与 Python 行为匹配
<details>
<summary><small>点击展开</small></summary>

在 PyO3 0.15 中，`__getitem__`、`__setitem__` 和 `__delitem__` 在 `#[pymethods]` 中仅为 `#[pyclass]` 生成 _映射_ 实现。为了与 Python 行为匹配，这些方法现在同时生成 _映射_ **和** _序列_ 实现。

这意味着实现这些 `#[pymethods]` 的类现在也将被视为序列，就像 Python `class` 一样。可能会导致行为上的小差异：
 - PyO3 将允许这些类的实例转换为 `PySequence` 和 `PyMapping`。
 - 如果类没有实现，Python 将提供 `__iter__` 的默认实现，该实现会重复调用 `__getitem__`，使用整数（从 0 开始）直到引发 `IndexError`。

为了详细说明这一点，考虑以下 Python 类：

```python
class ExampleContainer:

    def __len__(self):
        return 5

    def __getitem__(self, idx: int) -> int:
        if idx < 0 or idx > 5:
            raise IndexError()
        return idx
```

此类实现了 Python [序列](https://docs.python.org/3/glossary.html#term-sequence)。

`__len__` 和 `__getitem__` 方法也用于实现 Python [映射](https://docs.python.org/3/glossary.html#term-mapping)。在 Python C-API 中，这些方法并不共享：序列的 `__len__` 和 `__getitem__` 由 `sq_length` 和 `sq_item` 插槽定义，而映射的等效方法是 `mp_length` 和 `mp_subscript`。对于 `__setitem__` 和 `__delitem__` 也有类似的区别。

由于 Python 没有这样的区别，因此实现这些方法将同时填充映射和序列插槽。具有实现 `__len__` 的 Python 类，例如，将同时填充 `sq_length` 和 `mp_length` 插槽。

PyO3 在 0.16 中的行为已更改为默认更接近于这种 Python 行为。
</details>

### `wrap_pymodule!` 和 `wrap_pyfunction!` 现在正确尊重隐私
<details>
<summary><small>点击展开</small></summary>

在 PyO3 0.16 之前，`wrap_pymodule!` 和 `wrap_pyfunction!` 宏可以使用定义 `fn` 的可达性不符合 Rust 隐私规则的模块和函数。

例如，以下代码在 0.16 之前是合法的，但在 0.16 中被拒绝，因为 `wrap_pymodule!` 宏无法访问 `private_submodule` 函数：

```rust,compile_fail
mod foo {
    use pyo3::prelude::*;

    #[pymodule]
    fn private_submodule(_py: Python<'_>, m: &PyModule) -> PyResult<()> {
        Ok(())
    }
}

use pyo3::prelude::*;
use foo::*;

#[pymodule]
fn my_module(_py: Python<'_>, m: &PyModule) -> PyResult<()> {
    m.add_wrapped(wrap_pymodule!(private_submodule))?;
    Ok(())
}
```

要修复此问题，请使私有子模块可见，例如使用 `pub` 或 `pub(crate)`。

```rust,ignore
mod foo {
    use pyo3::prelude::*;

    #[pymodule]
    pub(crate) fn private_submodule(_py: Python<'_>, m: &PyModule) -> PyResult<()> {
        Ok(())
    }
}

use pyo3::prelude::*;
use pyo3::wrap_pymodule;
use foo::*;

#[pymodule]
fn my_module(_py: Python<'_>, m: &PyModule) -> PyResult<()> {
    m.add_wrapped(wrap_pymodule!(private_submodule))?;
    Ok(())
}
```
</details>

## 从 0.14.* 升级到 0.15

### 序列索引的变化
<details>
<summary><small>点击展开</small></summary>

对于所有接受序列索引的类型（`PyList`、`PyTuple` 和 `PySequence`），API 已被统一为仅接受 `usize` 索引，以与 Rust 的索引约定保持一致。负索引，之前即使在接受 `isize` 的 API 中也仅偶尔支持，现在在任何地方都不再支持。

此外，`get_item` 方法现在始终返回 `PyResult`，而不是在无效索引上引发 panic。`Index` 特性已被实现，并提供与 Rust 向量相同的 panic 行为。

请注意，*切片* 索引（由 `PySequence::get_slice` 和其他接受）仍然继承 Python 行为，限制索引到实际长度，而不会在超出范围的索引上引发 panic/返回错误。

使用 Rust 索引约定的额外好处是，这些类型现在也可以支持 Rust 的索引运算符，作为一致的 API 的一部分：

```rust,ignore
#![allow(deprecated)]
use pyo3::{Python, types::PyList};

Python::with_gil(|py| {
    let list = PyList::new(py, &[1, 2, 3]);
    assert_eq!(list[0..2].to_string(), "[1, 2]");
});
```
</details>

## 从 0.13.* 升级到 0.14

### `auto-initialize` 特性现在是选择性启用
<details>
<summary><small>点击展开</small></summary>

对于嵌入 Python 的 Rust 项目，PyO3 不再自动初始化 Python 解释器，除非启用了 [`auto-initialize` 特性](features.md#auto-initialize)。
</details>

### 新的 `multiple-pymethods` 特性
<details>
<summary><small>点击展开</small></summary>

`#[pymethods]` 已重新设计，采用更简单的默认实现，移除了对 `inventory` crate 的依赖。这减少了大多数用户的依赖和编译时间。

新默认实现的限制是它不支持同一 `#[pyclass]` 的多个 `#[pymethods]` 块。如果您需要此功能，必须启用 `multiple-pymethods` 特性，这将切换 `#[pymethods]` 到基于 inventory 的实现。
</details>

### 弃用的 `#[pyproto]` 方法
<details>
<summary><small>点击展开</small></summary>

一些协议（即 `__dunder__`）方法，例如 `__bytes__` 和 `__format__`，在 PyO3 中已经可以通过两种方式实现：通过 `#[pyproto]`（例如 `PyObjectProtocol` 对于此处列出的方法），或通过直接在 `#[pymethods]` 中编写它们。这仅适用于少数 `#[pyproto]` 方法（出于技术原因，涉及 PyO3 当前与 Python C-API 的交互方式）。

为了只有一种方法来实现这些方法，`#[pyproto]` 的形式已被弃用。

要迁移，只需将受影响的方法从 `#[pyproto]` 移动到 `#[pymethods]` 块中。

之前：

```rust,compile_fail
use pyo3::prelude::*;
use pyo3::class::basic::PyObjectProtocol;

#[pyclass]
struct MyClass {}

#[pyproto]
impl PyObjectProtocol for MyClass {
    fn __bytes__(&self) -> &'static [u8] {
        b"hello, world"
    }
}
```

之后：

```rust
use pyo3::prelude::*;

#[pyclass]
struct MyClass {}

#[pymethods]
impl MyClass {
    fn __bytes__(&self) -> &'static [u8] {
        b"hello, world"
    }
}
```
</details>

## 从 0.12.* 升级到 0.13

### 最低 Rust 版本提高到 Rust 1.45
<details>
<summary><small>点击展开</small></summary>

PyO3 `0.13` 使用了在 Rust 1.40 和 Rust 1.45 之间稳定的新 Rust 语言特性。如果您使用的 Rust 编译器版本低于 Rust 1.45，则需要更新工具链才能继续使用 PyO3。
</details>

### 运行时更改以支持 CPython 限制 API
<details>
<summary><small>点击展开</small></summary>

在 PyO3 `0.13` 中，添加了对编译 CPython 限制 API 的支持。这对 _所有_ PyO3 用户有许多影响，描述如下。

其中最大的影响是，所有由 PyO3 创建的类型都是 CPython 所称的“堆”类型。这一点的具体影响包括：

- 如果您希望从 Rust _子类化_ 其中一种类型，则必须将其标记为 `#[pyclass(subclass)]`，就像您希望允许从 Python 代码对子类化它一样。
- 类型对象现在是可变的 - Python 代码可以在其上设置属性。
- 对于没有 `#[pyclass(module="mymodule")]` 的类型，`__module__` 不再返回 `builtins`，而是引发 `AttributeError`。
</details>

## 从 0.11.* 升级到 0.12

### `PyErr` 已被重新设计
<details>
<summary><small>点击展开</small></summary>

在 PyO3 `0.12` 中，`PyErr` 类型已被重新实现，以显著更好地与标准 Rust 错误处理生态系统兼容。具体而言，`PyErr` 现在实现了 `Error + Send + Sync`，这些是用于错误类型的标准特性。

虽然这导致删除了一些 API，但结果 `PyErr` 类型现在应该更容易使用。以下部分详细列出了更改及如何迁移到新 API。
</details>

#### `PyErr::new` 和 `PyErr::from_type` 现在要求其参数为 `Send + Sync`
<details>
<summary><small>点击展开</small></summary>

对于大多数用法，无需更改。如果您尝试从不是 `Send + Sync` 的值构造 `PyErr`，则需要先创建 Python 对象，然后使用 `PyErr::from_instance`。

同样，任何实现了 `PyErrArguments` 的类型现在也需要是 `Send + Sync`。
</details>

#### `PyErr` 的内容现在是私有的
<details>
<summary><small>点击展开</small></summary>

不再可以访问 `PyErr` 的字段 `.ptype`、`.pvalue` 和 `.ptraceback`。
您现在应该使用新方法 `PyErr::ptype`、`PyErr::pvalue` 和 `PyErr::ptraceback`。
</details>

#### `PyErrValue` 和 `PyErr::from_value` 已被移除
<details>
<summary><small>点击展开</small></summary>

由于这些是 `PyErr` 的内部实现的一部分，已被重新设计，因此这些 API 不再存在。

如果您使用了此 API，建议使用 `PyException::new_err`（请参见 [异常类型部分](#exception-types-have-been-reworked)）。
</details>

#### `Into<PyResult<T>>` 对 `PyErr` 的实现已被移除
<details>
<summary><small>点击展开</small></summary>

此实现是多余的。只需直接构造 `Result::Err` 变体。

之前：
```rust,compile_fail
let result: PyResult<()> = PyErr::new::<TypeError, _>("error message").into();
```

之后（也使用新重构的异常类型；见以下部分）：
```rust
# use pyo3::{PyResult, exceptions::PyTypeError};
let result: PyResult<()> = Err(PyTypeError::new_err("error message"));
```
</details>

### 异常类型已被重新设计
<details>
<summary><small>点击展开</small></summary>

之前，异常类型是零大小的标记类型，仅用于构造 `PyErr`。在 PyO3 0.12 中，这些类型已被完整定义并可以像 `PyAny`、`PyDict` 等一样使用。这使得与 Python 异常对象进行交互成为可能。

新类型的名称也以 "Py" 前缀开头。例如，之前：

```rust,ignore
let err: PyErr = TypeError::py_err("error message");
```

之后：

```rust,ignore
# use pyo3::{PyErr, PyResult, Python, type_object::PyTypeObject};
# use pyo3::exceptions::{PyBaseException, PyTypeError};
# Python::with_gil(|py| -> PyResult<()> {
let err: PyErr = PyTypeError::new_err("error message");

// 使用 PyErr 的 Display，PyO3 0.12 新增
assert_eq!(err.to_string(), "TypeError: error message");

// 现在可以与异常实例进行交互，PyO3 0.12 新增
let instance: &PyBaseException = err.instance(py);
assert_eq!(
    instance.getattr("__class__")?,
    PyTypeError::type_object(py).as_ref()
);
# Ok(())
# }).unwrap();
```
</details>

### 移除 `FromPy` 
<details>
<summary><small>点击展开</small></summary>

为了简化 PyO3 转换特性，已移除 `FromPy` 特性。之前有两种方式定义类型到 Python 的转换：
`FromPy<T> for PyObject` 和 `IntoPy<PyObject> for T`。

现在只有一种方式定义转换，即 `IntoPy`，因此下游 crate 可能需要相应调整。

之前：
```rust,compile_fail
# use pyo3::prelude::*;
struct MyPyObjectWrapper(PyObject);

impl FromPy<MyPyObjectWrapper> for PyObject {
    fn from_py(other: MyPyObjectWrapper, _py: Python<'_>) -> Self {
        other.0
    }
}
```

之后：
```rust
# use pyo3::prelude::*;
# #[allow(dead_code)]
struct MyPyObjectWrapper(PyObject);

impl IntoPy<PyObject> for MyPyObjectWrapper {
    fn into_py(self, _py: Python<'_>) -> PyObject {
        self.0
    }
}
```

同样，使用 `FromPy` 特性的代码可以轻松重写为使用 `IntoPy`。

之前：
```rust,compile_fail
# use pyo3::prelude::*;
# Python::with_gil(|py| {
let obj = PyObject::from_py(1.234, py);
# })
```

之后：
```rust
# use pyo3::prelude::*;
# Python::with_gil(|py| {
let obj: PyObject = 1.234.into_py(py);
# })
```
</details>

### `PyObject` 现在是 `Py<PyAny>` 的类型别名
<details>
<summary><small>点击展开</small></summary>

从使用的角度来看，这几乎不会改变。如果您为 `PyObject` 和 `Py<T>` 实现了特性，您可能会发现可以直接删除 `PyObject` 的实现。
</details>

### `AsPyRef` 已被移除
<details>
<summary><small>点击展开</small></summary>

由于 `PyObject` 已更改为仅为类型别名，因此唯一剩余的 `AsPyRef` 的实现是 `Py<T>`。这消除了特性的必要性，因此 `AsPyRef::as_ref` 方法已移至 `Py::as_ref`。

这应该不需要代码更改，只需删除未使用 `pyo3::prelude::*` 的 `use pyo3::AsPyRef`。

之前：
```rust,ignore
use pyo3::{AsPyRef, Py, types::PyList};
# pyo3::Python::with_gil(|py| {
let list_py: Py<PyList> = PyList::empty(py).into();
let list_ref: &PyList = list_py.as_ref(py);
# })
```

之后：
```rust,ignore
use pyo3::{Py, types::PyList};
# pyo3::Python::with_gil(|py| {
let list_py: Py<PyList> = PyList::empty(py).into();
let list_ref: &PyList = list_py.as_ref(py);
# })
```
</details>

## 从 0.10.* 升级到 0.11

### 稳定 Rust
<details>
<summary