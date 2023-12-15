---
title: 【學習筆記】Python Magic Method 01：__new__、__init__、__del__。
date: 2023-05-15 20:00:00
tags: 學習
---

---

本系列會著重在紀錄學習Python的筆記，如果有任何問題或是錯誤的地方，可以直接留言或私訊我，自學錯誤很難看到問題點，還請各位多多指教。


Python Magic Method 直接翻譯就叫"魔法方法 or 魔術方法"，這個神奇的方法也就是雙底線開頭雙底線結尾，我們常見要定義class屬性時使用的__init__也是Magic Method～，然後我會看Python的document然後一個一個去了解Magic Method到底在做什麼以及怎麼使用。

---
### object.__new__(cls[, …])
+ new
+ Called to create a new instance of class cls. __new__() is a static method (special-cased so you need not declare it as such) that takes the class of which an instance was requested as its first argument.

+ 這__new__ 會在__init__ 之前執行， 其主要功能是實例__init__所指定的屬性。 如果__new__沒有return cls 則__new__不會被調用。 
以下是範例：
```python
# new會在init前執行
class MagicNew:
    def __new__(cls, *args, **kwargs):
        print("here is new")
        instance = object.__new__(cls)
        return instance

    def __init__(self, name):
        print("here is init")
        self.name = name

c = MagicNew("John")
# output:
# here is new
# here is init
```

+ 以下是使用情境為設計模式之單例模式： 
參考網址：https://zhuanlan.zhihu.com/p/35943253

```python
# 單例模式，該物件存在就不會在new一個新的出來
class Singleton:
    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = object.__new__(cls)
        return cls._instance

s1 = Singleton()
s2 = Singleton()
print(s1)
print(s2)
# output:
# <__main__.Singleton object at 0x104808d90>
# <__main__.Singleton object at 0x104808d90>
# 注意這邊的記憶體位置 s1 s2 會是同一個物件
```
---

### object.__init__(self[, …])
+ initialization
+ Called after the instance has been created (by __new__()), but before it is returned to the caller. The arguments are those passed to the class constructor expression.
+ this__new__() to create it, and__init__() to customize it
__init__就是大家常見的定義該物件屬性的方式了。

```python
# 在new出物件時可以透過init來指定該物件所擁有的屬性
class MagicInit:
    def __init__(self, name: str, age: int, say_something: str = "I am default"):
        self.name = name
        self.age = age
        self.say_something = say_something

p1 = MagicInit("John", 25)
p2 = MagicInit("John", 25, "change the default")
print(p1.name, p1.age, p1.say_something)
print(p2.name, p2.age, p2.say_something)
# output:
# John 25 I am default
# John 25 change the default
```

+ 不過據我所知好像有滿多奇妙的操作可以在__init__完成，畢竟new出一個物件的時候執行完__new__就會執行__init__，根據不同的使用情境或許能有不同的操作物件。（待學習….）

---
### object.__del__(self)
+ Called when the instance is about to be destroyed.
+ __del__當該物件被消除時會call到這個magic method 
+ 根據官方文件表示：
    + del x doesn't directly call x.__del__() - the former decrements the reference count for x by one, and the latter is only called when x's reference count reaches zero.
    + del <物件>時並不會直接去call __del__()而會先去扣他所關聯的計算，而物件的new出來初始reference count值為1，隨著使用(指定變數為該物件時)會+1，而del <物件>時該數會-1，直到該物件的reference count為0時才會去call __del__()。 （下面使用sys.getrefcount(<物件>)也算使用唷）
參考文章：https://www.796t.com/content/1542840125.html

```python
import time

class MagicDel:
    def __init__(self, name):
        self.name = name

    def __del__(self):
        print("刪除了")

p = MagicDel("John")
del p
print(p)
time.sleep(2)

# output:
# 刪除了
# Traceback (most recent call last):
# NameError: name 'p' is not defined
```
+ 一開始先new出一個物件出來後直接 del <物件>該物件就不會存在程式所以就會觸發__del__()。

```python
# 當今天使用該物件不只一次時
import time
import sys

class MagicDel:
    def __init__(self, name):
        self.name = name

    def __del__(self):
        print("刪除了")

# sys.getrefcount() 可以取得reference count
p = MagicDel("John")
print(sys.getrefcount(p))
p1 = p
print(sys.getrefcount(p))
del p1
print(sys.getrefcount(p))
print("等待兩秒鐘")
time.sleep(2)

# output:
# 2
# 3
# 2
# 等待兩秒鐘後
# 刪除了
```

+ 而當今天讓多個物件去使用這個class則會顯示如上圖，而在整個程式結束後才會觸發__del__()。
目前還不清楚什麼情況下會有機會去使用。（待學習….）