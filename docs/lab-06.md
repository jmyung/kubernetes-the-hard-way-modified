# LAB-06. Kubernetes Worker Nodes 부트스트래핑

- 이 랩에서는 두 개의 워커 노드를 부트스트랩 합니다.
- 다음 구성 요소가 각 노드에 설치됩니다.
  - [kubelet](https://kubernetes.io/docs/admin/kubelet)
  - [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies)
  - docker
  - ~~[runc](https://github.com/opencontainers/runc)~~
  - ~~[containerd](https://github.com/containerd/containerd)~~
  - ~~[container networking plugins](https://github.com/containernetworking/cni)~~
  - [gVisor](https://github.com/google/gvisor)


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
  https://github.com/kubernetes-incubator/cri-tools/releases/download/v1.0.0-beta.0/crictl-v1.0.0-beta.0-linux-amd64.tar.gz \
  https://github.com/containernetworking/plugins/releases/download/v0.6.0/cni-plugins-amd64-v0.6.0.tgz \
  https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.12.0/bin/linux/amd64/kubelet
```


설치 디렉토리 생성

```sh
sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
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
  sudo tar -xvf crictl-v1.0.0-beta.0-linux-amd64.tar.gz -C /usr/local/bin/
  sudo tar -xvf cni-plugins-amd64-v0.6.0.tgz -C /opt/cni/bin/
}
```

도커 설치
```sh
sudo apt install docker.io -y
```

### 2-2. Kubelet 구성

```
{
  sudo mv ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
  sudo mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
  sudo mv ca.pem /var/lib/kubernetes/
}
```

```
POD_CIDR=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/attributes/pod-cidr)
```

`kubelet-config.yaml` 설정 파일 생성

```yaml
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
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
podCIDR: "${POD_CIDR}"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${HOSTNAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${HOSTNAME}-key.pem"
EOF
```

> `resolvConf` 설정은 `systemd-resolved`를 실행하는 시스템에서 CoreDNS를 사용할 때 루프를 피하기 위해 사용됩니다.

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
