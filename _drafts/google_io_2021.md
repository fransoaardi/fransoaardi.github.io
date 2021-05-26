---
layout: post
title: Google I/O 2021
date: 2021-05-26 00:08:00 +0900
categories: [Seminar]
tags: [google, io]
toc: true
comments: true
---

# introduction

Google I/O 에 참가하기 위해서는, 경쟁률을 뚫고.. lucky draw 과정을 거쳐야한다. 2020년도 google i/o 참가를 하기 위해 거액의 돈을 내고, 당첨까지 되어서 갈 뻔했지만, 코로나로 Google I/O 2020 은 취소되어, 가지 못했다.

작년의 한을 풀고자, 집에서 열심히 키노트와 세션들을 들어봐야지 라고 생각하며 들은 내용을 아래에 정리해보려고 한다.

google io 를 기념하며 최대한 오프라인과 비슷한 경험을 해보라며, 게임도 만들어놨는데 그것도 재밌게 해봤네요.

키노트 시청 후, 아래와 같이 정리를 좀 해봤습니다. 전체 내용을 담은건 아니라(정리하다보니 많은 부분을 담긴 했습니다) 빠진내용은 직접 시청해보시는게 좋습니다.
자세한 내용은 각 분야에 대해 정통하신 여러분들이.. deep 한 부분은 알아서 챙겨주시면 되겠습니다. 
- https://events.google.com/io/session/88b34a4e-6170-4f18-a321-4260fb559e60?lng=ko

# content

- LaMDA (Language Model for Dialogue Applications) (18분 부터)
"명왕성", "종이비행기" 와 대화하는 데모를 보여주는데, 상당히 자연스럽게 대화가 이어졌고, 이미 학습된 데이터에 대한 질의응답이 이뤄졌습니다.
예를들면, 종이비행기에게 "what's the worst place you've ever landed(너가 착륙해본 제일 최악의 장소가 어디야?)" 라는 질문에, "진흙탕에 빠졌을때에요. 다행히 손상은 크지 않았고, 몇분정도 그대로 있었다. 상당히 짜증났다." 라는 식으로 답변합니다. 
관련해서 아래와 같은 글이 올라왔네요(keynote 발표와 함께 올라온것같습니다)
- https://www.blog.google/technology/ai/lamda

- MUM (Multitask Unified Model) (43분 부터)
BERT 의 1000 배 성능.. 이런건 제가 잘 모르겠구요, 영상에 보시면 "Mt.Adam 에 다녀왔었는데, Mt.Fuji 를 가려고 한다. 어떤걸 준비하면 되겠어?" 라는 질문의 의도를 파악하고 답변을 하거나,
multimodal 을 지원해서, 등산화 사진을 올려놓고, "can I use these to hike Mt.Fuji?" 와 같은 질문에 대해 대답도 합니다. 
역시나 관련해서 아래와 같은 글이 올라왔습니다.
- https://blog.google/products/search/introducing-mum/

- Google workspace
    - Smart Canvas: docs 를 같이 보면서 화상회의를 할 수 있음
    - live caption, translation 얘기가 발표 전반적으로 얘기가 나옵니다. 

- TPU v4 출시: v3 의 2배 성능이라고 합니다. 

- Quantum computing
    3 단계의 milestone 을 공유하면서 현재는 1단계를 넘어섰다 라고 얘기합니다. 
```
1. beyond classical
2. error-corrected logical qubit
3. error-corrected quantum computer
```

Android 12 소식 (1시간 18분부터)
안드로이드 12가 가을에 나온다고 합니다. 세가지가 키워드라고 하는데요, 
```
- deeply personal
- private and secure
- better together with the phone at the center
```
권한 관련한 별도의 대시보드와, 권한을 바로 꺼버릴 수 있는.. shortcut 까지 제공될것 같고,
크롬북, android auto 등의 ecosystem 과 mobile 기기의 연동을 강조하는데, 특히 차량 smart key 얘기와 bmw 와의 협업 얘기가 있었습니다.

material ui 에서 "material you" 라는 color template 을 몇개 설정하여 테마를 잡고가는.. 그런 컨셉(원래 있던거 아닌지..) 을 얘기했었는데요,
안드로이드도 material 적용해서 ui theme 을 많이 바꿀것이고, 예를들면 배경 사진 고르면, 자동으로 color template 을 뽑아서.. 적용해준다 이런 얘기가 있었습니다. 

