# 在Python中使用带有特征约束的Rust函数

PyO3允许将某些函数和类从Rust轻松转换为Python（请参见[转换表](conversions/tables.md)）。然而，将需要特定特征实现作为参数的Rust代码转换并不总是简单明了。

本教程解释了如何将一个接受特征作为参数的Rust函数转换为Python使用，该函数的类实现了与特征相同的方法。

这有什么用？

### 优点
- 使你的Rust代码可供Python用户使用
- 在Rust中编写复杂算法，借助借用检查器

### 缺点
- 不如原生Rust快（需要进行类型转换，并且代码的一部分在Python中运行）
- 需要调整代码以进行暴露

## 示例

让我们以一个优化求解器的基本实现作为例子，该求解器在给定模型上运行。

假设我们有一个函数`solve`，它在一个模型上运行并改变其状态。该函数的参数可以是任何实现了`Model`特征的模型：

```rust
# #![allow(dead_code)]
pub trait Model {
    fn set_variables(&mut self, inputs: &Vec<f64>);
    fn compute(&mut self);
    fn get_results(&self) -> Vec<f64>;
}

pub fn solve<T: Model>(model: &mut T) {
    println!("魔法求解器将模型变为已解决状态");
}
```
假设我们有以下约束：
- 我们不能更改该代码，因为它在许多Rust模型上运行。
- 我们还有许多Python模型无法解决，因为该求解器在该语言中不可用。
在Python中重写它将是繁琐且容易出错的，因为一切都已经在Rust中可用。

我们如何通过PyO3将这个求解器暴露给Python呢？

## Python类的特征约束实现

如果一个Python类实现了与`Model`特征相同的三个方法，那么将其适配以使用求解器似乎是合乎逻辑的。然而，无法将`PyObject`传递给它，因为它并未实现Rust特征（即使Python模型具有所需的方法）。

为了实现特征，我们必须在Rust中为Python模型的调用编写一个包装器。方法签名必须与特征相同，记住Rust特征不能为了使代码在Python中可用而更改。

我们想要暴露的Python模型如下，它已经包含了所有所需的方法：

```python
class Model:
    def set_variables(self, inputs):
        self.inputs = inputs
    def compute(self):
        self.results = [elt**2 - 3 for elt in self.inputs]
    def get_results(self):
        return self.results
```

以下包装器将从Rust调用Python模型，使用一个结构体将模型作为`PyAny`对象持有：

```rust
# #![allow(dead_code)]
use pyo3::prelude::*;
use pyo3::types::PyList;

# pub trait Model {
#   fn set_variables(&mut self, inputs: &Vec<f64>);
#   fn compute(&mut self);
#   fn get_results(&self) -> Vec<f64>;
# }

struct UserModel {
    model: Py<PyAny>,
}

impl Model for UserModel {
    fn set_variables(&mut self, var: &Vec<f64>) {
        println!("Rust调用Python以设置变量");
        Python::with_gil(|py| {
            self.model
                .bind(py)
                .call_method("set_variables", (PyList::new(py, var),), None)
                .unwrap();
        })
    }

    fn get_results(&self) -> Vec<f64> {
        println!("Rust调用Python以获取结果");
        Python::with_gil(|py| {
            self.model
                .bind(py)
                .call_method("get_results", (), None)
                .unwrap()
                .extract()
                .unwrap()
        })
    }

    fn compute(&mut self) {
        println!("Rust调用Python以执行计算");
        Python::with_gil(|py| {
            self.model
                .bind(py)
                .call_method("compute", (), None)
                .unwrap();
        })
    }
}
```

现在这部分已经实现，让我们将模型包装器暴露给Python。我们添加PyO3注解并添加构造函数：

