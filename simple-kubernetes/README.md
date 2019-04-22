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
