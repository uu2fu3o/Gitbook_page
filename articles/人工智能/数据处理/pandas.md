按照路线学习数据处理的第一章，选择pandas

> 关于为什么选择pandas
>
> pandas学习起来比较简单，适合入门
>
> 当然，你也可以选择其他几个库(Numpy,Matplotlib,Scipy,Scikit-learn)进行学习，不过作为入门来说，pandas完全足够了

## 创建数据

我们关注pandas提供的两个核心对象**DataFrame**和**Series**

### DataFrame

直译过来叫作数据框，简单来说是一个表，其中包含了许多条目

举个例子，比如我想创建每个人对不同事物的评价，可以看到如下简单的代码

```python
import pandas as pd

print(pd.DataFrame({'uu2': ['good', 'notbad'],
                    'uu3': ['bad', 'good']},
                   index=['foodA','foodB']))
```

![image-20240410202918809](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240410202918809.png)

如图，这里降键设为我们的人名，值为评价，如果我们不额外设置index,这里应该是简单的0,1......类型，通过设置index(索引)，我们可以为其赋值

### Series

相较于DataFrame来说，Series是数据值的序列。比如

```python
print(pd.Series(['good','bad']))
```

![image-20240410203858638](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240410203858638.png)

对比上方的图，我们可以简单的理解为Series是Dataframe的其中一列的数据，当然，Series同样可以设置索引

```python
print(pd.Series(['good','bad'],
                index=['foodA','foodB'],
                name='Test'))
```

![image-20240410204225071](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240410204225071.png)

可能你会关注这里的name是指什么，显然series和dataframe还有个区别就在于series并没有自己的列名，只有一个整体的name

## 读取数据文件

虽然pandas创建数据非常方便，但是通常我们使用的是外部的数据，包括但不限于csv表格，excel等格式文件。

创建一个csv文件，csv编辑器推荐ron'd data，可以自行搜索

我们试着读取一个csv文件，看看其输出是什么样子的

```python
test_review = pd.read_csv('test.csv')
print(test_review)
```

![image-20240411001148705](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240411001148705.png)

或者使用shape属性来查看文件的大小

![image-20240411001253275](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240411001253275.png)

很直观可以看出csv的大小是20行，10列，不过在实际的训练当中，这个数字会非常大

通过.head()，我们可以获取指定的行数

```python
test_review = pd.read_csv('test.csv')
print(test_review.head(5))
```

read_csv函数有很多参数可以使用，比如我想指定自己的索引为csv表格的第一列

```python
test_review = pd.read_csv('test.csv',index_col=0)
print(test_review.head())
```

![image-20240411002056367](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240411002056367.png)

> 保存DataFrame数据文件可以使用.to_csv()属性

## 索引、选择和分配

```python
test_review = pd.read_csv('test.csv',index_col=0)
print(test_review['Column 02'])
or
print(test_review.index)
```

我们想要查看某一个属性有如上两种写法，需要注意的是第二种写法不能包含空格

![image-20240411194117901](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240411194117901.png)

### 索引

pandas 有自己的访问器运算符 `loc` 和 `iloc`

这两个属性都是行在前，列在后

例如，获取Dataframe的第一行数据

```python
test_review = pd.read_csv('test.csv')
print(test_review.iloc[0])
```

从1-3行获取index值，可以使用:设定值的范围

```python
test_review = pd.read_csv('test.csv')
print(test_review.iloc[:3,0])
```

![image-20240411201648631](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240411201648631.png)

或者获取指定位置的数据

```python
print(test_review.iloc[1:3,0])
or
print(test_review.iloc[[0,1],0])
```

> 负数也可用于选择，是从末尾开始计数

loc与iloc不同的是，iloc基于位置的索引，loc基于标签的索引

例如我想获取index列的第一个条目数据

```python
print(test_review.loc[0,'index'])
```

### 条件选择

和python中的条件选择没太大区别

