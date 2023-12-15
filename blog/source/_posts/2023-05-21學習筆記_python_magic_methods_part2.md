---
title: 【學習筆記】Python Magic Method 02：__repr__、__str__、__format__、__bytes__
date: 2023-05-21 20:00:00
tags: 
  - python
  - magic method
categories:
  - 學習
---


本系列會著重在紀錄學習Python的筆記，如果有任何問題或是錯誤的地方，可以直接留言或私訊我，自學錯誤很難看到問題點，還請各位多多指教。

藉由學習Magic Method能深度的了解Python，也讓我在寫code的時候能有不同的想法和選擇最近在水球軟體學院之軟體設計精通之旅上的一個作業『大老二』中就有部分使用Magic Method讓我的code寫起來舒適許多，課程相關的體驗和想法有機會再來寫篇文章。那我們就進入主題吧！

---

### object.\_\_repr__(self)
+ representation
+ Called by the repr() built-in function to compute the “official” string representation of an object.
### object.\_\_str__(self)
+ string
+ Called by str(object) and the built-in functions format() and print() to compute the “informal” or nicely printable string representation of an object.

以上兩個就一起講，他們兩個非常地相似，看官方文件重點捕捉\_\_repr__被標註為正式，而\_\_str__被標註為非正式，至於什麼是正式與非正式呢，有在部分文章中看到\_\_repr__是寫給機器看而\_\_str__是寫給人看這樣的說法。
+ \_\_repr__呼叫方式repr()或直接輸入該物件
+ \_\_str__呼叫方式有print() format() str()
+ 參考文章：https://ithelp.ithome.com.tw/articles/10194593

```python
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __str__(self):
        return f"Name:{self.name}, age:{self.age}"

    def __repr__(self):
        return f"姓名：{self.name}，年齡：{self.age}"


p = Person("John", 25)
print(p)
print(str(p))
print(repr(p))

# output:
# Name:John, age:25
# Name:John, age:25
# 姓名：John，年齡：25

# 而當直接在console中輸入p
# 會跳出：姓名：John，年齡：25
```

+ 如今天只有使用 \_\_str__，則repr()會顯示該物件的記憶體位置。
當今天只有使用 \_\_repr__，則呼叫 \_\_str__ 也會顯示 \_\_repr__ 內容。
以下範例：

```python
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __str__(self):
        return f"Name:{self.name}, age:{self.age}"


p = Person("John", 25)
print(p)
print(str(p))
print(repr(p))

# output:
# Name:John, age:25
# Name:John, age:25
# <__main__.Person object at 0x100e91fd0>
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __repr__(self):
        return f"姓名：{self.name}，年齡：{self.age}"


p = Person("John", 25)
print(p)
print(str(p))
print(repr(p))

# output:
# 姓名：John，年齡：25
# 姓名：John，年齡：25
# 姓名：John，年齡：25
```

---

### object.\_\_format__(self, format_spec)
+ Called by the format() built-in function, and by extension, evaluation of formatted string literals and the str.format() method, to produce a “formatted” string representation of an object.
+ \_\_format__呼叫用format()，可以客製化format後回傳的值

```python
class MagicFormat:
    def __init__(self, name):
        self.name = name

    def __format__(self, format_spec):
        return f"我的名字是：{self.name}"


f = MagicFormat("John")
print(f"My name is :{f.name}")
print(format(f))
print(f"{f}")
print("%s" % f)

# output
# My name is :John  這個單純只是摳該物件的屬性
# 我的名字是：John
# 我的名字是：John
# <__main__.MagicFormat object at 0x100a79fd0>
```

+ 最後一個之所以沒有辦法套用format是因為%s表示string而我上面的class並沒有定義\_\_str__或\_\_repr__所以印出來的會是他的記憶體位置。

+ 下面會是比較特殊的用法，看了看覺得滿有趣的。
+ 參考文章https://www.ctyun.cn/zhishi/p-172413

```python
data_dict = {
    'ymd': '{0.year}:{0.month}:{0.day}',
    'dmy': '{0.day}/{0.month}/{0.year}',
    'mdy': '{0.month}-{0.day}-{0.year}'
}


class MyText:
    def __init__(self, year, month, day):
        self.year = year
        self.month = month
        self.day = day

    def __format__(self, format_spec):
        fmt = data_dict[format_spec]
        return fmt.format(self)


d1 = MyText(2019, 9, 17)
print('{:ymd}'.format(d1))
print('{:dmy}'.format(d1))
print('{:mdy}'.format(d1))

# output
# 2019:9:17
# 17/9/2019
# 9-17-2019
```

+ 而這邊為什麼要寫0呢，我花了一點時間去理解到底為什麼是0，找到了這篇文章，文章裡面有表示format(index0, index1)而我return fmt.format(self)只有一個參數，所以會去讀到第一個值類似[0]。

+ 參考文章：https://bobbyhadz.com/blog/python-indexerror-replacement-index-1-out-of-range

```python
print(f"{} {}".format(index0, index1)) 
# 字串那邊的{}會根據索引對應到後面format()定義的參數
```

+ 而format其實還有很多特殊的功能，之後有機會在寫篇文章吧～

---

### object.\_\_bytes__(self)
+ Called by bytes to compute a byte-string representation of an object. This should return a bytes object.
+ 簡單的一句話，呼叫用bytes()，\_\_bytes__必須回傳bytes物件
+ 實際使用情境目前還沒有想法

```python
class Person:
  def __init__(self, name, age):
        self.name = name
        self.age = age
  
  def __bytes__(self):
    return b'here is bytes'

p = Person("John", 25)
print(bytes(p))

# output
# b'here is bytes'
```
目前是真的不清楚什麼時候會使用到他，跟 \_\_del__ 一樣呢。
