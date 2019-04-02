# 볼륨 컨테이너 활용을 통한 MySQL 데이터 저장하기
## 실행법
```
$ docker image build -t example/mysql-data:latest .
$ docker container run -d --name mysql-data example/mysql-data:latest
$ docker container run -d --rm --name mysql \
  -e "MYSQL_ALLOW_EMPTY_PASSWORD=yes" \
  -e "MYSQL_DATABASE=volume_test" \
  -e "MYSQL_USER=example" \
  -e "MYSQL_PASSWORD=example" \
  --volumes-from mysql-data \
  mysql:5.7
$ docker container exec -it mysql mysql -u root -p volume_test
```

패스워드를 빈 값으로 입력한다.

```
mysql> CREATE TABLE user (
	id int PRIMARY KEY AUTO_INCREMENT,
	name VARCHAR(255)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE utf8mb4_unicode_ci;

mysql> INSERT INTO user (name) VALUES ('gihyo'), ('docker'), ('Solomon Hykes');
```

mysql 도커를 종료하고 다시 실행해본다.

```
$ docker container stop mysql
$ docker container run -d --rm --name mysql \
  -e "MYSQL_ALLOW_EMPTY_PASSWORD=yes" \
  -e "MYSQL_DATABASE=volume_test" \
  -e "MYSQL_USER=example" \
  -e "MYSQL_PASSWORD=example" \
  --volumes-from mysql-data \
  mysql:5.7
```

위에서 생성한 데이터가 저장되어 있는지 확인한다.

```
$ docker container exec -it mysql mysql -u root -p volume_test
mysql> SELECT * FROM user;
```

데이터가 제대로 저장된 것을 확인할 수 있다.