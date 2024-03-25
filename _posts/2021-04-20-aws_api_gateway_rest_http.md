---
layout: post
title: AWS API Gateway 에서 EC2 에 서빙중인 application HTTP 로 호출하기
date: 2021-04-20 14:40:00 +0900
categories: [Wiki]
tags: [aws, api-gateway]
toc: true
comments: true
---

# introduction

회사에서 보안문제로, 몇개의 well-known 포트를 제외하고는(솔직히 이중에는 well-known 이 아닌것도 있었다) 외부망을 호출할 수 없게하여, EC2 에 다른 포트에 서빙중이었던 서비스에 접근하지 못한다고 proxy 역할을 할 다른 서버를 제공해줄것을 요청했다. 떠오른 방법은 4가지 였다.

1. 기존에 reverse proxy 용도로 개발해놨던, header, body 를 transparent 하게 전달하는 application server 를 띄우는것.
> 그러나 이전에 띄워놨던 이 서버도 대상 포트에 속하지 않아, 제공할 수 없었고, 다른 팀원이 이렇게 무분별하게 제공하면 SPOF(single point of failure) 가 될 수 있고, 수정하는 시점에 downtime 이 발생하지 않겠냐고 하여(현재 서비스 중이진 않고 prototype 단계로 크게 상관은 없었고 서버 대수를 늘려 rolling update 방식을 취한다면 생기지 않을 문제였지만 의견을 존중하기로 했다) 지양하기로 했다.
2. NGINX 를 띄운다.
> EC2 를 새로 띄우고 싶지 않았는데.. 오늘 하루종일 한 삽질을 생각하면 이렇게 할걸 그랬다
3. API-Gateway 와 Lambda 를 제공해서 Lambda 가 EC2 를 호출하게 한다.
> API call 이 일어날때마다 lambda call 도 함께 일어나서, 비용이 마음에 들지 않았다
4. API-Gateway 로 EC2 를 직접 호출하게 한다. 

이러한 의사결정 과정 끝에 4번 방법을 시도해보게 되었다.

실제로 4번 방법으로 구성을 성공했는데, 잘 정리된 문서가 없어서 많이 고생했었기 때문에, 다른분들은 그렇지 않았으면 하는 마음에 기록으로 남기기로 했다.

작업을 완료한 시점에 다시 생각해보건데, API-Gateway - EC2 호출 과정에 VPC-Link 라는 방식으로 연결하게 되었는데, NLB(network load balancer) 를 하나 생성해놓는 이 방식이 과연 api 호출마다 Lambda call 을 하는 방식보다 경제적인가는 다시 생각해볼 필요가 있다.

# references

정말 많은 구글링을 했다고 생각했지만, aws 의 문서는 결과적으로 아무런 도움이 되지 않았다.(개인적으로 aws 의 문서화 방식을 상당히 싫어한다) 

이 글을 작성하는데 참고한 링크를 첨부한다.

