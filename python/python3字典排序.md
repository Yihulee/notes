# `python3` 字典排序

说实话,对字典进行排序,这个说法本身就有问题,实际上,你无法对操纵字典说,字典,在你的底层实现里,你就得按照我指定的顺序来排列,如果这样的话,字典就丧失了它的速度优势,它也不是一个字典了.



好了,废话不多说,我这里稍微记录一下我的做法吧.



`python2`里面原来是有`dict.iteritems`这样一个函数的,但是在`python3`里面给消除掉了,`dict.itemitems`实际返回的是一个`list`类型的对象,对象里面全部是`tuple`元素,每个`tuple`都有两个元素,一个是`key`,一个是`value`.



在`python2`时代的做法是这么来排序的:

```python
>> dic
{'a':3 , 'b':2 , 'c': 1}
>> sorted(dic.iteritems(), key=lambda x:x[0], reverse=True) # 按照第0个元素降序排列
[('c', 1), ('b', 2), ('a', 3)]
>> sorted(dic.iteritems(), key=lambda x:x[0], reverse=False) # 按照第0个元素升序排列
[('a', 3), ('b', 2), ('c', 1)]
>> sorted(dic.iteritems(), key=lambda x:x[1], reverse=True) # 按照第1个元素降序排列
[('a', 3), ('b', 2), ('c', 1)]
>> sorted(dic.iteritems(), key=lambda x:x[1], reverse=False) # 按照第1个元素降序排列
[('c', 1), ('b', 2), ('a', 3)]
```

既然`iteritems()`函数取消了,我们构建一个类似的函数即可.

```python
def dict2list(dic:dict):
    ''' 将字典转化为列表 '''
    keys = dic.keys()
    vals = dic.values()
    lst = [(key, val) for key, val in zip(keys, vals)]
    return lst
```

以后在`python3`里面要对字典排个序什么的,也很简单:

```python
>> dic
{'a':3 , 'b':2 , 'c': 1}
>> sorted(dict2list(dic), key=lambda x:x[0], reverse=True) # 按照第0个元素降序排列
[('c', 1), ('b', 2), ('a', 3)]
>> sorted(dict2list(dic), key=lambda x:x[0], reverse=False) # 按照第0个元素升序排列
[('a', 3), ('b', 2), ('c', 1)]
>> sorted(dict2list(dic), key=lambda x:x[1], reverse=True) # 按照第1个元素降序排列
[('a', 3), ('b', 2), ('c', 1)]
>> sorted(dict2list(dic), key=lambda x:x[1], reverse=False) # 按照第1个元素降序排列
[('c', 1), ('b', 2), ('a', 3)]
```

至于效率,你都用`python`了,还扯什么效率呢.

当然,也请不要奢望说,返回一个排序后的字典,前面已经讲了,真的没有意义,你是在追求顺序的字典的话,可以使用有序字典,`python`里面有这样的包.

