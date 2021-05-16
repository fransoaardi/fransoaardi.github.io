---
layout: post
title: Interview Study
date: 2021-05-09 00:00:00 +0900
categories: [Wiki]
tags: [engineering]
toc: true
comments: true
---

# introduction

처음 들어본, 들어는 봤지만 설명하라니 할 수 없는 이러한 것들을 내가 모르는 것으로 정의하고 가볍게 정리해볼 계획으로 시작하였다.

# keywords

- HashSet/HashMap/HashTable/Dictionary
    - 다 같은뜻 아닌가?
- Bloom Filter
- SLA
- word embedding
- parse trees
- LSTM
- attention
- bidirectional models
- beam search
- ML approach vs n-gram models

RL
- Markov Decision Processes
- Value functions
- Bellman equations
- Policy gradient
- Q-learning
- Actor-critic

- precision/recall trade off
- 

# contents

## bloom filter

블룸 필터(Bloom filter)는 원소가 집합에 속하는지 여부를 검사하는데 사용되는 확률적 자료 구조이다. 1970년 Burton Howard Bloom에 의해 고안되었다. 블룸 필터에 의해 어떤 원소가 집합에 속한다고 판단된 경우 실제로는 원소가 집합에 속하지 않는 긍정 오류가 발생하는 것이 가능하지만, 반대로 원소가 집합에 속하지 않는 것으로 판단되었는데 실제로는 원소가 집합에 속하는 부정 오류는 절대로 발생하지 않는다는 특성이 있다. 집합에 원소를 추가하는 것은 가능하나, 집합에서 원소를 삭제하는 것은 불가능하다. 집합 내 원소의 숫자가 증가할수록 긍정 오류 발생 확률도 증가한다.

reference: [https://ko.wikipedia.org/wiki/%EB%B8%94%EB%A3%B8_%ED%95%84%ED%84%B0](https://ko.wikipedia.org/wiki/%EB%B8%94%EB%A3%B8_%ED%95%84%ED%84%B0)

- 원소가 집합에 존재하는지 확인을 하는 확률적인 접근 방식
- 블룸필터 결과 있다고 했는데 실제로는 없을 수 있지만(FalsePositive, FP) 없다고 했는데 실제로 있을 수(FalseNegative, FN)는 없다. 
- 집합에 원소 추가 가능, 삭제 불가능
- 집합 크기 늘어날수록 FP 발생 늘어남

블룸 필터는 m 비트 크기의 배열 구조가 있고, k 가지 서로 다른 해시함수(f1, f2 ... fk)를 사용한다고 가정한다.

예를들어 k 를 3이라고 하면 추가하려는 원소 e1, e2, e3 에 대해 블룸필터 m 에 f1(e1), f2(e1), f3(e1) 의 결과 비트의 자리를 1로 변경하면, e1 의 경우 f1, f2, f3 의 결과가 각각 index 2, 3, 5 의 자리를 1로 만들었다면, 새로운 원소 e4 의 결과(f1(e4), f2(e4), f3(e4)) 도 2, 3, 5 index 가 나온다면 블룸필터 결과 해당 원소는 set 에 존재한다고 생각할 수 있다. 물론 

좋은 점은, 원소가 set 에 존재하는지 확인은 k 개의 해시 함수 계산과 그 index 가 모두 1인지만 확인해보면 간단히 구현 가능할것 같은데, 원소가 늘어날수록 FP 확률이 늘어날 것이라, 적절한 m 의 값과, 해시 함수갯수 및 해시함수 구현이 중요해보인다.

## SLA (Service Level Agreement)

- 서비스 수준 협약서
서비스 수준 협약서(Service Level Agreement)는 서비스를 제공함에 있어서 공급자와 사용자간에 서비스에 대하여 측정지표와 목표 등에 대한 협약서이다. 일반적으로 여기에 포함될 수 있는 서비스 측정치들은 CPU의 가용시간, CPU 응답시간, 헬프 데스크 응답시간, 서비스 완료시간 등이다.

결국 서비스 개발에 있어서 어느정도의 가용성을 보장해야되고, 어느정도의 latency, 배포시간 downtime 등.. 관련해서 고민할 부분이 많아보인다.

reference: [https://ko.wikipedia.org/wiki/%EC%84%9C%EB%B9%84%EC%8A%A4_%EC%88%98%EC%A4%80_%ED%98%91%EC%95%BD%EC%84%9C](https://ko.wikipedia.org/wiki/%EC%84%9C%EB%B9%84%EC%8A%A4_%EC%88%98%EC%A4%80_%ED%98%91%EC%95%BD%EC%84%9C)

## Word Embedding (워드 임베딩)

N 개의 word set 을 갖고있다면, sparse representation 

## Precision/Recall Trade Off

- reference: [https://sumniya.tistory.com/26](https://sumniya.tistory.com/26)

| estimation \ reality  | True | False |
| --- | --- | --- |
| Positive | TP(정답) | FP(오답) |
| Negative | TN(오답) | FN(정답) |

- Precision: 정밀도
    - model 이 true 라고 예상한것 중 실제 true 의 비율
    - TP / TP + FP
- Recall: 재현율
    - 실제 정답중 model 이 true 라고 한 비율
    - TP / (TP + FN)

- FP 를 낮춰서, 정말로 확실한것만 판단한다면 precision 이 높아질 수 있다.
- precision, recall 둘다 봐야된다. (상호보완적)

- F1 score: precision, recall 의 조화평균


