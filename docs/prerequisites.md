# 사전 준비

## 1. GCP 가입

- 사이트 가입 : https://console.cloud.google.com/
- 필요 사항 : 해외 결제 가능한 신용 카드 등록
- 본 가이드는 1년간 무료로 주어지는 $300 크래딧 내에서 진행합니다.
- AWS에서 VM을 띄우셔도 상관없습니다. 단 방화벽 작업은 이하 내용을 참고하여 해제해주세요.


## 2. VM 띄우기

### 2-1. 마스터용 VM

```
for i in 0 1; do
  gcloud compute instances create controller-${i} \
    --async \
    --boot-disk-size 10GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --private-network-ip 10.240.0.1${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,controller
done
```
