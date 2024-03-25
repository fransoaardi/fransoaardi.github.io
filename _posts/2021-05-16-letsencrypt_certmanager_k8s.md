---
layout: post
title: Kubernetes 에서 Letsencrypt 와 Certmanager 를 이용한 무료 인증서 발급
date: 2021-05-16 22:25:00 +0900
categories: [Wiki]
tags: [k8s, letsencrypt, certmanager]
toc: true
comments: true
---

# introduction

회사에서 k8s 를 이용한 시스템을 구축할때, route53 을 이용해서 도메인을 생성했지만, gRPC 를 tls 설정을 해서 이용하기 위해 인증서 발급이 필요했다. 관련해서 고민을 하다가 letsencrypt 라는 비영리단체에서 발급하는 무료 인증서에 대해서 알게되었다. 

이전에 가상 인스턴스에 NGINX 를 직접 띄우고, letsencrypt 에서 부여한 well-known directory 에 파일 설정을 하는 mission 을 해결한 적이 있지만, 직접 설정했어야 됐는데, 다행히도 cert-manager 라는 프로젝트가 있어서, k8s 에서 3개월 만기인 letsencrypt 발급 인증서를 만료 1개월 전쯤 자동으로 갱신을 해주는 역할을 한다.

이렇게 설정해뒀던 cert-manager 에 문제가 생긴건지, 특정 도메인에 대해서 자동갱신이 안되기 시작했고, 이 현상을 troubleshoot 했던 내용을 담기위해 글을 작성한다.

추가로 letsencrypt 와 cert-manager 가 어떻게 동작하는지에 대해 간단히 정리한다.

**참고: troubleshoot 과정에서, 다소 어이없지만.. ingress 리소스를 삭제한 뒤 새로 생성하니 정상화 됐다.**

# references

