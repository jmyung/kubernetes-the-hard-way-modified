# LAB-07. 원격 액세스를 위한 kubectl 구성

이 랩에서는 `admin` 사용자 자격증명에 기반하여, `kubectl` 명령행 유틸리티를 위한 kubeconfig 파일을 생성합니다.

> 이 랩은 인증서를 생성했던 디렉토리에서 실행합니다.

## 1. Admin 쿠버네티스 구성 파일 생성

각 kubeconfig에는 쿠버네티스 API 서버가 연결되어 있어야 합니다. 고가용성을 지원하기 위해 로드밸런서에 할당된 IP 주소가 사용됩니다.

`admin` 사용자용 컨텍스트 생성

```sh
{
  KUBERNETES_PUBLIC_ADDRESS=$(gcloud compute addresses describe kubernetes-the-hard-way \
    --region $(gcloud config get-value compute/region) \
    --format 'value(address)')

  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem

  kubectl config set-context kubernetes-the-hard-way \
    --cluster=kubernetes-the-hard-way \
    --user=admin

  kubectl config use-context kubernetes-the-hard-way
}
```

## 2. 확인

원격 쿠버네티스 클러스터의 상태 확인

```sh
kubectl get componentstatuses
```

> 출력(예)

```sh
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health":"true"}
etcd-1               Healthy   {"health":"true"}
```

원격 쿠버네티스 클러스터의 노드 목록 조회

```sh
kubectl get nodes
```

> 출력(예)

```sh
NAME       STATUS     ROLES    AGE     VERSION
worker-0   NotReady   <none>   9m42s   v1.12.0
worker-1   NotReady   <none>   9m42s   v1.12.0
```
