# Swarm 클러스터를 활용한 TODO 어플리케이션 배포
앱의 전체 구조는 아래와 같다.
- 데이터 스토어 역할을 할 MySQL 서비스를 마스터-슬레이브 구조로 구축
- MySQL 과 데이터를 주고 받을 API 구현
- Nginx 를 웹 어플리케이션과 API 사이에서 리버스 프록시 역할을 하도록 설정
- API 를 사용해 서버 사이드 렌더링을 수행할 웹 어플리케이션을 구현
- 프론트엔드 쪽에 리버스 프록시 (Nginx) 배치

본 예제는 이전 예제 - swarm_dind 에서 시작한다.
```
$ docker exec -it manager docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
mbvdfh56jpljxket3sq8e1fhn *   361d334ebc08        Ready               Active              Leader              18.05.0-ce
cfe2kfxkfxpc809gnorju5cy7     520f0357c711        Ready               Active                                  18.05.0-ce
l2lpu3i9omn7hu75f47jhmp9e     9479d5ee1cfb        Ready               Active                                  18.05.0-ce
oxtu2wd75mfqj3oikpn6l979d     faa94f007257        Ready               Active                                  18.05.0-ce
```

위와 같은 상태로 준비가 되어 있어야 한다.

## mysql 세팅하기

### overlay 네트워크 만들기
```
$ docker container exec -it manager docker network create --driver=overlay --attachable todoapp
```

### MySQL 마스터, 슬레이브 이미지 만들기
```
(tododb) $ docker image build -t ch04/tododb:latest .
(tododb) $ docker image tag ch04/tododb:latest localhost:5000/ch04/tododb:latest
(tododb) $ docker image push localhost:5000/ch04/tododb:latest
```

### MySQL 스웜으로 배포하기
```
$ docker container cp stack/todo-mysql.yml manager:/stack/
$ docker container exec -it manager docker stack deploy -c /stack/todo-mysql.yml todo_mysql
ID                  NAME                MODE                REPLICAS            IMAGE                               PORTS
uw4ngn3wfsv7        todo_mysql_master   replicated          1/1                 registry:5000/ch04/tododb:latest
finnzllelkhb        todo_mysql_slave    replicated          2/2                 registry:5000/ch04/tododb:latest
```

### 초기 데이터 넣기

데이터 쓰기는 마스터에서만 가능하기에 마스터 노드의 정보를 가져온다.
```
$ docker container exec -it manager docker service ps todo_mysql_master --no-trunc --filter "desired-state=running"
```

docker 내부의 중첩 명령어를 쉽게 뽑기 위해 format 기능을 이용한다.
```
$ docker container exec -it manager \
> docker service ps todo_mysql_master \
> --no-trunc \
> --filter "desired-state=running" \
> --format "docker container exec -it {{.Node}} docker container exec -it {{.Name}}.{{.ID}} bash"
docker container exec -it 520f0357c711 docker container exec -it todo_mysql_master.1.nhttzm3643ze98oyp0nbnsiwm bash
```

미리 작성된 쉘 스크립트를 통해 초기 데이터를 집어 넣는다.

```
$ docker container exec -it 520f0357c711 docker container exec -it todo_mysql_master.1.nhttzm3643ze98oyp0nbnsiwm init-data.sh
```

데이터가 제대로 들어갔는지 확인해보자.
```
$ docker container exec -it 520f0357c711 docker container exec -it todo_mysql_master.1.nhttzm3643ze98oyp0nbnsiwm mysql -u gihyo -pgihyo tododb
mysql> SELECT * FROM todo \G
*************************** 1. row ***************************
     id: 1
  title: MySQL 도커 이미지 만들기
content: MySQL 마스터와 슬레이브를 환경 변수로 설정할 수 있는 MySQL 이미지 생성
 status: DONE
created: 2019-04-15 04:01:08
updated: 2019-04-15 04:01:08
*************************** 2. row ***************************
     id: 2
  title: MySQL 스택 만들기
content: MySQL 마스터 및 슬레이브 서비스로 구성된 스택을 스웜 클러스터에 구축한다
 status: DONE
created: 2019-04-15 04:01:08
...
```

