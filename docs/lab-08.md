# LAB-08. 네트워킹

## 1. 모든 워커 노드에서 IP forwarding 활성화

```sh
sudo sysctl net.ipv4.conf.all.forwarding=1
echo "net.ipv4.conf.all.forwarding=1" | sudo tee -a /etc/sysctl.conf
```

## 2. 클러스터에 Weave Net 설치

Weaveworks를 사용하여 Weave Net 설치
```sh
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')&env.IPALLOC_RANGE=10.200.0.0/16"
```

확인
```sh
kubectl get pods -n kube-system
```

> 출력 (예)

```
NAME              READY     STATUS    RESTARTS   AGE
weave-net-m69xq   2/2       Running   0          11s
weave-net-vmb2n   2/2       Running   0          11s
```

## 3. 네트워킹 기능을 테스트하기 위해 파드 실행

### 3-1. 디플로이먼트 생성

2개의 레플리카를 spec으로 하는 nginx 디플로이먼트 생성

```yaml
cat << EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      run: nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF
```

### 3-2. 서비스 생성

서비스에 대한 연결을 테스트 할 수 있도록 해당 디플로이먼트에 대한 서비스 생성

```sh
kubectl expose deployment/nginx
```

### 3-3. 테스트용 busybox 파드 실행

이 파드에서 다른 파드 및 서비스에 연결할 수 있는지 여부를 테스트합니다.

```sh
kubectl run busybox --image=radial/busyboxplus:curl --command -- sleep 3600
POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
```

### 3-4. 테스트

#### 3-4-1. 파드 (Pod) 테스트

두 개의 nginx 파드의 IP 주소 조회

```
kubectl get ep nginx
```

```
NAME      ENDPOINTS                       AGE
nginx     10.200.0.2:80,10.200.128.1:80   50m
```

nginx 파드 테스트

```sh
kubectl exec $POD_NAME -- curl <첫번째 nginx 파드 IP 주소>
kubectl exec $POD_NAME -- curl <두번째 nginx 파드 IP 주소>
```

#### 3-4-2. 서비스 테스트

서비스 조회
```sh
kubectl get svc
```

nginx 서비스 IP 주소 확인
```
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.32.0.1    <none>        443/TCP   1h
nginx        ClusterIP   10.32.0.123   <none>        80/TCP   23m
```

nginx 서비스 테스트
```sh
kubectl exec $POD_NAME -- curl <nginx 서비스 IP 주소>
```
