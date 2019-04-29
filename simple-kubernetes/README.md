# Kubernetes 를 사용하기
## 간단한 pod 만들기
```
$ kubectl apply -f simple-pod.yml
pod "simple-echo" created
```

만들어진 pod 조회하기

```
$ kubectl get pod
NAME          READY     STATUS    RESTARTS   AGE
simple-echo   2/2       Running   0          31s
```

컨테이너에 접근하기

```
$ kubectl exec -it simple-echo sh -c nginx
# 
```

컨테이너의 표준 출력 화면에 출력하기

```
$ kubectl logs -f simple-echo -c echo
2019/04/22 09:06:26 start server
```

pod 삭제하기

```
$ kubectl delete pod simple-echo
pod "simple-echo" deleted
```

매니페스트를 이용해 pod 삭제하기 (메니페스트에 작성된 모든 리소스 삭제)
```
$ kubectl delete -f simple-pod.yml
pod "simple-echo" deleted
```

## Replica Set
똑같은 정의를 갖는 pod 를 여러 개 생성하고 관리하기 위한 리소스
```
$ kubectl apply -f simple-replicaset.yml
replicaset.apps "echo" created

$ kubectl get pod
NAME         READY     STATUS    RESTARTS   AGE
echo-7dlv4   2/2       Running   0          30s
echo-qshkk   2/2       Running   0          30s
echo-slpxc   2/2       Running   0          30s
```

Replica Set 를 삭제하는 방법은 아래와 같다.
```
$ kubectl delete -f simple-replicaset.yml
replicaset.apps "echo" deleted

$ kubectl get pod
NAME         READY     STATUS        RESTARTS   AGE
echo-7dlv4   2/2       Terminating   0          1m
echo-qshkk   2/2       Terminating   0          1m
echo-slpxc   2/2       Terminating   0          1m
```

## Deployment
어플리케이션 배포의 기본이 되는 단위. replica set 를 관리하고 다루기 위한 리소스
```
$ kubectl apply -f simple-deployment.yml --record
deployment.apps "echo" created

$ kubectl get pod,replicaset,deployment --selector app=echo
NAME                        READY     STATUS    RESTARTS   AGE
pod/echo-8556ddbfb9-4r7wv   2/2       Running   0          23s
pod/echo-8556ddbfb9-4t9st   2/2       Running   0          23s
pod/echo-8556ddbfb9-c2f8k   2/2       Running   0          23s

NAME                                    DESIRED   CURRENT   READY     AGE
replicaset.extensions/echo-8556ddbfb9   3         3         3         23s

NAME                         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/echo   3         3         3            3           23s
```

revivsion 을 확인해보자.
```
$ kubectl rollout history deployment echo
deployments "echo"
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=simple-deployment.yml --record=true
```

### Replicaset 생애주기

#### 파드의 개수만 수정하면 replicaset 가 새로 생성되지 않는다.  
simple-deployment.yml 파일의 replicas 를 3 -> 4 로 변경하고 다음을 실행하자
```
$ kubectl apply -f simple-deployment.yml --record
deployment.apps "echo" configured

$ kubectl get pod
NAME                    READY     STATUS    RESTARTS   AGE
echo-8556ddbfb9-4r7wv   2/2       Running   0          1d
echo-8556ddbfb9-4t9st   2/2       Running   0          1d
echo-8556ddbfb9-c2f8k   2/2       Running   0          1d
echo-8556ddbfb9-vrwdd   2/2       Running   0          18s

$ kubectl rollout history deployment echo
deployments "echo"
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=simple-deployment.yml --record=true
```

REVISION 이 변경되지 않고 1 로 있음을 확인할 수 있다.

#### 컨테이너의 정의를 수정하면 replicaset 이 새롭게 설정된다.

`simple-deployment.yml` 파일의 이미지를 `gihyodocker/echo:patched` 로 수정한다.
```
		- name: nginx
          image: gihyodocker/nginx:latest
          env:
            - name: BACKEND_HOST
              value: localhost:8080
```
```
$ kubectl apply -f simple-deployment.yml --record
deployment "echo" configured
```

기존 pod 들은 순차적으로 종료된다. 최종적으로 새롭게 실행된 pod 들만 남는다.
```
$ kubectl get pod --selector app=echo
NAME                    READY     STATUS    RESTARTS   AGE
echo-6bfffbcf9f-6qlnk   2/2       Running   0          19h
echo-6bfffbcf9f-hkpvp   2/2       Running   0          19h
echo-6bfffbcf9f-m7npw   2/2       Running   0          19h
echo-6bfffbcf9f-szjsh   2/2       Running   0          19h

$ kubectl rollout history deployment echo
deployments "echo"
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=simple-deployment.yml --record=true
2         kubectl apply --filename=simple-deployment.yml --record=true
```

### 롤백 수행하기

리비전 정보를 확인해보자.
```
$ kubectl rollout history deployment echo --revision=1
deployments "echo" with revision #1
Pod Template:
  Labels:	app=echo
	pod-template-hash=4112886965
  Annotations:	kubernetes.io/change-cause=kubectl apply --filename=simple-deployment.yml --record=true
  Containers:
   nginx:
    Image:	gihyodocker/nginx:latest
    Port:	80/TCP
    Host Port:	0/TCP
    Environment:
      BACKEND_HOST:	localhost:8080
    Mounts:	<none>
   echo:
    Image:	gihyodocker/echo:latest
    Port:	8080/TCP
    Host Port:	0/TCP
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>
```

