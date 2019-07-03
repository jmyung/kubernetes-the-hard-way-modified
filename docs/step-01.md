# STEP-01. CA 프로비저닝 및 TLS 인증서 생성

## CFSSL 설치

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
