# LAB-05. Kubernetes Control Plane 부트스트래핑

- 이 랩에서는 쿠버네티스 두 개의 마스터 노드 VM 상에서 컨트롤 플레인을 부트스트랩 합니다.
- 각 노드에는 Kubernetes API Server, Scheduler 및 Controller Manager가 설치됩니다.
- Kubernetes API 서버를 원격 클라이언트에 노출시키는 로드 밸런서를 만듭니다.


## 0. 들어가기 전에

- [컨트롤 플레인](https://kubernetes.io/ko/docs/concepts/#%ec%bf%a0%eb%b2%84%eb%84%a4%ed%8b%b0%ec%8a%a4-%ec%bb%a8%ed%8a%b8%eb%a1%a4-%ed%94%8c%eb%a0%88%ec%9d%b8)
  - 쿠버네티스 클러스터를 제어하는 ​​서비스 (by 컨트롤 플레인 컴포넌트)
  - 쿠버네티스 오브젝트의 레코드와 상태를 관리
  - 클러스터에 대한 글로벌한 의사결정, 이벤트에 대한 탐지 및 응답 수행
- 컨트롤 플레인 컴포넌트 (마스터)
  - kube-apiserver : 쿠버네티스 API 제공. 사용자 - 클러스터 간 인터페이스
  - etcd : 쿠버네티스 클러스터 데이터 저장소
  - kube-scheduler : 가용한 워커 노드에 파드(pods) 스케줄링
  - kube-controller-manager : 다양한 기능을 제공하는 4가지 종류의 컨트롤러를 실행
  - ~~cloud-controller-manager : 클라우드 제공사업자와 상호작용 (AWS, GCP, Azure)~~


> 위 쿠버네티스 컴포넌트에 대한 설명은 [이곳](https://kubernetes.io/ko/docs/concepts/overview/components/)을 참고하세요.


## 1. 준비 사항

이 랩에서 명령은 마스터 노드인 `controller-0`,`controller-1` 모두 에서 실행되어야 합니다.

GCP 인스턴스 ssh 로그인
```
gcloud compute ssh controller-0
```

## 2. 쿠버네티스 Control Plane 프로비저닝

쿠버네티스 구성 디렉토리 생성

```
sudo mkdir -p /etc/kubernetes/config
```

### 2-1. 쿠버네티스 컨트롤러 바이너리 다운로드 및 설치

쿠버네티스 릴리스 바이너리 다운로드 (v.1.12.0)

```
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kubectl"
```

쿠버네티스 바이너리 설치

```
{
  chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
  sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
}
```

### 2-2. 쿠버네티스 API Server 구성하기

```
{
  sudo mkdir -p /var/lib/kubernetes/

  sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem \
    encryption-config.yaml /var/lib/kubernetes/
}
```

인스턴스 내부 IP 주소는 API 서버를 클러스터 멤버에게 알리는데 사용됩니다. 인스턴스의 내부 IP 주소를 가져옵니다.

```
INTERNAL_IP=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip)
```

위에서 확인된 마스터 노드 0, 1의 ip를 컨트롤러IP 변수에 각각 넣어줍니다. (예)
```
CONTROLLER0_IP=10.240.0.10
CONTROLLER1_IP=10.240.0.11
```

`kube-apiserver.service` systemd 파일 생성

```sh
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=2 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=Initializers,NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://$CONTROLLER0_IP:2379,https://$CONTROLLER1_IP:2379 \\
  --event-ttl=1h \\
  --experimental-encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config=api/all \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2 \\
  --kubelet-preferred-address-types=InternalIP,InternalDNS,Hostname,ExternalIP,ExternalDNS
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

> 1.13 버전부터는 encryption-provider-config 로 변경

> --kubelet-preferred-address-types=InternalIP,InternalDNS,Hostname,ExternalIP,ExternalDNS 추가

> 명 check : endpoint-reconciler-type=master-count 가 빠져있음

### 2-3. 쿠버네티스 Controller Manager 구성하기

```
sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

`kube-controller-manager.service` systemd 파일 생성

```sh
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 2-4. 쿠버네티스 Scheduler 구성하기

```
sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

`kube-scheduler.yaml` yaml 파일 생성

```yaml
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: componentconfig/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

`kube-scheduler.service` systemd 파일 생성

```
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 2-5. 마스터 노드 서비스 시작

```
{
  sudo systemctl daemon-reload
  sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
  sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
}
```

### 2-6. HTTP 헬스 체크 활성화

로드 밸런서는 두 개의 API 서버를 통해 트래픽을 분배하고 각 API 서버가 TLS 연결을 종료하고 클라이언트 인증서의 유효성을 검증하는 데 사용됩니다. 네트워크 로드 밸런서는 HTTP 헬스 체크만 지원합니다. 즉, API 서버에 의해 노출된 HTTPS 엔드포인트는 사용할 수 없습니다.

이 문제를 해결하기 위해, nginx 웹서버를 사용하여 HTTP 헬스 체크를 프록시 처리 할 수 ​​있습니다.
이 섹션에서는 nginx가 설치되어 `80` 포트에서 HTTP 헬스 체크를 허용하고 `https://127.0.0.1:6443/healthz`에 대한 API 서버 연결을 프록시합니다.

> `/healthz` API 서버 엔드포인트는 기본적으로 인증을 필요로 하지 않습니다.

HTTP 상태 검사를 처리 할 기본 웹서버 설치

```sh
sudo apt-get install -y nginx
```

```sh
cat > kubernetes.default.svc.cluster.local <<EOF
server {
  listen      80;
  server_name kubernetes.default.svc.cluster.local;

  location /healthz {
     proxy_pass                    https://127.0.0.1:6443/healthz;
     proxy_ssl_trusted_certificate /var/lib/kubernetes/ca.pem;
  }
}
EOF
```

```sh
{
  sudo mv kubernetes.default.svc.cluster.local \
    /etc/nginx/sites-available/kubernetes.default.svc.cluster.local

  sudo ln -s /etc/nginx/sites-available/kubernetes.default.svc.cluster.local /etc/nginx/sites-enabled/
}
```

```sh
sudo systemctl restart nginx
sudo systemctl enable nginx
```

### 2-7. 확인

```
kubectl get componentstatuses --kubeconfig admin.kubeconfig
```

```
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
```

nginx HTTP 헬스 체크 프록시 테스트

```
curl -H "Host: kubernetes.default.svc.cluster.local" -i http://127.0.0.1/healthz
```

```
HTTP/1.1 200 OK
Server: nginx/1.14.0 (Ubuntu)
Date: Sun, 30 Sep 2018 17:44:24 GMT
Content-Type: text/plain; charset=utf-8
Content-Length: 2
Connection: keep-alive

ok
```

## 2-8. Kubelet 인증을 위한 RBAC

이 섹션에서는 쿠버네티스 API 서버가 각 워커 노드의 Kubelet API에 액세스 할 수 있도록 RBAC 권한을 구성합니다.

파드(Pod)에서 메트릭, 로그 및 실행 명령을 검색하려면 Kubelet API에 액세스해야합니다.

> 이 가이드는 Kubelet `--authorization-mode` 플래그를 `Webhook`으로 설정합니다. Webhook 모드는 [SubjectAccessReview](https://kubernetes.io/docs/admin/authorization/#checking-api-access) API를 사용하여 접근 허가를 결정합니다.

```
gcloud compute ssh controller-0
```

Kubelet API에 접근 권한이 있는 `system:kube-apiserver-to-kubelet` [ClusterRole](https://kubernetes.io/docs/admin/authorization/rbac/#role-and-clusterrole) 을 만듭니다.

```yaml
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF
```

쿠버네티스 API 서버는 `--kubelet-client-certificate` 플래그로 정의된 클라이언트 인증서를 사용하여 `kubernetes` 사용자로 Kubelet에 인증합니다.

`system:kube-apiserver-to-kubelet` ClusterRole을 `kubernetes` 사용자에게 바인드

```yaml
cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

## 3. 쿠버네티스 프론트엔드 로드밸런서

이 섹션에서는 쿠버네티스 API 서버를 사용하기 위해 외부로드 밸런서를 프로비저닝합니다. 앞에서 생성한 고정IP는 이 로드밸런서 역할을 하는 인스턴스에 연결됩니다.

- load-balancer 인스턴스 로그인
```
gcloud compute ssh load-balancer
```

### 3-1. nginx 설치

```sh
{
  sudo apt-get install -y nginx
  sudo systemctl enable nginx
}
```

### 3-2. 두 마스터 노드에서 쿠버네티스 API 트래픽 밸런싱 구성

`nginx.conf` 편집
```sh
sudo mkdir -p /etc/nginx/tcpconf.d
sudo vi /etc/nginx/nginx.conf
```

마지막 줄에 추가
```
include /etc/nginx/tcpconf.d/*;
```

```
CONTROLLER0_INTERNAL_IP=10.240.0.10
CONTROLLER1_INTERNAL_IP=10.240.0.11
```

구성 파일을 만들어 쿠버네티스 API 로드밸런싱 구성
```sh
cat << EOF | sudo tee /etc/nginx/tcpconf.d/kubernetes.conf
stream {
    upstream kubernetes {
        server ${CONTROLLER0_INTERNAL_IP}:6443;
        server ${CONTROLLER1_INTERNAL_IP}:6443;
    }

    server {
        listen 6443;
        listen 443;
        proxy_pass kubernetes;
    }
}
EOF
```

nginx 설정 리로드
```sh
sudo nginx -s reload
```



### 4. 확인

`kubernetes-the-hard-way` 고정IP 주소를 확인

```sh
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')
```

쿠버네티스 버전 정보에 대한 HTTP 요청

```sh
curl --cacert ca.pem https://${KUBERNETES_PUBLIC_ADDRESS}:6443/version
```

> 출력

```json
{
  "major": "1",
  "minor": "12",
  "gitVersion": "v1.12.0",
  "gitCommit": "0ed33881dc4355495f623c6f22e7dd0b7632b7c0",
  "gitTreeState": "clean",
  "buildDate": "2018-09-27T16:55:41Z",
  "goVersion": "go1.10.4",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```
