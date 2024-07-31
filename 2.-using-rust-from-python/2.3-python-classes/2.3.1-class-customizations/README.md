# 类自定义

Python 的对象模型定义了多个协议以支持不同的对象行为，例如序列、映射和数字协议。Python 类通过实现“魔法”方法（如 `__str__` 或 `__repr__`）来支持这些协议。由于其名称前后都有双下划线，这些方法也被称为“dunder”方法。

PyO3 使得每个魔法方法都可以在 `#[pymethods]` 中实现，就像在常规 Python 类中一样，但有一些显著的区别：
- `__new__` 和 `__init__` 被 [`#[new]` 属性](../class.md#constructor) 替代。
- `__del__` 目前尚不支持，但未来可能会支持。
- `__buffer__` 和 `__release_buffer__` 当前不支持，取而代之的是 PyO3 支持 [`__getbuffer__` 和 `__releasebuffer__`](#buffer-objects) 方法（这些方法早于 [PEP 688](https://peps.python.org/pep-0688/#python-level-buffer-protocol)），未来可能会有所变化。
- PyO3 添加了 [`__traverse__` 和 `__clear__`](#garbage-collector-integration) 方法以控制垃圾回收。
- PyO3 所基于的 Python C-API 要求许多魔法方法具有特定的 C 函数签名，并被放置在类类型对象的特殊“槽”中。这限制了这些方法的参数和返回类型。它们的详细信息在下面的部分列出。

如果某个魔法方法不在上述列表中（例如 `__init_subclass__`），那么它应该在 PyO3 中正常工作。如果不是这种情况，请提交错误报告。

## PyO3 处理的魔法方法

如果 `#[pymethods]` 中的函数名称是已知需要特殊处理的魔法方法，它将自动放置在 Python 类型对象的正确槽中。函数名称遵循 `#[pymethods]` 的命名规则：如果存在 `#[pyo3(name = "...")]` 属性，则使用该属性，否则使用 Rust 函数名称。

PyO3 处理的魔法方法与 [此页面](https://docs.python.org/3/reference/datamodel.html#special-method-names) 上的标准 Python 方法非常相似——特别是它们是具有槽的子集，如 [此处定义](https://docs.python.org/3/c-api/typeobj.html)。

当 PyO3 处理魔法方法时，与其他 `#[pymethods]` 相比，有几个变化适用：
- Rust 函数签名被限制为匹配魔法方法。
- 不允许使用 `#[pyo3(signature = (...)]` 和 `#[pyo3(text_signature = "...")]` 属性。

以下部分列出了 PyO3 实现必要特殊处理的所有魔法方法。给定的签名应按以下方式解释：
- 所有方法的第一个参数是接收者，表示为 `<self>`。它可以是 `&self`、`&mut self` 或像 `self_: PyRef<'_, Self>` 和 `self_: PyRefMut<'_, Self>` 的 `Bound` 引用，如 [这里](../class.md#inheritance) 所述。
- 可选的 `Python<'py>` 参数始终允许作为第一个参数。
- 返回值可以选择性地包装在 `PyResult` 中。
- `object` 表示允许任何可以从 Python 对象中提取的类型（如果是参数）或可以转换为 Python 对象的类型（如果是返回值）。
- 其他类型必须与给定的匹配，例如 `pyo3::basic::CompareOp` 用于 `__richcmp__` 的第二个参数。
- 对于比较和算术方法，提取错误不会作为异常传播，而是导致返回 `NotImplemented`。
- 对于某些魔法方法，返回值不受 PyO3 的限制，而是由 Python 解释器检查。例如，`__str__` 需要返回一个字符串对象。这通过 `object (Python type)` 表示。

### 基本对象自定义

- `__str__(<self>) -> object (str)`
- `__repr__(<self>) -> object (str)`

- `__hash__(<self>) -> isize`

  相等的对象必须具有相同的哈希值。可以返回任何类型的值，最多 64 位，PyO3 会自动转换为 isize（包装无符号类型，如 `u64` 和 `usize`）。
  <details>
  <summary>禁用 Python 的默认哈希</summary>
  默认情况下，所有 `#[pyclass]` 类型都有来自 Python 的默认哈希实现。那些不应可哈希的类型可以通过将 `__hash__` 设置为 `None` 来覆盖。这与纯 Python 类的机制相同。实现方式如下：

  ```rust
  # use pyo3::prelude::*;
  #
  #[pyclass]
  struct NotHashable {}

  #[pymethods]
  impl NotHashable {
      #[classattr]
      const __hash__: Option<PyObject> = None;
  }
  ```
  </details>

- `__lt__(<self>, object) -> object`
- `__le__(<self>, object) -> object`
- `__eq__(<self>, object) -> object`
- `__ne__(<self>, object) -> object`
- `__gt__(<self>, object) -> object`
- `__ge__(<self>, object) -> object`

  分别实现 Python 的“丰富比较”运算符 `<`、`<=`、`==`、`!=`、`>` 和 `>=`。

  _注意，实现这些方法中的任何一个将导致 Python 不生成默认的 `__hash__` 实现，因此也考虑实现 `__hash__`。_
  <details>
  <summary>返回类型</summary>
  返回类型通常是 `bool` 或 `PyResult<bool>`，但可以返回任何 Python 对象。
  </details>

- `__richcmp__(<self>, object, pyo3::basic::CompareOp) -> object`

  在一个方法中实现 Python 比较操作（`==`、`!=`、`<`、`<=`、`>` 和 `>=`）。
  `CompareOp` 参数指示正在执行的比较操作。您可以使用 [`CompareOp::matches`] 将 Rust 的 `std::cmp::Ordering` 结果适配到请求的比较中。

  _此方法不能与任何 `__lt__`、`__le__`、`__eq__`、`__ne__`、`__gt__` 或 `__ge__` 方法一起实现。_

  _注意，实现 `__richcmp__` 将导致 Python 不生成默认的 `__hash__` 实现，因此在实现 `__richcmp__` 时考虑实现 `__hash__`。_
  <details>
  <summary>返回类型</summary>
  返回类型通常是 `PyResult<bool>`，但可以返回任何 Python 对象。

  如果您希望某些操作未实现，可以为某些操作返回 `py.NotImplemented()`：

  ```rust
  use pyo3::class::basic::CompareOp;

  # use pyo3::prelude::*;
  #
  # #[pyclass]
  # struct Number(i32);
  #
  #[pymethods]
  impl Number {
      fn __richcmp__(&self, other: &Self, op: CompareOp, py: Python<'_>) -> PyObject {
          match op {
              CompareOp::Eq => (self.0 == other.0).into_py(py),
              CompareOp::Ne => (self.0 != other.0).into_py(py),
              _ => py.NotImplemented(),
          }
      }
  }
  ```

  如果第二个参数 `object` 不是签名中指定的类型，生成的代码将自动 `return NotImplemented`。
  </details>

- `__getattr__(<self>, object) -> object`
- `__getattribute__(<self>, object) -> object`
  <details>
  <summary>`__getattr__` 和 `__getattribute__` 之间的区别</summary>
  与 Python 一样，只有在通过正常属性查找未找到属性时，才会调用 `__getattr__`。而 `__getattribute__` 则会在 *每个* 属性访问时被调用。如果它想要访问 `self` 上的现有属性，则需要非常小心，以免引入无限递归，并使用 `baseclass.__getattribute__()`。
  </details>

- `__setattr__(<self>, value: object) -> ()`
- `__delattr__(<self>, object) -> ()`

  重写属性访问。

- `__bool__(<self>) -> bool`

  确定对象的“真值”。

- `__call__(<self>, ...) -> object` - 在这里，可以定义任何参数列表，类似于常规的 `pymethods`。

### 可迭代对象

可以使用以下方法定义迭代器：

- `__iter__(<self>) -> object`
- `__next__(<self>) -> Option<object> or IterNextOutput` ([查看详情](#returning-a-value-from-iteration))

从 `__next__` 返回 `None` 表示没有更多项目。

示例：

```rust
use pyo3::prelude::*;

#[pyclass]
struct MyIterator {
    iter: Box<dyn Iterator<Item = PyObject> + Send>,
}

#[pymethods]
impl MyIterator {
    fn __iter__(slf: PyRef<'_, Self>) -> PyRef<'_, Self> {
        slf
    }
    fn __next__(mut slf: PyRefMut<'_, Self>) -> Option<PyObject> {
        slf.iter.next()
    }
}
```

在许多情况下，您会区分被迭代的类型（即 *可迭代对象*）和它提供的迭代器。在这种情况下，可迭代对象只需要实现 `__iter__()`，而迭代器必须同时实现 `__iter__()` 和 `__next__()`。例如：

```rust
# use pyo3::prelude::*;

#[pyclass]
struct Iter {
    inner: std::vec::IntoIter<usize>,
}

#[pymethods]
impl Iter {
    fn __iter__(slf: PyRef<'_, Self>) -> PyRef<'_, Self> {
        slf
    }

    fn __next__(mut slf: PyRefMut<'_, Self>) -> Option<usize> {
        slf.inner.next()
    }
}

#[pyclass]
struct Container {
    iter: Vec<usize>,
}

#[pymethods]
impl Container {
    fn __iter__(slf: PyRef<'_, Self>) -> PyResult<Py<Iter>> {
        let iter = Iter {
            inner: slf.iter.clone().into_iter(),
        };
        Py::new(slf.py(), iter)
    }
}

# Python::with_gil(|py| {
#     let container = Container { iter: vec![1, 2, 3, 4] };
#     let inst = pyo3::Py::new(py, container).unwrap();
#     pyo3::py_run!(py, inst, "assert list(inst) == [1, 2, 3, 4]");
#     pyo3::py_run!(py, inst, "assert list(iter(iter(inst))) == [1, 2, 3, 4]");
# });
```

有关 Python 迭代协议的更多详细信息，请查看库文档的 [“迭代器类型”部分](https://docs.python.org/library/stdtypes.html#iterator-types)。

#### 从迭代中返回值

到目前为止，本指南展示了如何使用 `Option<T>` 在迭代过程中实现值的生成。在 Python 中，生成器也可以返回一个值。这是通过引发 `StopIteration` 异常来完成的。要在 Rust 中表达这一点，请返回 `PyResult::Err`，并将 `PyStopIteration` 作为错误。

### 可等待对象

- `__await__(<self>) -> object`
- `__aiter__(<self>) -> object`
- `__anext__(<self>) -> Option<object>`

### 映射和序列类型

本节中的魔法方法可用于实现 Python 容器类型。Python 中有两种主要的容器类型：“映射”，如 `dict`，具有任意键，以及“序列”，如 `list` 和 `tuple`，具有整数键。

PyO3 所基于的 Python C-API 为序列和映射提供了单独的“槽”。在纯 Python 中编写 `class` 时，实施中没有这样的区分——例如，`__getitem__` 的实现将填充映射和序列形式的槽。

默认情况下，PyO3 复制 Python 的行为，填充映射和序列槽。这对于与 Python 匹配的“简单”情况是合理的，对于序列也是如此，因为映射槽无论如何都用于实现切片索引。

映射类型通常不希望填充序列槽。填充序列槽可能导致不希望的结果，例如：
- 映射类型将成功转换为 [`PySequence`]。这可能导致类型的消费者处理不当。
- Python 为序列提供了 `__iter__` 的默认实现，该实现使用从 0 开始的连续正整数调用 `__getitem__`，直到返回 `IndexError`。除非映射仅包含连续的正整数键，否则此 `__iter__` 实现可能不是预期的行为。

使用 `#[pyclass(mapping)]` 注解来指示 PyO3 仅填充映射槽，保留序列槽为空。这将适用于 `__getitem__`、`__setitem__` 和 `__delitem__`。

使用 `#[pyclass(sequence)]` 注解来指示 PyO3 为 `__len__` 填充 `sq_length` 槽，而不是 `mp_length` 槽。这将帮助像 `numpy` 这样的库将类识别为序列，但也会导致 CPython 在将任何负索引传递给 `__getitem__` 之前自动添加序列长度。（`__getitem__`、`__setitem__` 和 `__delitem__` 映射槽仍然用于序列的切片操作。）

- `__len__(<self>) -> usize`

  实现内置函数 `len()`。

- `__contains__(<self>, object) -> bool`

  实现成员测试运算符。
  如果 `item` 在 `self` 中，则应返回 true，否则返回 false。
  对于未定义 `__contains__()` 的对象，成员测试将简单地遍历序列，直到找到匹配项。

  <details>
  <summary>禁用 Python 的默认包含</summary>

  默认情况下，所有具有 `__iter__` 方法的 `#[pyclass]` 类型支持 `in` 运算符的默认实现。那些不希望这样做的类型可以通过将 `__contains__` 设置为 `None` 来覆盖。这与纯 Python 类的机制相同。实现方式如下：

  ```rust
  # use pyo3::prelude::*;
  #
  #[pyclass]
  struct NoContains {}

  #[pymethods]
  impl NoContains {
      #[classattr]
      const __contains__: Option<PyObject> = None;
  }
  ```
  </details>

- `__getitem__(<self>, object) -> object`

  实现对 `self[a]` 元素的检索。

  *注意：* 负整数索引不会被 PyO3 特殊处理。然而，对于具有 `#[pyclass(sequence)]` 的类，当通过 `PySequence::get_item` 访问负索引时，底层 C API 已经将索引调整为正数。

- `__setitem__(<self>, object, object) -> ()`

  实现对 `self[a]` 元素的赋值。
  仅在元素可以被替换时实现。

  关于负索引的行为与 `__getitem__` 相同。

- `__delitem__(<self>, object) -> ()`

  实现对 `self[a]` 元素的删除。
  仅在元素可以被删除时实现。

  关于负索引的行为与 `__getitem__` 相同。

* `fn __concat__(&self, other: impl FromPyObject) -> PyResult<impl ToPyObject>`

  连接两个序列。
  由 `+` 运算符使用，在尝试通过 `__add__` 和 `__radd__` 方法进行数值加法后。

* `fn __repeat__(&self, count: isize) -> PyResult<impl ToPyObject>`

  将序列重复 `count` 次。
  由 `*` 运算符使用，在尝试通过 `__mul__` 和 `__rmul__` 方法进行数值乘法后。

* `fn __inplace_concat__(&self, other: impl FromPyObject) -> PyResult<impl ToPyObject>`

  连接两个序列。
  由 `+=` 运算符使用，在尝试通过 `__iadd__` 方法进行数值加法后。

* `fn __inplace_repeat__(&self, count: isize) -> PyResult<impl ToPyObject>`

  连接两个序列。
  由 `*=` 运算符使用，在尝试通过 `__imul__` 方法进行数值乘法后。

### 描述符

- `__get__(<self>, object, object) -> object`
- `__set__(<self>, object, object) -> ()`
- `__delete__(<self>, object) -> ()`

### 数字类型

二元算术运算（`+`、`-`、`*`、`@`、`/`、`//`、`%`、`divmod()`、`pow()` 和 `**`、`<<`、`>>`、`&`、`^` 和 `|`）及其反射版本：

（如果 `object` 不是签名中指定的类型，生成的代码将自动 `return NotImplemented`。）

- `__add__(<self>, object) -> object`
- `__radd__(<self>, object) -> object`
- `__sub__(<self>, object) -> object`
- `__rsub__(<self>, object) -> object`
- `__mul__(<self>, object) -> object`
- `__rmul__(<self>, object) -> object`
- `__matmul__(<self>, object) -> object`
- `__rmatmul__(<self>, object) -> object`
- `__floordiv__(<self>, object) -> object`
- `__rfloordiv__(<self>, object) -> object`
- `__truediv__(<self>, object) -> object`
- `__rtruediv__(<self>, object) -> object`
- `__divmod__(<self>, object) -> object`
- `__rdivmod__(<self>, object) -> object`
- `__mod__(<self>, object) -> object`
- `__rmod__(<self>, object) -> object`
- `__lshift__(<self>, object) -> object`
- `__rlshift__(<self>, object) -> object`
- `__rshift__(<self>, object) -> object`
- `__rrshift__(<self>, object) -> object`
- `__and__(<self>, object) -> object`
- `__rand__(<self>, object) -> object`
- `__xor__(<self>, object) -> object`
- `__rxor__(<self>, object) -> object`
- `__or__(<self>, object) -> object`
- `__ror__(<self>, object) -> object`
- `__pow__(<self>, object, object) -> object`
- `__rpow__(<self>, object, object) -> object`

就地赋值运算（`+=`、`-=`、`*=`、`@=`、`/=`、`//=`、`%=`、`**=`、`<<=`、`>>=`、`&=`、`^=` 和 `|=`）：

- `__iadd__(<self>, object) -> ()`
- `__isub__(<self>, object) -> ()`
- `__imul__(<self>, object) -> ()`
- `__imatmul__(<self>, object) -> ()`
- `__itruediv__(<self>, object) -> ()`
- `__ifloordiv__(<self>, object) -> ()`
- `__imod__(<self>, object) -> ()`
- `__ipow__(<self>, object, object) -> ()`
- `__ilshift__(<self>, object) -> ()`
- `__irshift__(<self>, object) -> ()`
- `__iand__(<self>, object) -> ()`
- `__ixor__(<self>, object) -> ()`
- `__ior__(<self>, object) -> ()`

一元运算（`-`、`+`、`abs()` 和 `~`）：

- `__pos__(<self>) -> object`
- `__neg__(<self>) -> object`
- `__abs__(<self>) -> object`
- `__invert__(<self>) -> object`

强制转换：

- `__index__(<self>) -> object (int)`
- `__int__(<self>) -> object (int)`
- `__float__(<self>) -> object (float)`

### 缓冲对象

- `__getbuffer__(<self>, *mut ffi::Py_buffer, flags) -> ()`
- `__releasebuffer__(<self>, *mut ffi::Py_buffer) -> ()`
  从 `__releasebuffer__` 返回的错误将被发送到 `sys.unraiseablehook`。强烈建议永远不要从 `__releasebuffer__` 返回错误，如果确实有必要，请尽力在返回之前执行任何所需的释放操作。`__releasebuffer__` 不会被第二次调用；任何未释放的内容将会泄漏。

### 垃圾回收器集成

如果您的类型拥有对其他 Python 对象的引用，您需要与 Python 的垃圾回收器集成，以便 GC 知道这些引用。为此，实现两个方法 `__traverse__` 和 `__clear__`。这些对应于 Python C API 中的槽 `tp_traverse` 和 `tp_clear`。`__traverse__` 必须为每个对其他 Python 对象的引用调用 `visit.call()`。`__clear__` 必须清除对其他 Python 对象的任何可变引用（从而打破引用循环）。不可变引用不必清除，因为每个循环必须至少包含一个可变引用。

- `__traverse__(<self>, pyo3::class::gc::PyVisit<'_>) -> Result<(), pyo3::class::gc::PyTraverseError>`
- `__clear__(<self>) -> ()`

示例：

```rust
use pyo3::prelude::*;
use pyo3::PyTraverseError;
use pyo3::gc::PyVisit;

#[pyclass]
struct ClassWithGCSupport {
    obj: Option<PyObject>,
}

#[pymethods]
impl ClassWithGCSupport {
    fn __traverse__(&self, visit: PyVisit<'_>) -> Result<(), PyTraverseError> {
        if let Some(obj) = &self.obj {
            visit.call(obj)?
        }
        Ok(())
    }

    fn __clear__(&mut self) {
        // 清除引用，这会减少引用计数。
        self.obj = None;
    }
}
```

通常，`__traverse__` 的实现应该只包含对 `visit.call` 的调用。最重要的是，在 `__traverse__` 的实现中禁止安全访问 GIL，即 `Python::with_gil` 将会引发恐慌。

> 注意：这些方法是 C API 的一部分，PyPy 不一定会遵循它们。如果您为 PyPy 构建，应该测量内存消耗，以确保没有内存增长失控。请参阅 [PyPy bug 跟踪器上的此问题](https://github.com/pypy/pypy/issues/3848)。

[`PySequence`]: {{#PYO3_DOCS_URL}}/pyo3/types/struct.PySequence.html
[`CompareOp::matches`]: {{#PYO3_DOCS_URL}}/pyo3/pyclass/enum.CompareOp.html#method.matches