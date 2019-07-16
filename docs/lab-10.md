# LAB-10. 스모크 테스트

이 실습에서는 쿠버네티스 클러스터가 올바르게 작동하는지 확인하기 위해 테스트 해봅니다.

## 1. 데이터 암호화

이 섹션에서는, [시크릿 데이터](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#verifying-that-data-is-encrypted)를 암호화하는 기능을 확인합니다.

generic secret 오브젝트 생성

```sh
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"
```

etcd에 저장된 kubernetes-the-hard-way secret의 hexdump를 출력합니다.

```
gcloud compute ssh controller-0 \
  --command "sudo ETCDCTL_API=3 etcdctl get \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem\
  /registry/secrets/default/kubernetes-the-hard-way | hexdump -C"
```

> 출력

```
00000000  2f 72 65 67 69 73 74 72  79 2f 73 65 63 72 65 74  |/registry/secret|
00000010  73 2f 64 65 66 61 75 6c  74 2f 6b 75 62 65 72 6e  |s/default/kubern|
00000020  65 74 65 73 2d 74 68 65  2d 68 61 72 64 2d 77 61  |etes-the-hard-wa|
00000030  79 0a 6b 38 73 3a 65 6e  63 3a 61 65 73 63 62 63  |y.k8s:enc:aescbc|
00000040  3a 76 31 3a 6b 65 79 31  3a dd 3f 36 6c ce 65 9d  |:v1:key1:.?6l.e.|
00000050  b3 b1 46 1a ba ae a2 1f  e4 fa 13 0c 4b 6e 2c 3c  |..F.........Kn,<|
00000060  15 fa 88 56 84 b7 aa c0  7a ca 66 f3 de db 2b a3  |...V....z.f...+.|
00000070  88 dc b1 b1 d8 2f 16 3e  6b 4a cb ac 88 5d 23 2d  |...../.>kJ...]#-|
00000080  99 62 be 72 9f a5 01 38  15 c4 43 ac 38 5f ef 88  |.b.r...8..C.8_..|
00000090  3b 88 c1 e6 b6 06 4f ae  a8 6b c8 40 70 ac 0a d3  |;.....O..k.@p...|
000000a0  3e dc 2b b6 0f 01 b6 8b  e2 21 29 4d 32 d6 67 a6  |>.+......!)M2.g.|
000000b0  4e 6d bb 61 0d 85 22 ea  f4 d6 2d 0a af 3c 71 85  |Nm.a.."...-..<q.|
000000c0  96 27 c9 ec 90 e3 56 8c  94 a7 1c 9a 0e 00 28 11  |.'....V.......(.|
000000d0  18 28 f4 33 42 d9 57 d9  e3 e9 1c 38 e3 bc 1e c3  |.(.3B.W....8....|
000000e0  d2 47 f3 20 60 be b8 57  a7 0a                    |.G. `..W..|
000000ea
```

etcd 키 앞에는 `k8s:enc:aescbc:v1:key1` 이라는 접두사가 있어야합니다. 이는 `aescbc` 제공자가 `key1` 암호화 키로 데이터를 암호화하는 데 사용되었음을 나타냅니다.

## 디플로이먼트

이 섹션에서는 [디플로이먼트](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)를 만들고 관리하는 기능을 확인합니다.

[nginx](https://nginx.org/en/) 웹 서버에 대한 디플로이먼트를 만듭니다.

```
kubectl run nginx --image=nginx
```

`nginx` 디플로이먼트로 생성된 pod 목록 조회

```
kubectl get pods -l run=nginx
```

> 출력

```
NAME                    READY   STATUS    RESTARTS   AGE
nginx-dbddb74b8-6lxg2   1/1     Running   0          10s
```

### 포트 포워딩

이 섹션에서는 [포트 포워딩](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)을 사용하여 원격으로 어플리케이션에 액세스하는 기능을 확인합니다.


`nginx` 파드 이름을 변수에 할당

```
POD_NAME=$(kubectl get pods -l run=nginx -o jsonpath="{.items[0].metadata.name}")
```

로컬 머신의 `8080` 포트를 `nginx` 파드의 `80` 포트로 포워딩합니다.

```
kubectl port-forward $POD_NAME 8080:80
```

> 출력

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
```

새로운 터미널 창에서 포워딩 주소를 사용하여 HTTP 요청합니다.

```
curl --head http://127.0.0.1:8080
```

> 출력

```
HTTP/1.1 200 OK
Server: nginx/1.15.4
Date: Sun, 30 Sep 2018 19:23:10 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 25 Sep 2018 15:04:03 GMT
Connection: keep-alive
ETag: "5baa4e63-264"
Accept-Ranges: bytes
```

이전 터미널로 다시 전환하고 `nginx` 파드로 포트 포워딩을 중지합니다.

```
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
^C
```

### 로그

이 섹션에서는 [컨테이너 로그를 검색](https://kubernetes.io/docs/concepts/cluster-administration/logging/)하는 기능을 확인합니다.

`nginx` 파드 로그를 출력합니다.

```
kubectl logs $POD_NAME
```

> 출력

```
127.0.0.1 - - [30/Sep/2018:19:23:10 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.58.0" "-"
```

### 실행 (Exec)

이 섹션에서는 [컨테이너에서 명령을 실행](https://kubernetes.io/docs/tasks/debug-application-cluster/get-shell-running-container/#running-individual-commands-in-a-container)할 수 있는지 확인합니다.

`nginx` 컨테이너에서 `nginx -v` 명령을 실행하여 nginx 버전을 출력합니다.

```
kubectl exec -ti $POD_NAME -- nginx -v
```

> 출력

```
nginx version: nginx/1.15.4
```

## 서비스

이 섹션에서는 [서비스](https://kubernetes.io/docs/concepts/services-networking/service/)를 사용하여 어플리케이션을 노출하는 기능을 확인합니다.

```sh
kubectl expose deployment nginx --port 80 --type NodePort
```

> 클러스터가 [클라우드 공급자 통합](https://kubernetes.io/docs/getting-started-guides/scratch/#cloud-provider)으로 구성되어 있지 않으므로 LoadBalancer 서비스 유형을 사용할 수 없습니다.


`nginx` 서비스에 할당된 NodePort를 검색합니다.

```sh
NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')
```

`nginx` NodePort에 해당 포트로 원격 접근을 허용하는 방화벽 규칙을 추가합니다.

```
gcloud compute firewall-rules create kubernetes-the-hard-way-allow-nginx-service \
  --allow=tcp:${NODE_PORT} \
  --network kubernetes-the-hard-way
```

워커 노드 인스턴스의 외부 IP 주소 검색

```
EXTERNAL_IP=$(gcloud compute instances describe worker-0 \
  --format 'value(networkInterfaces[0].accessConfigs[0].natIP)')
```

외부 IP 주소와 `nginx` 노드 포트를 사용하여 HTTP 요청을 합니다.

```
curl -I http://${EXTERNAL_IP}:${NODE_PORT}
```

> 출력

```
HTTP/1.1 200 OK
Server: nginx/1.15.4
Date: Sun, 30 Sep 2018 19:25:40 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 25 Sep 2018 15:04:03 GMT
Connection: keep-alive
ETag: "5baa4e63-264"
Accept-Ranges: bytes
```

## 신뢰할 수 없는 (untrusted) 워크로드

이 섹션에서는 [gVisor](https://github.com/google/gvisor)를 사용하여 신뢰할 수 없는 워크로드를 실행할 수 있는지 확인합니다.

`신뢰할 수 없는` 파드 만들기

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: untrusted
  annotations:
    io.kubernetes.cri.untrusted-workload: "true"
spec:
  containers:
    - name: webserver
      image: gcr.io/hightowerlabs/helloworld:2.0.0
EOF
```

### 확인

이 섹션에서는 할당된 워커 노드를 검사하여 `신뢰할 수 없는 파드`를 gVisor (runsc)에서 실행하는지 확인합니다.

`untrusted` 파드가 실행 중인지 확인합니다.

```
kubectl get pods -o wide
```
```
NAME                       READY     STATUS    RESTARTS   AGE       IP           NODE
busybox-68654f944b-djjjb   1/1       Running   0          5m        10.200.0.2   worker-0
nginx-65899c769f-xkfcn     1/1       Running   0          4m        10.200.1.2   worker-1
untrusted                  1/1       Running   0          10s       10.200.0.3   worker-0
```

`신뢰할 수 없는` 파드에서 실행중인 노드 이름을 가져옵니다.

```
INSTANCE_NAME=$(kubectl get pod untrusted --output=jsonpath='{.spec.nodeName}')
```

워커 노드에 SSH 접속

```
gcloud compute ssh ${INSTANCE_NAME}
```

gVisor에서 실행되는 컨테이너를 조회

```
sudo runsc --root  /run/containerd/runsc/k8s.io list
```
```
I0930 19:27:13.255142   20832 x:0] ***************************
I0930 19:27:13.255326   20832 x:0] Args: [runsc --root /run/containerd/runsc/k8s.io list]
I0930 19:27:13.255386   20832 x:0] Git Revision: 50c283b9f56bb7200938d9e207355f05f79f0d17
I0930 19:27:13.255429   20832 x:0] PID: 20832
I0930 19:27:13.255472   20832 x:0] UID: 0, GID: 0
I0930 19:27:13.255591   20832 x:0] Configuration:
I0930 19:27:13.255654   20832 x:0]              RootDir: /run/containerd/runsc/k8s.io
I0930 19:27:13.255781   20832 x:0]              Platform: ptrace
I0930 19:27:13.255893   20832 x:0]              FileAccess: exclusive, overlay: false
I0930 19:27:13.256004   20832 x:0]              Network: sandbox, logging: false
I0930 19:27:13.256128   20832 x:0]              Strace: false, max size: 1024, syscalls: []
I0930 19:27:13.256238   20832 x:0] ***************************
ID                                                                 PID         STATUS      BUNDLE                                                                                                                   CREATED                OWNER
79e74d0cec52a1ff4bc2c9b0bb9662f73ea918959c08bca5bcf07ddb6cb0e1fd   20449       running     /run/containerd/io.containerd.runtime.v1.linux/k8s.io/79e74d0cec52a1ff4bc2c9b0bb9662f73ea918959c08bca5bcf07ddb6cb0e1fd   0001-01-01T00:00:00Z
af7470029008a4520b5db9fb5b358c65d64c9f748fae050afb6eaf014a59fea5   20510       running     /run/containerd/io.containerd.runtime.v1.linux/k8s.io/af7470029008a4520b5db9fb5b358c65d64c9f748fae050afb6eaf014a59fea5   0001-01-01T00:00:00Z
I0930 19:27:13.259733   20832 x:0] Exiting with status: 0
```

`신뢰할 수 없는` 파드의 ID를 가져옵니다.

```
POD_ID=$(sudo crictl -r unix:///var/run/containerd/containerd.sock \
  pods --name untrusted -q)
