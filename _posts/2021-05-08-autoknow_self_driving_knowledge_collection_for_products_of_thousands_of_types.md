---
layout: post
title: "[논문읽기] AutoKnow: Self-Driving Knowledge Collection for Products of Thousands of Types"
date: 2021-05-08 20:28:00 +0900
categories: [Paper]
tags: [ir, knowledge-graphs]
toc: true
comments: true
---

# pre-script

논문을 평하고 완벽하게 이해할 정도의 지식을 갖고있진 못하고, 논문 읽기에 익숙해지고 새로운 내용에 대한 공부와 이를 통해 알게된 내용 등을 스스로를 위해 간단하게 정리하는 의미가 크다. 아래 내용의 대부분은 읽은 논문의 해석이다.

따라서 본 글에는, 잘못된 번역과 잘못된 이해/정리의 내용을 포함할 수 있음을 미리 밝힌다.

# introduction

우연한 기회에 괜찮은 paper[(link)](https://assets.amazon.science/f7/d5/1c452719458b8803d1e397afd535/autoknow-self-driving-knowledge-collection-for-products-of-thousands-of-types.pdf) "AutoKnow: Self-Driving Knowledge Collection for
Products of Thousands of Types" 를 추천받아 읽게 되었다. 

Amazon 에서 낸 논문이고, 제품 도메인의 지식그래프(knowledge graph)를 자동으로 구축하려는 시도인 자사의 프로젝트 AutoKnow 에 대한 내용이다. 

ontology 와 같은 지식그래프는 보통 스포츠, 음악, 영화 와 같이 데이터가 잘 정리/구축 되어있는 도메인을 가지고 구축을 하고, 수동으로 전문가에 의해 생성이 된다. 그런데 Amazon 은 물건을 팔아야 하니, 제품 도메인(product domain)에 대해 지식그래프를 구축하고 싶은데, 제품의 종류도 너무 다양하고 참고해야되는 제품 프로필의 데이터 포맷도 제각각, 믿을 수 없는 정보들인 경우, 노이즈도 많고, 제품 프로필이 빈번하게 변경되고, ontology 생성 후의 entity 도 금방금방 바뀌게 되는.. 등등의 수많은 이유들로 성능이 잘 안나왔나 보다. 이런 데이터 자동화 하는 시도를 하게 되었고, 이것이 `AutoKnow` 라는 프로젝트이다. 이 프로젝트는 크게 `Data Imputation`, `Data Cleaning` 두가지 스텝으로 product profile 의 data sparsity 를 data imputation 과정을 통해서 채워서 유의미한 데이터들을 생성하고(정확한 방법론은 논문에 설명되어있지만 이해하지 못했다), 이후에 이러한 데이터들이 적절한지에 대해 검증할때 이상한 데이터들은 data cleaning 과정을 통해 배제하는 방식이다. 

회사 동료 중에 ontology 에 관심 많은 사람이 있어 이야기를 해보면, ontology 에서는 reasoning 이라는 과정을 통해 구축된 ontology triplets 가 적합한지를 검토하는 과정이 있다고 하는데, AutoKnow 방식으로 어떻게든 ontology 구축을 자동화 한다면 그 지식그래프의 유효성과 적합성을 판단해낼 수 있지 않을까 하는 생각이 든다. 물론 그 양이 너무 방대해서 매번 모델을 갱신하고 검증하는데 성능 이슈가 크게 발생할 것 같긴 하다.

제품의 상위레벨 엔티티 정도의 정보만을 manual 하게 생성한 ontology 를 구축해놓고, few-shot 처럼 제한된 어느 정도의 context 를 전달하는 방법론으로 자동화를 좀 더 수월하게 하거나 하는 방법은 없을까 가볍게 생각해봤다.

AutoKnow 를 이용해서 ontology 구축도 2.9배 확장하고 precision, recall 도 유의미하게 향상됐다는 얘기를 보면 대단한 효과로 보인다.

# 내용정리

## 1. Introduction

- `"어떻게 고품질의 포괄적인 knowledge graph 생성을 자동화 할것인가는 최근 몇년간 산업과 연구분야의 뜨거운 감자였다"`

주로 music, movie, sport 도메인에서 knowledge curation 이 성공적이었다.
- 원래 데이터가 풍부하고, 구조가 잘 잡혀있다
- 이미 잘 정리된 데이터 소스가 있다.

- 해당 도메인의 내용 복잡도가 처리할만하다.
- 수작업으로 ontology 를 구축하는데도 몇주면 가능하다.
- retail product(소매 제품) domain 에서는 힘들다. (워낙 많을테니)

- C1-Structure-sparsity: 영화쪽은 작성자가 데이터를 잘 입력하더라도, 판매자들이 편의를 위해(본인들이 귀찮아서) 데이터 작성을 잘 안하는 경우가 많음.

- C2-Domain-complexity: 도메인이 훨씬 복잡함, 관계 종류가 다양함 (sub-types, synonyms, overlapping types). 제품 특징들이 편차가 큼(TV vs 개밥). 특징들이 진화하기도 함(tv 에 wifi 기능이 있게될줄이야). 예전 ontology 를 계속 유지할 수 없으니, 자동화하는 솔루션이 필요하다.

- C3-Product-type-variety: 제품 종류가 많아서 모델 학습 등이 어렵다.

자율주행이 적은 사람의 개입으로 환경을 이해하는 방식처럼, AutoKnow 도 그렇고, 아래의 특징이 있다.

- Automatic
    - 적은 수동 작업이 필요함
- Multi-scalable
- Integrative

AutoKnow 를 사용해서 11K 의 제품타입과 1B 의 제품 knowledge fact 를 수집했고, 기존 ontology 를 2.9배 확장했고, precision 7.6, recall 16.4% 향상했다.

- retail domain 에 풍족한 사용자 행동 로그(customer behavior log) 를 활용해서 ontology 구축을 자동화 하는 컨셉인것같음.

## 2. Definition and system overview
### 2.1 Product Knowledge Graph

KG 는 set of triples in the form of (subject, predicate, object) 이다.

node 두개를 predicate(edge) 로 잇는 그래프라고 생각하면 된다.

### 2.2 System Architecture

- Ontology Suite:
    - Taxonomy(분류) enrichment
    - Relation discovery
- Data suite:
    - Data imputation(누락된 데이터 대신 값을 채우는 방법)
    - Data cleaning(data imputation 잘못된걸 지운다)
    - Synonym discovery

## 3. Enriching the ontology
### 3.1 Taxonomy Enrichment

- 전문가들이 online catalog 분류를 유지보수 하는데, 공수가 많이들고, 확장도 힘들다. 어떻게 새로운 제품타입을 발견하고, 기존 제품 분류에 어떻게 추가하는지가 어려운 과제이다.

### 3.2  Relation Discovery
- 고객의 쇼핑 결정에 영향을 많이 주는 요소는 별로 없다. 
- 과자 쇼핑에 브랜드가 영향은 많이주겠지만, 과일에선 그렇지 않다.
- 수천가지 타입중에서 어떤 특징이 주요한 영향을 주는지 찾아내는것이 중요한 문제다.

## 4. Enriching and cleaning knowledge
### 4.1 Data Imputation

- Problem definition:
C1-Structure-sparsity, C3-Product-type-variety challenge 를 해결해야된다. 

BiLSTM, CRF 를 이용한 sequential labeling 을 이용했다.

- Key techniques: 
특정 제품 타입에 기반한 예측을 하는 taxonomy-aware sequence tagging 방식을 제안했다. 

- Component Evaluation: 

### 4.2 Data Cleaning

- Problem definition:
retailer 들에 의해 제공된 데이터들은 잘못 이해했거나 제품 특징에 대한 의도적인 abuse 로 주로 error 에 취약하다. anomalies 를 감지하고 지우는것이 C1-structure-sparsity challenge 를 해결하는데 중요한 측면이다. Data imputation 과 비슷하게, C3-Product-type-variety challenge 수백만 타입들에 가깝게 하는것이 핵심이다. 다르게 말하면, "많은 숫자의 제품 타입들을 위한 제품 프로필 잘못된 데이터 일관성없음을 알아챌 것인가?" 라는 질문에 대답할 수 있어야된다.  

- Key techniques:
직관적으로 제품 프로필에 제공된 contexts 들과 attribute values 는 일관성이 있어야된다. 글로된 제품 프로필과 multi-head attention 방법을 통해 triple (PID, A, V) 가 유효한지를 따진 제품 분류 T 를 처리하는 transformer 기반의 neural net model 을 제안한다. 이러한 방법으로 대규모의 feature engineering 이 필요없는 원문의 텍스트 input 으로 부터 학습 가능하여 수천개의 타입들로 scaling 하는데 적합하다. 

cleaning model 을 학습하기 위해서, input 카탈로그로부터 학습 레이블을 자동생성하기 위한 distant supervision 을 도입했다. 우리는 여러 브랜드들에서 나타나는 고빈도의 값들의 예제를 생성하였고, 각각의 positive 예제에서 negative 예제를 만들기 위해 임의로 세 종류의 절차 중 하나를 적용했다. 1) 임의의 단어로 attribute 값을 치환하는것, 2) 임의로 제목에서 n-gram 을 선택, 3) 다른 attribute 의 값을 임의로 선택해서 바꿔치운다. 

