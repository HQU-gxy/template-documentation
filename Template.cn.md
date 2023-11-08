> [!NOTE]
> 这篇文档是对 [Template.md](Template.md) 的中文翻译。由 [Claude 2](https://www.anthropic.com/index/claude-2) 完成

# 模板设计

看来我们得重新发明一些轮子。基本上我们在做一个[商业智能](https://zh.wikipedia.org/wiki/%E4%B8%9A%E5%8A%A1%E6%99%BA%E8%83%BD)工具。

模板越可定制化,我们需要实现的[领域特定语言](https://zh.wikipedia.org/wiki/%E9%A0%98%E5%9F%9F%E7%89%B9%E5%AE%9A%E8%AA%9E%E8%A8%80)就越多。

## 变量表

### 类型

#### 原始类型

- `int` Python中的`int`或JavaScript中的`number`类型,其中宽度无关紧要。
- `float` 参见 [IEEE 754](https://zh.wikipedia.org/wiki/IEEE_754)
- `string` [`UTF-8`](https://zh.wikipedia.org/wiki/UTF-8) 编码的字符串。
- `bool`

#### 集合类型

我不打算引入太多集合类型。我们应该保持简单。

##### 列表

对应于原生Python `list` 或 JavaScript `Array`。

##### 系列

对应于 [`polars.Series`](https://pola-rs.github.io/polars/py-polars/html/reference/series/index.html)。

- `Series` 是原始类型的同质集合。`Series[int]` 是 `int` 的集合。

不应支持嵌套的Series。即`Series`应该是一个扁平的集合。你不应该把一个`Series`放在一个`Series`里,所以`Series[Series[int]]` 是不被允许的。

##### `DataFrame`

对应于 [`polars.DataFrame`](https://pola-rs.github.io/polars/py-polars/html/reference/dataframe/index.html)。

- `DataFrame` 是一个 [polars](https://www.pola.rs) 数据框架。
- 更好的做法是给每列一个名字。我不确定语法应该是什么。也许 `DataFrame["date":datetime, "field1":int, "field2":float]` 意思是一个具有三列的表格。第一列名为 `date`,它的类型是 `datetime`。第二列名为 `field1`,它的类型是 `int`。第三列名为 `field2`,它的类型是 `float`。

参见 [`pl.read_database`](https://pola-rs.github.io/polars/user-guide/io/database/)。

我认为支持元组不是一个好主意......异构集合可以只是两个不同的变量。然而,为了`zip`和`unzip`实现`tuple`是必要的。然而,如果我们支持`tuple`,就没有理由不支持`struct`。

#### 派生类型

- `datetime` 是一个遵循 [ISO 8601](https://zh.wikipedia.org/wiki/ISO_8601) 的字符串
- `any`。臭名昭著的 `any` 类型。我们不做运行时类型检查。

### 数据源

#### 常量

最基本的数据源是一个常量。它只是一个常量值。

#### 表达式

*(优先级较低)* 一个python表达式。它是一个字符串。我们可以使用 [AST](https://docs.python.org/3/library/ast.html) 来解析它。

一个简单的 [`eval`](https://python-reference.readthedocs.io/en/latest/docs/functions/eval.html) 就可以做到......

#### 数据库

##### 简单的方式

硬编码 `Series`......甚至更简单,我们从 API 获取所有变量。(如果有这样的 API)

##### 困难的方式

[Doris](https://doris.apache.org)是一个实时数据库。我假设它类似于一个 [InfluxDB](https://www.influxdata.com),这是一个时间序列数据库。

> Apache Doris采用MySQL协议,支持标准SQL,与MySQL语法高度兼容。用户可以通过各种客户端工具访问Doris,并支持与BI工具的无缝连接。

让我们看看其他软件是如何做的。参见 [Grafana 数据源](https://grafana.com/docs/grafana/latest/datasources/)。

也许我们可以借鉴 [查询构建器](https://grafana.com/docs/grafana/latest/datasources/mysql/#query-builder)。我们的目标是从一个表中获取一些**系列**。

我们也可以支持将数据读入 `DataFrame` 数据类型以进一步处理数据。 

有趣的是,Grafana支持[查询变量](https://grafana.com/docs/grafana/latest/datasources/mysql/#query-variable)。我们也应该支持这个。

```jsonc
[
  {
    "name": "age",
    "source":{
      "database": {
        "connection": "...", // 我们必须想办法表示一个数据库连接信息
        "query": "SELECT age FROM people", 
      }
    },
    "type": "Series[int]",
  },
  {
    "name": "hospital_name", 
    "source":{
      "constant": "某医院",
    },
    "type": "string",
  },
  {
    "name": "average_age",
    "source": {
      // 没有lambda或IIFE
      // 如果糟糕的复杂就定义一个新的函数
      "expression": "avg(age)",  
      // 显式捕获变量和函数?
      "variables": ["age"],
      "functions": ["avg"],
    },
    "type": "float",
  }
]
```

### 自定义格式化程序

只需提供一个函数来将值转换为字符串。像 [`toFixed`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Number/toFixed) 或 [toLocaleDateString](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Date/toLocaleDateString)。

实现某些像 *百分比* 的东西是必要的。

```python
def toPercentage(num:float) -> str:
  return "{}%".format(num * 100)
```

## 函数

### 简单的方法

像`sum`、`avg`、`count`、`max`、`min`这样的简单聚合函数。硬编码。(可能与*困难的方式*没有什么不同,但我们不会向用户公开接口)

### 困难的方式

我们需要一种自定义函数的方法。理想情况下[MapReduce](https://zh.wikipedia.org/wiki/MapReduce)应该是我们列操作的基础。

函数表只包含一个名称,但其具体实现应该在单独的文件中。我们可以有一个API来提供所有可用函数以及其类型信息。

出于安全原因,只有函数表中的函数可以在[`expression`](#expression)中使用。

```python
from typing import Callable, TypeVar
import polars as pl

T = TypeVar('T')

def filter_df(df: pl.DataFrame, col_name: str, fn: Callable[[T], bool]) -> pl.DataFrame:
    """
    根据应用于列的函数过滤DataFrame。

    参数:
        df (pl.DataFrame): 要过滤的DataFrame。
        col_name (str): 要应用函数的列的名称。  
        fn (Callable[[T], bool]): 要应用于该列的函数。

    返回:
        pl.DataFrame: 包含仅当函数返回True时的行的新DataFrame。
    """
    # 将函数应用于列
    filtered_col = df[col_name].apply(fn)

    # 根据新列中的值过滤行
    filtered_df = df.filter(filtered_col)

    return filtered_df

def col(df: pl.DataFrame, col_name: str) -> pl.Series:
    """
    从DataFrame中获取一列。

    参数:
        df (pl.DataFrame): 要获取列的DataFrame。
        col_name (str): 要获取的列的名称。
        
    返回:
        pl.Series: 该列。
    """
    return df[col_name]

def sum(col: pl.Series) -> float:
    """
    对一列求和。
    参数:
        col (pl.Series): 要求和的列。
    """
    return col.sum()

# ...
# 我们有很长的函数列表
```

#### 类型内聚性

我们必须实现一系列函数在不同类型之间转换。

#### 代码生成

如果用户不想编写代码,我们可以实现一个代码生成器。实现自定义代码应该使得对我们更容易。

```jsonc
{
  "generator": "filter",
  "name": "gt_2", 
  "rule": "gt", // 可用规则:gt、lt、eq、neq、gte、lte
  "col": "col1",
  "target": 2 
}
```

我们可以生成这样的函数并将其添加到我们的函数表中(可以分开)。

```python
def gt_2(df: pl.DataFrame) -> pl.DataFrame:
    return filter_df(df, "col1", lambda x: x > 2)
```

## 内容

内容的生成应通过JavaScript来处理。

### 文本

一个简单的HTML节点。不允许嵌套元素。通常它只是一个文本节点。

它只是一个简化的 [JSX](https://react.dev/learn/writing-markup-with-jsx) 的JSON表示。

- [react-jsx-parser](https://www.npmjs.com/package/react-jsx-parser)
- [Demystifying JSX: building your own JSX parser from scratch](https://blog.bitsrc.io/demystifying-jsx-building-your-own-jsx-parser-from-scratch-caecf58d7cbd)
- [jsx-parser: a lightweight jsx parser](https://github.com/RubyLouvre/jsx-parser)

```jsonc
{
  "type": "html", // 表示我们将其解析为一个简单的HTML节点
  "tagName": "h1", 
  "attributes": {
    "className": "title",
    "id": "title_one",
    "style": {
      "color": "red",
      "fontSize": "12px"
    }
  },
  "textContent": "${hospital_name}医院报告", // 查看模板字面量。可能需要一个 AST 解析器来解析它。
}
```

参见

- [模板字面量(模板字符串)](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Template_literals)

### 表格

我相信我们需要一个表格的中间表示。简单的HTML语法无法表示基于列的数据。

- [Tabulator](https://tabulator.info)
- [Grid.js](https://gridjs.io)
- [MUI table](https://mui.com/material-ui/react-table/)
- [Element UI: 分组表头](https://element.eleme.io/#/zh-CN/component/table#grouping-table-head)

需要一种将数据连接到UI框架的方式......这是简单的部分。基于GUI的表格编辑器可能很难实现。

*用户应该只定义表头。* 数据应该由[`数据源`](#数据源)提供。

#### 渲染表格

- [Apache Arrow in JS](https://arrow.apache.org/docs/js/)

### 图表

需要一种将数据连接到UI框架的方式。以及一个合适的图表编辑器。

### 图像

简单。

```jsonc
{
  "type": "html",
  "tagName": "img",
  "attributes": {
    "src": "https://example.com/image.png", 
    "alt": "image",
    "width": "100px",
    "height": "100px"
  }
}
```

`src`字段......这是我们的问题。如果我们自己处理图像上传,我们必须实现一个图像服务器。如果我们不处理,我们必须找到一种在其他地方上传图像并获取 URL 的方法。

可能需要一个图像表来存储 ID。

另一个问题是浮动图像。参见 [*深入研究 CSS `float` 属性*](https://blog.logrocket.com/deep-dive-css-float-property/)。 

## 样式

[用于Node.js的CSS解析器/字符串化器](https://github.com/reworkcss/css)。

JSON中的CSS。没什么特别的。
