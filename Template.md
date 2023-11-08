# Template Design

Guess we have to reinvent some wheel. Basically we are doing a [BI](https://en.wikipedia.org/wiki/Business_intelligence) tool.

The more customizable the template is, the more [DSL](https://en.wikipedia.org/wiki/Domain-specific_language) we need to implement.

## Variable Table

### Types

#### Primitive Types

- `int` Python `int` or JavaScript `number` type, where width doesn't matter.
- `float` See [IEEE 754](https://en.wikipedia.org/wiki/IEEE_754)
- `string` [`UTF-8`](https://en.wikipedia.org/wiki/UTF-8) encoded string.
- `bool`

#### Collection Types

I don't plan to introduce too many collection types. We should keep it simple.

##### List

Correspond to native Python `list` or JavaScript `Array`.

##### Series

Correspond to [`polars.Series`](https://pola-rs.github.io/polars/py-polars/html/reference/series/index.html).

- `Series` is a homogeneous collection of primitive types. `Series[int]` is a collection of `int`. 

Nested Series should NOT be supported. i.e. `Series` should be a flat collection. You should NOT put a `Series` inside a `Series`, so `Series[Series[int]]` is NOT allowed.

##### `DataFrame`

Correspond to [`polars.DataFrame`](https://pola-rs.github.io/polars/py-polars/html/reference/dataframe/index.html).

- `DataFrame` is a [polars](https://www.pola.rs) dataframe.  
- Better yet we could give every column a name. I'm not sure what the syntax should be. Maybe `DataFrame["date":datetime, "field1":int, "field2":float]` which means a table with three columns. The first column is named `date` and its type is `datetime`. The second column is named `field1` and its type is `int`. The third column is named `field2` and its type is `float`.

See [`pl.read_database`](https://pola-rs.github.io/polars/user-guide/io/database/).

I don't think support tuple is a good idea... heterogeneous collection could
just be two different variables. However, implementing `tuple` is necessary for
`zip` and `unzip`. However, there's no reason not to support `struct` if we
support `tuple`.

#### Derived Types

- `datetime` is a string following [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601)
- `any`. infamous `any` type. We don't do runtime type checking.

### Data Source

#### Constant

The most basic data source is a constant. It's just a constant value.

#### Expression

*(Low priority)* A python expression. It's a string. We could use [AST](https://docs.python.org/3/library/ast.html) to parse it. 

A simple [`eval`](https://python-reference.readthedocs.io/en/latest/docs/functions/eval.html) could do it...

#### Database

##### The easy way

Hard code the `Series`... Even easier, we get all of the variables from API. (If there's such API)

##### The hard way

[Doris](https://doris.apache.org) is a real-time database. I assume it's like a [InfluxDB](https://www.influxdata.com), which is a time series database.

> Apache Doris adopts MySQL protocol, supports standard SQL, and is highly compatible with MySQL dialect. Users can access Doris through various client tools and it supports seamless connection with BI tools.

Let's see how other software do it. See [Grafana data sources](https://grafana.com/docs/grafana/latest/datasources/).

Maybe we could borrow the [Query builder](https://grafana.com/docs/grafana/latest/datasources/mysql/#query-builder). Our target is to get some **Series** from a table. 

We could also support reading data into `DataFrame` data type to further process the data.

Interestingly Grafana supports [query variable](https://grafana.com/docs/grafana/latest/datasources/mysql/#query-variable). We should support this too.

```jsonc
[
  {
    "name": "age",
    "source":{
      "database": {
        "connection": "...", // we have to find a way to represent a database connection info
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
      // No lambda or IIFE
      // if shit goes to complex just define a new function
      "expression": "avg(age)",
      // explicit capture of variables and functions?
      "variables": ["age"],
      "functions": ["avg"],
    },
    "type": "float",
  }
]
```

### Custom Formatter

Just provide a function to convert the value to string. Like to [`toFixed`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/toFixed) or [toLocaleDateString](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/toLocaleDateString).

It's necessary to implement something like *percentage*.

```python
def toPercentage(num:float) -> str:
  return "{}%".format(num * 100)
```

## Functions

### The easy way

Simple aggregation functions like `sum`, `avg`, `count`, `max`, `min`. Hard encoded. (probably nothing different from *the hard way* but we don't expose the interface to user)

### The hard way

We need a way to customize the functions. Ideally [MapReduce](https://en.wikipedia.org/wiki/MapReduce)
should be our foundation for column operations.

Function table only contains a name but its concrete implementation should be in separated file. We could have a API to provide all of the available functions
and its type info.

For safety reason only functions in the function table could be used in [`expression`](#expression).  

```python
from typing import Callable, TypeVar
import polars as pl

T = TypeVar('T')

def filter_df(df: pl.DataFrame, col_name: str, fn: Callable[[T], bool]) -> pl.DataFrame:
    """
    Filter a DataFrame based on a function applied to a column.

    Args:
        df (pl.DataFrame): The DataFrame to filter.
        col_name (str): The name of the column to apply the function to.
        fn (Callable[[T], bool]): The function to apply to the column.

    Returns:
        pl.DataFrame: A new DataFrame containing only the rows where the function returns True.
    """
    # apply the function to the column
    filtered_col = df[col_name].apply(fn)

    # filter the rows based on the values in the new column
    filtered_df = df.filter(filtered_col)

    return filtered_df

def col(df: pl.DataFrame, col_name: str) -> pl.Series:
    """
    Get a column from a DataFrame.

    Args:
        df (pl.DataFrame): The DataFrame to get the column from.
        col_name (str): The name of the column to get.

    Returns:
        pl.Series: The column.
    """
    return df[col_name]

def sum(col: pl.Series) -> float:
    """
    Sum a column.
    Args: 
        col (pl.Series): The column to sum.
    """
    return col.sum()

# ... 
# we have a long list of functions
```

#### Type Cohesion

We have to implement a series of functions to convert between different types.

#### Code Generation

If user don't want to code, we could implement a code generator. Enabling custom
code should make it easier for us.

```jsonc
{
  "generator": "filter",
  "name": "gt_2",
  "rule": "gt", // available rules: gt, lt, eq, neq, gte, lte
  "col": "col1",
  "target": 2
}
```

we could generate such function and add it to our function table (could be separated).

```python
def gt_2(df: pl.DataFrame) -> pl.DataFrame:
    return filter_df(df, "col1", lambda x: x > 2)
```

## Content

Generation of content should be handled with JavaScript.

### Text

A simple HTML node. Nested elements are NOT allowed. Usually it's just a text node.

It's just a simplified JSON representation of [JSX](https://react.dev/learn/writing-markup-with-jsx).

- [react-jsx-parser](https://www.npmjs.com/package/react-jsx-parser)
- [Demystifying JSX: building your own JSX parser from scratch](https://blog.bitsrc.io/demystifying-jsx-building-your-own-jsx-parser-from-scratch-caecf58d7cbd)
- [jsx-parser: a lightweight jsx parser](https://github.com/RubyLouvre/jsx-parser)

```jsonc
{
  "type": "html", // indicates we parse it as a simple HTML node
  "tagName": "h1",
  "attributes": {
    "className": "title",
    "id": "title_one",
    "style": {
      "color": "red",
      "fontSize": "12px"
    }
  },
  "textContent": "${hospital_name}医院报告", // See Template literals. Might need a AST parser to parse it.
}
```

See 

- [Template literals (Template strings)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals)

### Table

I believe we need a intermediate representation for a table. Simple HTML syntax just can't represent column based data.

- [Tabulator](https://tabulator.info)
- [Grid.js](https://gridjs.io)
- [MUI table](https://mui.com/material-ui/react-table/)
- [Element UI: Grouping Table Head](https://element.eleme.io/#/en-US/component/table#grouping-table-head)

Need a way to connect data to UI framework... That's the simple part. A GUI
based table editor could be hard to implement.

*User should ONLY define the table header.* The data should be provided by the [`data source`](#data-source).

#### Table rendering

- [Apache Arrow in JS](https://arrow.apache.org/docs/js/)

### Chart

Need a way to connect data to UI framework. And a suitable chart editor.

### Image

Simple.

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

`src` field... That's our problem. If we handle image uploading by ourselves, we
have to implement a image server. If we don't, we have to find a way to upload images somewhere else
and get the URL.

Might need a image table to store IDs.

Another problem is floating images. See [*A deep dive into the CSS `float` property*](https://blog.logrocket.com/deep-dive-css-float-property/).

## Style

[CSS parser/stringifier for Node.js](https://github.com/reworkcss/css).

CSS in JSON. Nothing special.