- Component Evaluation:

### 4.3 Synonym Finding
synonym 분류
- spelling variants (Reese's vs reese)
- acronyms or abbreviation (FTO vs fair trade organic)
- semantic equivalence (lite vs low sugar)

방법에는 두가지 단계가 있는데, 첫번째는 높은 유사도를 갖는 제품 쌍을 얻어내는데 고객들이 함께 본 지표에 collaborative filtering 을 적용해서 그런 attribute 값들을 synonym 이 될 후보로 사용한다. 이러한 후보군 집합은 noise 가 많아, 많은 필터링이 필요하다. 두번째로, 후보쌍이 일치하는 의미를 갖는지 판단하기 위한 간단한 logistic regression 모델을 학습시킨다. 사용하는 features 에는 edit distance, pre-trained MT-DNN model score, 그리고 distinct vs common words 를 반영한 피쳐들을 사용한다. `features regarding distinct vs common words` 는 모델에서 핵심적인 역할을 하는데, 단어들의 세가지 쌍에 초점을 맞춘다: 첫번째 후보군에는 나오지만 두번째 후보군에는 나오지 않거나 그 반대인 경우, 그리고 두 쌍에 다 나오는 경우이다. (A U B 로 이해하면 될 것 같다) 세 쌍중 모든 두개 사이에 계산된 edit distance 와 embedding similarity 는 계산되어 features 로 사용된다. 

## 5. Experimental results

### 5.1 Input data and resulting graph

- RawData:
AutoKnow 는 제품타입 분류를 담은 Amazon 의 제품 카탈로그를 input 으로 사용한다. 식료품/건강/미용/유아 4가지의 도메인에서 제품을 선택한다. 이 도메인들은 sparse 한 데이터를 가지고 있지만 타입의 종류와 크기는 도메인 별로 차이가 있다. 우리는 임의로 선택된 달에 페이지에 나타난 제품들(빈도 높은 상품들을 의미하는듯)을 고려한다. Table 7 은 도메인들의 통계를 보인다. 각각의 도메인의 카탈로그 분류에는 수십만개의 종류가 있고, 타입별 제품들의 중앙값은 수천 수만 이다. 아마존 카탈로그는 수천가지의 attributes 를 갖지만, 각각의 제품에 거의 적용되진 않는다(해당하는 attribute 편차가 크다는 뜻). 따라서, 각각의 제품에 대해 보통 수십, 수백개의 값들이 있다. 해당 유형의 제품 중 하나 이상에 해당 속성에 대한 카탈로그 값이있는 경우 속성이 유형에 포함된다고 말한다. 통계에서 볼 수 있듯이 각 도메인에는 수천 개의 속성이 포함되며 제품 유형 당 속성의 중앙값은 100-250 이다.

- Building a Graph:
우리는 Apache Spark 2.4 를 Amazon EMR cluster 에 사용하고, python 3.6 을 각각의 호스트에 사용하는 분산환경에 AutoKnow 를 구현했다. 딥러닝은 텐서플로를 사용했고, AWS Deep Graph Library 는 AK(AutoKnow)-Taxonomy 의 그래프 신경망 네트워크를 구현하기 위해 사용했다. AK-Relations 는 Spark ML 을 사용해서 구현했다. AK-Imputation 은 AWS SageMaker 인스턴스를 학습에 사용했다.

Resulting PG(Product Graph):  
- Table 8 에는 우리가 만든 제품그래프의 주요 특징들을 다룬다. 표는 4개의 제품 도메인의 통합된 통계를 표현한다. 제품 타입 추출 이후에 6.7K 의 분류를 19K 로 2.9배 늘렸다. 몇몇 타입들은 다른 도메인에서 나타났고, 11K 의 유일한 타입들이다. 다음 섹션에서 structured knowledge 의 성능이 얼마나 향상됐는지를 다룬다.


### 5.2 Quality Evaluation

- Metrics: 
knowledge 의 precision, recall, F-metric 을 계산했다. 새로운 defect rate 라는 메트릭을 정의했는데, 이는 잘못되거나 누락된 (product, attribute) 쌍의 비율이다. 
세 종류의 triples 를 고려한다: 1) 제품타입을 갖는것(product-1, hasType, Television), 2) attribute 값을 갖는것 (product-2, hasBrand, Sony), 3) entity 간의 관계를 정의하는 것 (chocolate, isSynonymOf, choc). 우리는 각각의 타입의 precision 을 계산한다. recall 을 계산하는것은 힘든데, 특히 type triples 와 relation triples 는 모든 적합한 type 과 synonyms 를 찾아내는것이 불가능하기 때문이다. 때문에, attribute values 의 triples 만 계산한다. 제품의 인지도(popularity) 에 기반한 샘플링을 진행했다. 

