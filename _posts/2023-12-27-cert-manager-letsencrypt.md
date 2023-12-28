---
layout: posts
title:  "Let's Encrypt SSL with cert-manager"
date:   2023-12-27 00:00:00 +0900
categories: blog
---
# 1. 개요

[공식 사이트](https://cert-manager.io/docs/)

cert-manager를 사용해 Let's Encrypt 인증서 발급 및 갱신을 갱신자동화 하는 방법에 대해 기술한다.

# 2. 환경
- Domain : xodwk.kr - gabia.com을 통해 등록
- Kubernetes : RKE2 cluster - 공유기 하단에 서버 구성

# 3. cert-manager 
cert-manager는 certificates와 certificate issuers라는 Kubernetes의 CRD(custom resource definition)을 통해 인증서 발급, 갱신, 사용을 자동화한다.

## 3-1. Helm으로 설치

[공식 사이트](https://cert-manager.io/docs/installation/helm/)

기타 다른 방법으로의 설치는 공식 사이트의 다른 설치 방법을 참고한다.

cert-manager는 Helm Chart를 제공하고 있는데 다른 Helm Chart의 sub-chart로 만들어서 2번 설치되지 않도록 주의한다. 무슨 일이 발생할지 이해하고 대비할 수 없다면.

- Helm repo 추가
    ```bash
    $ helm repo add jetstack https://charts.jetstack.io
    ```
- repo update 
    ```bash
    $ helm repo update
    ```
- CRDs 설치

    CRDs를 설치할 때 주의 할 건 kubectl apply 를 통해 따로 설치하는 것이 안전하다는 것이다. helm install에서 옵션으로 같이 설치할 수는 있지만 이렇게 하게되면 해당 릴리즈를 삭제하거나 할 때 만들어져 있던 CRDs가 같이 삭제될 수 있다. 다시 말해 발급받았던 인증서들도 같이 날아갈 수 있다는 것이다. 백업은 당연히 해야하는 것이지만 예기치 않게 인증서가 삭제되는 것을 방지하려면 CRDs는 따로 설치한다. 자세한 내용은 [여기](https://cert-manager.io/docs/installation/helm/#crd-considerations)를 참조한다.

    ```bash
    $ kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.crds.yaml
    ```
- cert-manager 설치
    ```bash
    $ helm install \
    cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --create-namespace \
    --version v1.13.3 \
    ```
- 확인
    ```bash
    $ kubectl get pods --namespace cert-manager
    NAME                                       READY   STATUS    RESTARTS   AGE
    cert-manager-5c6866597-zw7kh               1/1     Running   0          2m
    cert-manager-cainjector-577f6d9fd7-tr77l   1/1     Running   0          2m
    cert-manager-webhook-787858fcdb-nlzsq      1/1     Running   0          2m
    ```
    cert-manager를 설치한 네임스페이스에서 해당 파드들이 보인다면 정상적으로 설치가 된 것이다. webhook도 잘 동작하는지 확인하기 위해서는 아래 명령을 실행했을 때 다음과 같은 결과가 나온다면 webhook도 정상적으로 동작한다고 할 수 있다.
    ```bash
    $ kubectl apply -f - <<EOF
    apiVersion: v1
    kind: Namespace
    metadata:
    name: cert-manager-test
    ---
    apiVersion: cert-manager.io/v1
    kind: Issuer
    metadata:
    name: test-selfsigned
    namespace: cert-manager-test
    spec:
    selfSigned: {}
    ---
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
    name: selfsigned-cert
    namespace: cert-manager-test
    spec:
    dnsNames:
        - example.com
    secretName: selfsigned-cert-tls
    issuerRef:
        name: test-selfsigned
    EOF
    ```
    ```bash
    $ kubectl describe certificate -n cert-manager-test
    ...
    Spec:
    Common Name:  example.com
    Issuer Ref:
        Name:       test-selfsigned
    Secret Name:  selfsigned-cert-tls
    Status:
    Conditions:
        Last Transition Time:  2019-01-29T17:34:30Z
        Message:               Certificate is up to date and has not expired
        Reason:                Ready
        Status:                True
        Type:                  Ready
    Not After:               2019-04-29T17:34:29Z
    Events:
    Type    Reason      Age   From          Message
    ----    ------      ----  ----          -------
    Normal  CertIssued  4s    cert-manager  Certificate issued successfully
    ```
    확인이 끝났으면 삭제한다.
    ```bash
    $ kubectl delete -f test-resources.yaml
    ```

## 3-2. Issuer 설정
먼저 Issuer를 설정한다. Issuer는 ACME로 설정할 건데 간당하게 설명하자면 인증서를 발급할 때는 challenge라는 것을 통해 해당 도메인에 대한 소유권을 확인하고 인증서를 발급 해준다. 크게 2가지 방법으로 HTTP01과 DNS01이 있다. Let's Encrypt만 그런 건지는 모르겠지만 HTTP01은 wildcard 인증서를 발급받을 수 없다. 다시 말해 특정 subdomain의 인증서만을 발급받을 수 있다. wildcard 인증서를 받아야겠다면 DNS01 방식의 challenge를 사용해야 한다. 문제는 DNS01 challenge를 풀려면 acme-challenge라는 서브 도메인에 Let's Encrypt에서 발행한 문자열을 넣는 방법으로 해당 도메인의 소유를 증명해야 하는데 cert-manager에서 공식적으로 지원하는 DNS01 provider에 gabia는 없다는 것이다. webhook으로 만드는 방법도 있는 것 같은데 dns를 수정하는 API를 gabia에서 제공하는지도 모르겠고 어차피 자동화 시켜놓는다면 서브도메인당 한 번만 해주면 되는 거니까 그냥 HTTP01 방식으로 자동화 하는 것을 추천한다.

그리고 ~~와일드카드 못 써서 하는 소리는 아니고~~ 보안을 생각한다면 wildcard 인증서가 마냥 좋은 것 만은 아니다. 

Issuer는 그냥 Issuer와 clusterIssuer가 있는데 그냥 Issuer는 발급한 namespace 안에서만 사용할 수 있고 clusterIssuer는 해당 cluster 내 모든 namespace에서 사용할 수 있다는 차이점이 있다고만 읽었고 확인은 못해봤다. clusterIssuer만 만들어서 모든 namespace에서 사용할 수 있다는 것만 확인해봤다.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-acme-issuer
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    # rate limit 걸리면 한동안 인증서 발급이 안 된다. staging으로 확인하고 아래 실제 prod를 사용한다.
    # server: https://acme-v02.api.letsencrypt.org/directory
    email: # 이메일 주소 입력
    solvers:
    - http01:
        ingress:
          class: nginx # 라고 넣었는데 아마도 optional인듯
```
```bash
$ kubectl apply -f [clusterIssuer].yaml
```
## 3-3. Certificate 설정
마지막으로 Certificate만 만들어 주면 설정은 끝난다. 
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: rke2-test-cert
  namespace: test
spec:
  secretName: rke2-test-tls #이 이름으로 secret이 생성된다.
  # commonName: xodwk.kr # deprecated라고 해서 삭제함.
  dnsNames:
    - nginx.test.xodwk.kr
    # - nginx2.test.xodwk.kr
  issuerRef:
    name: letsencrypt-acme-issuer
    kind: ClusterIssuer
    group: cert-manager.io
```
```bash
$ kubectl apply -f [certificate].yaml
```
## 3-4. 확인
설정이 마무리 되면 아래와 같은 CRDs를 확인할 수 있다.
```bash
$ kubectl get clusterissuers.cert-manager.io 
NAME                      READY   AGE
letsencrypt-acme-issuer   True    5h

$ kubectl get certificates.cert-manager.io -n test 
NAME             READY   SECRET          AGE
rke2-test-cert   True    rke2-test-tls   5h
```
이후 cert-manager는 해당 secret이 생성되는 namespace에 새로운 ingress를 만드는데 해당 ingress에서는 [이곳](https://letsencrypt.org/docs/challenge-types/)에서 말하는 바와 같이 http://<YOUR_DOMAIN>/.well-known/acme-challenge/<TOKEN> 이 주소에 token을 담아서 노출하고 확인이 되면 위 secretName에 설정된 이름의 tls 타입 secret이 설정된다.

```bash
$ kubectl get secrets -n test                      
NAME                                TYPE                 DATA   AGE
rke2-test-tls                       kubernetes.io/tls    2      10h
```

# 4. 마무리
이렇게 만들어진 secret을 ingress에 적용한다.

만들면서 대부분의 옵션을 생략했기 때문에 default로 90일 짜리 인증서가 발급이 되었다.

만료 15일 전에 cert-manager가 자동 갱신 할 것이다. 

90일 이후에 잘 갱신되는지 확인 후 다시 기록한다.

```bash

```
