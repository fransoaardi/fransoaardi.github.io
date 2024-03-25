# introduction
- eks 에 trendmicro 를 세팅하기 위해 ec2 시작 템플릿을 이용한 nodegroup(이하 ng) 을 생성하기로 했다.
- 기존 ng(ng1) -> 새로만든 ng(ng2) 로 옮기는데 k8s resource (pod, deploy 등..) 은 잘 넘어왔는데 동일한 domain 으로 호출이 안됨
- nlb 가 기존 ng 의 인스턴스 정보를 캐싱하고있어서 트래픽이 전달이 되지 않고있는거라고 생각함

# 결론
- nlb 은 eks 의 ng 변경에, 딜레이가 있긴 하지만 결국 연동이 된다.
- istio 는 안전/완전하지 않다. (이것도 이후에 반성하게될지도)
    - 고정된 node 갯수는 몰라도, node autoscale 에 잘 대응할 수 있을지는 모르겠음 

# 실험
## 실험환경
- istio: 1.7.3
- k8s: 1.18
- route53 에 aws nlb 를 upstream 으로 등록해놓음
- ingress 로는 istio 의 gateway 를 씀

## 실험1
- Q: ng1 의 크기를 변경하면 nlb 는 잘 scale out 하는지?
- A: yes
### 절차 
- 새로운 cluster 에 ng(2EA, ng1) 생성
- istio 설치: aws clb 생성 확인
    - istio-system/istio-ingressgateway 가 elb(clb) 를 생성하며 보안그룹을 생성함
    - eks cluster 보안그룹에 위에 생긴 보안그룹이 inbound 에 잡힘
- aws nlb 패치: aws nlb 생성확인, eks 보안그룹에 nlb 관련 annotation 추가됨, target그룹 정상 생성
    - https://kubernetes.io/ko/docs/concepts/services-networking/service/#aws-nlb-support

    > subnet 별로 nlb 가 갖는 eni 가 생겼고, 그 eni 의 public ip 임 
    ```
    $ dig +short newly-created-nlb-endpoint.elb.ap-northeast-2.amazonaws.com
    xx.xxx.xxx.197
    xx.xxx.xxx.136
    xx.xxx.xxx.172
    ```

- ng1 크기 변경 (2EA -> 4EA): target 그룹 정상 반영
- 새로운 ng(ng2) 생성, ng1 삭제: target 그룹 정상 반영

## 실험2
- Q: ng1 -> ng2 변경하면 nlb 는 따라서 잘 변경되는지?
- A: yes, 그런데 istio 의 문제인지, pod redeploy 가 필요한 부분이 있었음
### 절차
- ng1 이 생성되어있는 cluster 에 ng2 를 생성함
    - target group 에 ng2 가 추가됨 
- ng1 을 제거함
    - ng2 로 k8s 리소스들 옮겨감
    - target group 에 ng1 제거됨
    - route53 에 등록된 dns 딜레이 있은 후 제대로 동작시작 
    - api 동작에는 문제가 발생: 
        - log 를 보니, nats 를 못찾고있는데 (reconnect 기능이 꺼져있었던건지는 확인이 더 필요함), container 들어가서 직접 호출해보니 dnslookup 은 가능한 상태
        - nats 뿐만 아니라, 다른 서비스들을 전체적으로 못찾고있었음
        - pod redeploy 를 통해 해결함

## 실험3
- Q: 동일 cluster 에 istio 삭제+재설치로 새로운 nlb 를 할당받으면 어떤일이 생길까
- A: 예상대로 잘 동작한다. 단, 삭제는 아래 절차에 포함될 커맨드를 이용해서 잘 해야된다.

### 절차
- istio 가 설치된 cluster 에서 istio 를 지운다
> https://istio.io/latest/docs/setup/getting-started/#uninstall
- istio 를 istio-system namespace 만 지우는 방식으로 했었는데, namespace 에 있던 istio-ingressgateway svc, deploy 등이 삭제되는 효과는 있었지만, service discovery 에 영향을 주는 어떤 설정이 남아있었다고 추측하고있다. 이후 실행한 gateway 는 제대로 기능하지 못했다.
```
# pod 등, 리소스를 일단 다 지우는게 옳다 (injection 된 sidecar 인 envoy 들을 내리는 등.. )
# default 로 깔았기 때문에 profile 은 default 로 한다.
$ istioctl manifest generate --set profile=default | kubectl delete --ignore-not-found=true -f -
$ kubectl delete namespace istio-system
```
- istio 를 새로 설치한다
- nlb 를 새로 생성한다, 이때 cluster security group 에 새로 생긴 nlb 관련 annotation tag 가 잘 생겼는지 확인한다.
- 이후 특이사항 없이 동작한다.