- Type triples: 
Table 9 는 MTurk 근무자들이 각 도메인의 300개의 레이블에 측정한 제품 타입 triples 의 품질을 보여준다. MoE 는 confidence 95% 의 margin of error 이다. AutoKnow 는 평균 87.7% 의 precision 과 평균 2.9배 늘어난 제품 타입 숫자를 얻었다.

- Attribute triples: 
...
data cleaning 이 어떻게 data 품질을 높였는지 강조하기 위해, table 11 에는 table 10 에서 noisy value 를 없앤 결과를 나타냈다. 90% 의 precision 과 Attribute 1 에 대해 73.7 %, Attribute 2 에 대해 27.8 의 recall 을 얻었다. 합쳐서 2 attributes 에서 1.3M 의 values 를 지운것이다.

- Relation triples:
마지막으로, Table 12 에는 relation triple 을 나타내는데, 이는 제품 타입간 hypernym 관계와 attributes 사이 synonym 관계를 다룬다. 

## 6. Lessons we learnt

- Non-tree product hierarchies: 
    - multiple parents 를 놓쳤다, knife 는 cutlery 와 hunting kits 양쪽의 subtype 일 수 있다.
- Noisy data: 
    - 큰 볼륨의 노이즈는 imputation model 과 cleaning model 의 성능을 저해할 수 있다. 이거는 유아용 상품 도메인의 카탈로그에서의 적절하지 않은 많은 제품타입들에 의해, 저품질 지식에서 관측됐다. 우리는 모델 학습 전에 적극적인 data cleaning 을 제안한다. 
