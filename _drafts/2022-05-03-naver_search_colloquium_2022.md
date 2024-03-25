---
layout: post
title: Naver Search Colloquium 2022 후기
date: 2022-05-03 18:00:00 +0900
categories: [Seminar]
tags: [ir]
toc: true
comments: true
---

# Introduction

- 

# Contents

## Smart block ranking
- 1 Best, N Best, N More 라는 개념으로 나눴음
- Deep Retriever -> Personalized Ranker 순으로 처리

- Deep Matching
    - Block2Query (Adding Expansion terms to blocks to improve recall)
    - SPLADE (modern IR model)

## modern web search engines
- 

---
SPLADE

- can we replace bm25 with machine learning?
- Dense model vs Sparse Model

White box analysis of ColBERT
- Is IR theory still useful?
    - IR Theory vs Pre-trained LMs(LMs work better but don't know what they do)
    - 

Splade
- first stage retriever
    - goal: infer sparse representation directly
- sparse regularization
    - count the number of activations
- Splade 가 다른 BM25, ColBERT, Tas-B 보다 더 좋은 결과를 나타냈다
- 

---