```

`신뢰할 수 없는` 파드에서 실행중인 `webserver` 컨테이너의 ID를 변수화합니다.

```
CONTAINER_ID=$(sudo crictl -r unix:///var/run/containerd/containerd.sock \
  ps -p ${POD_ID} -q)
```

gVisor `runsc` 명령을 사용하여 `webserver` 컨테이너에서 실행중인 프로세스를 조회합니다.

```
sudo runsc --root /run/containerd/runsc/k8s.io ps ${CONTAINER_ID}
```

> 출력

```
I0930 19:31:31.419765   21217 x:0] ***************************
I0930 19:31:31.419907   21217 x:0] Args: [runsc --root /run/containerd/runsc/k8s.io ps af7470029008a4520b5db9fb5b358c65d64c9f748fae050afb6eaf014a59fea5]
I0930 19:31:31.419959   21217 x:0] Git Revision: 50c283b9f56bb7200938d9e207355f05f79f0d17
I0930 19:31:31.420000   21217 x:0] PID: 21217
I0930 19:31:31.420041   21217 x:0] UID: 0, GID: 0
I0930 19:31:31.420081   21217 x:0] Configuration:
I0930 19:31:31.420115   21217 x:0]              RootDir: /run/containerd/runsc/k8s.io
I0930 19:31:31.420188   21217 x:0]              Platform: ptrace
I0930 19:31:31.420266   21217 x:0]              FileAccess: exclusive, overlay: false
I0930 19:31:31.420424   21217 x:0]              Network: sandbox, logging: false
I0930 19:31:31.420515   21217 x:0]              Strace: false, max size: 1024, syscalls: []
I0930 19:31:31.420676   21217 x:0] ***************************
UID       PID       PPID      C         STIME     TIME      CMD
0         1         0         0         19:26     10ms      app
I0930 19:31:31.422022   21217 x:0] Exiting with status: 0
```
