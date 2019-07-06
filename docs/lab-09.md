# LAB-09. DNS 클러스터 추가 기능 배포

## Deploy kube-dns to the cluster.

kube-dns를 클러스터에 배포

```sh
kubectl create -f https://storage.googleapis.com/kubernetes-the-hard-way/kube-dns.yaml
```

확인

```sh
kubectl get pods -l k8s-app=kube-dns -n kube-system
```

출력(예)

```
NAME                        READY     STATUS    RESTARTS   AGE
kube-dns-598d7bf7d4-spbmj   3/3       Running   0          36s
```

## DNS 테스트


```sh
kubectl run nginx --image=nginx
kubectl expose deployment nginx --port 80
```

서비스 목록 조회

```sh
kubectl get svc
```

테스트용 busybox 파드 실행

```sh
kubectl run busybox --image=busybox:1.28 --command -- sleep 3600
POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```

DNS 작동 확인

```sh
kubectl exec $POD_NAME -- nslookup nginx
```

```
Server:    10.32.0.10
Address 1: 10.32.0.10 kube-dns.kube-system.svc.cluster.local

Name:      nginx
Address 1: 10.32.0.248 nginx.default.svc.cluster.local
```
