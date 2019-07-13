# LAB-04. etcd 클러스터 부트스트래핑

쿠버네티스 컴포넌트는 stateless하고 [etcd](https://github.com/coreos/etcd)에 클러스터 상태를 저장합니다.
이 랩에서는 두 노드의 etcd 클러스터를 부트스트랩 합니다.

- etcd
 - 기능 : 분산 시스템의 중요 데이터를 저장하기 위한 신뢰할 수 있는 key-value 저장소입니다.
 - 역할 : 클러스터 내 분산된 머신에 데이터를 저장하고, 모든 머신에서 데이터가 동기화되는지 확인하는 방법을 제공합니다.

- 쿠버네티스 내에서 etcd 역할
  - 쿠버네티스는 etcd를 사용하여 클러스터 상태에 대한 모든 내부 데이터를 저장합니다.
  - 저장된 데이터는 클러스터의 모든 마스터 노드에서 안정적으로 동기화되어야 합니다.

## 1. 마스터 노드 접속

이 랩의 명령은 각 마스터 노드에서 실행해야합니다. : `controller-0`, `controller-1`

```sh
gcloud compute ssh controller-0
```

## 2. etcd 클러스터 멤버를 부트스트랩하기

### 2-1. etcd 다운로드 및 설치

```sh
wget -q --show-progress --https-only --timestamping \
  "https://github.com/coreos/etcd/releases/download/v3.3.9/etcd-v3.3.9-linux-amd64.tar.gz"
```

`etcd` 서버와 `etcdctl` 커맨드 라인 유틸리티 설치

```
{
  tar -xvf etcd-v3.3.9-linux-amd64.tar.gz
  sudo mv etcd-v3.3.9-linux-amd64/etcd* /usr/local/bin/
}
```

### 2-2. `etcd` 서버 구성

```
{
  sudo mkdir -p /etc/etcd /var/lib/etcd
  sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
}
```

VM 인스턴스 내부 IP 주소는 클라이언트 요청을 처리하고 etcd 클러스터 peer와 통신하는데 사용됩니다.

VM 인스턴스 내부 IP 주소 조회
```
INTERNAL_IP=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/network-interfaces/0/ip)
```

> (AWS 인 경우) INTERNAL_IP=$(curl http://169.254.169.254/latest/meta-data/local-ipv4)


각 etcd 멤버는 etcd 클러스터 내에서 고유한 이름을 가져야합니다. 현재 VM의 호스트 이름과 일치하도록 etcd 이름을 설정합니다.

```
ETCD_NAME=$(hostname -s)
```

INITIAL_CLUSTER에 etcd 클러스터의 모든 서버 리스트를 설정합니다.

```sh
INITIAL_CLUSTER=controller-0=https://10.240.0.10:2380,controller-1=https://10.240.0.11:2380
```

`etcd.service` systemd 유닛 파일 생성

```
cat <<EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster ${INITIAL_CLUSTER} \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### 2-3. etcd 서버 시작

```sh
{
  sudo systemctl daemon-reload
  sudo systemctl enable etcd
  sudo systemctl start etcd
}
```

> 각 마스터 노드에서 위의 명령을 실행합니다 : `controller-0`,`controller-1`

## 3. 확인

etcd 클러스터 멤버 조회

```
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

> 출력 예

```
3a57933972cb5131, started, controller-0, https://10.240.0.12:2380, https://10.240.0.12:2379
f98dc20bce6225a0, started, controller-1, https://10.240.0.10:2380, https://10.240.0.10:2379
```