```python
test_review = pd.read_csv('test.csv',index_col=0)
print(test_review.contary == 11) //返回bool值，哪些条目为true
print(test_review.loc[test_review.contary== 11]) //打印true的条目
```

当然使用&连接符也是允许的

```python
print(test_review.loc[(test_review.contary== 11)&(test_review.score > 11)])
```

![image-20240411203701970](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240411203701970.png)

顺理成章的，管道符也是允许的 |

顺便来介绍一下pandas内置选择器isin

```python
print(test_review.loc[test_review.contary.isin(['test', 'isinis'])])
```

isnull(notnull)可以让你找出其中数据(不)为空的条目

### 分配数据

```python
test_review['contary'] ='test'
print(test_review)
```

使其为一个常量

![image-20240411204656672](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240411204656672.png)

或者可迭代的值

```python
test_review['contary'] =range(len(test_review), 0, -1)
```

## 摘要函数

```
.describe()
.mean() //某个属性的平均值
.unique() //某个列表的唯一值
.value_counts() //统计出现的频率
```

## Maps

数学上的映射

```python
test_review = pd.read_csv('test.csv')
review = test_review.score.mean()
print(test_review.score.map(lambda p:p - review))
```

通过简单的map映射，我们可以计算与平均值的差值

比较复杂的是apply(),这个方法允许我们对每一行或列进行更改，从而更改整个Dataframe

```python
test_review = pd.read_csv('test.csv')
review_score_mean = test_review.score.mean()
# print(test_review.score.map(lambda p:p - review))

def remean_score(row):
    row.score = row.score - review_score_mean
    return row

print(test_review.apply(remean_score, axis='columns'))
```

![image-20240412141944526](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240412141944526.png)

## Grouping

通过groupby()这个函数来进行操作

例如下面这条语句,达到了value_counts()相同的作用

```
reviews.groupby('points').points.count()
```

当然groupby()有更多细度的用法，比如多列分组，或使用agg()方法运行多个函数(这里的例子是随便的，只演示用法)

![image-20240412150728129](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240412150728129.png)

> Multi-indexes
>
> 在使用groupby()时会导致多索引的情况
>
> 一般使用reset_index()方法将其转换为常规的索引方法

## Sorting

sort_values(by=‘’，ascending=False)方法允许我们按照某个属性的数据进行排序，默认为升序

sort_index()则按照索引值进行排序

## DataTypes

DataFrame 或 Series 中列的数据类型称为 dtype

```python
print(test_review.score.dtype)
```

> 字符串类型没有自己的类型，他们是object类型

可以通过astepye()方法进行强制类型转换

## Missing data

缺失数据，即为NaN,这个值由于技术原因，dtype始终为float64

我们可以通过fillna()方法将确实值替换为我们自己定义的值

或是replace方法替换某一个值

```
reviews.country.replace("???","xxx")
```

## Renaming

rename()函数，更改索引或者列的名称

```
reviews.rename(columns={'score': 'points'})
//
reviews.rename(columns=dict(region_1='region', region_2='locale'))
```

![image-20240412161138215](https://raw.githubusercontent.com/uu2fu3o/blog-picture/master/cloud/image-20240412161138215.png)

或者修改pandas默认的索引

```python
reviews.rename(index={0: 'firstEntry', 1: 'secondEntry'})
```

行索引和列索引都可以有自己name属性

```
reviews.rename_axis("wines", axis='rows').rename_axis("fields", axis='columns')
```

## Combining

主要关注三个函数concat(),join(),merge()

最简单的组合方法是 `concat()` 。给定一个元素列表，此函数会将这些元素沿轴混合在一起。

不同的 DataFrame 或 Series 对象中拥有数据但具有相同的字段（列）时考虑concat()

```
pd.concat([A, B])
```

join()可以组合具有相同索引但并不相同的DataFrame对象

```
left.join(right, lsuffix='_CAN', rsuffix='_UK')
```

`lsuffix` 和 `rsuffix` 参数是必需的，这是为了避免相同的列名，如果列名本身就不相同，那就不需要了

