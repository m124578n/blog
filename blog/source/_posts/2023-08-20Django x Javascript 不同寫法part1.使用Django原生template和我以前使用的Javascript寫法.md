---
title:  Django x Javascript 不同寫法part1.使用Django原生template
date: 2023-07-23 20:00:00
tags: 
  - 學習
  - django
  - javascript
categories:
  - 學習
---

![](images/2023-08-20Django x Javascript 不同寫法part1.使用Django原生template和我以前使用的Javascript寫法/0_MmikwY_ANgH8Xj99.webp)
*image [source](https://unsplash.com/photos/black-flat-screen-computer-monitor-SyvsTmuuZyM)*


前陣子看到了這個

### [Youtube影片](https://www.youtube.com/watch?v=3vfum74ggHE&list=PLrgX5bzJJljqMbq7msQX4nzTtV3kqZOST&ab_channel=PatrickLoeber)

決定花了一點時間去研究練習Django網站的各種寫法，其中包括原生template、fetch、axios、react，後面其實都在寫Javascript居多，Python改成API後就沒事了XD。

照著影片把最基本的Django template版本先做出來大概會長這樣

```python
# view.py
...

def index(request):
    todos = Todo.objects.all()
    return render(request, 'base.html', {"todo_list": todos})


@require_http_methods(["POST"])
def add(request):
    title = request.POST.get("title", "")
    todo = Todo(title=title)
    todo.save()
    return redirect("index")


def update(request, todo_id):
    todo = Todo.objects.get(id=todo_id)
    todo.complete = not todo.complete
    todo.save()
    return redirect("index")


def delete(request, todo_id):
    todo = Todo.objects.get(id=todo_id)
    todo.delete()
    return redirect("index")
```

首頁＋增刪修，且任何動作都直接導回index做重新查詢

```html
// base.html
...

<body>
    <div style="margin-top: 50px;" class="ui container">
        <h1 class="ui center aligned header">To Do App</h1>

        <form class="ui form" action="/add" method="post">
            {% csrf_token %}
            <div class="field">
                <label>Todo Title</label>
                <input type="text" name="title" placeholder="Enter Todo..."><br>
            </div>
            <button class="ui blue button" type="submit">Add</button>
        </form>

        <hr>

        {% for todo in todo_list %}
        <div class="ui segment">
            <p class="ui big header">{{ todo.id }} | {{ todo.title }}</p>

            {% if todo.complete == False %}
            <span class="ui gray label">Not Complete</span>
            {% else %}
            <span class="ui green label">Completed</span>
            {% endif %}

            <a class="ui blue button" href="/update/{{ todo.id }}">Update</a>
            <a class="ui red button" href="/delete/{{ todo.id }}">Delete</a>
        </div>
        {% endfor %}
    </div>
</body>
```

很簡單且基礎的Jinja2模板，但使用上面的程式碼寫出來的網站不管做什麼事都會重新整理一遍（導回index），現在ajax已經是基本要求了所以開改！

首先先把Django改成API形式吧！這邊就沒用restful framework直接回json

```python
# view.py
...

def index(request):
    return render(request, 'base.html')


def api(request):
    todos = Todo.objects.all()
    return JsonResponse({"data":list(todos.values())})


@require_http_methods(["POST"])
def add(request):
    body = request.body.decode("utf-8")
    body = json.loads(body)
    title = body.get("title", "")
    todo = Todo(title=title)
    todo.save()
    return JsonResponse({"todo_id": todo.id, "complete": todo.complete, "todo_title": todo.title})


def update(request, todo_id):
    todo = Todo.objects.get(id=todo_id)
    todo.complete = not todo.complete
    todo.save()
    return JsonResponse({"todo_id": todo_id, "complete": todo.complete})


def delete(request, todo_id):
    todo = Todo.objects.get(id=todo_id)
    todo.delete()
    return JsonResponse({"todo_id": todo_id})
```

再來就是改base.html

```html
<body onload="get_all_list()">
    <div style="margin-top: 50px;" class="ui container">
        <h1 class="ui center aligned header">To Do App</h1>

        <form class="ui form">
            <div class="field">
                <label>Todo Title</label>
                <input name="title" id="title" placeholder="Enter Todo..." value=""><br>
            </div>
            <button class="ui blue button" type="button" onclick=add()>Add</button>
        </form>

        <hr>
        <div id="all">
        </div>

        <script>
            document.onkeydown = form_sumbit

            function form_sumbit(e){
                the_event = e || window.event
                code = the_event.keyCode || the_event.which || the_event.charCode
                if (code == 13){
                    add()
                    return false
                }
                return true
            }

            function get_all_list(){
                fetch("/api/")
                .then(function (response){
                    return response.json()
                })
                .then(function (myJosn){
                    data = myJosn["data"]
                    data.forEach(todo => {
                        html = ""
                        html += '<div id="all_todo'+todo.id+'"> <div class="ui segment"> '
                        html += '<p class="ui big header">'+todo.id+' | '+todo.title+'</p> '
                        if (todo.complete){
                            html += '<span class="ui green label" id="todo'+todo.id+'">Completed</span>'
                        }
                        else{
                            html += '<span class="ui gray label" id="todo'+todo.id+'">Not Complete</span> '
                        }
                        html += '<a class="ui blue button" onclick=update_("'+todo.id+'")>Update</a> '
                        html += '<a class="ui red button" onclick=delete_("'+todo.id+'")>Delete</a> </div></div>'
                        document.querySelector("#all").innerHTML += html
                    });
                })
            }

            function add(){
                title = document.querySelector("#title").value
                data = {
                    "title": title
                }
                fetch("/api/add/", {
                    headers: { 
                        "X-CSRFToken": "{{csrf_token}}",
                        "user-agent": "Mozilla/4.0 MDN Example",
                        "content-type": "application/json", 
                    },
                    method: "POST", 
                    body: JSON.stringify(data),
                })
                .then(function (response){
                return response.json()
                })
                .then(function (myJson){
                    todo_id = myJson["todo_id"]
                    complete = myJson["complete"]
                    title = myJson["todo_title"]
                    html = ""
                    html += '<div id="all_todo'+todo_id+'"> <div class="ui segment"> '
                    html += '<p class="ui big header">'+todo_id+' | '+title+'</p> '
                    html += '<span class="ui gray label" id="todo'+todo_id+'">Not Complete</span> '
                    html += '<a class="ui blue button" onclick=update_("'+todo_id+'")>Update</a> '
                    html += '<a class="ui red button" onclick=delete_("'+todo_id+'")>Delete</a> </div></div>'
                    document.querySelector("#all").innerHTML += html
                })
            }

            function update_(todo_id){
                fetch("/api/update/"+todo_id)
                .then(function (response){
                return response.json()
                })
                .then(function (myJson) {
                    todo_id = myJson["todo_id"]
                    complete = myJson["complete"]
                    ctodo_id = document.querySelector("#todo"+todo_id)
                    if (complete){
                        ctodo_id.classList.remove("gray")
                        ctodo_id.classList.add("green")
                        ctodo_id.innerHTML = "Completed"
                    }
                    else{
                        ctodo_id.classList.remove("green")
                        ctodo_id.classList.add("gray")
                        ctodo_id.innerHTML = "Not Complete"
                    }
                })
            }

            function delete_(todo_id){
                fetch("/api/delete/"+todo_id)
                .then(function (response){
                return response.json()
                })
                .then(function (myJson) {
                    todo_id = myJson["todo_id"]
                    this_node = document.querySelector("#all_todo"+id)
                    this_node.parentElement.removeChild(this_node)
                })
            }
        </script>
    </div>
</body>
```

WOW變超多的，我們一個一個拆開來看吧

首先先來看看get_all_list做了什麼

```javascript
function get_all_list(){
    fetch("/api/")
    .then(function (response){
        return response.json()
    })
    .then(function (myJosn){
        data = myJosn["data"]
        data.forEach(todo => {
            html = ""
            html += '<div id="all_todo'+todo.id+'"> <div class="ui segment"> '
            html += '<p class="ui big header">'+todo.id+' | '+todo.title+'</p> '
            if (todo.complete){
                html += '<span class="ui green label" id="todo'+todo.id+'">Completed</span>'
            }
            else{
                html += '<span class="ui gray label" id="todo'+todo.id+'">Not Complete</span> '
            }
            html += '<a class="ui blue button" onclick=update_("'+todo.id+'")>Update</a> '
            html += '<a class="ui red button" onclick=delete_("'+todo.id+'")>Delete</a> </div></div>'
            document.querySelector("#all").innerHTML += html
        });
    })
}
```
fetch把過去後回來的response要先過一層json才能使用，而這個就是把原先\{% for %\}迴圈拆成js的forEach去寫把每個todo串出來

然後是新增

```javascript
function add(){
    title = document.querySelector("#title").value
    data = {
        "title": title
    }
    fetch("/api/add/", {
        headers: { 
            "X-CSRFToken": "{{csrf_token}}",
            "user-agent": "Mozilla/4.0 MDN Example",
            "content-type": "application/json", 
        },
        method: "POST", 
        body: JSON.stringify(data),
    })
    .then(function (response){
    return response.json()
    })
    .then(function (myJson){
        todo_id = myJson["todo_id"]
        complete = myJson["complete"]
        title = myJson["todo_title"]
        html = ""
        html += '<div id="all_todo'+todo_id+'"> <div class="ui segment"> '
        html += '<p class="ui big header">'+todo_id+' | '+title+'</p> '
        html += '<span class="ui gray label" id="todo'+todo_id+'">Not Complete</span> '
        html += '<a class="ui blue button" onclick=update_("'+todo_id+'")>Update</a> '
        html += '<a class="ui red button" onclick=delete_("'+todo_id+'")>Delete</a> </div></div>'
        document.querySelector("#all").innerHTML += html
    })
}
```

使用fetch把input data丟過去，那Django會需要加上csrf_token，接回來的值一樣先過一層json就可以對裡面的資料做處理，就把剛剛新增的再往下加一筆

接著delete和update就一起吧

```javascript
function update_(todo_id){
    fetch("/api/update/"+todo_id)
    .then(function (response){
    return response.json()
    })
    .then(function (myJson) {
        todo_id = myJson["todo_id"]
        complete = myJson["complete"]
        ctodo_id = document.querySelector("#todo"+todo_id)
        if (complete){
            ctodo_id.classList.remove("gray")
            ctodo_id.classList.add("green")
            ctodo_id.innerHTML = "Completed"
        }
        else{
            ctodo_id.classList.remove("green")
            ctodo_id.classList.add("gray")
            ctodo_id.innerHTML = "Not Complete"
        }
    })
}

function delete_(todo_id){
    fetch("/api/delete/"+todo_id)
    .then(function (response){
    return response.json()
    })
    .then(function (myJson) {
        todo_id = myJson["todo_id"]
        this_node = document.querySelector("#all_todo"+id)
        this_node.parentElement.removeChild(this_node)
    })
}
```

update一樣丟fetch看你要改哪一個todo id，回傳值再去判斷哪一個id要變成的相對應顏色，而delete就是直接移除那一整個子節點

以上是我原先都在使用的方式，也很常混著寫Jinja2 x Javascript的寫法，接著去稍微看了一下別人怎麼串React後又在網路上爬了一些文章看到一些不同的寫法，React的state和components挺有趣的，下次再來介紹第二版，為axios＋components＋state的概念用純JS做的一版～
