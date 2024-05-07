## Line Charts

折线图

```python
# Set the width and height of the figure
plt.figure(figsize=(12,6))
# Add title
plt.title("Monthly Visitors to Avila Adobe")
# Line chart showing the number of visitors to Avila Adobe over time
sns.lineplot(data=museum_data['Avila Adobe'])
# Add label for horizontal axis
plt.xlabel("Date")
```

## Bar Charts and Heatmaps

条形图

```python
plt.figure(figsize=(8, 6))
sns.barplot(x=ign_data['Racing'],y=ign_data.index)
plt.xlabel("")
```

热图

```python
plt.figure(figsize=(10,10))
sns.heatmap(ign_data, annot=True) #annot=True -确保每个单元格的值显示在图表上
plt.xlabel("Genre")
```

## scatter plots

散点图

```python
sns.scatterplot(x=candy_data['sugarpercent'], y=candy_data['winpercent'])
#x设置行标量，依次
sns.regplot(x=candy_data['sugarpercent'], y=candy_data['winpercent'])
#添加回归线
sns.scatterplot(x=candy_data['pricepercent'],y=candy_data['winpercent'],hue=candy_data['chocolate'])
#这里使用chocolate做颜色分类，这能帮助我们研究三个变量
sns.lmplot(x="pricepercent", y="winpercent", hue="chocolate", data=candy_data)
#两条回归线
sns.swarmplot(x=candy_data['chocolate'],y=candy_data['winpercent'])
#用于创建分类散点图
```

## Histograms

直方图

```python
sns.histplot(iris_data['Petal Length (cm)'])
```

## Density plots

密度图，下面是一种核密度估计图(KDE)

```python
sns.kdeplot(data=iris_data['Petal Length (cm)'], shade=True)
#shade=True用于为下方区域着色
```

### 2D KDE

使用 `sns.jointplot` 命令创建一个二维（2D）KDE图

```python
sns.jointplot(x=iris_data['Petal Length (cm)'], y=iris_data['Sepal Width (cm)'], kind="kde")
```

### Color-coded plots

颜色编码图

```python
sns.histplot(data=iris_data, x='Petal Length (cm)', hue='Species')
#直方图
sns.kdeplot(data=iris_data, x='Petal Length (cm)', hue='Species', shade=True)
```

