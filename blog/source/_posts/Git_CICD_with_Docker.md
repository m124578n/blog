---
title: Git CICD with Docker
date: 2023-03-18 20:00:00
tags: 
  - github actions
  - docker
  - 水球軟體學院
categories:
  - 學習
---

![](images/Git_CICD_with_Docker/1_bZP17SmwRZihfAYDr5KBFg.webp)
*From [docker-logo](https://1000logos.net/docker-logo/)*

# 前言：

這個主題是在水球軟體學院中舉辦的Docker共學會最終回的成果發表文章，本人我也是第一次寫文章，主要目的應該會著重在自己的一個紀錄，內容如果不是那麼正確還請多多包容和指點以下正篇。

---

# 正篇：

Docker是一個容器化服務，他可以在一台電腦中切出好幾個環境分別執行不同的任務。

這邊會進行一個簡單的web server然後執行Github actions 或 Gitlab CICD達成自動化測試以及自動化部署。

稍後將會介紹到的項目有以下幾樣：

+ Dockerfile

+ docker-compose.yml

+ github/workflows 中的yml檔

---

## 首先
最基本的要架設一個網站會需要的服務為web server和database。

那們我將會以Python Django串接Nginx做為web server那database是使用Django預設的sqlite簡單演示。

### 資料夾階層會長這樣

```bash
+根目錄
|
+-+nginx/
|      |
|      +--Dockerfile
|      +--docker-nginx-web.conf
|      +--nginx.conf
+-+web/
|     |
|     +--Dockerfile
|     +--requirements.txt
|     +-+app/以下略
|
+--docker-compose.yml
```

### 這是Django也就是web裡面的Dockerfile

```bash
FROM python:3.8.5
LABEL maintainer="xxxx@gmail.com"

WORKDIR /web
COPY . /web/

RUN pip install --upgrade pip 
RUN pip install -r requirements.txt

WORKDIR /web/app

VOLUME /web

EXPOSE 8001
```

### 再來是nginx裡面的Dockerfile

```bash
FROM nginx:latest
LABEL maintainer="xxxx@gmail.com"


COPY nginx.conf /etc/nginx/nginx.conf
COPY docker-nginx-web.conf /etc/nginx/sites-available/

RUN mkdir -p /etc/nginx/sites-enabled/ && \
    ln -s /etc/nginx/sites-available/docker-nginx-web.conf /etc/nginx/sites-enabled/

CMD ["nginx", "-g", "daemon off;"]
```

### 然後是根目錄下的docker-compose.yml

```bash
version: '3.8'

services:
        app_web:
                build: ./web
                container_name: app_web
                restart: always
                command: ["/bin/bash","-c","uwsgi --ini uwsgi.ini"]
                volumes:
                        - web_data:/web/app
                ports:
                        - "8001:8001"
                environment:
                        - PYTHONUNBUFFERED=TURE
        app_nginx:
                build: ./nginx
                container_name: app_nginx
                restart: always
                volumes:
                        - web_data:/web/app
                ports:
                        - "80:80"
                depends_on:
                        - app_web
volumes:
        web_data:
```

那我們這邊docker-compose.yml裡面的build會去找尋web和nginx目錄下的Dockerfile並根據Dockerfile的內容去啟動container。

這邊下指令

```bash
docker-compose up --build -d
```

其實就會直接把兩個container建立起來了。

---

## Github actions

然後就可以開始寫Github actions的yml檔囉！

### 我們到Github的頁面點選Actions

![](images/Git_CICD_with_Docker/1_NEBsJwswssn2VffDuNyVSg.webp)

### 點選下方的Configure就會先幫你建立一個預設的yml

![](images/Git_CICD_with_Docker/1_eT-ez4_d77qUFV54NycBig.webp)


這邊就可以開始編輯自己的yml檔，但是Github上也有很多已經編輯好的yml檔會出現在Configure下方可以選用。

![](images/Git_CICD_with_Docker/1_Zk9WR4svGoxlIuGd59R8gw.webp)


那這邊我就先用預設的yml來編輯

那預設的yml呢它上面會有很詳細的註解說明每一個指令的功用，這邊只截取我需要的部分把它改寫成這樣

```bash
name: Django CI

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      max-parallel: 4
      matrix:
        python-version: [3.8]
    steps:
    - uses: actions/checkout@v3
    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v3
      with:
        python-version: ${{ matrix.python-version }}
    - name: Install Dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r web/requirements.txt
    - name: Run Tests
      run: |
        python  web/app/manage.py test
```

這邊是當你對這個repositories有push或是pull_request的時候就會觸發job的程序，上面那邊有${{matrix.python-version}}是我們可以對他執行多個版本的測試，這邊我只有跑一個python 3.8，那我們看到最後一行，這個就只是在跑Django的測試內容。

### 接下來是測試通過後要把這整包部署到你指定的位置上也就是CD的部分

```bash
name: Django CD
# 只有在 CI 的 workflow 完成時才會執行此 workflow
# https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#workflow_run
on:
  workflow_run:
    workflows: [ Django CI ]
    types:
      - completed

jobs:
  deploy:
    runs-on: ubuntu-latest

    # 注意前面 workflow_run 的 completed 意思是「完成」，不論執行結果成功或是失敗都算是「完成」
    # 但是一般來說測試如果失敗就應該暫停部屬至正式環境
    # 因此這裡加上一個 if 判斷，只有 CI 成功才會執行此 workflow
    # https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#workflow_run
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    
    steps:
      # 使用 appleboy/ssh-action@master 這個 action 遠端連線至正式環境
      # https://github.com/appleboy/ssh-action
      - name: Deployment
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.HOST }}
          key: ${{ secrets.TOKEN }}
          username: ec2-user
          # 執行部屬的指令
          #docker rmi -f $(docker images -q  -f dangling=true)
          #docker volume rm $(docker volume ls -q -f dangling=true)
          script: | 
            whoami
        
            sudo yum -y install docker
            sudo yum -y install git 
            sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
            sudo chmod 777 /usr/local/bin/docker-compose
            docker-compose version
            cd ~
            sudo rm -fr *
            sudo git clone https://github.com/xxx/testCICD.git
            cd testCICD
            sudo systemctl restart docker
            sudo chmod 777 /var/run/docker.sock
            docker-compose down
            docker rmi -f $(docker images -q  -f dangling=true)
            docker volume rm $(docker volume ls -q -f dangling=true)
            docker-compose up --build -d 
```

這邊我是參考了appleboy的ssh-action，這樣就可以透過ssh的方式連接到你的機器裡做上面所寫好的script了，那上面有兩個地方應該會有問題就是host跟key那兩個變數的新增位置在Settings

### Settings

![](images/Git_CICD_with_Docker/1_8Wz_y3La0uRxRcdfkytSvA.webp)


### Secrets and variables

裡面的Security點開Secrets and variables中的Actions

![](images/Git_CICD_with_Docker/1_PC6h-Bnv3I5Ow5j0U4NbRQ.webp)


你會看到

![](images/Git_CICD_with_Docker/1_y0993aLDo40RPOChDZMQkw.webp)


這邊就可以管理你在Github actions中的任何密鑰以及變數。

這邊這個CICD最終會部署到我在AWS EC2中架設的一個小機器裡面，裡頭還有很多細節也有很多我還沒搞清楚的地方，之後有機會在拆分主題來一一探討。

---

最後，我是一個轉職的工程師，目前剛轉滿半年多一點，還在努力學習中，如果有任何問題或建議都可以私訊我m23568n@gmail.com，我也常在 [水球軟體學院](https://discord.gg/waterballsa) 活動，歡迎大家一起加入這個大社群一起學習一起進步！！