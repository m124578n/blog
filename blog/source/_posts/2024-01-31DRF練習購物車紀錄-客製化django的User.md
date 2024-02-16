---
title: DRF練習購物車紀錄-客製化django的User
date: 2024-01-31 20:30:00
tags: 
  - python
  - learning
  - django
categories:
  - 學習
---

![](images/2024-01-31DRF練習購物車紀錄-客製化django的User/0_p7cVVFOd6CpUowfp.webp)

# 練習
使用DRF django rest framework來寫一個簡單的購物車api，前端的部分就斟酌斟酌的寫了～主要著重在DRF這個超集，一直沒有好好的用他來寫一個東東出來，新的一年就來試著寫寫吧！

首先當然是建立環境，虛擬環境很多種大家就挑自己喜歡的建吧～  
然後pip的部分如下

```py
asgiref==3.7.2
Django==5.0.1
django-filter==23.5
djangorestframework==3.14.0
djangorestframework-simplejwt==5.3.1
Markdown==3.5.2
PyJWT==2.8.0
pytz==2023.3.post1
setuptools==68.2.2
sqlparse==0.4.4
tzdata==2023.4
wheel==0.41.2
```
python的版本則是用3.12.0

# 資料夾
目前規劃如下，有account, cart, order, shop, product

![](images/2024-01-31DRF練習購物車紀錄-客製化django的User/1_aXOeWMJ8Ev1WeSSMXqkXzw.webp)

# 登入
首當其衝的就是登入了，目前考慮使用simplejwt和內建的api-auth兩個並行，api-auth是測試的時候比較方便，之後還要考慮轉換成simplejwt後一些request驗證怎麼執行～

```py
# settings.py

INSTALLED_APPS = [
    # ...
    'rest_framework',
    'rest_framework_simplejwt',
    # ...
]

REST_FRAMEWORK = {
    # ...
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    # ...
}
```
```py
# project下的urls.py
from django.contrib import admin
from django.urls import path, include
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenVerifyView,
)


urlpatterns = [
    path('admin/', admin.site.urls),
    path('api-auth/', include('rest_framework.urls')),
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    path('api/token/verify/', TokenVerifyView.as_view(), name='token_verify'),
]
```
之後跑

+ `python manage.py migrate`
+ `python manage.py runserver`

接著去 http://127.0.0.1:8000/api/token

![](images/2024-01-31DRF練習購物車紀錄-客製化django的User/1_ZK2T4KEA1e9pIgp0k91FZQ.webp)


帳號密碼呢，我們先用 `python manage.py createsuperuser`建立一個admin帳號來測試會拿到什麼token～建立好輸入帳號密碼應該會像下面這樣拿到token～

```py
HTTP 200 OK
Allow: POST, OPTIONS
Content-Type: application/json
Vary: Accept

{
    "refresh": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0b2tlbl90eXBlIjoicmVmcmVzaCIsImV4cCI6MTcwOTczNTE0MywiaWF0IjoxNzA1NDE1MTQzLCJqdGkiOiJkMWI0MzRkZjc5YTM0MjE1OWFlOGY3ODViOWZiZWFiNiIsInVzZXJfaWQiOjF9.eCAuJ9LMUUbAYDmtB9h9VBbjCthz-SucJlcTTNztJk4",
    "access": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0b2tlbl90eXBlIjoiYWNjZXNzIiwiZXhwIjoxNzA1NDI1OTQzLCJpYXQiOjE3MDU0MTUxNDMsImp0aSI6ImY3MjViYmU4ZWVjZTQ5ZWJiYjk4MzZiMDk5NmM5MTVkIiwidXNlcl9pZCI6MX0.tAlpCVanj942qMjgBdxcTwqpLmdFCmPsSOJqkgw2XmY"
}
```

這邊只是測試能否運行，實際要使用到JWT應該滿後期了～

# Account
一開始就從Account開始著手吧！首先要起一個account的app

+ `python manage.py startapp account`

DRF會有三個不同負責的區域

+ models
+ serializers
+ views

## models
我們會需要動用到django內建的User，所以我們的model會去繼承

```py
from django.db import models
from django.contrib.auth.models import AbstractUser


class Account(AbstractUser):
    
    class Sex(models.IntegerChoices):
        FEMALE = 0, "Female"
        MALE = 1, "Male"
        OTHER = 2, "Other"
    
    sex = models.PositiveSmallIntegerField(
        verbose_name="sex",
        choices=Sex,
        default=Sex.OTHER
        )
    
    phone = models.CharField(
        verbose_name="phone", 
        max_length=10
        )
    
    email = models.EmailField(
        verbose_name="email", 
        max_length=254
        )
    
    address = models.CharField(
        verbose_name="address", 
        max_length=100
        )
    
    def __str__(self):
        return self.username
```

這邊除了AbstractUser自帶的欄位外我自己新增了幾項～

## serializers
models是主要跟資料庫連結的class，而serializers則是把models跟view接在一起的通道

```py
class AccountSerializer(serializers.HyperlinkedModelSerializer):

    sex = serializers.SerializerMethodField()

    class Meta:
        model = Account
        fields = ["username", "sex", "email", "phone", "address", "url", "is_superuser", "is_staff"]

    def get_sex(self, obj):
        return obj.get_sex_display()
```

sex 性別額外拉出來是因為我想要它顯示的時候是顯示 male, female, other這樣，而不是0, 1, 2，至於這邊是怎麼對接上的有時間再來好好研究研究～

## views
views當然就是要準備回傳值的地方囉～

```py
from rest_framework import viewsets
from account.models import Account
from account.serializers import AccountSerializer


class AccountsView(viewsets.ModelViewSet):
    queryset = Account.objects.all().order_by("id")
    serializer_class = AccountSerializer
```

## urls
最後把我們寫好的view註冊到urls裡面
```py
from account import views
from rest_framework import routers


router = routers.DefaultRouter()
router.register(r'', views.AccountsView)

urlpatterns = [
     # ...
]

urlpatterns += router.urls
```

最後在

+ `python manage.py makemigrations`
+ `python manage.py migrate`
+ `python manage.py runserver`

就可以看到DRF預設可以操作api的頁面囉～

![](images/2024-01-31DRF練習購物車紀錄-客製化django的User/1_SogjtcatoJEyNwDA6VANyA.webp)
