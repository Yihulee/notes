# `numpy`点滴

这篇博文会陆陆续续更新的,因为`numpy`这个东西,还是熟一点的话,会更加得心应手.



## 约定

后面的东西中,都已经假设我们做了如下的导入:

```python
import numpy as np
```

## 1. 构建一个对称矩阵

其实并没有很好的办法,但是我们可以通过几个命令的组合来构建一个对角矩阵,举个栗子,假设我们有这样一个矩阵:

```python
In [88]: a = np.arange(9).reshape((3, 3))

In [89]: a
Out[89]:
array([[0, 1, 2],
       [3, 4, 5],
       [6, 7, 8]])
```

然后我们只保留上三角矩阵:

```python
In [91]: x = np.triu(a)

In [92]: x
Out[92]:
array([[0, 1, 2],
       [0, 4, 5],
       [0, 0, 8]])
```

当然,`np`中有一个类似的函数是保留下三角矩阵用的,叫做`tril`.

然后让`x`和`x`的转置加一下,立马构成了一个对称矩阵.

```python
In [93]: x + x.T
Out[93]:
array([[ 0,  1,  2],
       [ 1,  8,  5],
       [ 2,  5, 16]])
```

当然,我们看得到.对角线上的元素加了两遍,如果你乐意的话,减去一遍对角线的元素即可.

```python
In [96]: x.diagonal() # 获得对角线的元素的值
Out[96]: array([0, 4, 8])

In [97]: np.diag(x.diagonal()) # 根据对角线构建一个矩阵
Out[97]:
array([[0, 0, 0],
       [0, 4, 0],
       [0, 0, 8]])

In [98]: x + x.T - np.diag(x.diagonal())
Out[98]:
array([[0, 1, 2],
       [1, 4, 5],
       [2, 5, 8]])
```



## 2. 找出矩阵中最大值或者最小值的下标

有的时候我们还真有这样的一个需求,我这里记录一下我的做法:

```python
In [99]: x
Out[99]:
array([[0, 1, 2],
       [0, 4, 5],
       [0, 0, 8]])

In [100]: x == x.max()	# 获得一个掩模矩阵
Out[100]:
array([[False, False, False],
       [False, False, False],
       [False, False,  True]], dtype=bool)

# 接下来要使用到where
In [101]: np.where(x == x.max())
Out[101]: (array([2], dtype=int64), array([2], dtype=int64))
```

我们可以看得到,最后返回了一个二维的矩阵,而坐标`(2, 2)`恰好就是我们要求的值.

再给出一个手册上的例子:

```python
>>> goodvalues = [3, 4, 7]
>>> ix = np.in1d(x.ravel(), goodvalues).reshape(x.shape)
>>> ix
array([[False, False, False],
       [ True,  True, False],
       [False,  True, False]], dtype=bool)
>>> np.where(ix)
(array([1, 1, 2]), array([0, 1, 1]))
# (1, 0), (1, 1), (2, 1)
```

## 3. 填充对角线的元素

这个函数在进行矩阵运算的时候应该经常会用得到,它就是`fill_diagonal`函数.

```python
In [102]: x
Out[102]:
array([[0, 1, 2],
       [0, 4, 5],
       [0, 0, 8]])

In [103]: np.fill_diagonal(x, 5)

In [104]: x
Out[104]:
array([[5, 1, 2],
       [0, 5, 5],
       [0, 0, 5]])
```

### 获得对角线的元素

```python
In [105]: x
Out[105]:
array([[5, 1, 2],
       [0, 5, 5],
       [0, 0, 5]])

In [106]: x.diagonal()
Out[106]: array([5, 5, 5])
```

## 3. 获得矩阵中的最大值或者最小值

这个应该是非常常用的,我们还是来看例子吧:

```python
In [2]: x = np.arange(9).reshape(3, 3)

In [3]: x
Out[3]:
array([[0, 1, 2],
       [3, 4, 5],
       [6, 7, 8]])

In [4]: x.max() # 获得最大值
Out[4]: 8

In [5]: x.min() # 获得最小值
Out[5]: 0
```

如果某一天,你想获得某一类或者某一行中的最大值或者最小值,那又应该怎么办呢?

```python
In [6]: x.min(axis=1) # 获得第1维上的数据的最小值
Out[6]: array([0, 3, 6])

In [7]: x.min(axis=0) # 获得第0维上的数据的最小值
Out[7]: array([0, 1, 2])
```

在这里可能要稍微普及一下,什么叫做维(`axis`).

`axis=0`一般指代的是列,`axis=1`一般指代的是行.

更近一步的,不知道什么时候,你想知道每一行的最大值,最小值,或者每一列的最大值,最小值的下标,怎么把?

```python
In [8]: x.argmax() # 将x平展之后,最大值的下标
Out[8]: 8

In [9]: x.argmax(axis=0) # 每一列最大值的下标
Out[9]: array([2, 2, 2], dtype=int64) # (0, 2), (1, 2), (2, 2)

In [10]: x.argmax(axis=1) # 每一行最大值的下标
Out[10]: array([2, 2, 2], dtype=int64) # (0, 2), (1, 2), (2, 2)
```