slave 에서도 데이터가 제대로 보이는지 확인해보자.
```
$ docker container exec -it manager \
> docker service ps todo_mysql_slave \
> --no-trunc \
> --filter "desired-state=running" \
> --format "docker container exec -it {{.Node}} docker container exec -it {{.Name}}.{{.ID}} bash"
docker container exec -it faa94f007257 docker container exec -it todo_mysql_slave.1.3dagkrf163spqty6a6sw3ksfj bash
docker container exec -it 520f0357c711 docker container exec -it todo_mysql_slave.2.izhk6b3f96yerengmrxzlk3gj bash

$ docker container exec -it faa94f007257 docker container exec -it todo_mysql_slave.1.3dagkrf163spqty6a6sw3ksfj mysql -u gihyo -pgihyo tododb
mysql> SELECT * FROM todo \G
*************************** 1. row ***************************
     id: 1
  title: MySQL 도커 이미지 만들기
content: MySQL 마스터와 슬레이브를 환경 변수로 설정할 수 있는 MySQL 이미지 생성
 status: DONE
created: 2019-04-15 04:01:08
updated: 2019-04-15 04:01:08
*************************** 2. row ***************************
     id: 2
  title: MySQL 스택 만들기
content: MySQL 마스터 및 슬레이브 서비스로 구성된 스택을 스웜 클러스터에 구축한다
 status: DONE
created: 2019-04-15 04:01:08
...
```

## API 서비스 구축
todoapi 디렉토리 내의 Go 로 작성된 API 서버를 배포한다.

### 이미지 준비하기
```
(todoapi) $ docker image build -t ch04/todoapi:latest .
Sending build context to Docker daemon  14.34kB
Step 1/9 : FROM golang:1.9
 ---> ef89ef5c42a9
Step 2/9 : WORKDIR /
 ---> Using cache
 ---> 47b95888d115
 ...

(todoapi) $ docker image tag ch04/todoapi:latest localhost:5000/ch04/todoapi:latest
(todoapi) $ docker image push localhost:5000/ch04/todoapi:latest
The push refers to repository [localhost:5000/ch04/todoapi]
aaff1b119b0e: Pushed
b3267570327f: Pushed
987a76bef245: Pushed
0c570764e0a4: Pushed
3314937e0fbd: Pushed
186d94bd2c62: Mounted from example/echo
24a9d20e5bee: Mounted from example/echo
e7dc337030ba: Mounted from example/echo
920961b94eb3: Mounted from example/echo
fa0c3f992cbd: Mounted from example/echo
ce6466f43b11: Mounted from example/echo
719d45669b35: Mounted from example/echo
3b10514a95be: Mounted from example/echo
latest: digest: sha256:0ecfba3a814a66fa335c6f774cfedec9af59dc08eaf43dcb69767d2504e7fb2e size: 3055
```

### 서비스 배포하기
```
$ docker container cp stack/todo-app.yml manager:/stack/todo-app.yml
$ docker container exec -it manager docker stack deploy -c /stack/todo-app.yml todo_app
Creating service todo_app_api
$ docker container exec -it manager docker service logs -f todo_app_api
todo_app_api.2.cyd2j1r6b9k1@faa94f007257    | 2019/04/15 04:44:05 Listen HTTP Server
todo_app_api.1.o71uk9huorhw@361d334ebc08    | 2019/04/15 04:44:05 Listen HTTP Server
```

## Nginx
API 의 리버스 프록시를 위해 Nginx 를 사용한다.

