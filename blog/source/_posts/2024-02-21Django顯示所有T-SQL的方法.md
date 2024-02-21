---
title: Django顯示所有T-SQL的方法
date: 2024-02-21 11:30:00
tags: 
  - python
  - django
  - django orm
categories:
  - 學習
---

![](images/2024-02-21Django顯示所有T-SQL的方法/0_bc8v9wzbg2U72OHV.webp)

寫django的人應該或多或少都會使用ORM，而當ORM用太多怕忘記怎麼寫SQL或是想要優化無從下手，就該讓ORM幫我們組成的SQL現形啦！

可以使用以下方式

+ logging
+ django-debug-toolbar
+ connection

## 前置作業
先準備一個model Person

```py
# models.py
from django.db import models

# Create your models here.

class Person(models.Model):
    class Sex(models.IntegerChoices):
        MALE = 0
        FEMALE = 1
        OTHERS = 2
    name = models.CharField(max_length=255)
    sex = models.IntegerField(choices=Sex.choices)

    class Meta:
        db_table = "person"
```

然後註冊進admin page

```py
# admin.py
from django.contrib import admin

from .models import Person
# Register your models here.

admin.site.register(Person)
```

之後下

+ `python manage.py makemigrations`

+ `python manage.py migrate`

+ `python manage.py createsuperuser`

+ `python manage.py runserver`

到 http:127.0.0.1:8000/admin，輸入剛剛createsuperuser輸入的帳號密碼就可以看到下圖
![](images/2024-02-21Django顯示所有T-SQL的方法/1_7m4MwVZGEJXhDpZzokVwHg.webp)

## logging
顯示SQL的時刻到了，只需要在settings.py中新增這些

```py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'simple': {
            'format': '[{levelname}] [{asctime}] {message}',
            'style': '{'
        },
    },
    'handlers': {
        'console':{
            'level':'DEBUG',
            'class':'logging.StreamHandler',
            'formatter': 'simple',
        },
    },
    'loggers': {
        'django.db.backends': {
            'handlers': ['console'],
            'propagate': True,
            'level':'DEBUG',
        },
    }
}
```
+ formatters 是在format顯現的格式
+ handlers 是指定處理器
+ loggers 是選擇要記錄log的範圍

接著到admin page新增user，就能在很亂的console中獲得剛剛新增的SQL
![](images/2024-02-21Django顯示所有T-SQL的方法/1_x0cgOa5kNVSDwB446tNuDw.webp)


## django-debug-toolbar
接著是更方便查閱的套件 debug-toolbar

`pip install django-debug-toolbar`

安裝完後在settings.py中插入這些
```py
INSTALLED_APPS = [
    # ....
    'debug_toolbar',
]

if DEBUG:
    MIDDLEWARE.insert(0,  'debug_toolbar.middleware.DebugToolbarMiddleware')

INTERNAL_IPS = [
    "127.0.0.1",
]
```
並且在urls.py中加入
```py
if DEBUG:
    import debug_toolbar
    urlpatterns += [path('__debug__/', include(debug_toolbar.urls))]
```
回到admin page就能看到toolbar囉
![](images/2024-02-21Django顯示所有T-SQL的方法/1_PfznRk7i7ZV42l9Pq76bOw.webp)

而debug toolbar當今天views是回傳JsonResponse的話不會觸發，畢竟他會需要render出toolbar頁面

## connection
直接在目標view插入connection
```py
# views.py
from django.shortcuts import render
from .models import Person
from django.db import connection
# Create your views here.


def person(request):
    context = {}
    persons = Person.objects.all()
    context['persons'] = list(persons.values())
    print(connection.queries)
    print(persons.query)
    return render(request, 'index.html', context)
```
+ connection.queries 會顯示執行過的SQL語句

+ persons.query 則是顯示當前ORM的SQL（僅限select）