---
layout: post
title: Naver Search Colloquium 후기
date: 2021-05-07 01:20:00 +0900
categories: [Seminar]
tags: [ir]
toc: true
comments: true
---

# introduction

네이버에서 제공하는 검색 콜로키움 세미나가 있어서, 하루를 투자해서 가벼운 마음으로 들었다. 아래는 그냥 듣고는 금방 잊어버리게 될 것 같아서, 짧게짧게 그때의 생각들을 의식의 흐름대로 적어본 것이다. 나중에 공부할 내용을 적어놓기도 하고, 이후에 참고할 때를 위함이다. 

잘못된 내용을 적었을 수 있지만, 정리에 많은 공을 들이진 않으려고 한다.

각 세션별로 [링크](https://m.blog.naver.com/naver_search/222309902298) 에 담긴 Topic 소개를 복사해서 붙여넣었다. 

# references

- 발표 Topic 소개 / 스케쥴 [https://m.blog.naver.com/naver_search/222309902298](https://m.blog.naver.com/naver_search/222309902298)

# contents

## Neural Topic Model for Web Search (이종욱 교수)

- bert 를 쓰지 않고, 주제모델을 사용했는데, 주제모델이 계산량이 적어서 더 좋을것이라고 생각한 접근
- prodLDA ?
- NVDM ?

## Optimizing Training Data in Learning to Rank (송영인 님)

```
이 발표에서는 왜 학습 데이터의 최적화가 웹검색을 위한 랭킹 모형 구축에 필요한지, 그리고 기존 data dropout에 기반한 학습 데이터 최적화 방법을 랭킹 모형 학습에 사용하기 어려운 이유에 대해서 살펴본 후, 기존 방법을 개선할 수 있는 새로운 data dropout 기반 최적화 방법론을 제안합니다.

제안 방법은 1) Data Dropout 기반 학습 데이터 최적화 방법에서 요구되는 계산 량을 효율적으로 감소시키는 것과 2) 학습에 유용한 예제를 부적절하게 dropout시킴으로써 발생하는 성능 상의 손실을 막는데 개발 주안점을 두었으며, 복수의 공개 learning to rank 데이터셋과 네이버 검색 서비스로부터 수집한 실 세계 데이터셋 모두에서 안정적인 성능 향상을 보였습니다.
```

- LambdaMART ?

- 클릭기반 학습데이터는 수작업 데이터보다 품질이 좋지 않고, 노이즈가 심하다
- Data Dropout algorithm 에서 favorable, unfavorable 데이터가 있다고 가정한다. 따라서 unfavorable 을 제거해서 학습데이터를 최적화하는 방식을 취함. 
- Influence Function(IF) from Approximation to reduce computations by Koh et al, 2019: 전체 데이터셋에서 학습예제의 loss 변경을 해가면서 영향이 큰지를 판단해서, favorable/unfavorable 구분해서 제거할 수 있다.
- Influence Function 의 계산량이 많아서, approximation 을 이용해서 data dropout 에 적용함. 그런데 이마저도 비싼 계산코스트가 든다.
- influence estimation 의 정확도가 낮다. (Three problems of dropout algorithm in pairwise learning to rank: L2R)

- 기존 two round dropout 이 아닌, iterative data dropout 이라는 방법을 제안하고, 이 경우 노이즈에도 강하고 여러 강점이 있더라.

- 문서단위의 새로운 influence estimation 방식 제안.

## AI-Driven Shopping Search and Price Comparison for Billion-Scale Products (하종우, 최승권 님)

```
네이버 쇼핑에서는 다양한 판매자들이 등록한 상품과 가격 비교를 위해서 생성한 카탈로그를 서비스하고 있습니다. 발표의 전반부에서는 사용자의 검색어에 적합한 상품과 카탈로그르 제공하기 위한 세 가지 딥러닝 모델들에 대하여 소개합니다. 발표의 후반부에서는 실질적으로 동일한 상품을 정확하게 클러스터링 하기 위해서 개발한 다양한 딥러닝 모델들에 대하여 소개합니다. 그리고 이러한 과정에 있어서 10억 단위의 상품 스케일에 대응하기 위한 노력에 대해서 간략히 소개합니다.
```

- "Billion-scale similarity search with GPUs by Jeff Johnson, Matthjis Douze and Herve Jegou"

- 가격비교를 위한 카탈로그 매칭

- free form product information: 사람이 입력하기때문에 좋지 않다. 변수가 많다. 공통의 특징을 찾아내는게 어렵다.

- "End-to-end learning of Deep visual representations for Image retrieval by Albert Gordo et al. 2017"

## NAVER federated learning: challenges and opportunities (강경윤 님)

```
품질 향상을 위한 머신 러닝의 기본은 학습 데이터 확보인데, 최근 정보보호에 대한 관심이 높아지면서 데이터 확보에 한계가 생겼다. 본 과제는 연합학습(Federated Learning)을 통해 데이터 전송 없이 개별 장치에서 자체적으로 학습하고, 학습된 백터만을 이용해 새로운 모델을 만들어 냄으로써 문제를 해결한 과정을 소개한다. 연합학습을 위한 기술력과 서비스 적용 가능성을 검증하기 위해 자사의 키보드 애플리케이션에 이모지 추천 기능을 통해 실험을 수행하였다. 개발된 프레임워크와 모델링의 과정을 설명하고, 이 과정 중 발생한 이슈들을 해결하는 방법을 제시하였다. 또한, 사용자 배포 후 이용자 클릭에서 4배의 향상된 결과를 확인하였다. 연합학습을 통한 새로운 방법으로 데이터 전송에 대한 부담 없이 모델의 품질을 향상시킬 수 있는 방법을 제시하고자 한다.
```

- on device 에서 학습, 데이터를 보내지 않고, 학습된 벡터만 보냄

- 개인적으로 재밌었던 내용이라, 몇가지 질문을 했다.
- Q1: federated learning 을 할때 server 에 parent model 이 있을것이고, edge 에 child model 이 있을텐데, 학습된 child model 의 vector 값을 parent model 에 학습하는 경우에 비율은 어떻게 하는가?

- A1: 실험을 통해서 변경하고 있고, 아직 정답을 찾진 못했다. 
> 이후 슬라이드를 보니 0.95:0.05 와 같은.. heuristic 한 접근을 취하고 있었다

- Q2: edge device 에서 model 학습 과정을 거칠텐데, edge device 의 배터리 사용량 증가되는 정도에 대한 실험결과가 있는지? 
> 사용자의 동의를 받지 않았을것 같은 생각도 들었고, 법적인 것들도 물어보고 싶었지만 간접적으로 질문했다

- A2: 학습이 그렇게 오래 걸리진 않고, cpu core 를 1개만 사용하도록 강제하였다. 
> 학습 과정에서 리소스를 쥐어짜내는 방식은 아니었겠지만, 이 조차도 사용자 입장에서는(특히 배터리가 부족하다거나 하는) 알게되면 짜증나지 않을까 싶었다.


## NAVER Knowledge Graph Introduction (현동석 님)

```
이 세션에서는 지식그래프 기술에 대해 이야기합니다. 얼마나 다양한 분야에서 지식그래프가 사용되고 있고 네이버는 검색 영역에서 지식그래프를 어떻게 구축하고 연구하며 사용자에게 검색 결과로 제공하고 있는 지 살펴봅니다. 여러분은 이 세션을 통해 네이버의 지식그래프 연구가 어떤 단위로 나누어져 진행되고 있으며 사용하는 세부 기술에 대해 알 수 있습니다.
```

- 지식: 정보와 정보에 대한 이해(사람의 이해에서 기계의 이해로의 확장)
- 그래프: 관계의 수학적 표현

- data -> information -> knowledge

- 기존의 검색에서는 semantic 에 대해 따지지는 않음

- kg 에서는 semantics 를 따짐. 

- 지식그래프 모델링 단계에서 로그를 이용함
- 온톨로지를 미리 구축해놓고 비교/검증 을 이용함
    - reasoning, 새로운 확장 등
- knowledge retrieval

- 응답성/ 확장성/ 유연성
- 저장공간의 분산/ 엔티티 저장과 추출을 나눔
- 분산환경에서의 데이터 업데이트 시의 영향도를 파악중

- 필요한 연산의 종류에 맞게 서빙엔진을 사용하게됨.
    - 전통적인 검색방식 혹은, 지식그래프를 활용한 방법
- 영어중심이라, 한글 중심의 지식그래프 생성/연구가 필요함

## Knowledge Snippet: Direct Answer to a Search Query (신동욱 님)

```
지식 스니펫은 수많은 문서 중에서 사용자가 검색한 의도에 부합하는 정보를 자동으로 추출하여 보여주는 서비스입니다. 사용자의 질의에 대한 풍부한 정보를 검색 결과에 보여줌으로써 사용자가 원하는 정보를 좀 더 빠르게 확인할 수 있도록 검색 편의성을 제공해왔습니다. 지식 스니펫을 추출하는 과정은 크게 3가지 단계로 구성됩니다. 먼저 사용자의 검색 의도를 파악하여 지식 스니펫으로 대응할 질의인지 여부를 판단하고 질의와 연관도가 높은 문서를 검색합니다. 그 후 해당 문서에서 사용자의 검색 의도에 부합하는 정보를 추출합니다. 지식 스니펫은 본문형, 리스트형, 테이블형 3가지 유형으로 제공되고 각 유형을 추출하는 과정을 소개하도록 하겠습니다.
```

Query Processing
- Query Normalization:
    - subject/ property 로 구분함
- Answer Type Detection

Document Processing
- 웹 문서에 불필요한 부분 날리고, 본문영역에서 테이블/리스트/텍스트를 추출함

Snippet Extraction
- Table Snippet Extraction
    - 웹 문서에 있는 테이블이 유의미한 정보를 갖고 있는지 아닌지 추정함
    - 정보 테이블을 positive, 나열 테이블은 negative 로 판단함
- List Snippet Extraction
    - 정보를 포함한 리스트형 지식 스니펫 추출 (예를들어, 혼인신고 서류, 절차 관련)
- Paragraph Snippet Extraction
    - Passage Classification
    - Machine Reading Comprehension Model
    
Fact Checking
- 사용자 검색의도에 맞는 지식스니펫이 추출된건지에 대한 검증방법 연구가 필요함

## Conversational AI with Big Generative LM (김경덕 님)

```
NAVER의 Conversational AI는 스피커를 비롯한 스마트 디바이스, 네이버 앱, 차량 환경에 내장된 음성 대화 인터페이스이며, 우리는 이를 좀 더 자연스러운 대화가 가능하고, 좀 더 다양한 응답을 안정적으로 하게 할 수 있는 기술을 개발하고 있다. 이를 위해 거대한 생성 기반 언어모델을 이용하여 한 문장이 아닌 전체 대화를 이해하기 위한 대화 이해 모델, 거대 생성 기반 언어 모델을 제어하기 위한 기술, 그리고 semantic search를 이용한 시스템 응답 선택 기술을 소개한다.
```

- BERT vs GPT(Encoder 만 vs Decoder 만 이용)
    - GPT: 자연어 생성, 요약 문제에서 활용

- 주어진 입력에 기반한 이후의 텍스트를 생성함
- 보다 자연스럽고, 많은 지식을 응답할 수 있는 능력
- few shot learning 기반의 생성모델
- 텍스트만 봤을때는 문제가 없어보여도, 사용자의 컨텍스트에서 해당 결과를 받으면 문제가 생김
- plug and play LM by Uber
- objective, chitchat, question answering 세가지 파트로 나눠서 re-ranker 를 이용해서 뽑아내는 구조
- 레이턴시가 너무 빨라서 신기했음

## Meta-learning for the low-resource domain adaption (박천복 님)

```
본 발표에서는 Meta-learning을 활용하여 병렬 코퍼스 없이 적은 도메인의 데이터만 주어진 도메인 번역에 대해서 효과적으로 할 수 있는 알고리즘을 연구한 결과를 소개한다. 추가적으로 Meta-learning 알고리즘을 변형하여 참고할 수 있는 많은 양의 다른 도메인 데이터로부터 얻을 수 있는 지식을 명시적으로 파라미터에 넣도록 하고 파인튜닝시에 모델의 과적합을 방지하도록 변경을 하였다. 이를 통해 다양한 도메인에 대해서 실험적으로 좋은 성능을 보여준다. 추가적으로 개선된 버전의 Meta-learning 알고리즘은 더 빠르게 학습되고 더 빠르게 파인튜닝이 가능하다.
```

- 예를들어 한국-네팔어 의료도메인 번역은 희소한 코퍼스이다. 이런경우에는?

- 일단 모노링구얼 코퍼스를 찾아서 학습을 시킨다

- 도메인 공통 knowledge, 도메인 특정 knowledge 두가지를 가지고 meta learning 

- meta learning


## Search Applications with Big Generative LM (김선훈 님)

```
생성 기반의 대형 언어 모델을 검색 관련 다양한 어플리케이션으로 적용해 본 경험을 소개합니다. 특히, 기존의 사전 학습된 언어 모델에 대한 fine-tuning 방식이 아닌, in-context few-shot learning 방식으로도 의미 있는 결과물이 나온다는 것과, 각 태스크 별 성능을 높이기 위해 시도했던 경험을 공유합니다. 추가적으로, 생성 기반의 모델이 가지고 있는 문제점과, few-shot learning 방식이 가지고 있는 한계점에 대해서도 공유를 드리려고 합니다.
```

- In-context Learning

- fewshot?

- Null-Query Rewriting:
    - 결과가 없는 검색어: null-query
    - 오타가 있거나, 띄어쓰기나, 자소단위 섞이거나, 잘못된 정보가 있는경우
    - gpt3 를 이용해서 null-query 로 변환하는
    - null-query 만으로는 판단하기 힘들어서, prev-query (동일 세션 내의) 를 함께 전달함