- 동작방식: [https://letsencrypt.org/ko/how-it-works/](https://letsencrypt.org/ko/how-it-works/)
- troubleshoot: [https://cert-manager.io/docs/faq/troubleshooting/](https://cert-manager.io/docs/faq/troubleshooting/)
- troubleshoot-acme: [https://cert-manager.io/docs/faq/acme/](https://cert-manager.io/docs/faq/acme/)
- ACME 의 RFC: [https://datatracker.ietf.org/doc/html/rfc8555](https://datatracker.ietf.org/doc/html/rfc8555)

# 동작방식

아래에서 letsencrypt 는 CA(Certification Authority) 로 동작한다.

## 도메인 인증

- DNS 기록을 제공한다.
- 요청하는 도메인의 well-known URI 에 HTTP 리소스를 제공한다.

위의 두가지 과제 중 하나와 더불어, 아래 그림과 같이 임시 값을 제공한다.
![미션](/assets/img/posts/2021-05-16-letsencrypt_certmanager_k8s/howitworks_challenge_1.png)

이미지 출처: [https://letsencrypt.org/ko/how-it-works/](https://letsencrypt.org/ko/how-it-works/)

그러면 cert-manager 는 준비를 마치고, letsencrypt 쪽에 준비가 다 되었다고 통지하고, letsencrypt 는 과제가 충족되었는지를 검증한다.

![검증](/assets/img/posts/2021-05-16-letsencrypt_certmanager_k8s/howitworks_authorization_2.png)

이미지 출처: [https://letsencrypt.org/ko/how-it-works/](https://letsencrypt.org/ko/how-it-works/)

## 인증서 발급 및 해지

### 발급

위의 도메인 인증 과정이 끝나면, 인증과정에서 사용한 키의 쌍을 갖게되는데, letsencrypt CA 에 인증서를 발행하도록 요구하는 PKCS#10 인증서 서명 요청을 구성한다. 그 결과로 letsencrypt 는 인증서를 발급해서 돌려준다. 

![발급](/assets/img/posts/2021-05-16-letsencrypt_certmanager_k8s/howitworks_certificate_issue_3.png)

이미지 출처: [https://letsencrypt.org/ko/how-it-works/](https://letsencrypt.org/ko/how-it-works/)

### 해지

아래의 이미지와 같이, 발급과 동일하게 키를 가지고 해지를 요청하고, letsencrypt 는 해지채널 (CRL/OCSP) 에 게시해서 다른 곳에서 인증서 유효성 체크를 하는 경우에, 해지된 인증서임을 확인할 수 있게 하는 과정을 갖는것으로 보인다.

![해지](/assets/img/posts/2021-05-16-letsencrypt_certmanager_k8s/howitworks_revocation_4.png)

이미지 출처: [https://letsencrypt.org/ko/how-it-works/](https://letsencrypt.org/ko/how-it-works/)

# ACME

ACME 는 `Automatic Certificate Management Environment` 로, 문자 그대로 인증서를 자동으로 관리하는 프로토콜이다. 

아래 내용은 RFC8555 의 초록 발췌인데, **CA(Certificate Authority) 와 applicant 가 인증서 발급과 인증서 검증에 대한 방법을 자동화 하기 위한 프로토콜을 다룬다**고 명시한다

```
          Automatic Certificate Management Environment (ACME)

Abstract

   Public Key Infrastructure using X.509 (PKIX) certificates are used
   for a number of purposes, the most significant of which is the
   authentication of domain names.  Thus, certification authorities
   (CAs) in the Web PKI are trusted to verify that an applicant for a
   certificate legitimately represents the domain name(s) in the
   certificate.  As of this writing, this verification is done through a
   collection of ad hoc mechanisms.  This document describes a protocol
   that a CA and an applicant can use to automate the process of
   verification and certificate issuance.  The protocol also provides
   facilities for other certificate management functions, such as
   certificate revocation.
```

# yamls

## Cluster Issuer

Clusterissuer 는 클러스터 단위의 인증서를 발급받기 위한 리소스라고 생각하면 된다. 이후에 Ingress resource 생성 yaml 에 생성한 clusterissuer 의 name 을 명시해야된다.

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: cert-manager
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: abc@gmail.com 
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
      - http01:
          ingress:
            class:  nginx
```

## Ingress 

아래는 ingress 에 nginx-ingress 를 사용하는 모습인데, 아래 `metadata.annotations` 과 `spec.tls` 부분을 염두하면 된다.

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: name-here
  namespace: release
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - your.domain.here
    secretName: name-of-secret-to-make
```

# troubleshoot

2번 정도 알아서 잘 갱신되던 인증서가 제대로 갱신이 되지 않는 증상을 함께 troubleshoot 한 적이 있다. 결과적으로 ingress resource 를 삭제하고 새로 생성하여, 인증서를 발급받는 certificate request 과정이 새로 돌면서 정상화 됐었지만, 최대한 확인할 수 있는 아래 단계들을 기술한다. 물론, 근본적인 해결책을 제시하지는 못하고, 이정도까지는 확인해서 조금이라도 로그를 더 얻어내어 구글링을 해보기 위함이다.

1. ClusterIssuer (혹은 Issuer) 가 잘못되어있는지를 체크한다.

```bash
$ kubectl describe clusterissuer
```

> 잘못되었으면 당연히 clusterissuer 생성에 대한 yaml 파일의 점검이 필요하다.

2. Certificate 가 생성되어 있는지를 확인한다.

```bash
$ kubectl describe certificates.cert-manager.io 
```

> ingress 는 리소스를 생성할때 clusterissuer 에게 certificate 를 발급하기 위한 일련의 도메인 인증 과정을 요청한다. 이슈는 크게 세 가지인데, ingress 의 설정이 잘못되었거나, certificate 발급 과정에서 시간이 좀 걸리거나, certificate 발급 과정에서 실제로 문제가 발생했거나 이다.
ingress 설정은 정상이나 certificate 발급과정이 오래걸리거나 문제가 있는 경우에는 아래 trouble shoot 을 계속 따라하며 로그를 확인하는것이 중요하다.

3. CertificateRequest 에 문제가 없는지 확인한다.

```bash
$ kubectl describe certificaterequests.cert-manager.io 
```

4. Orders 가 잘 전달되었는지 확인한다.

```bash
$ kubectl describe orders.acme.cert-manager.io
```

위의 2,3,4 과정은 결국 `Status` 필드가 False 인 것을 발견하고, 그 로그를 확인해서 구글링을 하는 과정인데, 정말로 설정 오류가 아니라면, 트래픽이 몰려서인지, 성의없는 에러 로그를 남기며 안되는 경우도 있었다.

# 주의사항

- 테스트 과정에서 너무 많은 횟수의 발급이 요청된 경우, letsencrypt 자체에서 rate limit 을 걸어서, 정해진 기간이 지나기 전까지 새로운 인증서의 발급이 제한된다. 따라서, 테스트를 할때는 cluster issuer 를 production(`https://acme-v02.api.letsencrypt.org/directory`) 이 아닌 stage(`https://acme-staging-v02.api.letsencrypt.org/directory`) 쪽 서버로 연결하여 테스트 하도록 한다.

- 실제로, 테스트 및 인프라 세팅 과정에서 7일 동안 인증서 발급을 못한적이 있는데, dev 쪽 설정이라 다행이었다.

# 결과

[https://crt.sh](https://crt.sh) 에 가서 도메인 이름으로 확인해보면, 잘 발급 된 경우 아래와 같이 표에 나타나는 것을 확인할 수 있다.

![정상적으로 발급된 인증서](/assets/img/posts/2021-05-16-letsencrypt_certmanager_k8s/crt_sh_result.png)