- More than text: 
    - 제품 프로필은 제품 지식의 유일한 source 가 아니다. 22 쌍의 랜덤하게 샘플링된 연구에서 71.3% 는 Amazon 제품 프로필에서, 추가로 3.8% 는 Amazon 제품 이미지에서, 24.9% 는 제조사 홈페이지 등과 같은 외부 소스에서 가져와야됐다. 이것은, AutoKnow 의 이미지 처리와 웹문서 추출을 강화하는게 중요하다는것을 암시한다.

## 7. Related work

산업에서의 지식그래프 시스템은 수작업으로 정의된 온톨로지와, 위키피디아를 비롯한 여러 구조가 잘 잡혀있는 소스들에서 엄선된 지식에 의존한다. 연구 시스템은 웹문서 추출을 수행하지만, 결국 다시 미리 정의한 온톨로지를 참고한다. 반면에 이 work 는 다양한 웹페이지에 다양한 노이즈 정도와 sparsity 를 갖고있는 제품 정보로부터 제품 knowledge 를 추출한다. 분류 지식을 기계 학습 모델에 통합하고 감독을 위해 고객 행동 신호를 활용하는 것은, 이 작업 전반에 걸쳐 모델 성능을 개선하기 위해 사용되는 두 가지 주제다.

End-to-end 시스템에 더하여, 독립적인 컴포넌트를 위한 해결책들이 있는데, 이는 온톨로지 정의, entity identification, 관계 추출, 관계 임베딩, 연결, 지식 결합 이다. 우리는 이러한 기술을 적절할때  제품 도메인에 특수한 문제 전달을 향상하기 위해 적용한다.

## 8. Conclusions

- 이 paper 는 knowledge graph 생성하는 과정의 경험을 다룸
- ontology 생성 자동화, 변경이 많은 수많은 제품들에 대한 knowledge enrichment/cleaning 에 ML 방법론들을 적용했다.