```rust
# #![allow(dead_code)]
# pub trait Model {
#   fn set_variables(&mut self, inputs: &Vec<f64>);
#   fn compute(&mut self);
#   fn get_results(&self) -> Vec<f64>;
# }
# use pyo3::prelude::*;

#[pyclass]
struct UserModel {
    model: Py<PyAny>,
}

#[pymodule]
fn trait_exposure(m: &Bound<'_, PyModule>) -> PyResult<()> {
    m.add_class::<UserModel>()?;
    Ok(())
}

#[pymethods]
impl UserModel {
    #[new]
    pub fn new(model: Py<PyAny>) -> Self {
        UserModel { model }
    }
}
```

现在我们将PyO3注解添加到特征实现中：

```rust,ignore
#[pymethods]
impl Model for UserModel {
    // 之前的特征实现
}
```

然而，之前的代码将无法编译。编译错误如下：
`error: #[pymethods] cannot be used on trait impl blocks`

真是糟糕！然而，我们可以在这些函数周围编写第二个包装器，以直接调用它们。这个包装器还将执行Python和Rust之间的类型转换。

```rust
# #![allow(dead_code)]
# use pyo3::prelude::*;
# use pyo3::types::PyList;
#
# pub trait Model {
#   fn set_variables(&mut self, inputs: &Vec<f64>);
#   fn compute(&mut self);
#   fn get_results(&self) -> Vec<f64>;
# }
#
# #[pyclass]
# struct UserModel {
#     model: Py<PyAny>,
# }
#
# impl Model for UserModel {
#  fn set_variables(&mut self, var: &Vec<f64>) {
#      println!("Rust调用Python以设置变量");
#      Python::with_gil(|py| {
#          self.model.bind(py)
#              .call_method("set_variables", (PyList::new(py, var),), None)
#              .unwrap();
#      })
#  }
#
#  fn get_results(&self) -> Vec<f64> {
#      println!("Rust调用Python以获取结果");
#      Python::with_gil(|py| {
#          self.model
#              .bind(py)
#              .call_method("get_results", (), None)
#              .unwrap()
#              .extract()
#              .unwrap()
#      })
#  }
#
#  fn compute(&mut self) {
#      println!("Rust调用Python以执行计算");
#      Python::with_gil(|py| {
#          self.model
#              .bind(py)
#              .call_method("compute", (), None)
#              .unwrap();
#      })
#
#  }
# }

#[pymethods]
impl UserModel {
    pub fn set_variables(&mut self, var: Vec<f64>) {
        println!("从Python调用Rust设置变量");
        Model::set_variables(self, &var)
    }

    pub fn get_results(&mut self) -> Vec<f64> {
        println!("从Python调用Rust获取结果");
        Model::get_results(self)
    }

    pub fn compute(&mut self) {
        println!("从Python调用Rust计算");
        Model::compute(self)
    }
}
```
这个包装器处理PyO3要求和特征之间的类型转换。为了满足PyO3的要求，这个包装器必须：
- 返回类型为`PyResult`的对象
- 在方法签名中仅使用值，而不是引用

让我们运行Python文件：

```python
class Model:
    def set_variables(self, inputs):
        self.inputs = inputs
    def compute(self):
        self.results = [elt**2 - 3 for elt in self.inputs]
    def get_results(self):
        return self.results

if __name__=="__main__":
  import trait_exposure

  myModel = Model()
  my_rust_model = trait_exposure.UserModel(myModel)
  my_rust_model.set_variables([2.0])
  print("从Python打印值: ", myModel.inputs)
  my_rust_model.compute()
  print("通过Rust从Python打印值: ", my_rust_model.get_results())
  print("直接从Python打印值: ", myModel.get_results())
```

这将输出：

```block
从Python调用Rust设置变量
从Rust调用Python设置变量
从Python打印值:  [2.0]
从Python调用Rust计算
从Rust调用Python执行计算
从Python调用Rust获取结果
从Rust调用Python获取结果
通过Rust从Python打印值:  [1.0]
直接从Python打印值:  [1.0]
```

我们现在成功地将实现了`Model`特征的Rust模型暴露给Python！

现在我们将暴露`solve`函数，但在此之前，让我们谈谈类型错误。

## Python中的类型错误