undo 를 실행하면 바로 직적 리비전으로 롤백된다.
```
$ kubectl rollout undo deployment echo
deployment.apps "echo"

$ kubectl rollout history deployment echo
deployments "echo"
REVISION  CHANGE-CAUSE
2         kubectl apply --filename=simple-deployment.yml --record=true
3         kubectl apply --filename=simple-deployment.yml --record=true
```

deployment 및 replicaset 와 pod 삭제하기
```
$ kubectl delete -f simple-deployment.yml
deployment.apps "echo" deleted
```

## 서비스
클러스터 안에서 pod 의 집합 (주로 replica set) 에 대한 경로나 서비스 디스커버리를 제공하는 리소스
```
$ kubectl apply -f simple-replicaset-with-label.yml
replicaset.apps "echo-summer" created

$ kubectl get pod -l app=echo -l release=spring
NAME                READY     STATUS    RESTARTS   AGE
echo-spring-7t4rf   2/2       Running   0          14s

$ kubectl get pod -l app=echo -l release=summer
NAME                READY     STATUS    RESTARTS   AGE
echo-summer-7dq4t   2/2       Running   0          24s
echo-summer-fxf7k   2/2       Running   0          24s
```

이제 relase=summer 인 pod 만 접근할 수 있는 서비스를 생성해보자.
```
$ kubectl apply -f simple-service.yml
service "echo" created

$ kubectl get svc echo
NAME      TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
echo      ClusterIP   10.99.223.242   <none>        80/TCP    7s
```

서비스는 기본적으로 클러스터 안에서만 접근할 수 있으므로 디버깅용 임시 컨테이너를 배포하고 curl 명령으로 HTTP 요청을 확인해 보자.
```
$ kubectl run -it --rm debug --image=gihyodocker/fundamental:0.1.0 --restart=Never -- bash -il
If you don't see a command prompt, try pressing enter.
debug:/# curl http://echo/
Hello Docker!!debug:/#

$ kubectl get pod
NAME                READY     STATUS    RESTARTS   AGE
echo-spring-7t4rf   2/2       Running   0          13m
echo-summer-7dq4t   2/2       Running   0          16m
echo-summer-fxf7k   2/2       Running   0          16m

$ kubectl logs -f echo-summer-7dq4t -c echo
2019/04/26 05:38:26 start server
2019/04/26 05:52:50 received request
```

### ingress
클러터스의 외부 노출과 정교한 HTTP/HTTPS 라우팅을 동시에 사용 가능하게 함.
```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
namespace "ingress-nginx" created
configmap "nginx-configuration" created
configmap "tcp-services" created
configmap "udp-services" created
serviceaccount "nginx-ingress-serviceaccount" created
clusterrole.rbac.authorization.k8s.io "nginx-ingress-clusterrole" created
role.rbac.authorization.k8s.io "nginx-ingress-role" created
rolebinding.rbac.authorization.k8s.io "nginx-ingress-role-nisa-binding" created
clusterrolebinding.rbac.authorization.k8s.io "nginx-ingress-clusterrole-nisa-binding" created
deployment.apps "nginx-ingress-controller" created

$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/cloud-generic.yaml
service "ingress-nginx" created

$ kubectl -n ingress-nginx get service,pod
NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx   LoadBalancer   10.110.123.192   localhost     80:32050/TCP,443:30138/TCP   1m

NAME                                            READY     STATUS    RESTARTS   AGE
pod/nginx-ingress-controller-78474696b4-zgtvk   1/1       Running   0          1m
```

`simple-service.yml` 을 다음과 같이 수정한다.
```
apiVersion: v1
kind: Service
metadata:
  name: echo
spec:
  selector:
    app: echo
  ports:
    - name: http
      port: 80
```

`simple-ingress.yml` 을 만들고 다음과 같이 입력한다.
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: echo
spec:
  rules:
    - host: ch05.gihyo.local
      http:
        paths:
          - path: /
            backend:
              serviceName: echo
              servicePort: 80
```

이제 ingress 를 반영한다.
```
$ kubectl apply -f simple-ingress.yml
ingress.extensions "echo" created

$ kubectl get ingress
NAME      HOSTS              ADDRESS     PORTS     AGE
echo      ch05.gihyo.local   localhost   80        25s
```

이제 로컬에서 http 요청을 통해 응답을 확인할 수 있다.
```
$ curl http://localhost -H 'Host: ch05.gihyo.local'
Hello Docker!!
```

ingress 층에서 HTTP 요청에 대해 다양한 제어를 할 수 있다.  
`simple-ingress.yml` 을 다음과 같이 수정하자.
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: echo
  annotations:
    nginx.ingress.kubernetes.io/server-snippet: |
      set $agentflag 0;
      if ($http_user_agent ~* "(Mobile)") {
        set $agentflag 1;
      }
      if ($agentflag = 1) {
        return 301 http://gihyo.jp/;
      }
spec:
  rules:
    - host: ch05.gihyo.local
      http:
        paths:
          - path: /
            backend:
              serviceName: echo
              servicePort: 80

```

그리고 나서 변경 사항을 적용하고 Mobile 이 들어간 User-Agent 로 요청을 하면 다음과 같은 결과를 볼 수 있다.
```
$ kubectl apply -f simple-ingress.yml
ingress.extensions "echo" configured

$ curl http://localhost -H 'Host: ch05.gihyo.local' -H 'User-Agent: Mobile'
<html>
<head><title>301 Moved Permanently</title></head>
<body>
<center><h1>301 Moved Permanently</h1></center>
<hr><center>nginx/1.15.10</center>
</body>
</html>
```

User-Agent 를 보내지 않으면 원래의 echo 응답을 확인할 수 있다.
```
$ curl http://localhost -H 'Host: ch05.gihyo.local'
Hello Docker!!
```