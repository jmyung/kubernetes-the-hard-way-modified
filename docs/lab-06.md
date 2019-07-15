# LAB-06. Kubernetes Worker Nodes 부트스트래핑

- 워커 노드 (Worker Nodes) : 쿠버네티스에서 관리하는 컨테이너 어플리케이션의 실제 작업을 담당합니다.
- 이 랩에서는 두 개의 워커 노드를 부트스트랩 합니다.
- 컨트롤 플레인 컴포넌트 (워커 노드)
  - **[kubelet](https://kubernetes.io/docs/admin/kubelet)** : 워커 노드에서 실행되는 에이전트. Kubelet은 파드 스펙(PodSpec)을 받아, 파드에서 컨테이너가 동작하도록 관리합니다.
  - **[kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies)** : 워커 노드에서 실행되는 네트워크 프록시로서, 호스트의 네트워크 규칙(iptables)을 관리하고 요청에 대한 포워딩을 책임집니다. (NodePort로 들어온 트래픽을 클러스터 내의 적절한 파드로 리다이렉팅)
  - **컨테이너 런타임** : 컨테이너 실행을 담당하는 소프트웨어


- 다음 구성 요소가 각 노드에 설치됩니다.
  - kubelet
  - kube-proxy
  - docker
  - ~~[runc](https://github.com/opencontainers/runc)~~
  - ~~[containerd](https://github.com/containerd/containerd)~~
  - ~~[container networking plugins](https://github.com/containernetworking/cni)~~
  - [gVisor](https://github.com/google/gvisor)

- 아키텍쳐
  ![architecture](kthw.png "architecture")


## 1. 준비 사항

이 랩에서 명령은 워커 노드인 `worker-0`,`worker-1` 모두 에서 실행되어야 합니다.

GCP 인스턴스 ssh 로그인
```
gcloud compute ssh worker-0
```

## 2. 쿠버네티스 워커 노드 프로비저닝

OS dependencies 설치

```sh
{
  sudo apt-get update
  sudo apt-get -y install socat conntrack ipset
}
```

> socat 바이너리는 `kubectl port-forward` 명령을 지원합니다.


### 2-1. 쿠버네티스 워커 바이너리 다운로드 및 설치

```sh
wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kubelet
```


설치 디렉토리 생성

```sh
sudo mkdir -p \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

워커 바이너리 설치

```sh
{
  chmod +x kubectl kube-proxy kubelet
  sudo mv kubectl kube-proxy kubelet /usr/local/bin
}
```

도커 설치
```sh
sudo apt install docker.io -y
```

### 2-2. Kubelet 구성

```sh
{
  sudo mv ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
  sudo mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
  sudo mv ca.pem /var/lib/kubernetes/
}
```

`kubelet-config.yaml` 설정 파일 생성

```yaml
cat << EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${HOSTNAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${HOSTNAME}-key.pem"
EOF
```

`kubelet.service` systemd 파일 생성

```sh
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=docker \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

> kubelet이 hostname을 제대로 못가져 오는 경우, --hostname-override=${HOSTNAME}, --allow-privileged=true 를 사용해야할 수 도 있습니다.

### 2-3. 쿠버네티스 Proxy 구성

```
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

`kube-proxy-config.yaml` 설정 파일 생성

```yaml
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF
```

`kube-proxy.service` systemd 파일 생성

```
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 2-4. 워커 서비스 시작

```
{
  sudo systemctl daemon-reload
  sudo systemctl enable kubelet kube-proxy
  sudo systemctl start kubelet kube-proxy
}
```

> 각 워커 노드에서 위의 명령들을 실행해야합니다 : `worker-0`,`worker-1`

## 3. 확인

> 워커 노드 인스턴스에서는 다음 명령을 수행할 수 있는 권한이 없습니다. 마스터 노드에서 다음 명령을 실행합니다.

등록된 Kubernetes 노드를 조회

```
gcloud compute ssh controller-0 \
  --command "kubectl get nodes --kubeconfig admin.kubeconfig"
```

> 출력

```
NAME       STATUS     ROLES    AGE    VERSION
worker-0   NotReady   <none>   103s   v1.12.0
worker-1   NotReady   <none>   103s   v1.12.0
```

- 네트워크 설정이 아직 남아있기 때문에, `NotReady` 상태입니다.
