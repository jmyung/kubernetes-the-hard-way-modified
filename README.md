# kubernetes-the-hard-way-modified

본 가이드는 [켈시 하이타워](https://github.com/kelseyhightower)의 [쿠버네티스 하드웨이](https://github.com/kelseyhightower/kubernetes-the-hard-way)를 각색하여 제작했습니다.

## 순서

0. [사전준비](./docs/prerequisites.md)
1. [CA 프로비저닝 및 TLS 인증서 생성](./docs/lab-01.md)
2. [인증을 위한 Kubernetes config 파일 생성](./docs/lab-02.md)
3. 데이터 암호화 구성 및 키 생성
4. etcd 클러스터 부트스트래핑
5. Kubernetes Control Plane 부트스트래핑
6. Kubernetes Worker Nodes 부트스트래핑
7. 원격 액세스를 위한 kubectl 구성
8. 네트워킹
9. DNS 클러스터 추가 기능 배포
10. 스모크 테스트


## 켈시 하이타워 가이드 와의 차이점
- 마스터, 워커를 2개씩만 띄움
- `containerd` 대신 `docker` 사용
- 로드밸런서를 GCP서비스를 사용하지 않고 `직접 구성`
- CNI를 `weave-net` 사용
