---
title: Django x Javascript 不同寫法part2. 更新State的方式進行JS渲染
date: 2023-08-27 20:00:00
tags: 
  - 學習
  - django
  - javascript
categories:
  - 學習
---

![](images/2023-08-27DjangoxJavascript不同寫法part2.更新State的方式進行JS渲染/0_Yd6S5EqUbIdDYODn.webp)
*image [source](https://unsplash.com/photos/a-macbook-with-lines-of-code-on-its-screen-on-a-busy-desk-m_HRfLhgABo)*

上次的文章中是我以前常用的寫法，而今天要說的是我前陣子看到這篇文章發現的新大陸

### [[week 21] 前端框架 - 先別急著學 React - HackMD](https://hackmd.io/@Heidi-Liu/note-fe302-review?source=post_page-----4e0621202043--------------------------------)

我覺得挺有趣的就試著把上次那版改成這種方式下去實作！

Django的程式碼跟上週一樣所以今天不會有python的code，就請參考上篇文章！！

那這次我是使用axios跟fetch大同小異，只是需要而外安裝（引入）也有使用到一些JQuery，話不多說先上code吧～

```javascript
<script src="https://code.jquery.com/jquery-3.7.0.js" integrity="sha256-JlqSTELeR4TLqP0OG9dxM7yDPqX1ox/HfgiSLBj8+kM=" crossorigin="anonymous"></script>
<script src="https://cdn.jsdelivr.net/npm/axios/dist/axios.min.js"></script>
//....

let state = {
    todos: []
}

function get_all_list(){
    axios.get("/api/")
        .then(response => {
            response.data.data.forEach(todo => {
                state = {
                    todos: [...state.todos, {
                        id: todo.id,
                        content: todo.title,
                        isDone: todo.complete
                    }]
                }
            })
            updateState(state)
        })
    }

// 更新 state
function updateState(newState) {
    state = newState;
    render()
}

// state => UI
function render() {
    // 先把畫面清空
    $('.todos').empty();
    console.log(state.todos)
    $('.todos').append(
    // 把每個 todo 的 HTML 集合起來放到畫面上
    state.todos.map(todo => Todo(todo)).join('')
    );
}

// Todo component
function Todo({id, content, isDone}) {
    return `
    <div class="ui segment todo" data-id="${id}">
        <p class="ui big header"> ${id} | ${content} </p>

        ${Span({
            className: isDone ? 'ui green label' : 'ui gray label',
            content: isDone ? 'Complete' : 'Not Completed'
        })}

        ${Button({
            className: 'blue btn-update',
            content: 'Update'
        })}

        ${Button({
            className: 'red btn-delete',
            content: 'Delete'
        })}

    </div>
    `
}

// Span component
function Span(props){
    return `<span class="${props.className}">${props.content}</span>`
}

// Button component
function Button(props) {
    return `
    <a class="ui ${props.className} button">${props.content}</a>
    `
}

// 新增 todo
$('.btn-add').click(() => {
    const content = $('.input-todo').val();
    if (!content) return;
    $('.input-todo').val('');
    axios.post("/api/add/", 
        {
            "title": content
        },
        {
            headers: { 
            "X-CSRFToken": "{{csrf_token}}",
            },
        }
    )
    .then(response => {
        todo_id = response.data["todo_id"]
        title = response.data["todo_title"]
        complete = response.data["complete"]
        // 更新 state
        updateState({
            todos: [...state.todos, {
                id: todo_id,
                content: title,
                isDone: complete
            }]
        })
    })
})

// 刪除 todo
$('.todos').on('click', '.btn-delete', e => {
    const id = Number($(e.target).parents('.todo').attr('data-id'));
    axios.get("/api/delete/"+id)
    .then(response => {
        d_id = response.data["todo_id"]
        updateState({
            todos: state.todos = state.todos.filter(todo => todo.id !== d_id)
        })
    })
})

// 未完成 <-> 已完成
$('.todos').on('click', '.btn-update', e => {
    const id = Number($(e.target).parents('.todo').attr('data-id'));
    axios.get("/api/update/"+id)
    .then(response => {
        u_id = response.data["todo_id"]
        complete = response.data["complete"]
        updateState({
            todos: state.todos.map(todo => {
                if (todo.id !== u_id) return todo;
                return {
                ...todo,
                isDone: complete
                }
            })
        })
    })
})
```

跟上次相比是不是很不一樣，我自己覺得這樣子的寫法更加的直觀和易讀易懂！

那我們一樣拆開來看，首先我們要生成Todo的component

```javascript
// Todo component
function Todo({id, content, isDone}) {
    return `
    <div class="ui segment todo" data-id="${id}">
        <p class="ui big header"> ${id} | ${content} </p>

        ${Span({
            className: isDone ? 'ui green label' : 'ui gray label',
            content: isDone ? 'Complete' : 'Not Completed'
        })}

        ${Button({
            className: 'blue btn-update',
            content: 'Update'
        })}

        ${Button({
            className: 'red btn-delete',
            content: 'Delete'
        })}

    </div>
    `
}

// Span component
function Span(props){
    return `<span class="${props.className}">${props.content}</span>`
}

// Button component
function Button(props) {
    return `
    <a class="ui ${props.className} button">${props.content}</a>
    `
}
```

我的Todo component裡面還包括了一個Span component和兩個Button component那他們會依據帶進去的參數而有不同的樣式

接著再到get_all_list()

```javascript
let state = {
    todos: []
}

function get_all_list(){
    axios.get("/api/")
        .then(response => {
            response.data.data.forEach(todo => {
                state = {
                    todos: [...state.todos, {
                        id: todo.id,
                        content: todo.title,
                        isDone: todo.complete
                    }]
                }
            })
            updateState(state)
        })
    }

// 更新 state
function updateState(newState) {
    state = newState;
    render()
}

// state => UI
function render() {
    // 先把畫面清空
    $('.todos').empty();
    console.log(state.todos)
    $('.todos').append(
    // 把每個 todo 的 HTML 集合起來放到畫面上
    state.todos.map(todo => Todo(todo)).join('')
    )
}
```

一開始的狀態先給一個空array，在get_all_list()用axios去打api拿取現在所有的Todo datas，拿到datas後在一個一個把他們塞進去todos array裡面，最後再交由updateState去把現在的state更新掉然後render，render()的工作很簡單會先把現在html上所有的todos元素清空，然後在一筆一筆塞進去～

再來我們來看看新增

```javascript
// 新增 todo
$('.btn-add').click(() => {
    const content = $('.input-todo').val();
    if (!content) return;
    $('.input-todo').val('');
    axios.post("/api/add/", 
        {
            "title": content
        },
        {
            headers: { 
              "X-CSRFToken": "{{csrf_token}}",
            },
        }
    )
    .then(response => {
        todo_id = response.data["todo_id"]
        title = response.data["todo_title"]
        complete = response.data["complete"]
        // 更新 state
        updateState({
            todos: [...state.todos, {
                id: todo_id,
                content: title,
                isDone: complete
            }]
        })
    })
})
```

很簡單的去判斷button有沒有沒click，然後取input的值丟axios，那response會回傳該todo的data，就把他updateState一次就OK了！

接下來的修改和刪除也是同樣的概念，打api後response丟給updateState就完事啦～

```javascript
// 刪除 todo
$('.todos').on('click', '.btn-delete', e => {
    const id = Number($(e.target).parents('.todo').attr('data-id'));
    axios.get("/api/delete/"+id)
    .then(response => {
        d_id = response.data["todo_id"]
        updateState({
            todos: state.todos = state.todos.filter(todo => todo.id !== d_id)
        })
    })
})

// 未完成 <-> 已完成
$('.todos').on('click', '.btn-update', e => {
    const id = Number($(e.target).parents('.todo').attr('data-id'));
    axios.get("/api/update/"+id)
    .then(response => {
        u_id = response.data["todo_id"]
        complete = response.data["complete"]
        updateState({
            todos: state.todos.map(todo => {
                if (todo.id !== u_id) return todo;
                return {
                ...todo,
                isDone: complete
                }
            })
        })
    })
})
```

刪除就是把存在state裡的todo id移除掉，而修改則是把該todo id抓出來改變他的isDone屬性。

至此就大功告成啦，對Javascript不熟悉的我經過這個練習大概可以知道state component的概念！下次可能就是直接用react改寫看看囉！

軟體和程式的世界真的很有趣，可以用不同的做法達到相同的目的，而且瞬息萬變，可能明天又能有新的東西可以學習，想想就興奮呢！！
