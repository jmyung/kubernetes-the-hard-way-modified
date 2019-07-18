# kubernetes-the-hard-way-modified

본 가이드는 [켈시 하이타워](https://github.com/kelseyhightower)의 [쿠버네티스 하드웨이](https://github.com/kelseyhightower/kubernetes-the-hard-way)를 각색하여 제작했습니다.


## 아키텍쳐

![architecture](./docs/kthw.png "architecture")


## 순서

- [사전준비](./docs/prerequisites.md)


0. [시작하기 전에](./docs/lab-00.md)
1. [CA 프로비저닝 및 TLS 인증서 생성](./docs/lab-01.md)
2. [인증을 위한 Kubernetes config 파일 생성](./docs/lab-02.md)
3. [데이터 암호화 구성 및 키 생성](./docs/lab-03.md)
4. [etcd 클러스터 부트스트래핑](./docs/lab-04.md)
5. [Kubernetes Control Plane 부트스트래핑](./docs/lab-05.md)
6. [Kubernetes Worker Nodes 부트스트래핑](./docs/lab-06.md)
7. [원격 액세스를 위한 kubectl 구성](./docs/lab-07.md)
8. [네트워킹](./docs/lab-08.md)
9. [DNS 클러스터 추가 기능 배포](./docs/lab-09.md)
10. [스모크 테스트](./docs/lab-10.md)


## 켈시 하이타워 가이드 와의 차이점
- 마스터, 워커를 2개씩만 띄움 (실습 목적)
- `containerd` 대신 `docker` 사용
- 로드밸런서를 GCP 서비스를 사용하지 않고 `nginx로 직접 구성`
- CNI 플러그인 (Pod network add-on) : `weave-net` 사용
