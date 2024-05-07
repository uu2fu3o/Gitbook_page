# Feature Engineering

特征工程的目标是使我们手头的数据更加匹配我们的目标

要使功能有用，它必须与模型能够学习的目标有关系。例如，线性模型只能学习线性关系。因此，当使用线性模型时，目标是变换特征，使其与目标的关系线性化。

## 互信息

**互信息**是信息论中用以评价两个随机变量之间的依赖程度的一个度量。

举个例子x=下雨,y=阴天，在已知y的情况下，x更有可能会发生

### 互信息及其度量

> 关于为什么需要互信息
>
> 帮助我们了解一个feature是否和我们的target相关，更有助于我们选择features吧

互信息描述了不确定性的关系。两个量之间的互信息（MI）是对一个量的知识减少另一个量的不确定性的程度的度量。

> 关于不确定性，使用熵的概念来衡量
>
> 熵是衡量一个系统的稳定程度。其实就是一个系统所有变量信息量的期望或者说均值。当一个系统越接近于不稳定或事件的不确定性越高，对应的熵就越大
>
> 一个变量的熵大致意味着：“平均来说，你需要多少个是或否的问题来描述这个变量的出现。“。互信息是该feature关于回答多少问题的期望

> 应用互信息需要注意几个点：
>
> MI可以帮助你理解一个特征作为目标预测器的相对潜力。
>
> MI无法检测特征之间的交互。它是一个单变量度量。
>
> 特征的实际有用性取决于您使用它的模型。

```python
import matplotlib.pyplot as plt
import pandas as pd
from sklearn.feature_selection import mutual_info_regression
from sklearn.impute import SimpleImputer

plt.style.use("seaborn-v0_8-whitegrid")
home_data = pd.read_csv("melb_data.csv")
X = home_data.drop(["Price"],axis=1)
y = home_data.Price

# 使用 pandas 的 factorize() 方法将分类的条目转化为整数
for colname in X.select_dtypes("object"):
    X[colname], _ = X[colname].factorize()

# 所有的离散变量dtype需要为int
discrete_features = X.dtypes == int
my_imputer = SimpleImputer()
X_2 = pd.DataFrame(my_imputer.fit_transform(X))
X_2.columns = X.columns
def make_mi_scores(X, y, discrete_features):
    mi_scores = mutual_info_regression(X, y, discrete_features=discrete_features) #price是实值，因此用mutual_info_regression
    mi_scores = pd.Series(mi_scores, name="MI Scores", index=X.columns)
    mi_scores = mi_scores.sort_values(ascending=False)
    return mi_scores

mi_scores = make_mi_scores(X_2,y,discrete_features)
print(mi_scores[::3])
```

这里是一段简单计算mi的代码

![image-20240423232444891](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240423232444891.png)

> 为什么要区别连续特征和离散特征
>
> 两者计算mi的方式不同，并且我们需要特殊处理离散变量，因为有分类变量的存在

> mi值低并不意味着我们不再需要该变量，该变量可能为其他特征有交互，这需要我们更深层的判断

## Creating Features 

### 数学变换

数值特征之间的关系通常通过数学公式表示，我们可以对数值进行列的计算，并将其作为新列

```python
autos["stroke_ratio"] = autos.stroke / autos.bore
```

或者你可以使用更复杂的算法

```python
autos["displacement"] = (
    np.pi * ((0.5 * autos.bore) ** 2) * autos.stroke * autos.num_of_cylinders
)
```

### 计数

描述某种事物存在或不存在的特征通常是成组的，比如说，一种疾病的风险因素。您可以通过创建计数来聚合这些要素。

这些特征将是二进制的（ `1` 表示存在， `0` 表示不存在）或布尔值（ `True` 或 `False` ）

我们通常用两个方法sum or Rumrame的内置的比 `gt` 方法

```python
accidents["RoadwayFeatures"] = accidents[roadway_features].sum(axis=1)
concrete["Components"] = concrete[components].gt(0).sum(axis=1)
#gt应该不难理解 ==> greater-than
```

### 建立和分解

同样是对列进行的操作，允许我们对列中的str进行分割或者拼接

```python
customer[["Type", "Level"]] = (  # Create two new features
    customer["Policy"]           # from the Policy feature
    .str                         # through the string accessor
    .split(" ", expand=True)     # by splitting on " "
                                 # and expanding the result into separate columns
)
#分割操作，从1列变为2列
autos["make_and_style"] = autos["make"] + "_" + autos["body_style"]
#拼接
```

### 群变换

>  简单概括一下，我认为是跨类别的信息聚合。相互作用的类别体现在一个列上

使用聚合函数，组转换组合了两个特性：一个提供分组的分类特性和另一个您希望聚合其值的特性。对于"各州平均收入"，您可以选择 `State` 用于分组功能， `mean` 用于聚合功能， `Income` 用于聚合功能。为了在Pandas中计算这个，我们使用 `groupby` 和 `transform` 方法：

```python
customer["AverageIncome"] = (
    customer.groupby("State")  # for each state
    ["Income"]                 # select the income
    .transform("mean")         # and compute its mean
)
#当然还有其他方法，例如max 、 min 、 median 、 var 、 std 和 count
```

或者频率计算

```python
customer["StateFreq"] = (
    customer.groupby("State")
    ["State"]
    .transform("count")
    / customer.State.count()
)
```

> 小技巧
>
> X_2 = pd.get_dummies(df.BldgType, prefix="Bldg")可以直接使用pandas库进行one-hot编码
