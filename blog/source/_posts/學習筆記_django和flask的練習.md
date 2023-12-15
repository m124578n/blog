---
title: 【學習筆記】Django和Flask的練習
date: 2023-06-14 20:00:00
tags: 
  - django
  - flask
categories:
  - 學習
---

![](images/django_flask_practice/Django-vs-Flask-1.jpg)
*From [django-vs-flask-which-framework-to-choose](https://www.turing.com/blog/django-vs-flask-which-framework-to-choose/)*

最近在練習 [Django](https://www.djangoproject.com/) 和 [Flask](https://flask.palletsprojects.com/en/latest/) 的restful相關套件實作方法，來對比一下兩者使用的相關套件等。

## django
django要實作restful API有超集的套件是 [django rest framework](https://www.django-rest-framework.org/) 跟著官方的tutorial下去實作能大概理解如何快速使用該框架，練習使用到的相關套件：

+ [django-rest-framework](https://www.django-rest-framework.org/)：restful的框架包含serializer序列化器
+ [drf-spectacular](https://drf-spectacular.readthedocs.io/en/latest/)：swagger相關套件
+ [django-rest-framework-simplejwt](https://django-rest-framework-simplejwt.readthedocs.io/en/latest/)：JWT套件

## flask

flask則是要自己一個一個疊樂高、拼拼圖的那種感覺，各式各樣的flask-OOO套件互相搭配組合，我自己練習用的大概有：

+ [flask-restful](https://flask-restful.readthedocs.io/en/latest/)：resuful的框架
+ [flask-sqlalchemy](https://flask-sqlalchemy.palletsprojects.com/en/latest/)：ORM
+ [flask-migrate](https://flask-migrate.readthedocs.io/en/latest/)：資料庫版控
+ [flask-marshmallow](https://flask-marshmallow.readthedocs.io/en/latest/)：序列化器

## 練習django
練習django的過程中會發現view的寫法會有以下幾種：

+ GenericAPIView
+ APIView
+ ViewSets

三者的差異目前還沒有太熟悉，還需要加強研究！
這篇文章寫得挺詳細的
https://zhuanlan.zhihu.com/p/72527077

### django rest framework
而且django rest framework由於都已經幫你把很多地方都封裝起來，要進行客製化會需要花一點時間去看source code，相對來說如果只是一般的CRUD使用ViewSets能節省不少寫code的時間！！

---

## 練習flask

flask倒是看著官方文件和一些實作範例就能很順利開啟一個專案，只不過過程中會有一點問題，在尚未使用flask-migrate時，資料表model.py在create table的時候網路上的範例會教使用python shell去import model的db在執行db.create_all()這邊會報錯現在寫法會要求要使用with app.app.context()才有辦法去執行，這邊還待研究，不過照著官方的文件做就絕對沒問題的！

---

## 做個總結
目前還需要更深入了解的：

1. django 三種view的差異性以及客製化要如何改寫(override function)
2. django rest framework中的serializer的實作原理以及如何客製化
3. flask 中的專案資料夾層級要怎麼規劃及設計
4. flask 尚未完成簡單的CRUD範例
5. flask 中app.context是什麼，目前看到很多翻譯寫『應用上下文』？

在現職中沒有機會碰到這些框架及套件的我，只能下班加緊腳步學習了，增加自己的競爭力，希望下份專案或工作中能活用自己所學的一切！！

---

# [每天進步1%，一年竟能成長37倍！](https://www.storm.mg/lifestyle/3360705?page=1)