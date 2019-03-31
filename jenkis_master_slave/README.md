# jenkins 의 마스터와 슬레이브 구조를 docker compose 로 구성
## 실행법
1) 도커 컨테이너 실행
```
$ docker-compose up
```

2) 실행된 명령어 중간의 password 따로 기록해두기

3) http://localhost:8080 에 브라우저로 접근

4) 2) 에서 기록한 password 입력하고 기본 세팅 진행

5) 마스터의 ssh 키 생성
```
$ docker container exec -it master ssh-keygen -t rsa -C ""
```
입력 나오는 부분에 그냥 모두 엔터 입력

6) `docker-compose.yml` 의 `${JENKINS_SLAVE_SSH_PUBKEY}` 부분 바꿔치기
```
$ docker container exec -it master cat /var/jenkins_home/.ssh/id_rsa.pub
```

7) http://localhost:8080 에서 노드관리를 메뉴를 통해 slave 등록