그리고 watch os 관련해서.. 삼성과 협업을 한다고 한것으로 보아, 타이젠은 이제 끝났나봅니다. 헬스케어 관련해서 fitbit 과도 협업을 한다니 흥미롭습니다.
- https://biz.chosun.com/it-science/ict/2021/05/19/ITHTXU4EAVBZZENH7XKKSQHUXI/

Camera 알고리즘 개선
- 어두운 피부색에도 잘 어울리도록.
- "카메라는 객관적이라고 알려져있지만, 사실은 그렇지 않다. 이제는 사람이 아니라 도구를 개선할때이다" 

Google photo
- little patterns: 비슷한 패턴이 보이는 사진들을 모아서 보여줌 (curation)
- cinematic photos: A, B 사진이 있을때 A~B 과정을 증강해서 영상으로 만들어버림

Google maps
- 도쿄에서 indoor ar navigation 을 시작한다고 합니다.
- detailed street maps: 50 개 도시에 신호등이랑 횡단보도까지 보여준다고 합니다.
- 8시에는 카페랑 베이커리를, 5시에는 저녁먹을곳을 보여주고, 새로운 도시에 도착하면, landmarks 들을 더 잘 보여주는.. context 에 맞는 시도를 한다고 합니다.
- area busyness: 해당 지역이 실시간으로 얼마나 붐비는지 확인할 수 있다고 합니다.

Google shopping
- 쇼피파이와 협업을 하고.. 검색과정에서 자연스럽게 제품 구매로 이어지는 걸 강조했습니다.
- 사진찍어서 찾기도하고.. 다양한 경로로 찾았습니다. 구글 렌즈를 많이 밀었습니다.
- 구글 메인에, 내가 비교했던 제품들이 "my carts" 에 관리될수도 있다고 합니다. 

헬스케어
    - 유방암 검사에 AI 를 도입하여, 더 필요한 사람들이 치료받을 수 있도록 하는 screening 기법 도입
    - 유럽에서 피부질환을 사진을 찍어서 검색할 수 있도록 하는 기능

project starline(대화하려는 상대를 초고화질 카메라를 이용해서 3d imaging 하고 바로 앞에 있는것처럼 하는..)
    - https://blog.google/technology/research/project-starline

Sustainability
    - 2030년까지 0 CO2 emission 을 달성하겠다고 합니다. 
    - 신사옥도 위에는 태양광 패널, 기둥은 지열을 이용한다고 합니다.
    - 지열을 적극 활용해서(geo thermal) 데이터센터 발전에 활용하겠다 라고 합니다.


Lit: Lit는 개발자가 모든 프레임워크에서 신속하게 작동하는 경량 구성요소를 빌드하도록 돕는 Google의 오픈소스 라이브러리 세트입니다. Lit를 사용하면 공유 가능한 구성요소, 애플리케이션, 설계 시스템 등을 빌드할 수 있습니다.
- https://codelabs.developers.google.com/codelabs/lit-2-for-react-devs#0

Chrome 90 devtools 새로운 기능들
- https://www.youtube.com/watch?v=kOodTLAjPsE&list=PLNYkxOF6rcIBDSojZWBv4QJNoT4GNYzQD&index=2&ab_channel=GoogleChromeDevelopersGoogleChromeDevelopers

--

# Google IO Keynote

## Smart Canvas

## Translation

## LaMDA (language model of dialogue application)
- 명왕성, 종이비행기와 대화하는 데모
- fairness, accuracy, security
- multimodal 은 아직 진행중인것같음

## TPU V4
- v3 보다 2배 빠름 

## Quantum computing 의 milestone
1. beyond classical
- 지금은 여기라고함
2. error-corrected logical qubit
3. error-corrected quantum computer 를 위한 개발이 진행중

## Auto delete
- 계정 보안 관련. 
- secure by default
- password free future
password manager
> chrome + android 연동.. 
- private by design

---

## google search

make information accessible and useful

MUM (multitask unified model)
- 1000 x than BERT
- understand multi modal

can I use these to hike Mt.Fuji? 쿼리를 
등산화 사진을 입력하면서 한다. (이게 말이되나..)

AR in Search

---

## google maps