### 이미지 준비하기
```
(todonginx) $ docker image build -t ch04/nginx:latest .
Sending build context to Docker daemon  9.728kB
Step 1/13 : FROM nginx:1.13
1.13: Pulling from library/nginx
f2aa67a397c4: Pull complete
3c091c23e29d: Pull complete
4a99993b8636: Pull complete
Digest: sha256:b1d09e9718890e6ebbbd2bc319ef1611559e30ce1b6f56b2e3b479d9da51dc35
Status: Downloaded newer image for nginx:1.13
...
(todonginx) $ docker image tag ch04/nginx:latest localhost:5000/ch04/nginx:latest
(todonginx) $ docker image push localhost:5000/ch04/nginx:latest
The push refers to repository [localhost:5000/ch04/nginx]
6bb8020395e6: Pushed
13a452aae3f8: Pushed
78c0fcde3847: Pushed
066ca0575df5: Pushed
...
```

### Nginx 배포하기
```
$ docker container cp stack/todo-app.yml manager:/stack/todo-app.yml
$ docker container exec -it manager docker stack deploy -c /stack/todo-app.yml todo_app
Creating service todo_app_nginx
Updating service todo_app_api (id: i8fdm231v33d4ps4sd4gykqq9)
```

## 웹 서비스

### 이미지 준비하기
```
(todoweb) $ docker image build -t ch04/todoweb:latest .
Sending build context to Docker daemon  425.5kB
Step 1/9 : FROM node:9.2.0
9.2.0: Pulling from library/node
85b1f47fba49: Pull complete
ba6bd283713a: Pull complete
817c8cd48a09: Pull complete
47cc0ed96dc3: Pull complete
8888adcbd08b: Pull complete
...
(todoweb) $ docker image tag ch04/todoweb:latest localhost:5000/ch04/todoweb:latest
(todoweb) $ docker image push localhost:5000/ch04/todoweb:latest
The push refers to repository [localhost:5000/ch04/todoweb]
cdead678e723: Pushed
9edca5df8a96: Pushed
521f9159611e: Pushed
97b2abdb11fb: Pushed
65d5fca7a4f1: Pushed
...
```

### 정적 파일 준비하기
```
(todonginx) $ cp etc/nginx/conf.d/public.conf.tmpl etc/nginx/conf.d/nuxt.conf.tmpl
(todonginx) $ cp Dockerfile Dockerfile-nuxt
(todonginx) $ docker image build -f Dockerfile-nuxt -t ch04/nginx-nuxt:latest .
(todonginx) $ docker image tag ch04/nginx-nuxt:latest localhost:5000/ch04/nginx-nuxt:latest
(todonginx) $ dokcer image push localhost:5000/ch04/nginx-nuxt:latest
```

### Nginx 를 통한 접근 허용하기
```
$ docker container cp stack/todo-frontend.yml manager:/stack/todo-frontend.yml
$ docker exec -it manager docker stack deploy -c /stack/todo-frontend.yml todo_frontend
Creating service todo_frontend_web
Creating service todo_frontend_nginx
```

### 인그레스로 서비스 노출하기
```
$ docker container cp stack/todo-ingress.yml manager:/stack/todo-ingress.yml
$ docker container exec -it manager docker stack deploy -c /stack/todo-ingress.yml todo_ingress
```

### 배포된 서비스 확인하기
```
$ curl -I http://localhost:8000/
HTTP/1.1 200 OK
Server: nginx/1.13.12
Date: Mon, 15 Apr 2019 06:45:41 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 19049
X-Powered-By: Express
ETag: "4a69-Zg4WKQenGJveb+YRuQ9WEHQuwwE"
Vary: Accept-Encoding

$ curl -I http://localhost:8000/_nuxt/app.84213ab389afece29614.js
HTTP/1.1 200 OK
Server: nginx/1.13.12
Date: Mon, 15 Apr 2019 06:45:55 GMT
Content-Type: application/javascript; charset=utf-8
Content-Length: 27272
Last-Modified: Mon, 15 Apr 2019 05:27:21 GMT
ETag: "5cb41639-6a88"
Accept-Ranges: bytes
```

/ 경로는 express 가 렌더링해주고, 정적 파일은 Nginx 에서 바로 보내주는 것을 확인할 수 있다.

브라우저로 localhost:8000 에 접근해보자.