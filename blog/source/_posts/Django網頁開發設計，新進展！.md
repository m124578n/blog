---
title: Django網頁開發設計，新進展！
date: 2023-04-23 20:00:00
tags: 
  - django
  - python
categories:
  - 其他
---

![](images/Django開發新進展/0__po9b2MR2fXklzUT.gif)
*image [memes.tw](https://memes.tw/gif-post?maker=16)*

延續前幾篇的想法下，又多冒出了一些新的點子！
而最新的點子好像才有辦法解決掉我一開始的煩惱。

---

一開始的目標是我要我的view是乾淨的、舒服的，於是我把一些相關的判斷或對資料庫的處理額外拉了類別出來，就前幾篇來說會是Member類別有login這個方法，然而這樣我的view裡面依舊有一些基本的判斷，例如：判斷request的method、拿取前端拋回來的資料⋯⋯依舊會讓我的view乘載過多的資訊，導致頁面一多看起來就亂亂的，於是參考了 [水球軟體學院](https://discord.com/invite/waterballsa) 社群夥伴的想法，我終於打造出一版我目前滿意的寫法，大致順序如下：

+ 把原本view裡面的資訊包成一個Service類別
+ 在Service類別中處理原來會跟Member類別的行為
+ 大功告成

```python
#views.py
def login(request):
  return LoginService(request)
```
這樣變得非常的乾淨呢！！再來只要把LoginService做好～

```python
class LoginService:
  def __init__(self, request):
    self.request = request

  def login(self):
    if self.request.method == "GET":
      ctx = {}
      return render(self.request, "會員登入.html", ctx)
    if self.request.method == "POST":
      data = {}
      會員帳號 = self.request.POST.get("會員帳號", "")
      會員密碼 = self.request.POST.get("會員密碼", "")
      member = 會員()
      if member.login(會員帳號, 會員密碼):
        data['message'] = "密碼正確！！"
      else:
        data['message'] = "噗噗！！帳號密碼有誤～"
      return JsonResponse(data)
```
完美！！這樣就可以很完美的把我的view變乾淨。

---

但今天歹擠謀假甘單（事情沒有那麼簡單），我的網站不只有member這個角色，還有store這個角色呢～照這樣看來我不就又要開一個view又要開一個LoginService呢？？（當然view要開因為不同頁面），於是我突發奇想蹦出了這個作法：

直接把store類別 / 物件直接當作參數丟進去！！

```python
def login(request):
  return LoginService(request, Store()).login()
```
```python
def login(request):
  return LoginService(request, Member()).login()
```
```python
class LoginService:
  def __init__(self, request, user):
    self.request = request
    self.user = user
  
  def login(self):
    if self.request.method == "GET":
      ctx = {}
      return render(self.request, "會員登入.html", ctx)
    if self.request.method == "POST":
      data = {}
      會員帳號 = self.request.POST.get("會員帳號", "")
      會員密碼 = self.request.POST.get("會員密碼", "")
      #這裡就直接修改掉！！
      if self.user.login(會員帳號, 會員密碼):
        data['message'] = "密碼正確！！"
      else:
        data['message'] = "噗噗！！帳號密碼有誤～"
      return JsonResponse(data)
```
可喜可賀！！這樣我的LoginService就可以同時給member和store使用了，假設未來又多了新的角色我也可以直接使用LoginService（除非他今天登入不用帳號密碼了），於是乎我就這樣開心的繼續寫著程式～～

---

心得：很感謝社群的夥伴願意分享想法，不然我自己一個人要想出這些東西可能還要花上好一段時間，我也相信我這個一定不是最佳解答，但至少以現在來說我可以繼續開心的寫程式，而不會有一個疙瘩黏在那邊，如果你有更好的想法也歡迎一起交流交流～～
