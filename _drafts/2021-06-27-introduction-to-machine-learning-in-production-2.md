---
layout: post
title: Introduction to Machine Learning in Production (2)
date: 2021-06-27 20:10:00 +0900
categories: [Lecture]
tags: [mlops]
toc: true
comments: true
---

# introudction

`deeplearning.ai` 에서 제공하는 ML-Ops 관련 강의 [https://www.coursera.org/learn/introduction-to-machine-learning-in-production](https://www.coursera.org/learn/introduction-to-machine-learning-in-production) 를 들어보기로 했다.

이 내용은 과정의 Week 2 를 듣고 기억에 남는 부분들을 정리한 내용이다. 

사용된 이미지들의 출처는 위의 강의의 영상 캡처이다.

# contents

## 1. Selecting and Training a Model

AI System = Code(algorithm/model) + Data

model 개발에 있어서 training set 에서 좋은 결과를 내는것은 꼭 필요하다. 이후에 dev/test sets 에서 좋은 결과를 내는것과 business metrics/project goals 에 좋은 결과를 내는것이 있다. research team 에서는 보통 dev/test sets 에서 좋은 결과를 내는데 목적을 갖지만 이것이 business metrics 와 project goals 를 충족할것이라고 확신할 수는 없다.

### Why low average error isn't good enough

- test set 에서 좋은 결과를 보였다고 좋은 모델이고 바로 적용할 수 없다. business metrics 와 realworld problem 을 풀어내기에 올바르지 않을 수 있기 때문이다.

- performance on disproportionately important examples

`apple pie recipe`, `latest movies` 와 같은 `informational and transactional queries` 는 `the best` answer 를 줄 것을 기대 안하고 적당한 정보도 괜찮지만 `Youtube` 와 같이 정확한 정보를 기대하는 `navigational queries` 에서는 잘못된 정보, 틀린 정보과 심각하게 받아들여질 수 있다.

- performance on key slices of the dataset

신용대출 관련 문제에서 인종/성별/지역/언어 등으로 결과에 차별이 생기거나 하면 안된다.

- rare classes

skewed data distribution 인 경우가 있다. 애초에 희소해서 1%의 확률로만 발생한다면 무조건 negative 로 return 하는 모델은 accuracy 가 99% 나 될것이다.

`I did well on the test set` 이라는 말로 방어하지 말고, test set 에서 잘 동작하는것은 실제 문제 해결에 있어서 부족할 수 있다.

### Establish a baseline

Ways to establish a baseline
- Human level performance (HLP)
- Literature search for state-of-the-art/open source
- Quick-and-dirty implementation
- Performance of older system

간혹 근거없이 높은 baseline 이 그어진 경우에 스트레스만 받고 해내지 못하는 경우가 있을 수 있으니 시간을 갖고 타당한 baseline 을 확보한 뒤에 프로젝트를 진행하는게 좋다.

### Tips for getting started

- Literature search to see what's possible (courses, blogs, open-source projects)
- Find open-source implementations if available.
- A reasonable algorithm with good data will often outperform a great algorithm with no so good data.

연구 목적이 아니라면 최신 논문 찾아볼 필요 없이, 좋은 데이터를 포함한 구현체가 있다면 사용해보고 좋은 결과를 얻을 수도 있다. 

Deployment constraints when picking a model
- cpu 사용량, memory 제한 등을 미리 따져볼 필요가 있을까?
- 만약 baseline 이 준비되었고, build/deploy 해볼 계획이라면 필요하다.
- baseline 을 만들기 위함이고, 가능할지 판단해보는 정도라면 미리 계획해볼 필요 없다.

Sanity-check for code and algorithm
- 큰 데이터셋으로 학습을 시켜보기 전에, 작은 training dataset 을 학습시켜보는게 좋다.
- 한개에 대해서도 제대로 결과를 줄 수 없다면 학습시킬 필요가 없다.
- 빠르고 간편하게 버그를 찾아낼 수 있다.

## Error analysis and performance auditing

### Error analysis example

error analysis 를 통해서 할 수 있는 가장 핵심적인 것

> What's the most efficient use of your time in terms of what you should do to imrove your learning algorithm's performance.

## Prioritizing what to work on

4 가지 과제가 있다고 할때, Gap to HLP 와 해당 과제의 데이터 비율을 이용해서 어떤 과제의 성능 개선을 할지 정해볼 수 있다.

예를들어, speech recognition 과제에서 car noise 가 있는 데이터는 4% 의 Gap to HLP 를 갖지만 데이터가 전체에서 4% 의 비중을 차지한다면, 4%의 성능 개선은 0.04 * 0.04 = 0.0016 의 개선일 뿐이고, Clean speech 과제가 전체 데이터의 60%를 차지한다면 1% 의 성능개선 뿐이라도, 0.01 * 0.6 = 0.006 의 큰 개선이 일어난다.


수학 공식 같은건 없지만, 아래의 항목들을 따져서 우선순위를 정할 수 있다.

- How much room for improvement there is.
- How frequently that category appears.
- How easy is to improve accuracy in that category.
- How important it is to improve in that category.

### Skewed datasets

positive/negative ratio 가 50:50 과는 거리가 먼 dataset 을 skewed datasets 라고 한다.

skewed datasets 에서는 그냥 skewed 된 쪽으로 결과를 몰아버리면 accuracy 가 높은 시스템이 만들어진다. 물론, 원하는 결과는 아닐것이다.

Confusion matrix: precision and recall
- accuracy 보다는 precision 과 recall 을 최적화 하는 방식을 취한다.
- 만약에 결과를 항상 0 를 내는 모델이라면 precision 이 `0/(0+0)` 이고 recall 은 `0/(0+86)` 도 0 이기 때문에 좋은 모델이 아니다.
- 만약에 precision 과 recall 이 서로 하나씩 높은 모델 두개의 성능은 어떻게 판단하는가?: F1 score 을 사용해서 precision, recall 의 조화평균(harmonic mean) 값을 계산해서 비교해본다.
- 공장에서는 recall 값이 중요하게 작용한다.

### Performance Auditing

- Brainstorm the ways the system might go wrong.
- Establish metrics to assess performance against these issues on appropriate slices of data.

## Data Iteration

### Data-centric AI Development

Model-centric view 와 Data-centric view 가 있다.

이전에 academia 에서는 제한된 데이터인 benchmark data 를 가지고 모델의 성능을 높이는 작업들을 해왔었고, 이것이 model-centric view 에 부합한다. 데이터는 두고(`hold the data fixed`) 코드/모델을 개선하는 (`iteratively improve the code/model`) 방법론이다.

data-centric view 는, 데이터의 품질은 아주 중요하고 데이터 품질을 높일수 있는 툴을 사용한다면 다양한 모델들을 더 잘 동작하게끔 할 것이라는 관점이다. 따라서, 모델코드는 두고(`hold the code fixed`) 데이터의 품질을 높이는 방법이다.

### Data Augmentation

Goal: create realistic examples that the algorithm does poorly on, but humans do well on.

사람도 machine 도 둘다 못할 저품질 데이터를 만들지 말라.

Checklist:
    - Does it sound realistic?
    - Is the x -> y mapping clear?
    - Is the algorithm currently doing poorly on?

### Can adding data hurt?

