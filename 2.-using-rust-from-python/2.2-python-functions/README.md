# Python 函数

`#[pyfunction]` 属性用于从 Rust 函数定义一个 Python 函数。一旦定义，该函数需要使用 `wrap_pyfunction!` 宏添加到 [模块](./module.md) 中。

以下示例在名为 `my_extension` 的 Python 模块中定义了一个名为 `double` 的函数：

```rust
use pyo3::prelude::*;

#[pyfunction]
fn double(x: usize) -> usize {
    x * 2
}

#[pymodule]
fn my_extension(m: &Bound<'_, PyModule>) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(double, m)?)
}
```

本指南的这一章节解释了 `#[pyfunction]` 属性的完整用法。在这一部分中，涵盖了以下主题：

- [函数选项](#function-options)
  - [`#[pyo3(name = "...")]`](#name)
  - [`#[pyo3(signature = (...))]`](#signature)
  - [`#[pyo3(text_signature = "...")]`](#text_signature)
  - [`#[pyo3(pass_module)]`](#pass_module)
- [每个参数的选项](#per-argument-options)
- [高级函数模式](#advanced-function-patterns)
- [`#[pyfn]` 简写](#pyfn-shorthand)

还有关于以下主题的附加部分：

- [函数签名](./function/signature.md)

## 函数选项

`#[pyo3]` 属性可用于修改生成的 Python 函数的属性。它可以接受以下选项的任意组合：

  - <a id="name"></a> `#[pyo3(name = "...")]`

    覆盖暴露给 Python 的名称。

    在以下示例中，Rust 函数 `no_args_py` 将作为 Python 函数 `no_args` 添加到 Python 模块 `module_with_functions` 中：

    ```rust
    use pyo3::prelude::*;

    #[pyfunction]
    #[pyo3(name = "no_args")]
    fn no_args_py() -> usize {
        42
    }

    #[pymodule]
    fn module_with_functions(m: &Bound<'_, PyModule>) -> PyResult<()> {
        m.add_function(wrap_pyfunction!(no_args_py, m)?)
    }

    # Python::with_gil(|py| {
    #     let m = pyo3::wrap_pymodule!(module_with_functions)(py);
    #     assert!(m.getattr(py, "no_args").is_ok());
    #     assert!(m.getattr(py, "no_args_py").is_err());
    # });
    ```

  - <a id="signature"></a> `#[pyo3(signature = (...))]`

    在 Python 中定义函数签名。请参见 [函数签名](./function/signature.md)。

  - <a id="text_signature"></a> `#[pyo3(text_signature = "...")]`

    覆盖 PyO3 生成的在 Python 工具中可见的函数签名（例如通过 [`inspect.signature`]）。请参见 [函数签名子章节中的相应主题](./function/signature.md#making-the-function-signature-available-to-python)。

  - <a id="pass_module" ></a> `#[pyo3(pass_module)]`

    设置此选项以使 PyO3 将包含模块作为第一个参数传递给函数。然后可以在函数体中使用该模块。第一个参数 **必须** 是类型 `&Bound<'_, PyModule>`、`Bound<'_, PyModule>` 或 `Py<PyModule>`。

    以下示例创建了一个函数 `pyfunction_with_module`，该函数返回包含模块的名称（即 `module_with_fn`）：

    ```rust
    use pyo3::prelude::*;
    use pyo3::types::PyString;

    #[pyfunction]
    #[pyo3(pass_module)]
    fn pyfunction_with_module<'py>(
        module: &Bound<'py, PyModule>,
    ) -> PyResult<Bound<'py, PyString>> {
        module.name()
    }

    #[pymodule]
    fn module_with_fn(m: &Bound<'_, PyModule>) -> PyResult<()> {
        m.add_function(wrap_pyfunction!(pyfunction_with_module, m)?)
    }
    ```

## 每个参数的选项

`#[pyo3]` 属性可以用于单个参数，以修改它们在生成函数中的属性。它可以接受以下选项的任意组合：

  - <a id="from_py_with"></a> `#[pyo3(from_py_with = "...")]`

    在选项上设置此项以指定一个自定义函数，将函数参数从 Python 转换为所需的 Rust 类型，而不是使用默认的 `FromPyObject` 提取。函数签名必须是 `fn(&Bound<'_, PyAny>) -> PyResult<T>`，其中 `T` 是参数的 Rust 类型。

    以下示例使用 `from_py_with` 将输入的 Python 对象转换为其长度：

    ```rust
    use pyo3::prelude::*;

    fn get_length(obj: &Bound<'_, PyAny>) -> PyResult<usize> {
        obj.len()
    }

    #[pyfunction]
    fn object_length(#[pyo3(from_py_with = "get_length")] argument: usize) -> usize {
        argument
    }

    # Python::with_gil(|py| {
    #     let f = pyo3::wrap_pyfunction!(object_length)(py).unwrap();
    #     assert_eq!(f.call1((vec![1, 2, 3],)).unwrap().extract::<usize>().unwrap(), 3);
    # });
    ```

## 高级函数模式

### 在 Rust 中调用 Python 函数

您可以将 Python 的 `def` 定义的函数和内置函数传递给 Rust 函数 [`PyFunction`] 对应于常规 Python 函数，而 [`PyCFunction`] 描述了诸如 `repr()` 的内置函数。

您还可以使用 [`Bound<'_, PyAny>::is_callable`] 来检查您是否拥有可调用对象。`is_callable` 对于函数（包括 lambda）、方法和具有 `__call__` 方法的对象将返回 `true`。您可以使用 [`Bound<'_, PyAny>::call`] 调用该对象，第一个参数为 args，第二个参数为 kwargs（或 `None`）。还有 [`Bound<'_, PyAny>::call0`] 没有参数和 [`Bound<'_, PyAny>::call1`] 仅有位置参数。

### 在 Python 中调用 Rust 函数

将 Rust 函数转换为 Python 对象的方法因函数而异：

- 命名函数，例如 `fn foo()`：添加 `#[pyfunction]`，然后使用 [`wrap_pyfunction!`] 获取相应的 [`PyCFunction`]。
- 匿名函数（或闭包），例如 `foo: fn()` 可以：
  - 使用 `#[pyclass]` 结构将函数作为字段存储并实现 `__call__` 来调用存储的函数。
  - 使用 `PyCFunction::new_closure` 直接从函数创建对象。

[`Bound<'_, PyAny>::is_callable`]: {{#PYO3_DOCS_URL}}/pyo3/prelude/trait.PyAnyMethods.html#tymethod.is_callable
[`Bound<'_, PyAny>::call`]: {{#PYO3_DOCS_URL}}/pyo3/prelude/trait.PyAnyMethods.html#tymethod.call
[`Bound<'_, PyAny>::call0`]: {{#PYO3_DOCS_URL}}/pyo3/prelude/trait.PyAnyMethods.html#tymethod.call0
[`Bound<'_, PyAny>::call1`]: {{#PYO3_DOCS_URL}}/pyo3/prelude/trait.PyAnyMethods.html#tymethod.call1
[`wrap_pyfunction!`]: {{#PYO3_DOCS_URL}}/pyo3/macro.wrap_pyfunction.html
[`PyFunction`]: {{#PYO3_DOCS_URL}}/pyo3/types/struct.PyFunction.html
[`PyCFunction`]: {{#PYO3_DOCS_URL}}/pyo3/types/struct.PyCFunction.html

### 访问 FFI 函数

为了使 Rust 函数可以从 Python 调用，PyO3 生成一个 `extern "C"` 函数，其确切签名取决于 Rust 签名。（PyO3 选择最佳的 Python 参数传递约定。）然后，它在此 FFI 包装函数中嵌入对 Rust 函数的调用。此包装器处理从输入 `PyObject` 中提取常规参数和关键字参数。

`wrap_pyfunction` 宏可用于直接获取给定 `#[pyfunction]` 和 `Bound<PyModule>` 的 `Bound<PyCFunction>`：`wrap_pyfunction!(rust_fun, module)`。

## `#[pyfn]` 简写

有一种 `#[pyfunction]` 和 `wrap_pymodule!` 的简写：函数可以放置在模块定义内部，并用 `#[pyfn]` 注释。为了简化 PyO3，预计 `#[pyfn]` 可能会在未来的版本中被移除（参见 [#694](https://github.com/PyO3/pyo3/issues/694)）。

以下是 `#[pyfn]` 的示例：

```rust
use pyo3::prelude::*;

#[pymodule]
fn my_extension(m: &Bound<'_, PyModule>) -> PyResult<()> {
    #[pyfn(m)]
    fn double(x: usize) -> usize {
        x * 2
    }

    Ok(())
}
```

`#[pyfn(m)]` 只是 `#[pyfunction]` 的语法糖，并接受本章其余部分中记录的所有相同选项。上面的代码扩展为以下内容：

```rust
use pyo3::prelude::*;

#[pymodule]
fn my_extension(m: &Bound<'_, PyModule>) -> PyResult<()> {
    #[pyfunction]
    fn double(x: usize) -> usize {
        x * 2
    }

    m.add_function(wrap_pyfunction!(double, m)?)
}
```

[`inspect.signature`]: https://docs.python.org/3/library/inspect.html#inspect.signature