- indoor navigation ar 을 시작한다고함.. (도쿄에서)
- detailed street maps
    - 50 개 도시에.. (신호등이랑 횡단보도 넣는듯)
8시에는 카페랑 베이커리를, 5시에는 저녁먹을곳을 보여줌 
새로운 도시에 도착하면, landmarks 들을 더 잘 보여줌

area busyness 
- 그 지역이 실시간으로 얼마나 붐비는지 확인할 수 있음 


## shopping graph (knowledge graph)
- 쇼피파이와 협업
- 검색과정에서 자연스럽게 제품 구매로 이어짐
- 사진찍어서 찾기도하고.. 
- 내가 비교했던 제품들이 my carts 에 관리될수도 있음 


## google photo
### little patterns
- 비슷한 패턴이 보이는 사진들을 모아서 보여줌
    - curation
### cinematic photos
- A, B 사진이 있을때 A~B 과정을 증강해서 영상으로 만들어버림
    - 이렇게까지 해야되나..

## Material design
- Material you 
    - material pallette 를 개인화해서, 모든 앱에 통용될 수 있게 하는 시도.

## android
- android 12 
    - deeply personal
    - private and secure
    - better together with the phone at the center

- 새로운 UI 
- color, shape by material you
    - wallpaper 의 색깔에서 뽑아내서 만들어냄

- google pay?

- 22% cpu reduction
    - everything faster

keep information safe
- file based encryption
- tls by default
- secure dns

granting access
- privacy dashboard 가 생김 (어떤 앱에서 갖다썼는지 볼수 있음)
- access(camera + @) 끌수있는 버튼이 생김

private compute core
- 

multi device
- digital car key

watch os(with samsung)
- unified platform
- new consumer experience
- world-class health and fitness
    - fitbit

타이젠이 통합된것같다.. wow.. 

camera 알고리즘 개선
- 어두운 피부색에도 잘 어울리도록.
- "카메라는 객관적이라고 알려져있지만, 사실은 그렇지 않다. 이제는 사람이 아니라 도구를 개선할때이다" 

---

## Health

- 
- 유럽에서 이미지를 이용한 피부질환 관련 검색기능이 추가됨 

---
Project starline

바로 앞에있는것처럼 보이기 위해 3d imaging 하고, 재구성하는것.. 
고화질 카메라 이용

---
Sustainability

2030: operate 24/7 carbon free 

carbon intelligent computing platform
geothermal power plant 를 이용한 datacenter 설립. (nevada)

---

building a more helpful google for everyone.

---

개발자 기조연설

We are problem-solvers, drawn to the hardest problems and using technology to tackle them

Android:
tools
    android studio
    kotlin
    jetpack
    jetpack compose

web 
    - privacy technology
    - platform capability
    - developer tool

privacy
    sandbox: 
    - user privacy by default

광고 게시자와 광고주 간, 3rd-party-cookies 를 이용한 방식 대신 
attribution reporting api 를 두고, chrome 이 
어느정도의 난독화 해서 보내는것 같음. 

declarative link capturing

- file system access api 

- file handling api (작업중)
    - 

- web HID protocol
- web serial api

- webgl google maps
    - webgl overlay view 
    - webgl tilting

- webassembly and simd

- content visibility: auto

- BF cache (back, forward cache)

- pre-rendering

- css flexbox, grid 
    - 개선중 

--
native fuzzing 기능 추가중

- go.dev
- static analysis, static check
- errorgroup pkg

---
AI in the cloud

vertex AI?
- 

---
google maps and web gl

-cloud-based maps styling
 - create, manage, and deploy map styles from the google cloud console
 - poi filtering and density control
 - marker collision handling
 - zoom-level customization

vector map
    - render with webGL

tilt, rotation

webgl overlay view

---
 web runtime performance

 최적화는 처음부터? 아니면 마지막에?
 - 마지막에 시간이 주어지지 않을 수 있어서, 그때그때 하는데, 물론 it depends.

 로컬에서 80MB 의 JS 를 reloading 하고있는데, 어떻게하면 빠르게할 수 있을까?
 - 개발환경과 프로덕션 환경을 최대한 비슷하게 맞춘다, 설마 프로덕션에서 80MB 씩 reload 하진 않을테니..
 - 프로덕션 뿐만 아니라 개발환경의 최적화도 상당히 중요하다.
 