如果在使用Python时发生类型错误，会发生什么，以及如何改善错误消息？

### Python函数参数中的错误类型

假设在第一种情况下，你将在Python文件中使用`my_rust_model.set_variables(2.0)`而不是`my_rust_model.set_variables([2.0])`。

Rust签名期望一个向量，这对应于Python中的列表。如果我们传递一个单一值而不是向量，会发生什么？

在Python执行时，我们得到：

```block
File "main.py", line 15, in <module>
   my_rust_model.set_variables(2)
TypeError
```

这是一个类型错误，Python指向了它，因此很容易识别和解决。

### Python方法签名中的错误类型

现在假设我们模型类的一个方法的返回类型错误，例如`get_results`方法，期望返回一个`Vec<f64>`在Rust中，对应于Python中的列表。

```python
class Model:
    def set_variables(self, inputs):
        self.inputs = inputs
    def compute(self):
        self.results = [elt**2 -3 for elt in self.inputs]
    def get_results(self):
        return self.results[0]
        #return self.results <-- 这是预期的输出
```

这个调用导致以下恐慌：

```block
pyo3_runtime.PanicException: called `Result::unwrap()` on an `Err` value: PyErr { type: Py(0x10dcf79f0, PhantomData) }
```

这个错误代码对一个不知道Rust的Python用户并没有帮助，或者对一个不知道使用了PyO3来接口Rust代码的人来说。

然而，由于我们负责将Rust代码提供给Python，我们可以对此做一些事情。

问题在于我们在任何可以的地方都调用了`unwrap`，因此PyO3的任何恐慌将直接转发给最终用户。

让我们修改执行类型转换的代码，以便为Python用户提供有用的错误消息：

我们在`get_results`方法中使用了以下调用来执行类型转换：

```rust
# #![allow(dead_code)]
# use pyo3::prelude::*;
# use pyo3::types::PyList;
#
# pub trait Model {
#   fn set_variables(&mut self, inputs: &Vec<f64>);
#   fn compute(&mut self);
#   fn get_results(&self) -> Vec<f64>;
# }
#
# #[pyclass]
# struct UserModel {
#     model: Py<PyAny>,
# }

impl Model for UserModel {
    fn get_results(&self) -> Vec<f64> {
        println!("Rust调用Python以获取结果");
        Python::with_gil(|py| {
            self.model
                .bind(py)
                .call_method("get_results", (), None)
                .unwrap()
                .extract()
                .unwrap()
        })
    }
#     fn set_variables(&mut self, var: &Vec<f64>) {
#         println!("Rust调用Python以设置变量");
#         Python::with_gil(|py| {
#             self.model.bind(py)
#                 .call_method("set_variables", (PyList::new(py, var),), None)
#                 .unwrap();
#         })
#     }
#
#     fn compute(&mut self) {
#         println!("Rust调用Python以执行计算");
#         Python::with_gil(|py| {
#             self.model
#                 .bind(py)
#                 .call_method("compute", (), None)
#                 .unwrap();
#         })
#     }
}
```

让我们分解一下，以便进行更好的错误处理：

```rust
# #![allow(dead_code)]
# use pyo3::prelude::*;
# use pyo3::types::PyList;
#
# pub trait Model {
#   fn set_variables(&mut self, inputs: &Vec<f64>);
#   fn compute(&mut self);
#   fn get_results(&self) -> Vec<f64>;
# }
#
# #[pyclass]
# struct UserModel {
#     model: Py<PyAny>,
# }

impl Model for UserModel {
    fn get_results(&self) -> Vec<f64> {
        println!("从Rust调用Python获取结果");
        Python::with_gil(|py| {
            let py_result: Bound<'_, PyAny> = self
                .model
                .bind(py)
                .call_method("get_results", (), None)
                .unwrap();

            if py_result.get_type().name().unwrap() != "list" {
                panic!(
                    "预期get_results()方法签名为列表，实际为{}",
                    py_result.get_type().name().unwrap()
                );
            }
            py_result.extract()
        })
        .unwrap()
    }
#     fn set_variables(&mut self, var: &Vec<f64>) {
#         println!("Rust调用Python以设置变量");
#         Python::with_gil(|py| {
#             let py_model = self.model.bind(py)
#                 .call_method("set_variables", (PyList::new(py, var),), None)
#                 .unwrap();
#         })
#     }
#
#     fn compute(&mut self) {
#         println!("Rust调用Python以执行计算");
#         Python::with_gil(|py| {
#             self.model
#                 .bind(py)
#                 .call_method("compute", (), None)
#                 .unwrap();
#         })
#     }
}
```

