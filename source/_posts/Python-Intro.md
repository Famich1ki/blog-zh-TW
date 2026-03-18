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

# Comprehensions

### List Comprehension

```python
nums = [1, 2, 3, 4]

list_1 = [n * n for n in nums if n % 2 == 0]
list_2 = list(map(lambda n: n * n, filter(lambda n: n % 2 == 0, nums)))
list_3 = [(a, b) for a in 'ab' for b in range(2)]
print(list_1)
print(list_2)
print(list_3)

#[4, 16]
#[4, 16]
#[('a', 0), ('a', 1), ('b', 0), ('b', 1)]
```

### Set Comprehension

```python
nums = [1, 2, 3, 4, 5, 1, 3]
set_1 = {n for n in nums}
set_2 = set(map(lambda n: n, nums))
print(set_1)
print(set_2)

#{1, 2, 3, 4, 5}
#{1, 2, 3, 4, 5}
```

### Dictionary Comprehension

```python
name = ['Tom', 'Bob', 'Andy']
age = [24, 18, 30]

dict_1 = {k: v for k, v in zip(name, age)}
print(list(zip(name, age)))
print(dict_1)

#[('Tom', 24), ('Bob', 18), ('Andy', 30)]
#{'Tom': 24, 'Bob': 18, 'Andy': 30}
```

### Generator Comprehension

```python
list = [1, 2, 3, 4, 5]

def square(list):
    for num in list:
        yield num * num

for i in square(list):
    print(i)
    
#1
#4
#9
#16
#25
```

There is a `yield` in square function, which means it is a **generator function**. when it is called, it returns a `generator` object .

During the **for** loop where calling the **square** function, Python actually executes logic similar to this:

```python
g = square(list)

while True:
    i = next(g)
    print(i)
```

`next()` is a **build-in function** that retrieves the next value produced by a generator.

The first time `next(g)` is called, the generator runs until it reaches `yield`, returning `1`.
The function then pauses at that position.

The next time `next(g)` is called, execution resumes from where it paused, so the variable `num` continues from `2` instead of starting again from `1`.

# `sorted()`

Preparation for data structure

```python
class Student:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __repr__(self):
        return f'{self.name} {self.age}'


students = [Student('Cindy', 23), Student('Bob', 21), Student('John', 24)]
```

There are several ways to use `sorted()` function.

- Using a normal function as the key.

- Using lambda expression to specify the key.

- Using the `attrgetter()` built-in function to retrieve the attribute used for sorting.

```python
def get_name(student)
    return student.name

from operator import attrgetter

sorted_students_1 = sorted(students, key=get_name)
sorted_students_2 = sorted(students, key=lambda student: student.age, reverse=True)
sorted_students_3 = sorted(students, key=attrgetter('age'), reverse=True)

print(sorted_students_1)
print(sorted_students_2)
print(sorted_students_3)

#[Bob 21, Cindy 23, John 24]
#[John 24, Cindy 23, Bob 21]
#[John 24, Cindy 23, Bob 21]
```

# Generator

generator is an essential feature of Python. Anyone who wants to become familiar with Python should understand how they work.

Why is generator so important ? Let's look at the example below.

```python
'''Suppose we want to get the squeare of every number in a list, 
one simple way is to iterate through the list and append every square value to a new list'''

list = [1, 2, 3, 4, 5]
list_square = []

for num in list:
    list_square.append(num * num)
    
print(list_square)
# [1, 4, 9, 16, 25]
```

In this case, all values are computed at once. If there are 10,000,000 number in the list and having such large amount of values in memory would be low-efficient and redundant.

To improve efficiency, we can use a generator.

```python
def square(list):
    for num in list:
        yield num * num
    
square_numbers = square(list)
print(square_numbers)

print(next(square_numbers))
print(next(square_numbers))
print(next(square_numbers))
print(next(square_numbers))
print(next(square_numbers)) 
# ps: if there is one more next(), a StopIteration exception will be raised 
# because of running out of number.

#<generator object square at 0x00000242DB6DA400>
#1
#4
#9
#16
#25
```

When using a generator, values are produced one at a time only when they are requested, instead of computing all values in advance.

The generator can also be iterated using a for loop:

```python
for num in square_numbers:
    print(num)
```

Additionally, we can also create a  generator by using parentheses.

```python
nums = (n * n for n in [1, 2, 3, 4, 5])
print(nums)
print(list(nums)) # convert generator into a list

#<generator object <genexpr> at 0x000001F765779700>
#[1, 4, 9, 16, 25]
```

Now let's look at the difference in memory usage and execution time between a list and a generator.

```python
import sys
import time

N = 10_000_000

start = time.time()

numbers = [i * i for i in range(N)]

end = time.time()

print('List:')
print('duration:', end - start)
print('memory usage:', sys.getsizeof(numbers))

start = time.time()

numbers = (i * i for i in range(N))

end = time.time()

print('Generator:')
print('duration:', end - start)
print('memory usage:', sys.getsizeof(numbers))

#List:
#duration: 0.501561164855957
#memory usage: 89095160
#Generator:
#duration: 0.04305219650268555
#memory usage: 208
```



