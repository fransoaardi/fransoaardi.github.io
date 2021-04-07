---
layout: post
title: k8s 가 container runtime 로 docker 지원을 중단
date: 2021-04-07 17:27:00 +0900
categories: [Wiki]
tags: [k8s, container]
toc: true
comments: true
---

# introduction

요 근래 medium 을 통해 추천되는 글들에, docker 를 대체할 container 기술들에 대한 이야기들이 많이 보였다. 물론, 거의 대부분의 경우 container 기술 이라는 표현 보다는 훨씬 더 많은 경우 docker 로 통칭되었다. 하지만, 다른 여러 글들을 읽다보니 이제는 docker 라고 부르기에는 입지가 점점 좁아지는건 아닌가 하는 생각이 들었다. 물론, docker 는 microsoft 와 2020 년 초에 partnership 을 맺어서 ms 의 어깨에 올라타긴 했지만...

redhat 에서 밀고있는 `podman` 이라는, `alias docker=podman` 이라고 하면 똑같이 사용할 수 있을 정도로 머리를 잘쓴 구현체가 있는데, 애석하게도 osx 에서 실험해보고 싶었지만, linux vm 을 할당받아서 쓰라는 말들이 보이는것으로 봐서, linux 에서만 사용 가능한 상태인것 같고, 굳이 `podman` 을 사용해보기 위해 새로운 환경을 구성하고 싶지는 않았다(`brew install podman` 이후 에러가 발생한걸 보고, 아니다 싶었다. 게을렀다).

