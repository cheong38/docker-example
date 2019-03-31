# Docker 가장 기초 예제
go 언어로 만들어진 http echo server 를 docker 로 실행해보는 예제

## 실행법
```
$ docker image build -t example/echo:latest .
$ docker container run -p 9000:8080 example/echo:latest
$ curl http://localhost:9000/
```

## Compose 를 활용한 실행법
```
docker-compose up --build
```
