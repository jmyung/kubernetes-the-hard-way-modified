# LAB-03. 데이터 암호화 구성 및 키 생성


Kubernetes는 클러스터 상태, 어플리케이션 구성 및 secret을 비롯한 다양한 데이터를 저장합니다. Kubernetes는 클러스터 데이터를 암호화하는 기능을 지원합니다.

이 랩에서는 Kubernetes Secrets 암호화에 적합한 암호화 키 및 암호화 구성을 생성합니다.

## 1. 암호화 키 생성

```sh
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

## 2. encryption-config.yaml 생성

```yaml
cat > encryption-config.yaml <<EOF
apiVersion: v1
kind: EncryptionConfig
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

encryption-config.yaml 파일을 각 마스터 노드에 복사합니다.

```sh
for instance in controller-0 controller-1; do
  gcloud compute scp encryption-config.yaml ${instance}:~/
done
```