- [vpc-link 구성 관련](https://aws.amazon.com/ko/blogs/korea/amazon-api-gateway-vpc-link/)

# detail

사실 API-GW 에서 EC2 에 서빙중인 application API 를 호출하는것은 간단하다. API-GW 에서 `API 생성` -> `REST API(private 아님)` 를 생성하고, endpoint 에 외부로 노출된 app API 를 연동하면 된다. 하지만, EC2 는 VPC(virtual private cloud) 안에 있었고, API-GW 와 EC2 사이에는 보안그룹(sg) 가 있었고, sg 의 inbound 에 API-GW 를 설정할 수 없었다. 물론 sg 를 public 하게 열어버리면 아무런 문제가 되진 않았지만, api-gw 를 호출하기 위해 sg 를 없애는건 말이 안된다. 

따라서 API-GW 에서 EC2 를 호출하게 하기 위한 VPC-Link 구축이 필요하다. VPC-Link 란 결국 EC2 를 target 으로 하는 대상그룹(target-group)을 생성하고, 앞에 NLB 를 생성해서 대상그룹을 연동하고, 이 NLB 를 API-GW 에서 호출할 수 있도록 연결해주는 방식이다. 

리소스를 아끼는 방향으로 API-Gateway : VPC Link : NLB : Target Group 을 1:1:1:N 형태로 구성하려고 한다.

# progress

- `EC2` 좌측메뉴에 `로드밸런싱 > 대상 그룹` 에서 `Create Target Group` 을 누른다.
- EC2 instance 를 대상그룹으로 할 것이라서 `Instances` 를 선택, **Protocol 은 HTTP 가 아닌 TCP 로 설정한다**

![대상그룹 생성](/assets/img/posts/2021-04-20-aws_api_gateway_rest_http/create-targetgroup.png)

- API-Gateway : VPC Link : NLB : Target Group 을 1:1:1:N 으로 하여 리소스를 아끼는 방향으로 구축하려고 하기 때문에 target group 을 여러개 만들고, 다른 port 로 구성한다. (8080, 8081, 8082 를 이용했다)

![대상그룹, 인스턴스 매핑](/assets/img/posts/2021-04-20-aws_api_gateway_rest_http/map-targetgroup-ec2instance.png)
- NLB 생성하며 위에서 생성한 target group 으로 forward 구성을 한다.

![NLB 생성](/assets/img/posts/2021-04-20-aws_api_gateway_rest_http/create-nlb.png)

![NLB 생성, 대상그룹 매핑](/assets/img/posts/2021-04-20-aws_api_gateway_rest_http/map-nlb-targetgroup.png)

- VPC-Link 를 생성하며 위에서 생성한 NLB 를 연동한다.

![VPC-Link 생성 화면](/assets/img/posts/2021-04-20-aws_api_gateway_rest_http/create-vpclink.png)

- API-Gateway 를 생성하고, 리소스를 3개 만들었고, 리소스 3개에 각각 메소드를 위에서 생성한 VPC-Link 로 연동한다. 연동할때 엔드포인트 URL 에 생성한 NLB dns name 을 입력(https 가 아닌 http 이다, 주의한다), 포트는 NLB 의 리스너 부분을 참고하여 작성한다. 

![VPC-Link 생성](/assets/img/posts/2021-04-20-aws_api_gateway_rest_http/map-apigw-vpclink.png)

![NLB 리스너](/assets/img/posts/2021-04-20-aws_api_gateway_rest_http/nlb-listener.png)

- 생성한 API-Gateway 의 리소스는, 스테이지 라는 과정이 필요한데, 쉽게 생각하면 배포관리 이다. 예를들어 지금 생성한 `/dev, /sprint, /release` 각각의 리소스 경로는 API-Gateway 의 dns name 을 `api-gw.com` 이라고 가정하면 `api-gw.com/dev`, `api-gw.com/sprint` ... 와 같이 제공되는것이 아니라, 스테이지 이름이 경로에 추가된다. 따라서, `api-gw.com/{stageName}/dev` 와 같이 제공된다. 추가로, API-key 를 생성하고 사용하기로 했다면, Header 에 `x-api-key: {apiKeyOnly}` 를 추가해서 API-Gateway 를 호출하면 된다.

# furthermore

NLB 는 처리용량, 실행시간에 따라 과금되는 리소스이다. 서울 리전 (ap-northeast-2) 기준으로 아래와 같다.
```
$0.006 per used Network load balancer capacity unit-hour (or partial hour)
$0.0225 per Network LoadBalancer-hour (or partial hour)
```
데이터 처리량은 따지지 않고, partial hour 로 생각해보면, 24 hr * 30 day = 720 hr/month
720(hour) * 0.0225($/hour) * 1100(원/$)= 17,820 원 정도라서 괜찮아 보인다.

다행인건 대상그룹을 여러개 생성하는 것은 비용이 발생하진 않아서, 한개의 NLB 에 여러개의 대상그룹을 걸고 서비스하는 방식으로 비용을 줄일 수는 있다. 예를들어 dev, stage, release 세가지 phase 를 서비스 하기 위해서 VPC-Link 를 3개 만들고, 각각의 NLB(3개) 를 생성하는 방식이 떠오르지만, 이렇게 하면 VPC-Link 1개에, NLB 1개로 처리할 수 있었다.

아래의 [람다를 호출하는 비용](https://aws.amazon.com/ko/lambda/pricing/)은 아래는 상당히 과격한 사용량의 계산인데, 보통의 경우 위의 NLB 생성 비용보다 훨씬 저렴할것으로 예상된다. (람다를 쓸걸 그랬나..)
```
함수에 512MB 메모리를 할당하고 한 달에 3백만 회 실행하며 매번 1초간 실행되었다면 요금은 다음과 같이 계산됩니다.
(이하 중략)
월별 컴퓨팅 요금 = 1,100,000 * 0.00001667 USD = 18.34 USD
```

비용과 구현의 간편성을 따진다면 람다를 이용하는게 나아보이지만, 이후에 성능까지 고려를 한다면 다시한번 생각해보는게 좋겠다.