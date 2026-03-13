---
title: Python Intro
date: 2026-03-13 15:41:19
tags:
  - Python
categories:
  - [Python, Fundamentals]
cover: https://pics.findfuns.org/python.jpg
---



# 列表 (List)

### 使用 `join()` 方法將列表中的所有元素連接成一個字串。

    list = ["this", "is", "an", "example", "of", "using", "join()", "method"]

    seperator = " - "
    new_list = seperator.join(list)

    print(new_list)
    # this - is - an - example - of - using - join() - method


### 使用 `split()` 方法將字串拆分為列表。

    print(new_list.split(seperator))
    # ['this', 'is', 'an', 'example', 'of', 'using', 'join()', 'method']


# 集合 (Set)

`intersection()` , `difference()` , `union()`

    set_1 = {1, 2, 3}
    set_2 = {2, 3, 4}

    print(set_1.intersection(set_2))
    print(set_1.difference(set_2))
    print(set_2.difference(set_1))
    print(set_1.union(set_2))

    #{2, 3}
    #{1}
    #{4}
    #{1, 2, 3, 4}


# 字典 (Dictionary)

### 索引存取 VS `get()`

`get()` 在鍵不存在時會回傳 `None`，而不是拋出 `KeyError`。

    dict_1 = {
        "name": "4pril",
        "age": 24,
    }

    print(dict_1.get("phone") is None)
    #True
    print(dict_1["phone"])
    #Traceback (most recent call last):
    #  File "C:\Users\Administrator\PyCharmMiscProject\test.py", line 24, in <module>
    #    print(dict_1["phone"])
    #          ~~~~~~^^^^^^^^^
    #KeyError: 'phone'


### `keys()` , `values()` , `items()`

    print(dict_1.keys())
    #dict_keys(['name', 'age'])
    print(dict_1.values())
    #dict_values(['4pril', 24])
    print(dict_1.items())
    #dict_items([('name', '4pril'), ('age', 24)])

    for k, v in dict_1.items():
        print(k, v, sep=" - ")

    #name - 4pril
    #age - 24


# `format()`

    name = '4pril'

    print("name is {}".format(name))
    # name is 4pril


# `*` 和 `**`

    def func(*args, **kwargs):
        print(args)
        print(kwargs)

    func(1,2,3, name = '4pril', age = 24)
    print('--------------------------------------')

    func([1,2,3], {'name': '4pril', 'age': 24})
    print('--------------------------------------')

    func(*[1,2,3], **{'name': '4pril', 'age': 24})
    print('--------------------------------------')


    #(1, 2, 3)
    #{'name': '4pril', 'age': 24}
    #--------------------------------------
    #([1, 2, 3], {'name': '4pril', 'age': 24})
    #{}
    #--------------------------------------
    #(1, 2, 3)
    #{'name': '4pril', 'age': 24}
    #--------------------------------------