通过这样做，你捕获Python计算的结果并检查其类型，以便在执行解包之前能够提供更好的错误消息。

当然，这并不能涵盖所有可能的错误输出：用户可能返回一个字符串列表而不是浮点数列表。在这种情况下，由于PyO3，运行时恐慌仍然会发生，但错误消息对于非Rust用户来说将更难以解读。

将Rust代码暴露给Python的开发者决定投入多少精力来处理Python类型错误和改进错误消息。

## 最终代码

现在让我们暴露`solve()`函数，使其可以从Python访问。

无法直接将`solve`函数暴露给Python，因为无法执行类型转换。它需要一个实现了`Model`特征的对象作为输入。

然而，`UserModel`已经实现了该特征。因此，我们可以编写一个函数包装器，该包装器将`UserModel`（已经暴露给Python）作为参数，以调用核心函数`solve`。

还需要将结构体设为公共。

```rust
# #![allow(dead_code)]
use pyo3::prelude::*;
use pyo3::types::PyList;

pub trait Model {
    fn set_variables(&mut self, var: &Vec<f64>);
    fn get_results(&self) -> Vec<f64>;
    fn compute(&mut self);
}

pub fn solve<T: Model>(model: &mut T) {
    println!("魔法求解器将模型变为已解决状态");
}

#[pyfunction]
#[pyo3(name = "solve")]
pub fn solve_wrapper(model: &mut UserModel) {
    solve(model);
}

#[pyclass]
pub struct UserModel {
    model: Py<PyAny>,
}

#[pymodule]
fn trait_exposure(m: &Bound<'_, PyModule>) -> PyResult<()> {
    m.add_class::<UserModel>()?;
    m.add_function(wrap_pyfunction!(solve_wrapper, m)?)?;
    Ok(())
}

#[pymethods]
impl UserModel {
    #[new]
    pub fn new(model: Py<PyAny>) -> Self {
        UserModel { model }
    }

    pub fn set_variables(&mut self, var: Vec<f64>) {
        println!("从Python调用Rust设置变量");
        Model::set_variables(self, &var)
    }

    pub fn get_results(&mut self) -> Vec<f64> {
        println!("从Python调用Rust获取结果");
        Model::get_results(self)
    }

    pub fn compute(&mut self) {
        Model::compute(self)
    }
}

impl Model for UserModel {
    fn set_variables(&mut self, var: &Vec<f64>) {
        println!("Rust调用Python以设置变量");
        Python::with_gil(|py| {
            self.model
                .bind(py)
                .call_method("set_variables", (PyList::new(py, var),), None)
                .unwrap();
        })
    }

    fn get_results(&self) -> Vec<f64> {
        println!("从Rust调用Python获取结果");
        Python::with_gil(|py| {
            let py_result: Bound<'_, PyAny> = self
                .model
                .bind(py)
                .call_method("get_results", (), None)
                .unwrap();

            if py_result.get_type().name().unwrap() != "list" {
                panic!(
                    "预期get_results()方法签名为列表，实际为{}",
                    py_result.get_type().name().unwrap()
                );
            }
            py_result.extract()
        })
        .unwrap()
    }

    fn compute(&mut self) {
        println!("Rust调用Python以执行计算");
        Python::with_gil(|py| {
            self.model
                .bind(py)
                .call_method("compute", (), None)
                .unwrap();
        })
    }
}
```