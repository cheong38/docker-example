# Docker in Docker (dind) 를 이용한 docker swarm 설정하기
본 예제는 다음과 같은 컴포넌트로 구성되어 있음

- registry x 1
- manager x 1
- worker x 3

## 실행법
```
$ docker-compose up -d
$ docker container ls
```

아직은 클러스터로 설정된 상태는 아니다.

```
$ docker container exec -it manager docker swarm init
Swarm initialized: current node (mbvdfh56jpljxket3sq8e1fhn) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-0ihk84jpwm9jzsk07wwkh3l6svkxpi4lr485yx2fj06bleoso8-adhedscmrw7f5kq917i3l3jfw 172.20.0.3:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

join 토큰을 이용해 3대의 노드를 스웜 클러스터에 worker 로 등록한다.

```
$ docker container exec -it worker01 docker swarm join --token SWMTKN-1-0ihk84jpwm9jzsk07wwkh3l6svkxpi4lr485yx2fj06bleoso8-adhedscmrw7f5kq917i3l3jfw 172.20.0.3:2377
$ docker container exec -it manager docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
mbvdfh56jpljxket3sq8e1fhn *   361d334ebc08        Ready               Active              Leader              18.05.0-ce
l2lpu3i9omn7hu75f47jhmp9e     9479d5ee1cfb        Ready               Active                                  18.05.0-ce
```

나머지 노드들도 마찬가지로 추가한다.

```
$ docker container exec -it worker02 docker swarm join --token SWMTKN-1-0ihk84jpwm9jzsk07wwkh3l6svkxpi4lr485yx2fj06bleoso8-adhedscmrw7f5kq917i3l3jfw 172.20.0.3:2377
$ docker container exec -it worker03 docker swarm join --token SWMTKN-1-0ihk84jpwm9jzsk07wwkh3l6svkxpi4lr485yx2fj06bleoso8-adhedscmrw7f5kq917i3l3jfw 172.20.0.3:2377
```

이제 registry 에 도커 이미지를 등록한다.  
이미지는 앞서 만들었던 `example/echo:latest` 를 이용한다.

```
$ docker image tag example/echo:latest localhost:5000/example/echo:latest
$ docker image push localhost:5000/example/echo:latest
```

이제 worker01 에서 이미지를 다운로드하도록 하자.

```
$ docker container exec -it worker01 docker image pull registry:5000/example/echo:latest
```

스웜을 사용하여 컨테이너를 배포하기

```
$ docker container exec -it manager \
> docker service create --replicas 1 --publish 8000:8080 --name echo registry:5000/example/echo:latest
```

서비스의 컨테이너 수 조절하기
```
$ docker container exec -it manager docker service scale echo=6
$ docker container exec -it manager docker service ps echo
```

배포된 서비스 삭제하기
```
$ docker container exec -it manager docker service rm echo
$ docker container exec -it manager docker service ls
```

## 스택

우선 overlay 네트워크를 만든다.
```
$ docker container exec -it manager docker network create --driver=overlay --attachable ch03
```

web api 환경 구성을 위한 docker-compose yml 파일을 컨테이너에 복사해 넣는다.
```
$ docker container exec -it manager mkdir /stack
$ docker container cp stack/ch03-webapi.yml manager:/stack/
```

stack 배포하기
```
$ docker container exec -it manager docker stack deploy -c /stack/ch03-webapi.yml echo
```

스택 echo 의 서비스 목록 확인하기
```
$ docker container exec -it manager docker stack services echo
```

스택에 배포된 컨테이너 확인하기
```
$ docker container exec -it manager docker stack ps echo
```

## 스택 시각화하기

visualizer yml 파일을 복사한다.
```
$ docker container cp stack/visualizer.yml manager:/stack/
```

visualizer 를 배포한다.
```
$ docker container exec -it manager docker stack deploy -c /stack/visualizer.yml visualizer
```

브라우저를 통해 http://localhost:9000 으로 접속한다.

## HA proxy 로 외부 접근 허용하기

ingress yml 파일을 복사한다.
```
$ docker container cp stack/ch03-ingress.yml manager:/stack/
```

ingress 를 배포한다.
```
$ docker container exec -it manager docker stack deploy -c /stack/ch03-ingress.yml ingress
```

이제 8000 번 포트로 접속해본다.
```
$ curl http://localhost:8000
Hello Docker!!
```