오늘 우연한 기회에 k8s 에서 container runtime 으로 사용하던 docker 지원이 중단 된다는 [이 글](https://kubernetes.io/ko/blog/2020/12/02/dont-panic-kubernetes-and-docker/) 을 읽고는(벌써 4개월이 지난 글이긴 하다), 왜 요즘 이런 docker 를 빼는 움직임이 보였는지 대충 이해하게 되었고, 관련해서 포스팅을 하기로 했다.

# tl;dr;

[이 글](https://kubernetes.io/ko/blog/2020/12/02/dont-panic-kubernetes-and-docker/)에서 하는 이야기를 요약해보면 아래와 같다.

- k8s 의 default container runtime 이었던 docker 가 v1.20 부터 deprecated 되고, v1.23 부터는 container runtime 으로 활용할 수 없게된다. docker 는 CRI 를 구현하지 않았고, dockershim 으로 변환하는데 비효율적이고 관리가 힘들기 때문이다.
- 그렇지만 docker 로 build 한 image 들이 사용 불가능한 상태가 되는것은 아니다. docker built image 는 OCI 를 충족하기 때문이다.
- k8s admin 이라면 앞으로 어떤 container runtime 을 사용할지 선택이 필요한데, 후보로는 `containerd` 와 `cri-o` 가 있다.

# docker, k8s, oci, cncf

docker 는 OCI(open container initiative), k8s 는 CNCF(cloud native computing foundation) 의 대표주자로, 태생이 다르다.
OCI 는 container image, container runtime 과 같은, 공통된 container format 을 갖자는 취지에서 시작했고, CNCF 는 cloud native 한 지속가능한 생태계를 구축하자 라는 서로 다르지만, 좋은 취지에서 시작했다. 서로 잡아먹을만한 이유는 없고, 실제로 OCI 에 CNCF 의 대표격인 google 도 포함되어 있다.

물론, docker 가 `docker swarm` 을 야심차게 공개하며, container orchestration 에 대한 야망을 드러낸 적이 있었던것으로 기억하는데, 그때 이미 구글을 앞세운 k8s 가 빠르게 인기를 얻어가고 있었고, 언젠가 docker 가 k8s 의 런타임으로 잘 지원하겠다고 하여 사태가 종료되었던 기억이 있다.

# detail

[docker 의 구조](https://docs.docker.com/get-started/overview/)는 docker cli (우리가 `docker ps -a` 를 호출할때 사용하는 그것 ) 가 호출하는 명령어에 따라 이를 듣고있는 docker daemon(`dockerd`) 이 container, registry, images 등을 호출한다.

k8s 를 직접 설치하지 않고, EKS, GKE 와 같은 PaaS 를 이용하였다면 확인해볼 기회가 없었을 수 있지만, k8s 의 node 에는 docker 와 같은 container runtime 이 설치된다. 즉, `kubectl apply -f deploy.yaml` 과 같이 pod 를 띄우는 상황에, node 에 있는 default container runtime 이 실행되는데, 이 경우 v1.20 이전까지는 default container runtime 인 docker 가 실행되는 구조이다. 

간접적으로 확인해볼 수 있는 방법은, `kubectl get no -owide` 해보면 `CONTAINER-RUNTIME` 필드에 `docker:19.3.6` 과 같이 표시된다. 

node 에는 docker runtime 이 설치가 되어있어야 된다는 얘긴데, docker 는 docker engine 을 설치해서 docker-cli, docker-daemon 과 같은 여러 기능들이 동시에 설치되고, 이는 k8s 에서 container runtime 으로 사용하기에는 적합하지 않다. 일부 기능을 사용하기 위해 monolithic app 전체를 설치하는것과 같은 상황으로 비유할 수 있다. 

docker 에는 문제가 있는데, k8s 에서 [CRI(container runtime interface)](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/container-runtime-interface.md) 를 이용하는데, docker 는 CRI 를 dockershim 을 통해 docker 가 사용하는 interface 로 변경을 하는 구조라서 비효율적이고, k8s community 에서 이를 [없애자는 proposal](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/2221-remove-dockershim) 이 올라왔다. k8s v1.20 에서 docker 는 사용 가능하지만 deprecated warning 이 발생하고, v1.23 부터는 docker 는 CRI 를 구현하지 않기 때문에 사용할 수 없고, CRI 를 구현한 `cri-o` 혹은 `conatinerd` 로 변경할 수 있다고 한다.

`cri-o` 는 redhat 에서 밀고있는 container runtime 이고, `containerd` 는 CNCF 를 졸업(graduated)한 프로젝트인데, 특이한건 cri-o 역시도 CNCF 소속이고, 물론 아직(2021/04/07 기준) incubating 단계이다.

`containerd` 는 원래 docker 내부에서 사용하고 있었기 때문에, `containerd` 로 옮긴다면 크게 변경 할 일이 없다고 한다. 
`cri-o` 는 이미 redhat 의 openshift 에서 사용중이던 runtime 이라서 안정성은 확보된 것 같지만, docker 에서 바로 옮기는데 risk 가 있어보인다.

`kubectl` 을 이용해서 명령을 하면, k8s 가 `containerd`, `cri-o` 와 같은 CRI 구현체(high level runtime) 들에게 gRPC 를 이용해서 통신하고, OCI container 구현체(low level runtime) 인 `runc`, `crun`, `gVisor` 등 에게 container 를 실행하도록 한다. `runc` 는 원래 `containerd` 에서 호출하던 방식이고, golang 으로 개발되어있는데, `crun` 은 c 로 개발해서 48% 정도 성능 개선이 일어났다고 하는데, 자세한건 더 찾아봐야 된다.

# expected problems

앞서 얘기했던 `podman` 을 사용해볼까? 라는 흥미 위주의 부분과 별개로, 확인이 필요한 부분이 몇가지가 있다. 아래 부분 확인해 보고 이후에 내용을 추가할 예정이다.

1. docker 의 특수한 기능을 사용하는 경우 (docker configuration) 는 다른 runtime 과 호환이 되지 않을 수 있다.
2. 회사에서 nvidia-docker 를 이용해서 GPU 를 사용하는 container 를 k8s 를 이용해서 실행하는 경우가 있는데, [nvidia 의 링크](https://developer.nvidia.com/nvidia-container-runtime) 중 `Supported Container Technologies` 에 `LXC` 와 `CRI-O` 가 적혀있긴 하지만, github 등을 참고했을때 nvidia 와 관련된 상세문서를 찾을 수 없었다. 호환이 될지 확인이 필요하다.
3. [이 글](https://kubernetes.io/blog/2020/12/02/dockershim-faq/#what-should-i-look-out-for-when-changing-cri-implementations)에 runtime 을 CRI 구현체로 이전할때 체크리스트가 제공되는데, docker socket 을 사용하거나, logging configuration 관련된 내용은 체크하라고 한다. 회사에서 구현한 서비스에 로깅을 k8s daemonset 으로 fluentbit 를 실행하고 노드별로 pod 들에서 stdout 으로 발생시킨 로그들을 es 로 보내는 방식을 사용하고 있는데, docker 의 logging driver 를 사용하진 않았던 것으로 기억하지만 영향도 확인은 필요하다.

# references

> k8s container runtime 에서 deprecated 된 docker 에 대한 k8s blog 글

- https://kubernetes.io/ko/blog/2020/12/02/dont-panic-kubernetes-and-docker/

> docker 는 왜 deprecated 됐는가를 알려주는 글 (dockershim 을 이용한 비효율적인 구조 때문이다)

- https://kubernetes.io/blog/2020/12/02/dockershim-faq/#what-should-i-look-out-for-when-changing-cri-implementations

> remove dockershim proposal

- https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/2221-remove-dockershim

> k8s container runtime 에서 deprecated 되는 docker. 개발자/관리자가 해야될 일과 상세한 내용을 다룬 꼭 읽어볼만한 글

- https://dev.to/inductor/wait-docker-is-deprecated-in-kubernetes-now-what-do-i-do-e4m