---
layout: post
title: "MLflow: Stanford MLSys Seminar Episode 2"
date: 2021-04-19 09:50:00 +0900
categories: [Seminar]
tags: [mlflow, mlsys]
toc: true
comments: true
---

# introudction

팀원들과 스터디 차원에서 일주일에 한편씩 보기 시작한 stanford 주최 mlsys seminar 의 내용 정리이다. 아래 내용 대부분은 [세미나 영상](https://www.youtube.com/watch?v=nCQ9WqXPIS4) 에서 제공하는 슬라이드쇼의 내용을 담았다.

이번 회차는 Matei Zaharia 라는 Spark 를 개발하고, Databricks 회사를 차린, mlsys seminar 의 고정패널인 사람의 세미나였다.

Databricks 에서 제공하는 MLflow 에 관한 내용인데, 직접 사용해보진 않았고(앞으로도 사용할것 같진 않다) 팀원 한명이 사용해본 적 있다고 했지만, 그마저도 초창기 버전이었다. 그래도 이전에 작성해놓은걸 보여줬는데, 느낌은 jupyter notebook 이 연동된 개발환경이고, 데이터 가공 과정에서 mlflow 에서 제공하는 hdfs 및 spark 를 사용할 수 있었다. 아래에서 얘기할 model 의 versioning 이라거나 API 를 REST 로 제공하는 등의 편의 기능에 대해서 확인해보진 못했지만, 어떤 생각을 갖고 이러한 툴을 제공하게 되었는지 알 수 있는 기회였다.

# references

https://www.youtube.com/watch?v=nCQ9WqXPIS4


# contents

ml 은 전통적인 소프트웨어와는 다른 방식을 따른다.

| # | traditional | ml |
| --- |--- | --- |
| goal | meet a functional specification | optimize a metric |
| quality | depends only on app code | depends on input data and tuning parameters |
| | pick one sw stack | compare & combine many libraries and algorithms for the same task |


ml platforms
- google tfx, fb fblearner, uber michelangelo

concerns:(all through inconsistency)
- data, experiment, model management 
- reproducibility, deployment for inference, test & monitoring 

platform 구축해놓으면 bottleneck 으로 작용할 수 있다. 예를들어 새로운 논문이 나와서, 빠르게 시도해보고 싶어도 platform 에서 tensorflow 로 제한해놨다면, 논문에서 pytorch 를 제공했다면, 실험해보지 못할 수 있다.

Based on an open interface design philosophy: make it easy arbitrary ML code & tools into the platform
- Simple command-line and REST APIs rather than environment-specific
- Easy to add to existing software


MLFlow 의 components

- Tracking: Experiment and data management
  - parameters, metrics, output files, code version 을 통해서 테스트 하면서 중간과정 잊어버리지 않고 관리한다.
- Projects: Reproducible execution
  - 프로젝트 구성을 해놓고, 환경 타지 않고 실행될 수 있도록 명세한다.
- Models: Model packaging and deployment
- Model Registry: Model management
  - 모델들을 잘 버저닝 해놓고, 현재 서빙중인 버전을 다른 모델로 쉽게 변환할 수 있는 ui 를 제공하는 부분이 인상적이었다.

MLflow 를 사용해본 팀원의 말(초기버전을 사용했어서 지금은 다를거라고 했다)에 따르면 mlflow 의 autolog 는 생각처럼 잘 동작하지 않았고, 직접 지정해서 로깅하는것이 필요했다고 한다. 또한, REST API 로 제공해주는 기능이 따로있는건 아니고, flask 로 API 지정 한 webserver 를 띄워주는 방식이라, container 화 해준다 정도로 이해하려고 한다.

Machine learning is being applied to critical problems in industry, but requires careful management when the stakes are high. ML Platforms are emergin abstraction to help with this, and an open interface is very important for broad usability

Expected use cases

- track experim during model design 
- track performance of continuous training, deployment pipelines
- deploy the same model for batch and real-time scoring 
- run pipelines deterministically in different envis
- ci/cd using model registry stages and apis 

---

domain 별로 따로 학습하고 싶어하는 니즈가 있음
서로 다른 도메인에서 privacy issue 가 발생할 수 있음
- 코카콜라의 데이터를 갖고 이메일 자동완성을 개발했더니, 펩시에서 기업기밀을 유출할 수 있는 상황

ml platforms are an emerging abstraction to help with this
- and an open interface is very important for broad usability

kubeflow vs mlflow
- kubeflow 는 deployment 에 치중되어있다. mlflow 는 사용성을 높이려고 한다.
> 하지만, kubeflow 가 점점 defacto 가 되어가는 느낌을 지울 수 없다.
