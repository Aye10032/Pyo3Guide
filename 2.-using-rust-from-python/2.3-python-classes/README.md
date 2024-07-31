# Python 类

PyO3 通过 Rust 的 proc macro 系统提供了一组属性，用于将 Python 类定义为 Rust 结构体。

主要属性是 `#[pyclass]`，它被放置在 Rust 的 `struct` 或 `enum` 上，以为其生成一个 Python 类型。通常，它们还会有一个用 `#[pymethods]` 注解的 `impl` 块，用于为生成的 Python 类型定义 Python 方法和常量。（如果启用了 [`multiple-pymethods`] 特性，每个 `#[pyclass]` 允许有多个 `#[pymethods]` 块。）`#[pymethods]` 还可以实现 Python 魔法方法，例如 `__str__`。

本章将讨论这些属性提供的功能和配置。以下是本章相关部分的链接列表：

- [`#[pyclass]`](#defining-a-new-class)
  - [`#[pyo3(get, set)]`](#object-properties-using-pyo3get-set)
- [`#[pymethods]`](#instance-methods)
  - [`#[new]`](#constructor)
  - [`#[getter]`](#object-properties-using-getter-and-setter)
  - [`#[setter]`](#object-properties-using-getter-and-setter)
  - [`#[staticmethod]`](#static-methods)
  - [`#[classmethod]`](#class-methods)
  - [`#[classattr]`](#class-attributes)
  - [`#[args]`](#method-arguments)
- [魔法方法和槽](class/protocols.md)
- [将类作为函数参数](#classes-as-function-arguments)

## 定义新类

要定义自定义 Python 类，请将 `#[pyclass]` 属性添加到 Rust 结构体或枚举上。
```rust
# #![allow(dead_code)]
use pyo3::prelude::*;

#[pyclass]
struct MyClass {
    inner: i32,
}

// 一个“元组”结构体
#[pyclass]
struct Number(i32);

// PyO3 支持仅包含单元变体的枚举
// 这些简单的枚举行为类似于 Python 的枚举 (enum.Enum)
#[pyclass(eq, eq_int)]
#[derive(PartialEq)]
enum MyEnum {
    Variant,
    OtherVariant = 30, // PyO3 支持自定义判别式。
}

// PyO3 支持仅包含单元变体的枚举中的自定义判别式
#[pyclass(eq, eq_int)]
#[derive(PartialEq)]
enum HttpResponse {
    Ok = 200,
    NotFound = 404,
    Teapot = 418,
    // ...
}

// PyO3 还支持具有结构体和元组变体的枚举
// 这些复杂的枚举与上述简单枚举的行为略有不同
// 它们旨在与实例检查和匹配语句模式一起使用
// 变体可以混合使用
// 结构体变体具有命名字段，而元组枚举为字段生成通用名称，顺序为 _0, _1, _2, ...
// 除此之外，这两种类型在功能上是相同的
#[pyclass]
enum Shape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
    RegularPolygon(u32, f64),
    Nothing(),
}
```

上述示例为 `MyClass`、`Number`、`MyEnum`、`HttpResponse` 和 `Shape` 生成了 [`PyTypeInfo`] 和 [`PyClass`] 的实现。要查看这些生成的实现，请参阅本章末尾的 [实现细节](#implementation-details)。

### 限制

为了将 Rust 类型与 Python 集成，PyO3 需要对可以用 `#[pyclass]` 注解的类型施加一些限制。特别是，它们必须没有生命周期参数，没有泛型参数，并且必须实现 `Send`。每个限制的原因如下所述。

#### 无生命周期参数

Rust 生命周期由 Rust 编译器用于推理程序的内存安全性。它们是一个仅在编译时存在的概念；在运行时无法从动态语言如 Python 访问 Rust 生命周期。

一旦 Rust 数据暴露给 Python，就无法保证 Rust 编译器对数据的生存时间做出任何保证。Python 是一种引用计数语言，这些引用可以被持有任意长的时间，而 Rust 编译器无法追踪。正确表达这一点的唯一方法是要求任何 `#[pyclass]` 不借用任何短于 `'static` 生命周期的数据，即 `#[pyclass]` 不能有任何生命周期参数。

当您需要在 Python 和 Rust 之间共享数据的所有权时，考虑使用引用计数的智能指针，如 [`Arc`] 或 [`Py`]，而不是使用带有生命周期的借用引用。

#### 无泛型参数

Rust 的 `struct Foo<T>` 具有泛型参数 `T`，每次与不同的具体类型 `T` 一起使用时，都会生成新的编译实现。这些新实现由编译器在每个使用位置生成。这与在 Python 中包装 `Foo` 不兼容，因为需要有一个与 Python 解释器集成的 `Foo` 的单一编译实现。

目前，最佳替代方案是编写一个宏，该宏为您想要的每个实例化扩展为一个新的 `#[pyclass]`：

```rust
# #![allow(dead_code)]
use pyo3::prelude::*;

struct GenericClass<T> {
    data: T,
}

macro_rules! create_interface {
    ($name: ident, $type: ident) => {
        #[pyclass]
        pub struct $name {
            inner: GenericClass<$type>,
        }
        #[pymethods]
        impl $name {
            #[new]
            pub fn new(data: $type) -> Self {
                Self {
                    inner: GenericClass { data: data },
                }
            }
        }
    };
}

create_interface!(IntClass, i64);
create_interface!(FloatClass, String);
```

#### 必须是 Send

由于 Python 对象在 Python 解释器中可以在多个线程之间自由共享，因此无法保证哪个线程最终会释放该对象。因此，所有用 `#[pyclass]` 注解的类型必须实现 `Send`（除非用 [`#[pyclass(unsendable)]`](#customizing-the-class) 注解）。

## 构造函数

默认情况下，无法从 Python 代码创建自定义类的实例。
要声明构造函数，您需要定义一个方法并用 `#[new]` 注解它。只能指定 Python 的 `__new__` 方法，`__init__` 不可用。

```rust
# #![allow(dead_code)]
# use pyo3::prelude::*;
# #[pyclass]
# struct Number(i32);
#
#[pymethods]
impl Number {
    #[new]
    fn new(value: i32) -> Self {
        Number(value)
    }
}
```

另外，如果您的 `new` 方法可能失败，您可以返回 `PyResult<Self>`。

```rust
# #![allow(dead_code)]
# use pyo3::prelude::*;
# use pyo3::exceptions::PyValueError;
# #[pyclass]
# struct Nonzero(i32);
#
#[pymethods]
impl Nonzero {
    #[new]
    fn py_new(value: i32) -> PyResult<Self> {
        if value == 0 {
            Err(PyValueError::new_err("cannot be zero"))
        } else {
            Ok(Nonzero(value))
        }
    }
}
```

如果您想返回一个现有对象（例如，因为您的 `new` 方法缓存了它返回的值），`new` 可以返回 `pyo3::Py<Self>`。

如您所见，Rust 方法名称在这里并不重要；通过这种方式，您仍然可以使用 `new()` 作为 Rust 级别的构造函数。

如果没有声明带有 `#[new]` 的方法，则只能从 Rust 创建对象实例，而不能从 Python 创建。

有关参数，请参见下面的 [`方法参数`](#method-arguments) 部分。

## 将类添加到模块

下一步是创建模块初始化器并将我们的类添加到其中：

```rust
# #![allow(dead_code)]
# use pyo3::prelude::*;
# #[pyclass]
# struct Number(i32);
#
#[pymodule]
fn my_module(m: &Bound<'_, PyModule>) -> PyResult<()> {
    m.add_class::<Number>()?;
    Ok(())
}
```

## Bound<T> 和内部可变性

通常，将 `#[pyclass]` 类型 `T` 转换为 Python 对象并从 Rust 代码访问它是有用的。[`Py<T>`] 和 [`Bound<'py, T>`] 智能指针是 PyO3 API 中表示 Python 对象的方式。有关它们的更多详细信息，请参阅指南中的 [Python 对象](./types.md#pyo3s-smart-pointers) 部分。

大多数 Python 对象不提供独占的 (`&mut`) 访问（请参见 [Python 的内存模型部分](./python-from-rust.md#pythons-memory-model)）。然而，作为 Python 对象包装的 Rust 结构体（称为 `pyclass` 类型）通常 *确实* 需要 `&mut` 访问。由于 GIL，PyO3 *可以* 保证对它们的独占访问。

一旦对象的所有权被传递给 Python 解释器，Rust 借用检查器无法推理 `&mut` 引用。这意味着借用检查是在运行时完成的，使用的方案与 `std::cell::RefCell<T>` 非常相似。这被称为 [内部可变性](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html)。

熟悉 `RefCell<T>` 的用户可以像使用 `RefCell<T>` 一样使用 `Py<T>` 和 `Bound<'py, T>`。

对于不太熟悉 `RefCell<T>` 的用户，这里是 Rust 借用规则的提醒：
- 在任何给定时间，您可以拥有一个可变引用或任意数量的不可变引用（但不能同时拥有）。
- 引用永远不能超出它们所引用的数据的生命周期。

`Py<T>` 和 `Bound<'py, T>`，像 `RefCell<T>` 一样，通过在运行时跟踪引用来确保这些借用规则。

```rust
# use pyo3::prelude::*;
#[pyclass]
struct MyClass {
    #[pyo3(get)]
    num: i32,
}
Python::with_gil(|py| {
    let obj = Bound::new(py, MyClass { num: 3 }).unwrap();
    {
        let obj_ref = obj.borrow(); // 获取 PyRef
        assert_eq!(obj_ref.num, 3);
        // 只有在所有 PyRefs 被释放后，您才能获取 PyRefMut
        assert!(obj.try_borrow_mut().is_err());
    }
    {
        let mut obj_mut = obj.borrow_mut(); // 获取 PyRefMut
        obj_mut.num = 5;
        // 在 PyRefMut 被释放之前，您无法获取任何其他引用
        assert!(obj.try_borrow().is_err());
        assert!(obj.try_borrow_mut().is_err());
    }

    // 您可以将 `Bound` 转换为 Python 对象
    pyo3::py_run!(py, obj, "assert obj.num == 5");
});
```

`Bound<'py, T>` 限制在 GIL 生命周期 `'py` 内。要使对象的生命周期更长（例如，将其存储在 Rust 侧的结构体中），请使用 `Py<T>`。`Py<T>` 需要一个 `Python<'_>` 令牌以允许访问：

```rust
# use pyo3::prelude::*;
#[pyclass]
struct MyClass {
    num: i32,
}

fn return_myclass() -> Py<MyClass> {
    Python::with_gil(|py| Py::new(py, MyClass { num: 1 }).unwrap())
}

let obj = return_myclass();

Python::with_gil(move |py| {
    let bound = obj.bind(py); // Py<MyClass>::bind 返回 &Bound<'py, MyClass>
    let obj_ref = bound.borrow(); // 获取 PyRef<T>
    assert_eq!(obj_ref.num, 1);
});
```

### 冻结类：选择退出内部可变性

如上所述，运行时借用检查默认情况下是启用的。但是，类可以通过声明自己为 `frozen` 来选择退出。它仍然可以通过标准 Rust 类型如 `RefCell` 或 `Mutex` 使用内部可变性，但不受 PyO3 提供的实现的约束，可以根据字段逐个选择最合适的策略。

被标记为 `frozen` 且也为 `Sync` 的类，例如，它们使用 `Mutex` 而不是 `RefCell`，可以在不需要 Python GIL 的情况下通过 `Bound::get` 和 `Py::get` 方法访问：

```rust
use std::sync::atomic::{AtomicUsize, Ordering};
# use pyo3::prelude::*;

#[pyclass(frozen)]
struct FrozenCounter {
    value: AtomicUsize,
}

let py_counter: Py<FrozenCounter> = Python::with_gil(|py| {
    let counter = FrozenCounter {
        value: AtomicUsize::new(0),
    };

    Py::new(py, counter).unwrap()
});

py_counter.get().value.fetch_add(1, Ordering::Relaxed);

Python::with_gil(move |_py| drop(py_counter));
```

冻结类可能会成为默认值，从而引导 PyO3 生态系统更有意识地应用内部可变性。最终，这应该能够进一步优化 PyO3 的内部实现，并避免下游代码在不实际需要时承担内部可变性的成本。

## 自定义类

{{#include ../pyclass-parameters.md}}

这些参数在本指南的各个部分中都有介绍。

### 返回类型

通常，`#[new]` 方法必须返回 `T: Into<PyClassInitializer<Self>>` 或 `PyResult<T> where T: Into<PyClassInitializer<Self>>`。

对于可能失败的构造函数，您还应将返回类型包装在 PyResult 中。
请参阅下表以确定构造函数应返回的类型：

|                             | **不能失败**           | **可能失败**                      |
|-----------------------------|---------------------------|-----------------------------------|
|**无继承**                   | `T`                       | `PyResult<T>`                     |
|**继承(T 继承 U)**          | `(T, U)`                  | `PyResult<(T, U)>`                |
|**继承（一般情况）**         | [`PyClassInitializer<T>`] | `PyResult<PyClassInitializer<T>>` |

## 继承

默认情况下，`object`，即 `PyAny` 被用作基类。要覆盖此默认值，请使用 `pyclass` 的 `extends` 参数，并提供基类的完整路径。
目前，仅支持从 Rust 中定义的类和 PyO3 提供的内置类继承；尚不支持从其他 Python 中定义的类继承
（[#991](https://github.com/PyO3/pyo3/issues/991)）。

为了方便起见，`(T, U)` 实现了 `Into<PyClassInitializer<T>>`，其中 `U` 是 `T` 的基类。
但是，对于更深层次的嵌套继承，您必须显式返回 `PyClassInitializer<T>`。

要从子类获取父类，请在方法中使用 [`PyRef`] 而不是 `&self`，或使用 [`PyRefMut`] 而不是 `&mut self`。
然后，您可以通过 `self_.as_super()` 作为 `&PyRef<Self::BaseClass>` 访问父类，或通过 `self_.into_super()` 作为 `PyRef<Self::BaseClass>`（对于 `PyRefMut` 情况也是类似的）。为了方便起见，`self_.as_ref()` 也可以用来直接获取 `&Self::BaseClass`；然而，这种方法不允许您访问继承层次中更高的基类，您需要链式调用多个 `as_super` 或 `into_super`。

```rust
# use pyo3::prelude::*;

#[pyclass(subclass)]
struct BaseClass {
    val1: usize,
}

#[pymethods]
impl BaseClass {
    #[new]
    fn new() -> Self {
        BaseClass { val1: 10 }
    }

    pub fn method1(&self) -> PyResult<usize> {
        Ok(self.val1)
    }
}

#[pyclass(extends=BaseClass, subclass)]
struct SubClass {
    val2: usize,
}

#[pymethods]
impl SubClass {
    #[new]
    fn new() -> (Self, BaseClass) {
        (SubClass { val2: 15 }, BaseClass::new())
    }

    fn method2(self_: PyRef<'_, Self>) -> PyResult<usize> {
        let super_ = self_.as_super(); // 获取 &PyRef<BaseClass>
        super_.method1().map(|x| x * self_.val2)
    }
}

#[pyclass(extends=SubClass)]
struct SubSubClass {
    val3: usize,
}

#[pymethods]
impl SubSubClass {
    #[new]
    fn new() -> PyClassInitializer<Self> {
        PyClassInitializer::from(SubClass::new()).add_subclass(SubSubClass { val3: 20 })
    }

    fn method3(self_: PyRef<'_, Self>) -> PyResult<usize> {
        let base = self_.as_super().as_super(); // 获取 &PyRef<'_, BaseClass>
        base.method1().map(|x| x * self_.val3)
    }

    fn method4(self_: PyRef<'_, Self>) -> PyResult<usize> {
        let v = self_.val3;
        let super_ = self_.into_super(); // 获取 PyRef<'_, SubClass>
        SubClass::method2(super_).map(|x| x * v)
    }

    fn get_values(self_: PyRef<'_, Self>) -> (usize, usize, usize) {
        let val1 = self_.as_super().as_super().val1;
        let val2 = self_.as_super().val2;
        (val1, val2, self_.val3)
    }

    fn double_values(mut self_: PyRefMut<'_, Self>) {
        self_.as_super().as_super().val1 *= 2;
        self_.as_super().val2 *= 2;
        self_.val3 *= 2;
    }

    #[staticmethod]
    fn factory_method(py: Python<'_>, val: usize) -> PyResult<PyObject> {
        let base = PyClassInitializer::from(BaseClass::new());
        let sub = base.add_subclass(SubClass { val2: val });
        if val % 2 == 0 {
            Ok(Py::new(py, sub)?.to_object(py))
        } else {
            let sub_sub = sub.add_subclass(SubSubClass { val3: val });
            Ok(Py::new(py, sub_sub)?.to_object(py))
        }
    }
}
# Python::with_gil(|py| {
#     let subsub = pyo3::Py::new(py, SubSubClass::new()).unwrap();
#     pyo3::py_run!(py, subsub, "assert subsub.method1() == 10");
#     pyo3::py_run!(py, subsub, "assert subsub.method2() == 150");
#     pyo3::py_run!(py, subsub, "assert subsub.method3() == 200");
#     pyo3::py_run!(py, subsub, "assert subsub.method4() == 3000");
#     pyo3::py_run!(py, subsub, "assert subsub.get_values() == (10, 15, 20)");
#     pyo3::py_run!(py, subsub, "assert subsub.double_values() == None");
#     pyo3::py_run!(py, subsub, "assert subsub.get_values() == (20, 30, 40)");
#     let subsub = SubSubClass::factory_method(py, 2).unwrap();
#     let subsubsub = SubSubClass::factory_method(py, 3).unwrap();
#     let cls = py.get_type_bound::<SubSubClass>();
#     pyo3::py_run!(py, subsub cls, "assert not isinstance(subsub, cls)");
#     pyo3::py_run!(py, subsubsub cls, "assert isinstance(subsubsub, cls)");
# });
```

您可以继承原生类型，如 `PyDict`，如果它们实现了 [`PySizedLayout`]({{#PYO3_DOCS_URL}}/pyo3/type_object/trait.PySizedLayout.html)。
在为 Python 限制 API（即 PyO3 的 `abi3` 特性）构建时不支持此功能。

要在 Rust 类型和其原生基类之间转换，您可以将 `slf` 作为 Python 对象。要访问 Rust 字段，请使用 `slf.borrow()` 或 `slf.borrow_mut()`，要访问基类，请使用 `slf.downcast::<BaseClass>()`。

```rust
# #[cfg(not(Py_LIMITED_API))] {
# use pyo3::prelude::*;
use pyo3::types::PyDict;
use std::collections::HashMap;

#[pyclass(extends=PyDict)]
#[derive(Default)]
struct DictWithCounter {
    counter: HashMap<String, usize>,
}

#[pymethods]
impl DictWithCounter {
    #[new]
    fn new() -> Self {
        Self::default()
    }

    fn set(slf: &Bound<'_, Self>, key: String, value: Bound<'_, PyAny>) -> PyResult<()> {
        slf.borrow_mut().counter.entry(key.clone()).or_insert(0);
        let dict = slf.downcast::<PyDict>()?;
        dict.set_item(key, value)
    }
}
# Python::with_gil(|py| {
#     let cnt = pyo3::Py::new(py, DictWithCounter::new()).unwrap();
#     pyo3::py_run!(py, cnt, "cnt.set('abc', 10); assert cnt['abc'] == 10")
# });
# }
```

如果 `SubClass` 没有提供基类初始化，则编译将失败。
```rust,compile_fail
# use pyo3::prelude::*;

#[pyclass]
struct BaseClass {
    val1: usize,
}

#[pyclass(extends=BaseClass)]
struct SubClass {
    val2: usize,
}

#[pymethods]
impl SubClass {
    #[new]
    fn new() -> Self {
        SubClass { val2: 15 }
    }
}
```

从 Python 创建新实例时，原生基类的 `__new__` 构造函数会被隐式调用。确保在 `#[new]` 方法中接受您希望基类获取的参数，即使它们在该 `fn` 中未使用：

```rust
# #[allow(dead_code)]
# #[cfg(not(Py_LIMITED_API))] {
# use pyo3::prelude::*;
use pyo3::types::PyDict;

#[pyclass(extends=PyDict)]
struct MyDict {
    private: i32,
}

#[pymethods]
impl MyDict {
    #[new]
    #[pyo3(signature = (*args, **kwargs))]
    fn new(args: &Bound<'_, PyAny>, kwargs: Option<&Bound<'_, PyAny>>) -> Self {
        Self { private: 0 }
    }

    // 一些使用 `private` 的自定义方法...
}
# Python::with_gil(|py| {
#     let cls = py.get_type_bound::<MyDict>();
#     pyo3::py_run!(py, cls, "cls(a=1, b=2)")
# });
# }
```

在这种情况下，`args` 和 `kwargs` 允许通过传递初始项来创建子类实例，例如 `MyDict(item_sequence)` 或 `MyDict(a=1, b=2)`。

## 对象属性

PyO3 支持两种方式为您的 `#[pyclass]` 添加属性：
- 对于没有副作用的简单结构字段，可以直接在 `#[pyclass]` 中的字段定义上添加 `#[pyo3(get, set)]` 属性。
- 对于需要计算的属性，您可以在 [`#[pymethods]`](#instance-methods) 块中定义 `#[getter]` 和 `#[setter]` 函数。

我们将在以下部分中介绍每种方法。

### 使用 `#[pyo3(get, set)]` 的对象属性

对于简单情况，其中成员变量仅被读取和写入且没有副作用，您可以在 `#[pyclass]` 字段定义中使用 `pyo3` 属性声明 getter 和 setter，如下例所示：

```rust
# use pyo3::prelude::*;
# #[allow(dead_code)]
#[pyclass]
struct MyClass {
    #[pyo3(get, set)]
    num: i32,
}
```

上述代码将使 `num` 字段作为 `self.num` Python 属性可供读取和写入。要以不同名称公开属性，请在其余选项旁边指定，例如 `#[pyo3(get, set, name = "custom_name")]`。

属性可以是只读或只写，分别使用 `#[pyo3(get)]` 或 `#[pyo3(set)]`。

要使用这些注解，您的字段类型必须实现一些转换特征：
- 对于 `get`，字段类型必须同时实现 `IntoPy<PyObject>` 和 `Clone`。
- 对于 `set`，字段类型必须实现 `FromPyObject`。

例如，如果内部类型也实现了该特征，则为 `Cell` 类型提供了这些特征的实现。这意味着您可以在包装在 `Cell` 中的字段上使用 `#[pyo3(get, set)]`。

### 使用 `#[getter]` 和 `#[setter]` 的对象属性

对于不满足 `#[pyo3(get, set)]` 特征要求或需要副作用的情况，可以在 `#[pymethods]` `impl` 块中定义描述符方法。

这可以通过使用 `#[getter]` 和 `#[setter]` 属性来完成，如下例所示：

```rust
# use pyo3::prelude::*;
#[pyclass]
struct MyClass {
    num: i32,
}

#[pymethods]
impl MyClass {
    #[getter]
    fn num(&self) -> PyResult<i32> {
        Ok(self.num)
    }
}
```

getter 或 setter 的函数名称默认用作属性名称。有几种方法可以覆盖名称。

如果函数名称以 `get_` 或 `set_` 开头，分别用于 getter 或 setter，则描述符名称将成为去掉该前缀后的函数名称。这在 Rust 关键字（如 `type`）的情况下也很有用
（自 Rust 2018 起可以使用 [原始标识符](https://doc.rust-lang.org/edition-guide/rust-2018/module-system/raw-identifiers.html)）。

```rust
# use pyo3::prelude::*;
# #[pyclass]
# struct MyClass {
#     num: i32,
# }
#[pymethods]
impl MyClass {
    #[getter]
    fn get_num(&self) -> PyResult<i32> {
        Ok(self.num)
    }

    #[setter]
    fn set_num(&mut self, value: i32) -> PyResult<()> {
        self.num = value;
        Ok(())
    }
}
```

在这种情况下，定义了一个属性 `num`，并可以从 Python 代码中作为 `self.num` 访问。

由 `#[setter]` 或 `#[pyo3(set)]` 定义的属性在 `del` 操作上将始终引发 `AttributeError`。对定义自定义 `del` 行为的支持正在跟踪 [#1778](https://github.com/PyO3/pyo3/issues/1778)。

## 实例方法

要定义与 Python 兼容的方法，必须为您的结构体注解一个 `#[pymethods]` 块。PyO3 为此块中的所有函数生成与 Python 兼容的包装器，并有一些变体，如描述符、类方法、静态方法等。

由于 Rust 允许任意数量的 `impl` 块，您可以轻松地将方法分为可供 Python（和 Rust）访问的方法和仅供 Rust 访问的方法。但是，要为同一结构体拥有多个用 `#[pymethods]` 注解的 `impl` 块，您必须启用 PyO3 的 [`multiple-pymethods`] 特性。

```rust
# use pyo3::prelude::*;
# #[pyclass]
# struct MyClass {
#     num: i32,
# }
#[pymethods]
impl MyClass {
    fn method1(&self) -> PyResult<i32> {
        Ok(10)
    }

    fn set_method(&mut self, value: i32) -> PyResult<()> {
        self.num = value;
        Ok(())
    }
}
```

对这些方法的调用受到 GIL 的保护，因此可以使用 `&self` 和 `&mut self`。
返回类型必须是 `PyResult<T>` 或 `T`，其中某个 `T` 实现了 `IntoPy<PyObject>`；后者在方法无法引发 Python 异常时是允许的。

可以在方法签名中指定 `Python` 参数，在这种情况下，`py` 参数由方法包装器注入，例如：

```rust
# use pyo3::prelude::*;
# #[pyclass]
# struct MyClass {
# #[allow(dead_code)]
#     num: i32,
# }
#[pymethods]
impl MyClass {
    fn method2(&self, py: Python<'_>) -> PyResult<i32> {
        Ok(10)
    }
}
```

从 Python 的角度来看，此示例中的 `method2` 不接受任何参数。

## 类方法

要为自定义类创建类方法，方法需要用 `#[classmethod]` 属性注解。
这相当于 Python 装饰器 `@classmethod`。

```rust
# use pyo3::prelude::*;
# use pyo3::types::PyType;
# #[pyclass]
# struct MyClass {
#     #[allow(dead_code)]
#     num: i32,
# }
#[pymethods]
impl MyClass {
    #[classmethod]
    fn cls_method(cls: &Bound<'_, PyType>) -> PyResult<i32> {
        Ok(10)
    }
}
```

声明一个可以从 Python 调用的类方法。

* 第一个参数是调用该方法的类的类型对象。
  这可能是派生类的类型对象。
* 第一个参数隐式具有类型 `&Bound<'_, PyType>`。
* 有关 `parameter-list` 的详细信息，请参阅 `方法参数` 部分的文档。
* 返回类型必须是 `PyResult<T>` 或 `T`，其中某个 `T` 实现了 `IntoPy<PyObject>`。

### 接受类参数的构造函数

要创建一个接受位置类参数的构造函数，您可以将 `#[classmethod]` 和 `#[new]` 修饰符结合使用：
```rust
# #![allow(dead_code)]
# use pyo3::prelude::*;
# use pyo3::types::PyType;
# #[pyclass]
# struct BaseClass(PyObject);
#
#[pymethods]
impl BaseClass {
    #[new]
    #[classmethod]
    fn py_new(cls: &Bound<'_, PyType>) -> PyResult<Self> {
        // 获取在此类的子类上声明的抽象属性（假设）。
        let subclass_attr: Bound<'_, PyAny> = cls.getattr("a_class_attr")?;
        Ok(Self(subclass_attr.unbind()))
    }
}
```

## 静态方法

要为自定义类创建静态方法，方法需要用 `#[staticmethod]` 属性注解。返回类型必须是 `T` 或 `PyResult<T>`，其中某个 `T` 实现了 `IntoPy<PyObject>`。

```rust
# use pyo3::prelude::*;
# #[pyclass]
# struct MyClass {
#     #[allow(dead_code)]
#     num: i32,
# }
#[pymethods]
impl MyClass {
    #[staticmethod]
    fn static_method(param1: i32, param2: &str) -> PyResult<i32> {
        Ok(10)
    }
}
```

## 类属性

要创建类属性（也称为 [类变量][classattr]），可以用 `#[classattr]` 属性注解一个没有任何参数的方法。

```rust
# use pyo3::prelude::*;
# #[pyclass]
# struct MyClass {}
#[pymethods]
impl MyClass {
    #[classattr]
    fn my_attribute() -> String {
        "hello".to_string()
    }
}

Python::with_gil(|py| {
    let my_class = py.get_type_bound::<MyClass>();
    pyo3::py_run!(py, my_class, "assert my_class.my_attribute == 'hello'")
});
```

> 注意：如果方法具有 `Result` 返回类型并返回 `Err`，则 PyO3 在类创建期间将引发 panic。

如果类属性仅用 `const` 代码定义，则还可以注解关联常量：

```rust
# use pyo3::prelude::*;
# #[pyclass]
# struct MyClass {}
#[pymethods]
impl MyClass {
    #[classattr]
    const MY_CONST_ATTRIBUTE: &'static str = "foobar";
}
```

## 将类作为函数参数

使用 `#[pyfunction]` 定义的自由函数通过与实例方法的 self 参数相同的机制与类交互，即它们可以接受 GIL 绑定的引用、GIL 绑定的引用包装器或 GIL 独立的引用：

```rust
# #![allow(dead_code)]
# use pyo3::prelude::*;
#[pyclass]
struct MyClass {
    my_field: i32,
}

// 当底层 `Bound` 无关紧要时，获取引用。
#[pyfunction]
fn increment_field(my_class: &mut MyClass) {
    my_class.my_field += 1;
}

// 当借用应为自动时，获取引用包装器，
// 但希望与底层 `Bound` 进行交互。
#[pyfunction]
fn print_field(my_class: PyRef<'_, MyClass>) {
    println!("{}", my_class.my_field);
}

// 当需要手动管理借用时，获取对底层 Bound 的引用
#[pyfunction]
fn increment_then_print_field(my_class: &Bound<'_, MyClass>) {
    my_class.borrow_mut().my_field += 1;

    println!("{}", my_class.borrow().my_field);
}

// 当您希望将引用存储在其他地方时，获取 GIL 独立的引用。
#[pyfunction]
fn print_refcnt(my_class: Py<MyClass>, py: Python<'_>) {
    println!("{}", my_class.get_refcnt(py));
}
```

如果类可以被克隆，则也可以按值传递它们，即如果它们实现了 `FromPyObject`，则会自动实现 `Clone`，例如通过 `#[derive(Clone)]`：

```rust
# #![allow(dead_code)]
# use pyo3::prelude::*;
#[pyclass]
#[derive(Clone)]
struct MyClass {
    my_field: Box<i32>,
}

#[pyfunction]
fn dissamble_clone(my_class: MyClass) {
    let MyClass { mut my_field } = my_class;
    *my_field += 1;
}
```

请注意，在类上使用 `#[derive(FromPyObject)]` 通常没有用，因为它试图通过查找任何给定 Python 值的属性来构造一个新的 Rust 值。

## 方法参数

类似于 `#[pyfunction]`，可以使用 `#[pyo3(signature = (...))]` 属性来指定 `#[pymethods]` 接受参数的方式。请参阅 [`函数签名`](./function/signature.md) 的文档以了解此属性接受的参数。

以下示例定义了一个类 `MyClass`，其中的方法 `method`。该方法具有设置 `num` 和 `name` 的默认值的签名，并指示 `py_args` 应收集所有额外的位置参数，`py_kwargs` 应收集所有额外的关键字参数：

```rust
# use pyo3::prelude::*;
use pyo3::types::{PyDict, PyTuple};
#
# #[pyclass]
# struct MyClass {
#     num: i32,
# }
#[pymethods]
impl MyClass {
    #[new]
    #[pyo3(signature = (num=-1))]
    fn new(num: i32) -> Self {
        MyClass { num }
    }

    #[pyo3(signature = (num=10, *py_args, name="Hello", **py_kwargs))]
    fn method(
        &mut self,
        num: i32,
        py_args: &Bound<'_, PyTuple>,
        name: &str,
        py_kwargs: Option<&Bound<'_, PyDict>>,
    ) -> String {
        let num_before = self.num;
        self.num = num;
        format!(
            "num={} (was previously={}), py_args={:?}, name={}, py_kwargs={:?} ",
            num, num_before, py_args, name, py_kwargs,
        )
    }
}
```

在 Python 中，这可能会像这样使用：

```python
>>> import mymodule
>>> mc = mymodule.MyClass()
>>> print(mc.method(44, False, "World", 666, x=44, y=55))
py_args=('World', 666), py_kwargs=Some({'x': 44, 'y': 55}), name=Hello, num=44, num_before=-1
>>> print(mc.method(num=-1, name="World"))
py_args=(), py_kwargs=None, name=World, num=-1, num_before=44
```

`#[pyo3(text_signature = "...")`](./function/signature.md#overriding-the-generated-signature) 选项也适用于 `#[pymethods]`。

```rust
# #![allow(dead_code)]
use pyo3::prelude::*;
use pyo3::types::PyType;

#[pyclass]
struct MyClass {}

#[pymethods]
impl MyClass {
    #[new]
    #[pyo3(text_signature = "(c, d)")]
    fn new(c: i32, d: &str) -> Self {
        Self {}
    }
    // self 参数应写为 $self
    #[pyo3(text_signature = "($self, e, f)")]
    fn my_method(&self, e: i32, f: i32) -> i32 {
        e + f
    }
    // 类方法参数同样，使用 $cls
    #[classmethod]
    #[pyo3(text_signature = "($cls, e, f)")]
    fn my_class_method(cls: &Bound<'_, PyType>, e: i32, f: i32) -> i32 {
        e + f
    }
    #[staticmethod]
    #[pyo3(text_signature = "(e, f)")]
    fn my_static_method(e: i32, f: i32) -> i32 {
        e + f
    }
}
#
# fn main() -> PyResult<()> {
#     Python::with_gil(|py| {
#         let inspect = PyModule::import_bound(py, "inspect")?.getattr("signature")?;
#         let module = PyModule::new_bound(py, "my_module")?;
#         module.add_class::<MyClass>()?;
#         let class = module.getattr("MyClass")?;
#
#         if cfg!(not(Py_LIMITED_API)) || py.version_info() >= (3, 10)  {
#             let doc: String = class.getattr("__doc__")?.extract()?;
#             assert_eq!(doc, "");
#
#             let sig: String = inspect
#                 .call1((&class,))?
#                 .call_method0("__str__")?
#                 .extract()?;
#             assert_eq!(sig, "(c, d)");
#         } else {
#             let doc: String = class.getattr("__doc__")?.extract()?;
#             assert_eq!(doc, "");
#
#             inspect.call1((&class,)).expect_err("`text_signature` on classes is not compatible with compilation in `abi3` mode until Python 3.10 or greater");
#          }
#
#         {
#             let method = class.getattr("my_method")?;
#
#             assert!(method.getattr("__doc__")?.is_none());
#
#             let sig: String = inspect
#                 .call1((method,))?
#                 .call_method0("__str__")?
#                 .extract()?;
#             assert_eq!(sig, "(self, /, e, f)");
#         }
#
#         {
#             let method = class.getattr("my_class_method")?;
#
#             assert!(method.getattr("__doc__")?.is_none());
#
#             let sig: String = inspect
#                 .call1((method,))?
#                 .call_method0("__str__")?
#                 .extract()?;
#             assert_eq!(sig, "(e, f)");  // inspect.signature 跳过 $cls 参数
#         }
#
#         {
#             let method = class.getattr("my_static_method")?;
#
#             assert!(method.getattr("__doc__")?.is_none());
#
#             let sig: String = inspect
#                 .call1((method,))?
#                 .call_method0("__str__")?
#                 .extract()?;
#             assert_eq!(sig, "(e, f)");
#         }
#
#         Ok(())
#     })
# }
```

请注意，`#[new]` 上的 `text_signature` 在 `abi3` 模式下与 Python 3.10 或更高版本不兼容。

### 方法接收者和生命周期省略

PyO3 支持使用共享的 `&self` 和唯一的 `&mut self` 引用编写实例方法。这与 [生命周期省略][lifetime-elision] 相关，因为此类接收者的生命周期被分配给所有省略的输出生命周期参数。

这是一般 Rust 代码的良好默认值，因为返回值更可能从接收者借用，而不是从其他参数借用（如果它们根本包含任何生命周期）。但是，在基于 PyO3 的代码中返回绑定引用 `Bound<'py, T>` 时，GIL 生命周期 `'py` 通常应从作为参数传递的 GIL 令牌 `py: Python<'py>` 派生。

具体来说，像

```rust,ignore
fn frobnicate(&self, py: Python) -> Bound<Foo>;
```

将不起作用，因为它们被推断为

```rust,ignore
fn frobnicate<'a, 'py>(&'a self, py: Python<'py>) -> Bound<'a, Foo>;
```

而不是预期的

```rust,ignore
fn frobnicate<'a, 'py>(&'a self, py: Python<'py>) -> Bound<'py, Foo>;
```

通常应写为

```rust,ignore
fn frobnicate<'py>(&self, py: Python<'py>) -> Bound<'py, Foo>;
```

对于 `#[pyfunction]`，不存在此问题，因为接收者生命周期的特殊情况不适用，确实像

```rust,ignore
fn frobnicate(bar: &Bar, py: Python) -> Bound<Foo>;
```

将产生编译器错误 [E0106 "missing lifetime specifier"][compiler-error-e0106]。

## `#[pyclass]` 枚举

PyO3 中的枚举支持有两种风格，具体取决于枚举具有何种变体：简单和复杂。

### 简单枚举

简单枚举（又称 C 风格枚举）仅具有单元变体。

PyO3 为每个变体添加一个类属性，因此您可以在 Python 中访问它们，而无需定义 `#[new]`。PyO3 还提供 `__richcmp__` 和 `__int__` 的默认实现，因此可以使用 `==` 进行比较：

```rust
# use pyo3::prelude::*;
#[pyclass(eq, eq_int)]
#[derive(PartialEq)]
enum MyEnum {
    Variant,
    OtherVariant,
}

Python::with_gil(|py| {
    let x = Py::new(py, MyEnum::Variant).unwrap();
    let y = Py::new(py, MyEnum::OtherVariant).unwrap();
    let cls = py.get_type_bound::<MyEnum>();
    pyo3::py_run!(py, x y cls, r#"
        assert x == cls.Variant
        assert y == cls.OtherVariant
        assert x != y
    "#)
})
```

您还可以将简单枚举转换为 `int`：

```rust
# use pyo3::prelude::*;
#[pyclass(eq, eq_int)]
#[derive(PartialEq)]
enum MyEnum {
    Variant,
    OtherVariant = 10,
}

Python::with_gil(|py| {
    let cls = py.get_type_bound::<MyEnum>();
    let x = MyEnum::Variant as i32; // 精确值由编译器分配。
    pyo3::py_run!(py, cls x, r#"
        assert int(cls.Variant) == x
        assert int(cls.OtherVariant) == 10
    "#)
})
```

PyO3 还为枚举提供 `__repr__`：

```rust
# use pyo3::prelude::*;
#[pyclass(eq, eq_int)]
#[derive(PartialEq)]
enum MyEnum{
    Variant,
    OtherVariant,
}

Python::with_gil(|py| {
    let cls = py.get_type_bound::<MyEnum>();
    let x = Py::new(py, MyEnum::Variant).unwrap();
    pyo3::py_run!(py, cls x, r#"
        assert repr(x) == 'MyEnum.Variant'
        assert repr(cls.OtherVariant) == 'MyEnum.OtherVariant'
    "#)
})
```

PyO3 定义的所有方法都可以被重写。例如，以下是如何重写 `__repr__`：

```rust
# use pyo3::prelude::*;
#[pyclass(eq, eq_int)]
#[derive(PartialEq)]
enum MyEnum {
    Answer = 42,
}

#[pymethods]
impl MyEnum {
    fn __repr__(&self) -> &'static str {
        "42"
    }
}

Python::with_gil(|py| {
    let cls = py.get_type_bound::<MyEnum>();
    pyo3::py_run!(py, cls, "assert repr(cls.Answer) == '42'")
})
```

枚举及其变体也可以使用 `#[pyo3(name)]` 重命名。

```rust
# use pyo3::prelude::*;
#[pyclass(eq, eq_int, name = "RenamedEnum")]
#[derive(PartialEq)]
enum MyEnum {
    #[pyo3(name = "UPPERCASE")]
    Variant,
}

Python::with_gil(|py| {
    let x = Py::new(py, MyEnum::Variant).unwrap();
    let cls = py.get_type_bound::<MyEnum>();
    pyo3::py_run!(py, x cls, r#"
        assert repr(x) == 'RenamedEnum.UPPERCASE'
        assert x == cls.UPPERCASE
    "#)
})
```

可选地使用 `#[pyo3(ord)]` 添加枚举变体的顺序。
*注意：在传递 `ord` 参数时，必须实现 `PartialOrd` 特征。如果未实现，将引发编译时错误。*

```rust
# use pyo3::prelude::*;
#[pyclass(eq, ord)]
#[derive(PartialEq, PartialOrd)]
enum MyEnum{
    A,
    B,
    C,
}

Python::with_gil(|py| {
    let cls = py.get_type_bound::<MyEnum>();
    let a = Py::new(py, MyEnum::A).unwrap();
    let b = Py::new(py, MyEnum::B).unwrap();
    let c = Py::new(py, MyEnum::C).unwrap();
    pyo3::py_run!(py, cls a b c, r#"
        assert (a < b) == True
        assert (c <= b) == False
        assert (c > a) == True
    "#)
})
```

您不能将枚举用作基类或让枚举从其他类继承。

```rust,compile_fail
# use pyo3::prelude::*;
#[pyclass(subclass)]
enum BadBase {
    Var1,
}
```

```rust,compile_fail
# use pyo3::prelude::*;

#[pyclass(subclass)]
struct Base;

#[pyclass(extends=Base)]
enum BadSubclass {
    Var1,
}
```

`#[pyclass]` 枚举当前与 Python 中的 `IntEnum` 不兼容。

### 复杂枚举

如果枚举具有任何非单元（结构体或元组）变体，则该枚举是复杂的。

PyO3 仅支持复杂枚举中的结构体和元组变体。目前不支持单元变体（建议使用空元组枚举）。

PyO3 为每个变体添加一个类属性，可以用于构造值和匹配模式。PyO3 还为每个变体的所有字段提供 getter 方法。

```rust
# use pyo3::prelude::*;
#[pyclass]
enum Shape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
    RegularPolygon(u32, f64),
    Nothing { },
}

# #[cfg(Py_3_10)]
Python::with_gil(|py| {
    let circle = Shape::Circle { radius: 10.0 }.into_py(py);
    let square = Shape::RegularPolygon(4, 10.0).into_py(py);
    let cls = py.get_type_bound::<Shape>();
    pyo3::py_run!(py, circle square cls, r#"
        assert isinstance(circle, cls)
        assert isinstance(circle, cls.Circle)
        assert circle.radius == 10.0

        assert isinstance(square, cls)
        assert isinstance(square, cls.RegularPolygon)
        assert square[0] == 4 # 获取 _0 字段
        assert square[1] == 10.0 # 获取 _1 字段

        def count_vertices(cls, shape):
            match shape:
                case cls.Circle():
                    return 0
                case cls.Rectangle():
                    return 4
                case cls.RegularPolygon(n):
                    return n
                case cls.Nothing():
                    return 0

        assert count_vertices(cls, circle) == 0
        assert count_vertices(cls, square) == 4
    "#)
})
```

警告：`Py::new` 和 `.into_py` 当前不一致。请注意，构造的值 _不是_ 特定变体的实例。因此，仅建议使用 `.into_py` 构造值。

```rust
# use pyo3::prelude::*;
#[pyclass]
enum MyEnum {
    Variant { i: i32 },
}

Python::with_gil(|py| {
    let x = Py::new(py, MyEnum::Variant { i: 42 }).unwrap();
    let cls = py.get_type_bound::<MyEnum>();
    pyo3::py_run!(py, x cls, r#"
        assert isinstance(x, cls)
        assert not isinstance(x, cls.Variant)
    "#)
})
```

可以使用 `#[pyo3(constructor = (...))]` 属性自定义每个生成类的构造函数。它使用与 [`#[pyo3(signature = (...))]`](function/signature.md) 属性相同的语法，并支持相同的选项。要应用此属性，只需将其放置在 `#[pyclass]` 复杂枚举中的变体上，如下所示：

```rust
# use pyo3::prelude::*;
#[pyclass]
enum Shape {
    #[pyo3(constructor = (radius=1.0))]
    Circle { radius: f64 },
    #[pyo3(constructor = (*, width, height))]
    Rectangle { width: f64, height: f64 },
    #[pyo3(constructor = (side_count, radius=1.0))]
    RegularPolygon { side_count: u32, radius: f64 },
    Nothing { },
}

# #[cfg(Py_3_10)]
Python::with_gil(|py| {
    let cls = py.get_type_bound::<Shape>();
    pyo3::py_run!(py, cls, r#"
        circle = cls.Circle()
        assert isinstance(circle, cls)
        assert isinstance(circle, cls.Circle)
        assert circle.radius == 1.0

        square = cls.Rectangle(width = 1, height = 1)
        assert isinstance(square, cls)
        assert isinstance(square, cls.Rectangle)
        assert square.width == 1
        assert square.height == 1

        hexagon = cls.RegularPolygon(6)
        assert isinstance(hexagon, cls)
        assert isinstance(hexagon, cls.RegularPolygon)
        assert hexagon.side_count == 6
        assert hexagon.radius == 1
    "#)
})
```

## 实现细节

`#[pyclass]` 宏依赖于大量条件代码生成：每个 `#[pyclass]` 可以选择性地具有 `#[pymethods]` 块。

为了支持这种灵活性，`#[pyclass]` 宏扩展为一大块样板代码，设置 ["dtolnay 特化"](https://github.com/dtolnay/case-studies/blob/master/autoref-specialization/README.md) 的结构。这种实现模式使 Rust 编译器能够在存在时使用 `#[pymethods]` 实现，并在不存在时回退到默认（空）定义。

这种简单的技术适用于零个或一个实现的情况。为了支持同一 `#[pyclass]` 的多个 `#[pymethods]`（在 [`multiple-pymethods`] 特性中），使用 [`inventory`](https://github.com/dtolnay/inventory) crate 提供的注册机制。它在库加载时收集 `impl`，但并不支持所有平台。有关更多详细信息，请参见 [inventory: how it works](https://github.com/dtolnay/inventory#how-it-works)。

`#[pyclass]` 宏大致扩展为以下代码。`PyClassImplCollector` 是 PyO3 内部用于 dtolnay 特化的类型：

```rust
# #[cfg(not(feature = "multiple-pymethods"))] {
# use pyo3::prelude::*;
// 注意：当启用 `multiple-pymethods` 特性时，实现略有不同。
# #[allow(dead_code)]
struct MyClass {
    # #[allow(dead_code)]
    num: i32,
}

impl pyo3::types::DerefToPyAny for MyClass {}

unsafe impl pyo3::type_object::PyTypeInfo for MyClass {
    const NAME: &'static str = "MyClass";
    const MODULE: ::std::option::Option<&'static str> = ::std::option::Option::None;
    #[inline]
    fn type_object_raw(py: pyo3::Python<'_>) -> *mut pyo3::ffi::PyTypeObject {
        <Self as pyo3::impl_::pyclass::PyClassImpl>::lazy_type_object()
            .get_or_init(py)
            .as_type_ptr()
    }
}

impl pyo3::PyClass for MyClass {
    type Frozen = pyo3::pyclass::boolean_struct::False;
}

impl<'a, 'py> pyo3::impl_::extract_argument::PyFunctionArgument<'a, 'py> for &'a MyClass
{
    type Holder = ::std::option::Option<pyo3::PyRef<'py, MyClass>>;

    #[inline]
    fn extract(obj: &'a pyo3::Bound<'py, PyAny>, holder: &'a mut Self::Holder) -> pyo3::PyResult<Self> {
        pyo3::impl_::extract_argument::extract_pyclass_ref(obj, holder)
    }
}

impl<'a, 'py> pyo3::impl_::extract_argument::PyFunctionArgument<'a, 'py> for &'a mut MyClass
{
    type Holder = ::std::option::Option<pyo3::PyRefMut<'py, MyClass>>;

    #[inline]
    fn extract(obj: &'a pyo3::Bound<'py, PyAny>, holder: &'a mut Self::Holder) -> pyo3::PyResult<Self> {
        pyo3::impl_::extract_argument::extract_pyclass_ref_mut(obj, holder)
    }
}

impl pyo3::IntoPy<PyObject> for MyClass {
    fn into_py(self, py: pyo3::Python<'_>) -> pyo3::PyObject {
        pyo3::IntoPy::into_py(pyo3::Py::new(py, self).unwrap(), py)
    }
}

impl pyo3::impl_::pyclass::PyClassImpl for MyClass {
    const IS_BASETYPE: bool = false;
    const IS_SUBCLASS: bool = false;
    const IS_MAPPING: bool = false;
    const IS_SEQUENCE: bool = false;
    type BaseType = PyAny;
    type ThreadChecker = pyo3::impl_::pyclass::SendablePyClass<MyClass>;
    type PyClassMutability = <<pyo3::PyAny as pyo3::impl_::pyclass::PyClassBaseType>::PyClassMutability as pyo3::impl_::pycell::PyClassMutability>::MutableChild;
    type Dict = pyo3::impl_::pyclass::PyClassDummySlot;
    type WeakRef = pyo3::impl_::pyclass::PyClassDummySlot;
    type BaseNativeType = pyo3::PyAny;

    fn items_iter() -> pyo3::impl_::pyclass::PyClassItemsIter {
        use pyo3::impl_::pyclass::*;
        let collector = PyClassImplCollector::<MyClass>::new();
        static INTRINSIC_ITEMS: PyClassItems = PyClassItems { slots: &[], methods: &[] };
        PyClassItemsIter::new(&INTRINSIC_ITEMS, collector.py_methods())
    }

    fn lazy_type_object() -> &'static pyo3::impl_::pyclass::LazyTypeObject<MyClass> {
        use pyo3::impl_::pyclass::LazyTypeObject;
        static TYPE_OBJECT: LazyTypeObject<MyClass> = LazyTypeObject::new();
        &TYPE_OBJECT
    }

    fn doc(py: Python<'_>) -> pyo3::PyResult<&'static ::std::ffi::CStr> {
        use pyo3::impl_::pyclass::*;
        static DOC: pyo3::sync::GILOnceCell<::std::borrow::Cow<'static, ::std::ffi::CStr>> = pyo3::sync::GILOnceCell::new();
        DOC.get_or_try_init(py, || {
            let collector = PyClassImplCollector::<Self>::new();
            build_pyclass_doc(<MyClass as pyo3::PyTypeInfo>::NAME, pyo3::ffi::c_str!(""), collector.new_text_signature())
        }).map(::std::ops::Deref::deref)
    }
}

# Python::with_gil(|py| {
#     let cls = py.get_type_bound::<MyClass>();
#     pyo3::py_run!(py, cls, "assert cls.__name__ == 'MyClass'")
# });
# }
```


[`PyTypeInfo`]: {{#PYO3_DOCS_URL}}/pyo3/type_object/trait.PyTypeInfo.html

[`Py`]: {{#PYO3_DOCS_URL}}/pyo3/struct.Py.html
[`Bound<'_, T>`]: {{#PYO3_DOCS_URL}}/pyo3/struct.Bound.html
[`PyClass`]: {{#PYO3_DOCS_URL}}/pyo3/pyclass/trait.PyClass.html
[`PyRef`]: {{#PYO3_DOCS_URL}}/pyo3/pycell/struct.PyRef.html
[`PyRefMut`]: {{#PYO3_DOCS_URL}}/pyo3/pycell/struct.PyRefMut.html
[`PyClassInitializer<T>`]: {{#PYO3_DOCS_URL}}/pyo3/pyclass_init/struct.PyClassInitializer.html

[`Arc`]: https://doc.rust-lang.org/std/sync/struct.Arc.html
[`RefCell`]: https://doc.rust-lang.org/std/cell/struct.RefCell.html

[classattr]: https://docs.python.org/3/tutorial/classes.html#class-and-instance-variables

[`multiple-pymethods`]: features.md#multiple-pymethods

[lifetime-elision]: https://doc.rust-lang.org/reference/lifetime-elision.html
[compiler-error-e0106]: https://doc.rust-lang.org/error_codes/E0106.html