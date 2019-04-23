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
