# 사전 준비

- 실습 간 슬랙을 활용합니다.
  - 가입링크 : http://34.94.33.120/ 를 클릭하여 이메일을 기재하시면 초대메일이 발송됩니다.
  - 슬랙 주소 : https://peanut-butter-group.slack.com
  - Channels > `#k8s-the-hard-way` 채널 조인
- 참석 전에 반드시, 쿠버네티스 클러스터를 설치할 VM을 미리 구성해주시기 바랍니다.
  - 이하 절차 수행
  - 문의 사항은 슬랙에 올려주세요.

## 1. GCP (Google Cloud Platform) 가입

- 사이트 가입 : https://console.cloud.google.com/
- 필요 사항 : 해외 결제 가능한 신용 카드 등록
- 본 가이드는 1년간 무료로 주어지는 $300 크래딧 내에서 진행합니다.
- AWS에서 VM을 띄우셔도 상관없습니다. 단 [방화벽 작업](https://github.com/jmyung/kubernetes-the-hard-way-modified/blob/master/docs/prerequisites.md#vpc-%EC%83%9D%EC%84%B1-%EC%84%9C%EB%B8%8C%EB%84%B7-%EC%83%9D%EC%84%B1-%EB%B0%A9%ED%99%94%EB%B2%BD-%ED%95%B4%EC%A0%9C)은 이하 내용을 참고하여 해제해주세요.


## 2. VM 띄우기

### 2-1. 프로젝트 생성

- Select a project > New Project
- `Project name` 기입 후 `CREATE` 버튼 클릭
- `Cloud Shell` 버튼 클릭하여 이하 진행 (https://cloud.google.com/shell/)
- 프로젝트ID 확인 (기입한 프로젝트명과 다를 수 있음)

### 2-2. 리전 설정 및 GCP API 활성화

- 프로젝트ID 변수화
```sh
PROJECT_ID="프로젝트ID"
```

- 리전 설정 및 GCP API 활성화
```sh
{
  gcloud config set project ${PROJECT_ID}
  gcloud config set compute/region us-west2
  gcloud config set compute/zone us-west2-c
  gcloud services enable compute.googleapis.com
}
```

- config-update.sh
```sh
mkdir setup
cd setup
vi config-update.sh
```

```sh
PROJECT_ID="프로젝트ID"
gcloud config set project ${PROJECT_ID}
gcloud config set compute/region us-west2
gcloud config set compute/zone us-west2-c
```
저장후
```
chmod +x config-update.sh
```

### 2-3. VM 생성

#### 2-3-1. 네트워크 생성

##### VPC 생성, 서브넷 생성, 방화벽 해제
```sh
{
  gcloud compute networks create kubernetes-the-hard-way --subnet-mode custom
  gcloud compute networks subnets create kubernetes \
    --network kubernetes-the-hard-way \
    --range 10.240.0.0/24
  gcloud compute firewall-rules create kubernetes-the-hard-way-allow-internal \
    --allow tcp,udp,icmp \
    --network kubernetes-the-hard-way \
    --source-ranges 10.240.0.0/24,10.200.0.0/16
  gcloud compute firewall-rules create kubernetes-the-hard-way-allow-external \
    --allow tcp:22,tcp:6443,icmp \
    --network kubernetes-the-hard-way \
    --source-ranges 0.0.0.0/0
}
```

##### 확인

```sh
gcloud compute firewall-rules list --filter="network:kubernetes-the-hard-way"
```



#### 2-3-2. 마스터 VM 생성

```sh
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

#### 2-3-3. 워커 VM 생성

```sh
for i in 0 1; do
  gcloud compute instances create worker-${i} \
    --async \
    --boot-disk-size 10GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --metadata pod-cidr=10.200.${i}.0/24 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet kubernetes \
    --tags kubernetes-the-hard-way,worker
done
```

#### 2-3-4. 로드밸런서 VM 생성
- VM 생성
```sh
gcloud compute instances create load-balancer \
  --async \
  --boot-disk-size 10GB \
  --can-ip-forward \
  --image-family ubuntu-1804-lts \
  --image-project ubuntu-os-cloud \
  --machine-type n1-standard-1 \
  --private-network-ip 10.240.0.30 \
  --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
  --subnet kubernetes \
  --tags kubernetes-the-hard-way,load-balancer
```
- 정적 Public IP 생성
```sh
gcloud compute addresses create kubernetes-the-hard-way \
  --region $(gcloud config get-value compute/region)
```
- IP 확인
```sh
gcloud compute addresses list --filter="name=('kubernetes-the-hard-way')"
```
- 정적 IP를 로드밸런서 VM에 붙이기
```sh
gcloud compute instances delete-access-config load-balancer --access-config-name "external-nat"
gcloud compute instances add-access-config load-balancer --access-config-name "external-nat" --address [바로위에 확인된 IP 기입]
```

#### 2-3-4. 확인
```sh
gcloud compute instances list
```

### 2-9. 자주쓰는 쉘 만들어놓기 (필수 아님)
- start.sh
```sh
for instance in controller-0 controller-1 load-balancer worker-0 worker-1 ; do
  gcloud compute instances start ${instance}
done
```

- stop.sh
```sh
for instance in controller-0 controller-1 load-balancer worker-0 worker-1 ; do
  gcloud compute instances stop ${instance}
done
```

- init.sh
```sh
gcloud config set project {프로젝트명}
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

- CLEAN UP
  - 방법1
```sh
for instance in controller-0 controller-1 load-balancer worker-0 worker-1 ; do
  gcloud compute instances delete ${instance}
done
```
  - 방법2 : 콘솔에서 프로젝트를 delete 하면 생성한 자원들이 모두 삭제됩니다.
