# 为你的 Python 包提供类型和 IDE 提示

PyO3 提供了一个易于使用的接口，可以用 Rust 编写本地 Python 库。配套的 Maturin 允许你将它们构建并发布为一个包。然而，为了提供更好的用户体验，Python 库应该为所有公共实体提供类型提示和文档，以便 IDE 在开发过程中能够显示它们，并且类型分析工具如 `mypy` 可以利用这些信息来正确验证代码。

目前解决这个问题的最佳方案是手动维护 `*.pyi` 文件，并将它们与包一起发布。

有一个关于完成 [`experimental-inspect` 特性](./features.md#experimental-inspect) 的路线图草图，这可能最终会导致 PyO3 自动生成类型注释。这需要更多的测试和实现，请参见 [issue #2454](https://github.com/PyO3/pyo3/issues/2454)。

## `pyi` 文件简介

`pyi` 文件（即 `Python Interface` 的缩写）在与它们相关的大多数文档中被称为“存根文件”。在 [旧的 MyPy 文档](https://github.com/python/mypy/wiki/Creating-Stubs-For-Python-Modules) 中可以找到对它的非常好的定义：

> 存根文件仅包含模块公共接口的描述，而不包含任何实现。

在官方 Python 类型文档中还有 [关于类型存根的详细文档](https://typing.readthedocs.io/en/latest/source/stubs.html)。

大多数 Python 开发者在尝试使用 IDE 的“转到定义”功能时，可能已经遇到过它们。例如，几个标准异常的定义如下所示：

```python
class BaseException(object):
    args: Tuple[Any, ...]
    __cause__: BaseException | None
    __context__: BaseException | None
    __suppress_context__: bool
    __traceback__: TracebackType | None
    def __init__(self, *args: object) -> None: ...
    def __str__(self) -> str: ...
    def __repr__(self) -> str: ...
    def with_traceback(self: _TBE, tb: TracebackType | None) -> _TBE: ...

class SystemExit(BaseException):
    code: int

class Exception(BaseException): ...

class StopIteration(Exception):
    value: Any
```

正如我们所看到的，这些并不是包含实现的完整定义，而只是接口的描述。通常这就是库用户所需的全部。

### PEPs 的说法是什么？

在撰写此文档时，`pyi` 文件在三个 PEP 中被提及。

[PEP8 - Python 代码风格指南 - #函数注释](https://www.python.org/dev/peps/pep-0008/#function-annotations)（最后一点）建议所有第三方库创建者提供存根文件，作为类型检查工具了解包的来源。

> (...) 预计第三方库包的用户可能希望对这些包运行类型检查器。为此，[PEP 484](https://www.python.org/dev/peps/pep-0484) 建议使用存根文件：类型检查器优先读取的 .pyi 文件，而不是相应的 .py 文件。(...)

[PEP484 - 类型提示 - #存根文件](https://www.python.org/dev/peps/pep-0484/#stub-files) 对存根文件的定义如下。

> 存根文件是仅供类型检查器使用的文件，不在运行时使用。

它包含了对它们的规范（强烈推荐阅读，因为其中包含至少一项在正常 Python 代码中不使用的内容），以及关于存放存根文件的一些一般信息。

[PEP561 - 分发和打包类型信息](https://www.python.org/dev/peps/pep-0561/) 详细描述了如何构建能够进行类型检查的包。特别是，它包含了关于存根文件必须如何分发以便类型检查器使用的信息。

## 如何做到这一点？

[PEP561](https://www.python.org/dev/peps/pep-0561/) 识别了三种分发类型信息的方法：

* `inline` - 类型信息直接放在源 (`py`) 文件中；
* `separate package with stub files` - 类型信息放在分发在自己独立包中的 `pyi` 文件中；
* `in-package stub files` - 类型信息放在与源文件同一包中的 `pyi` 文件中。

第一种方法在 PyO3 中比较棘手，因为我们没有 `py` 文件。当进行调查并实施必要的更改后，本文件将会更新。

第二种方法比较简单，整个工作可以与主库代码完全分开。带有存根文件的包的示例仓库可以在 [PEP561 参考部分](https://www.python.org/dev/peps/pep-0561/#references) 中找到：[存根包仓库](https://github.com/ethanhs/stub-package)

第三种方法如下所述。

### 在你的 PyO3/Maturin 构建包中包含 `pyi` 文件

当源文件与存根文件在同一包中时，它们应该并排放置。我们需要一种方法来使用 Maturin 实现这一点。此外，为了将我们的包标记为支持类型，我们需要在包中添加一个名为 `py.typed` 的空文件。

#### 如果你没有其他 Python 文件

如果你不需要在包中添加任何其他 Python 文件，除了 `pyi` 文件，Maturin 提供了一种方法来为你完成大部分工作。正如 [Maturin 指南](https://github.com/PyO3/maturin/#mixed-rustpython-projects) 中所记录的，你只需在项目根目录中为你的模块创建一个名为 `<module_name>.pyi` 的存根文件，Maturin 将会处理其余的工作。

```text
my-rust-project/
├── Cargo.toml
├── my_project.pyi  # <<< 在此处为 my_project 模块中的 Rust 函数添加类型存根
├── pyproject.toml
└── src
    └── lib.rs
```

有关示例 `pyi` 文件，请参见 [`my_project.pyi` 内容](#my_projectpyi-content) 部分。

#### 如果你需要其他 Python 文件

如果你需要在包中添加其他 Python 文件，除了 `pyi` 文件，你也可以这样做，但这需要更多的工作。Maturin 提供了一种简单的方法来将文件添加到包中（[文档](https://github.com/PyO3/maturin/blob/0dee40510083c03607834c821eea76964140a126/Readme.md#mixed-rustpython-projects)）。你只需在 `Cargo.toml` 文件旁边创建一个以你的模块命名的文件夹（有关自定义，请参见上面链接的文档）。

文件夹结构如下：

```text
my-project
├── Cargo.toml
├── my_project
│   ├── __init__.py
│   ├── my_project.pyi
│   ├── other_python_file.py
│   └── py.typed
├── pyproject.toml
├── Readme.md
└── src
    └── lib.rs
```

让我们更详细地讨论包文件夹中的文件。

##### `__init__.py` 内容

由于我们现在指定了自己的包内容，因此我们必须提供 `__init__.py` 文件，以便该文件夹被视为一个包，我们可以从中导入内容。如果我们不指定 Python 源文件夹，我们可以始终使用 Maturin 为我们创建的相同内容。对于 PyO3 绑定，它将是：

```python
from .my_project import *
```

这样，我们的本地模块所暴露的所有内容都可以直接从包中导入。

##### `py.typed` 要求

正如 [PEP561](https://www.python.org/dev/peps/pep-0561/) 中所述：
> 希望支持其代码类型检查的包维护者必须在其支持类型的包中添加一个名为 py.typed 的标记文件。此标记文件递归适用：如果顶级包包含它，则所有子包也必须支持类型检查。

如果我们不包含该文件，一些 IDE 可能仍会使用我们的 `pyi` 文件来显示提示，但类型检查器可能不会。在这种情况下，MyPy 将引发错误：

```text
error: Skipping analyzing "my_project": found module but no type hints or library stubs
```

该文件只是一个标记文件，因此它应该是空的。

##### `my_project.pyi` 内容

我们的模块存根文件。本文档并不旨在描述如何编写它们，因为你可以找到很多相关文档，从已经引用的 [PEP484](https://www.python.org/dev/peps/pep-0484/#stub-files) 开始。

示例可能如下所示：

```python
class Car:
    """
    表示汽车的类。

    :param body_type: 车身类型的名称，例如掀背车、轿车
    :param horsepower: 发动机的马力
    """
    def __init__(self, body_type: str, horsepower: int) -> None: ...

    @classmethod
    def from_unique_name(cls, name: str) -> 'Car':
        """
        根据唯一名称创建一辆汽车

        :param name: 要创建的汽车的型号名称
        :return: 带有默认数据的 Car 实例
        """

    def best_color(self) -> str:
        """
        获取汽车的最佳颜色。

        :return: 我们伟大的算法认为这辆车的最佳颜色名称
        """
```