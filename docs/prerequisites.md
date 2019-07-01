# 사전 준비

## 1. GCP 가입

- 사이트 가입 : https://console.cloud.google.com/
- 필요 사항 : 해외 결제 가능한 신용 카드 등록
- 본 가이드는 1년간 무료로 주어지는 $300 크래딧 내에서 진행합니다.
- AWS에서 VM을 띄우셔도 상관없습니다. 단 방화벽 작업은 이하 내용을 참고하여 해제해주세요.


## 2. VM 띄우기

### 2-1. 프로젝트 생성

- Select a project > New Project
- Project name 기입 후 `CREATE` 버튼 클릭


### 2-2. 마스터용 VM

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


### 2-9. 자주쓰는 쉘 생성
- init.sh
```
gcloud config set project k-hard-way-1
gcloud config set compute/region us-west2
gcloud config set compute/zone us-west2-c
wget -q --show-progress --https-only --timestamping \
  https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 \
  https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x cfssl_linux-amd64 cfssljson_linux-amd64
sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl
sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
./start.sh
```

- region-update.sh
```
gcloud config set project k-hard-way-1
gcloud config set compute/region us-west2
gcloud config set compute/zone us-west2-c
```

- start.sh
```
for instance in controller-0 controller-1 load-balancer worker-0 worker-1 ; do
  gcloud compute instances start ${instance}
done
```

- stop.sh
```
for instance in controller-0 controller-1 load-balancer worker-0 worker-1 ; do
  gcloud compute instances stop ${instance}
done
```
