---
layout: post
title: "[논문읽기] AutoKnow: Self-Driving Knowledge Collection for Products of Thousands of Types"
date: 2021-05-06 00:00:00 +0900
categories: [Article]
tags: [ir, knowledge-graphs]
toc: true
comments: true
---

[논문 링크](https://assets.amazon.science/f7/d5/1c452719458b8803d1e397afd535/autoknow-self-driving-knowledge-collection-for-products-of-thousands-of-types.pdf)

# 내용

## Introduction

How to automatically build a knowledge graph with comprehensive
and high-quality data has been a hot topic for research and industry
practice in recent years

주로 music, movie, sport 도메인에서 knowledge curation 이 성공적이었다.
- 원래 데이터가 풍부하고, 구조가 잘 잡혀있다
- 이미 잘 정리된 데이터 소스가 있다.

해당 도메인의 내용 복잡도가 처리할만하다
수작업으로 ontology 를 구축하는데도 몇주면 가능하다.
retail product(소매 제품) domain 에서는 힘들다. (워낙 많을테니)

Structure-sparsity: 영화쪽은 작성자가 데이터를 잘 입력하더라도, 판매자들이 편의를 위해(본인들이 귀찮아서) 데이터 작성을 잘 안하는 경우가 많음.

Domain-complexity: 도메인이 훨씬 복잡함, 관계 종류가 다양함 (sub-types, synonyms, overlapping types). 제품 특징들이 편차가 큼(TV vs 개밥). 특징들이 진화하기도 함(tv 에 wifi 기능이 있게될줄이야). 예전 ontology 를 계속 유지할 수 없으니, 자동화하는 솔루션이 필요하다.

Product-type-variety: 제품 종류가 많아서 모델 학습 등이 어렵다.

---
자율주행이 적은 사람의 개입으로 환경을 이해하는 방식처럼, AutoKnow 도 그렇고, 아래의 특징이 있다.

- Automatic
    - 적은 수동 작업이 필요함
- Multi-scalable
- Integrative

AutoKnow 를 사용해서 11K 의 제품타입과 1B 의 제품 knowledge fact 를 수집했고, 기존 ontology 를 2.9배 확장했고, precision 7.6, recall 16.4% 향상했다.

- retail domain 에 풍족한 사용자 행동 로그(customer behavior log) 를 활용해서 ontology 구축을 자동화 하는 컨셉인것같음.

## Definition and system overview
### Product Knowledge Graph

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
### 4.2 Data Cleaning
### 4.3 Synonym Finding
synonym 분류
- spelling variants (Reese's vs reese)
- acronyms or abbreviation (FTO vs fair trade organic)
- semantic equivalence (lite vs low sugar)

## 5. Experimental results
### 5.1 Input data and resulting graph

## 6. Lessons we learnt
- Non-tree product hierarchies: 
    - multiple parents 를 놓쳤다, knife 는 cutlery 와 hunting kits 양쪽의 subtype 일 수 있다.
- Noisy data: 
    - 

## 7. Related work

## 8. Conclusions
- 이 paper 는 knowledge graph 생성하는 과정의 경험을 다룸
- ontology 생성 자동화, 변경이 많은 수많은 제품들에 대한 knowledge enrichment/cleaning 에 ML 방법론들을 적용했다.
