# LAB-02. 인증을 위한 Kubernetes config 파일 생성

In this lab you will generate Kubernetes configuration files, also known as kubeconfigs, which enable Kubernetes clients to locate and authenticate to the Kubernetes API Servers.

- 이 랩에서는 `kubeconfig`이라 불리는, [Kubernetes 설정 파일](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/)을 생성합니다.
- kubeconfig는 Kubernetes 클라이언트가 Kubernetes API 서버를 찾고 인증 할 수 있도록 해줍니다.

## 1. 클라이언트 인증 구성

이 섹션에서는 이하의 kubeconfig 파일을 생성합니다.
- `controller manager`
- `kubelet`
- `kube-proxy`
- `scheduler` 클라이언트
- `admin` 사용자

### 1-2. 정적 Public IP 조회

```sh
KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region) \
  --format 'value(address)')
```

### 1-3. kubelet kubeconfig 생성

Kubelet kubeconfig 파일을 생성할 때 Kubelet의 노드 이름과 일치하는 클라이언트 인증서를 사용해야합니다. 이렇게 하면 Kuubenetes Node Authorizer가 Kubelet을 올바르게 인증할 수 있습니다.

각 워커 노드에 대해 kubeconfig 파일 생성

```sh
for instance in worker-0 worker-1; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```

결과
```
worker-0.kubeconfig
worker-1.kubeconfig
```

### 1-4. kube-proxy kubeconfig 생성

`kube-proxy` 서비스에 대해 kubeconfig 파일 생성

```sh
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.pem \
    --client-key=kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
}
```

결과
```
kube-proxy.kubeconfig
```

### 1-5. kube-controller-manager kubeconfig 생성

`kube-controller-manager` 서비스에 대해 kubeconfig 파일 생성

```sh
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
}
```

결과
```
kube-controller-manager.kubeconfig
```

### 1-6. kube-scheduler kubeconfig 생성

`kube-scheduler` 서비스에 대해 kubeconfig 파일 생성

```sh
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
}
```

결과
```
kube-scheduler.kubeconfig
```

### 1-7. admin kubeconfig 생성

`admin` 사용자에 대해 kubeconfig 파일 생성

```sh
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default --kubeconfig=admin.kubeconfig
}
```

결과
```
admin.kubeconfig
```

## kubeconfig 파일 배포

`kubelet` 및 `kube-proxy` kubeconfig 파일을 각 워커 노드에 복사합니다.

```sh
for instance in worker-0 worker-1; do
  gcloud compute scp ${instance}.kubeconfig kube-proxy.kubeconfig ${instance}:~/
done
```

`kubelet` 및 `kube-proxy` kubeconfig 파일을 각 마스터 노드에 복사합니다.

```sh
for instance in controller-0 controller-1; do
  gcloud compute scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ${instance}:~/
done
```
