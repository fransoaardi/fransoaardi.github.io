---
layout: post
title: "Jekyll Serve 환경 구성(WSL2)" 
date: 2022-01-05 22:49:00 +0900
categories: [Wiki]
tags: [jekyll]
toc: true
comments: true
---

# Introduction

이 GitHub Pages 를 운영하면서 Jekyll 을 막연히 선택한 theme 에서 제공하던 GitHub Actions 를 이용한 CI/CD 를 사용했었다. 솔직히 말해 로컬에서 MarkDown 을 이용해서 작성한 후에 어떻게 렌더링 될지 확인을 하진 않았고, 배포 이후에 직접 페이지에 들어가 확인해보고, 포매팅이 깨진것을 뒤늦게 발견하곤 했다. 친구가 요근래 GitHub Pages 를 작성하기 시작하게 되면서 설정잡는걸 도와주다가, 미뤄뒀던 Jekyll Local Serve 설정을 했다.
솔직히 별 내용은 아닌데, 삽질의 기록을 남길 의도로 포스팅을 작성하였다.

# Contents

## Environments

후술 하였지만, 삽질 과정에서 Ubuntu 20.04 환경에서 Windows 10 의 WSL2(Ubuntu) 환경으로 옮겼다.

## Progress

솔직히 `jekyll serve` 하기 위한 설정은 간단하다. `bundle` 을 활용하기 위해 `ruby` 를 설치하면 되고, 명세된 dependency 들을 `bundle install` 을 통해서 설치해주면 된다.
하지만 왠지 모르게 로컬에 하나하나 설치하고 싶진 않았고 (업무에 Ruby 를 사용하지 않는다. 또한 Ruby 에 대한 지식이 없어서 확실하지 않은건 하고싶지 않았다) Docker container 를 이용하는 방법이 없을까 찾아봤다. 예상대로 방법이 있었고 간단했다. `jekyll/jekyll` image 를 활용한 container 를 실행하고, GitHub Pages repo 를 volume mount 해주고 container 내부에서 아래 command 를 차례로 수행해주면 된다.

```bash
$ bundle install
$ bundle exec jekyll serve
```

이론상으로는 간단했지만 이슈가 2 가지 발생했다.

## Troubleshoot

### First: Container 내부에서 `bundle install` 에 필요한 `https://index.rubygems.org/versions` 에 접근 불가

Container 에 `exec` 명령어를 이용해서 붙어서 `wget(curl 이 없었다)` 해보니 해당 페이지로 접근도 안되고, 다른 페이지에도 접근이 불가했다. 아마도 해당 machine 에는 VPN 이 연결되어 있었고, 연결 과정에서 `/etc/resolv.conf` 의 내용이 바뀌었다거나 하는, 특이한 상황이 있었을 것 같아서 일단은 다른 machine 을 이용하기로 했고, 예상대로 별 문제없이 동작했다.

### Second: `localhost` 혹은 `127.0.0.1` 을 이용한 접근이 불가

WSL1 에서는 localhost 를 처리하는 방식에 대해 WSL2 와는 차이가 있었나보다. WSL2 에서는 그런 문제가 없다고 하는데 구글링 과정에서 많은 혼선이 있었다.

다소 허무하지만, Jekyll 에서 참조하는 `_config.yaml` 에서 `host` 값은 [default 설정](http://jekyllrb.com/docs/configuration/default/)이 `127.0.0.1` 이다. 
아래의 결과에서 볼 수 있듯, loopback 설정이 되어있어 처럼 `localhost` 의 처리가 가능할 것 같은데, 이상하게 `host` 값을 `0.0.0.0` 으로 해주기 전까지는 동작하지 않았다. 

```bash
$ ifconfig
eth0      Link encap:Ethernet
          inet addr:172.17.0.2 
          ...
lo        Link encap:Local Loopback
          inet addr:127.0.0.1
          ...
```

이전에 `django` 와 같은 프레임워크를 사용할때 겪었던 문제인것 같은데, 우선은 `0.0.0.0` 으로 변경하여 처리하였다.

```bash
+# Serving
+host: 0.0.0.0
```

## Results

Repository 경로에서 아래와 같은 간단한 shell 실행으로 처리가 가능하다. 

물론, bundle install 할 것이 다소 많아서 시간이 걸리긴 하는데, 이후에 dynamic serving 으로 동작할거라 한번 정도는 기다릴만했다. 

> `--jobs` argument 로 병렬 처리를 이용해서 성능 향상이 기대된다고 했지만, 체감상은 차이가 없는 것 같다. 

```bash
#!/bin/bash
docker run --rm --volume="$PWD:/srv/jekyll" -p 4000:4000 -it jekyll/jekyll /bin/bash -c 'bundle install --jobs 4 --retry 3 --verbose && bundle exec jekyll serve'
```