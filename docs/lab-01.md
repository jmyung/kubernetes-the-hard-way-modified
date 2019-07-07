# LAB-01. CA 프로비저닝 및 TLS 인증서 생성

- 이 랩에서는 CloudFlare의 PKI 툴킷인 cfssl을 사용하여 PKI 인프라를 구축한 다음
- 이를 사용하여 CA(인증 기관)을 부트스트랩하고,
- etcd, kube-apiserver, kube-controller-manager, kube-scheduler, kubelet, kube-proxy 구성 요소에 대한 TLS 인증서를 생성합니다.

## 1. CFSSL 설치

```sh
{
  wget -q --show-progress --https-only --timestamping \
  https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 \
  https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
  chmod +x cfssl_linux-amd64 cfssljson_linux-amd64
  sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl
  sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
}
```

## 2. CA 프로비저닝

```sh
{

cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

}
```

결과
```
ca-key.pem
ca.pem
```

## 3. 클라이언트 및 서버 인증서 생성

### 3-1. admin 클라이언트 인증서 및 개인키 생성

```sh
{

cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin

}
```

결과
```
admin-key.pem
admin.pem
```

### 3-2. Kubelet 클라이언트 인증서 및 개인키 생성

- Kubernetes는 `Node Authorizer`라고하는 특수-목적의 인증 모드를 사용합니다.
- 이 모드에서는 Kubelet에서 만든 API 요청을 승인합니다.
- Node Authorizer에 의해 권한을 부여 받으려면, Kubelet이 `system:node:<nodeName>`이라는 username으로 `system:nodes` 그룹에 있는 것으로 식별하는 credential을 사용해야합니다.
- 이 섹션에서는 Node Authorizer 요구 사항을 충족하는 각 Kubernetes 워커 노드에 대한 인증서를 만듭니다.

```sh
for instance in worker-0 worker-1; do
cat > ${instance}-csr.json <<EOF
{
  "CN": "system:node:${instance}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

EXTERNAL_IP=$(gcloud compute instances describe ${instance} \
  --format 'value(networkInterfaces[0].accessConfigs[0].natIP)')

INTERNAL_IP=$(gcloud compute instances describe ${instance} \
  --format 'value(networkInterfaces[0].networkIP)')

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${EXTERNAL_IP},${INTERNAL_IP} \
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}
done
```

결과
```
worker-0-key.pem
worker-0.pem
worker-1-key.pem
worker-1.pem
```

### 3-3. Controller Manager 클라이언트 인증서

`kube-controller-manager` 클라이언트 인증서 및 개인 키 생성

```sh
{

cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

}
```

결과
```
kube-controller-manager-key.pem
kube-controller-manager.pem
```

### 3-4. Kube Proxy 클라이언트 인증서

`kube-proxy` 클라이언트 인증서 및 개인 키 생성

```sh
{

cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy

}
```

결과
```
kube-proxy-key.pem
kube-proxy.pem
```

### 3-5. Scheduler 클라이언트 인증서

`kube-scheduler` 클라이언트 인증서 및 개인 키 생성

```sh
{

cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

}
```

결과
```
kube-scheduler-key.pem
kube-scheduler.pem
```

### 3-6. Kubernetes API 서버 인증서

kubernetes-the-hard-way 정적 IP 주소는 Kubernetes API 서버 인증서의 hostname에 포함됩니다. 이렇게 하면 원격 클라이언트가 인증서를 확인할 수 있습니다.

`Kubernetes API` 서버 인증서 및 개인 키 생성

```sh
{

KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,10.240.0.10,10.240.0.11,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,kubernetes.default \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

}
```

결과
```
kubernetes-key.pem
kubernetes.pem
```

### 3-7. Service Account 키 페어

Kubernetes Controller Manager는 [managing service accounts](https://kubernetes.io/docs/admin/service-accounts-admin/) 문서에 설명된대로 키 페어를 사용하여 서비스 계정 토큰을 생성하고 서명합니다.

`service-account` 인증서 및 개인 키 생성

```sh
{

cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account

}
```

결과
```
service-account-key.pem
service-account.pem
```

## 4. 클라이언트 및 서버 인증서 배포

워커 노드에 배포
```sh
for instance in worker-0 worker-1; do
  gcloud compute scp ca.pem ${instance}-key.pem ${instance}.pem ${instance}:~/
done
```

마스터 노드에 배포
```sh
for instance in controller-0 controller-1; do
  gcloud compute scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem ${instance}:~/
done
```

> 다음 랩에서는 `kube-proxy`, `kube-controller-manager`, `kube-scheduler`, `kubelet` 클라이언트 인증서를 사용하여 클라이언트 인증 설정 파일을 생성합니다.
