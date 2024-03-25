---
layout: post
title: elasticsearch 를 이용한 텍스트 기반 행정동/법정동 주소검색
date: 2021-02-17 10:44:00 +0900
categories: [Project]
tags: [elk]
toc: true
comments: true
---

# introduction

- 주소검색에 대한 니즈가 생겼다. 행정동에 매핑된 데이터가 있기 때문에, 쿼리된 주소에 가장 적합한 행정동을 조회해와야 된다.
- 사용자는 행정동인지 법정동인지를 구분하지 않고 쿼리할 것이기 때문에, 이를 처리해줄 필요가 있다.
  - 예를들면, `역삼동` 이라고 검색을 하지만, `역삼동`은 법정동이기 때문에, 행정동인 `역삼1동` 혹은 `역삼2동` 의 데이터가 필요하다.
- 좌표값을 이용할 예정은 아니고(이후에 랭킹을 위해서는 사용할수 있긴하다), 쿼리 텍스트로 변환하려고 한다.
- elasticsearch 에 데이터를 색인하고, 이를 검색하는 방식으로 접근하기로 했다.

# process

## brainstorm

- 처음에 시행착오가 있었는데, 이미 갖고있었던 행정동 데이터를 이용해서 구축하려고 했으나, 앞서 말했던 문제점들이 예상되었다.
  - `역삼동` 데이터를 fuzzy 하게 찾거나 `역삼/동` 으로 tokenize 한다면 `역삼1동` 을 조회해올 수 있을거라 생각했다.
  - 법정동인 `경상남도 양산시 신기동` 을 행정동인 `경상남도 양산시 삼성동` 으로 매핑해내려면, 행정/법정동 매핑데이터가 '반드시' 필요하다

## 데이터 획득

- 구글링을 조금 해보니, 특이하게 법정동/행정동 데이터를 행정안전부의 `주민등록,인감,행정사` 부분에서 [데이터](https://www.mois.go.kr/frt/bbs/type001/commonSelectBoardArticle.do?bbsId=BBSMSTR_000000000052&nttId=82856)를 구할 수 있었다.

- 말소코드가 포함된 데이터도 있지만, 말소된 지역명을 사용하진 않을 계획이라, `jscode20210219.zip` 데이터를 받아서 첨부된 excel 파일을 이용하기로 했다. (`KIKmix.20210219.xlsx`)

## 데이터 가공

- `KIKmix.20210219.xlsx` 파일을 `pandas` 로 가공하기 위해서 익숙한 `csv` 포맷으로 변환해서 저장했다.

- `KIKmix.20210219.csv` 는 아래와 같은 필드들이 있는데, 이중에서 불필요한 코드들은 제거하고, `시도명`, `시군구명`, `읍면동명`, `동리명` 만 필터했다.

```
시도명	시군구명	읍면동명	동리명
서울특별시			서울특별시
서울특별시	종로구		종로구
서울특별시	종로구	청운효자동	청운동
서울특별시	종로구	청운효자동	신교동
서울특별시	종로구	청운효자동	궁정동
```

- 계획은, `시도명/시군구명/읍면동명`을 이용해서 행정동을 가공하고, `시도명/시군구명/동리명` 을 이용해서 법정동을 가공하는 것이었다. 하지만 이것은 잘못된 방법이고, 아래에 계속된다.

- `동리명` 이 어떻게 끝나는지 확인하고싶어서 출력해봤는데, 특이한 데이터를 발견했다, 이 데이터는 cleansing 대상이다.

```python3
dl = raw['동리명'].values.tolist() 
print(set([v[-1] for v in dl])) # '군', '동', ')', '구', '리', '시', '로', '가', '면', '읍'
```
    - `['기암리(岐岩)', '기암리(基岩)', '화산리(華山)', '화산리(花山)', '평사리(坪沙)', '평사리(平沙)']`

- 위에서 확인할 수 있듯, 법정동 주소가 될 `동리명` 필드는 `시군구읍면동로가리` 이렇게 광범위한 데이터가 있어서, 정리할 기준이 필요했다.

- 우선 아래와 같은 기준을 설정했다.
```
시, 군, 구 > 읍 면 동 == 로, 가 > 리 
(로, 가) 는 법정동에서만 등장
``` 

- 따라서, `시군구`는 제거하고, `읍면동로가` 는 그대로 이용했고, `리` 의 경우에는 `읍면동`명을 `동리명` 앞에 추가해주기로 했다.
    
- 혹시나, `동리명`이 `읍 or 면` 으로 끝나는 경우, `읍면동`명과 값이 다른 필드가 있을까 하여 확인해봤는데, 다행히 그렇지 않았다.

```python
print('행/법 다른 면', raw[(raw['읍면동명'].str.endswith('면')) &
                (raw['읍면동명'] != raw['동리명']) &
                (raw['동리명'].str.endswith('면'))])

print('행/법 다른 읍', raw[(raw['읍면동명'].str.endswith('읍')) &
                (raw['읍면동명'] != raw['동리명']) &
                (raw['동리명'].str.endswith('읍'))])
```

## 데이터 색인

- 데이터 가공은 했고, 색인을 위한 elasticsearch index 를 생성하였다.

- 주소에 더 적합할 tokenizer 가 있다면 더 좋았겠지만, 일단은 whitespace 로 tokenize 하는 `standard` 보다는 `nori_tokenizer` 가 더 나을것이라고 판단했다.

- nori_tokenizer 는 `mixed` 로 설정해서, tokenize 되더라도 원형도 추가해서 색인하도록 했다.
- user_dictionary 와 synonym 을 적극 활용했다.

```json
{
    "settings": {
        "index":{
            "analysis": {
                "tokenizer": {
                    "custom_tokenizer": {
                        "type": "nori_tokenizer",
                        "decompound_mode": "mixed",
                        "user_dictionary": "userdict_hb_search.txt"
                    }
                },
                "analyzer": {
                    "custom_analyzer": {
                        "type": "custom",
                        "tokenizer": "custom_tokenizer",
                        "filter": ["do_synonyms"]
                    }
                },
                "filter": {
                    "do_synonyms": {
                        "type": "synonym",
                        "synonyms_path": "synonyms_hb_search.txt"
                    }
                }
            }
        }
    },
    "mappings": {
        "properties": {
            "hj_addr": {
                "type": "text",
                "fields": {
                    "analyzed": {
                        "type": "text",
                        "analyzer": "custom_analyzer"
                    }
                }
            },
            "bj_addr": {
                "type": "text",
                "fields": {
                    "analyzed": {
                        "type": "text",
                        "analyzer": "custom_analyzer"
                    }
                }
            }
        }
    }
}
```

### synonym 과 user_dictionary

- synonym, user_dictionary 설정에 대한 내용은 elasticsearch 의 document 를 참고하면 된다.
- synonym, user_dictionary 를 사용한 이유는, 위에서 가공한 데이터는 '서울특별시' 와 같이 정형화된 데이터였고, 이를 tokenizer 가 `'서울'+'특별시'` 로 나누게 될 것이기 때문이다.
- 추가로, synonym 설정을 `"서울특별시, 서울"` 와 같이 한다면, `서울특별시` 가 합성어이고, 합성어중 하나인 `서울` 까지 synonym 으로 등록을 하면 error 가 발생되는 [issue](https://github.com/elastic/elasticsearch/issues/37751) 가 github 에 등록되어있다.
- 이 에러를 우회하는 방법으로, user_dictionary 에 등록해서 tokenize 하지 않도록 처리했다.
- 아래는 `"제주특별자치도, 제주, 제주도"` 를 등록할때의 문제가 발생한 로그이다.

```
{"error":{"root_cause":[{"type":"illegal_argument_exception","reason":"failed to build synonyms"}],"type":"illegal_argument_exception","reason":"failed to build synonyms","caused_by":{"type":"parse_exception","reason":"Invalid synonym rule at line 17","caused_by":{"type":"illegal_argument_exception","reason":"term: 제주도 analyzed to a token (제주) with position increment != 1 (got: 0)"}}},"status":400}
```

- synonym 작성

```
서울특별시, 서울
제주특별자치도, 제주, 제주도
(...)
```

- userdict 작성

```
서울특별시
제주특별자치도
제주도
(...)
```

### 데이터 index 에 저장

- elasticsearch python library 를 사용했고, pandas 의 dataframe 을 쉽고 빠르게 bulk 로 넣는 방법이 있다.

```python3
es = Elasticsearch(hosts=['es.host.address:9200'])  # es server
bulk(es, documents, index='hb_search', doc_type='_doc', raise_on_error=True)
```

## 데이터 조회

- 데이터 조회를 하기 위해 python elasticsearch library 를 이용했다.
- `multi_match` 를 이용하는게 가장 적합해 보이고, `type` 값을 여러가지로 실험해볼만하다.
- 일단, 구현 목적상 `hj_addr.analyzed`, `bj_addr.analyzed` 두개 필드에 대해 match 를 확인하는게 필요하다.
- 지금은 행정/법정 주소 필드만 갖고있는데, 다른 필드들을 추가하고, score boosting 을 변경해보는것도 방법일 것 같다.
- 다른 타입은 테스트 해보지 않았고, 기본값인 best_fields 와 most_fields 타입을 사용해보니, `'목동'` 쿼리의 예에서 볼 수 있듯, 행정/법정동 명에 동일하게 나타나는 경우 score 가 가장 높아질 수 있어, geo-spatial query 를 해도 `'양천구 목동'`을 얻어올 수 없을수도 있다고 생각했다. (물론 geo-spatial query 의 score 에 가중치를 더 주면 해결은 될 것이다)
    > `'목동'`: 중의성, `'대전광역시 중구 목동'` 행정동, 법정동 모두 `'목동'` 이라, score 가 가장 높게나옴.

- 참고: [multi-match 관련 es document](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-match-query.html)

```python
es = Elasticsearch(hosts=['es.host.address:9200'])  # es server

d = {
    "query": {
        "multi_match": {
            "query":      "역삼동",
            "type":       "most_fields",
            "fields":     ["hj_addr.analyzed", "bj_addr.analyzed"],
        }
    }
}
res = es.search(index="hb_search", body=d)
print(f'Got {res["hits"]["total"]["value"]} Hits:')
for hit in res['hits']['hits']:
    print(f"{hit}")
```

- result (원하는대로 나오긴 한다)

```
Got 4 Hits:
{'_index': 'hb_search', '_type': '_doc', '_id': '6sVqqXcBX30-OeLucJQ-', '_score': 25.360394, '_source': {'hj_addr': '서울특별시 강남구 역삼1동', 'bj_addr': '서울특별시 강남구 역삼동'}}
{'_index': 'hb_search', '_type': '_doc', '_id': '68VqqXcBX30-OeLucJQ-', '_score': 25.360394, '_source': {'hj_addr': '서울특별시 강남구 역삼2동', 'bj_addr': '서울특별시 강남구 역삼동'}}
{'_index': 'hb_search', '_type': '_doc', '_id': 'm8VqqXcBX30-OeLucqA4', '_score': 20.924463, '_source': {'hj_addr': '경기도 용인시 처인구 역삼동', 'bj_addr': '경기도 용인시 처인구 역북동'}}
{'_index': 'hb_search', '_type': '_doc', '_id': 'nMVqqXcBX30-OeLucqA4', '_score': 20.924463, '_source': {'hj_addr': '경기도 용인시 처인구 역삼동', 'bj_addr': '경기도 용인시 처인구 삼가동'}}

```

# 결과와 개선이 필요한 부분

- 아직 완벽한 솔루션이 아니다. 일단은 feasibility check 를 해보고 싶었던 부분인데, 접근 방식은 적절한것 같다.
- 형태소분석기(tokenizer), 동의어(synonym), 사용자사전(user_dictionary) 쪽에 개선할만한 부분이 있어보인다.
    - 도/시/구/동 등을 stopword 처리한다면 어떨까?
    - 동의어, 사용자사전 처리가 더 정교하게 된다면 어떨까?
- `'종로구 창신제1동'` 을 다른 dataset 에서는 `'창신1동'` 으로 사용할 가능성이 높아보인다. (기존 dataset 과 병합문제)
- 위의 `'역삼동'` 의 예에서, 중의성 문제가 발생한다. 이후에 대표 좌표를 매핑해서, geospatial querying 으로 re-rank 를 하는 방법이 필요해보인다.
- 도로명 처리의 경우 아직 데이터셋을 확보하지 못했고, 방법을 생각해보지 못했다.
    - 도로명의 경우, `'도로'`를 기준으로 하고, 도로가 행정경계를 넘나드는 경우가 있을것이라, 아직 조사해보진 않았다.

## 검증 예시

- `'목동'`: 중의성, `'대전광역시 중구 목동'` 행정동, 법정동 모두 `'목동'` 이라, score 가 가장 높게나옴.
- `'호곡리'`: 중의성, 원했던 데이터는 우정읍 까지인데, `'리'` 데이터까지 포함해서 대응할 수 있게함
- `'제주도 애월'`: 아직 `'제주'` 와 `'제주도'`를 synonym 처리를 안해서 그런지, `'제주 애월'`과 다르게 검색결과를 찾지 못함
- `'세종시'`: dictionary 추가 안해서 그런지, 검색결과가 상이함


# appendix

- elasticsearch 에반젤리스트인 [김종민님의 도큐먼트](https://esbook.kimjmin.net/06-text-analysis) 와 [지하철 노선 관련 한글 형태소분석 글](http://kimjmin.net/2019/08/2019-08-how-to-analyze-korean/)  참고하며 알게된 api 내용을 정리한다.

## analyze api

- 아래와 같이 query 에 대해 어떻게 analyze 될지에 대한 REST API 가 제공된다.
- api endpoint 를 보면 `hb_search` index 에 `_analyze` 를 한 것인데, index 명시 없이 es 에 _analyze 요청을 할 수도 있지만, `custom_analyzer` 는 `hb_search` index 에 생성한 analyzer 이므로, 분석이 안될것이고, `standard` 혹은 `nori` 와 같이 custom 하지 않은, plugin 으로 설치된 것들로만 분석 가능할것이다.
- 아래 예시는 `'서울특별시 관악구 신림1동'` 에 대한 분석 결과인데, 위의 index 생성시, `"decompound_mode": "mixed",` 로 설정했기 때문에, `'관악구'` 가 `'관악+구'`이기 때문에, `'관악'`, `'구'`, `'관악구'` 이렇게 3가지 token 이 생성된것을 확인할 수 있다.
- `'서울특별시'` 부분은 `user_dictionary` 에 포함된 단어라서 tokenize 되지 않았고, 아래 synonym 처리시, synonym 에 `서울특별시` 와 `서울` 을 동의어 처리 해놨기 때문에, `서울특별시`, `서울` 토큰이 둘다 생성된것을 확인할 수 있다.

```
curl 'http://es.host.address:9200/hb_search/_analyze' \
-H 'Content-Type: application/json' \
-d '{
    "analyzer": "custom_analyzer",
    "text": "서울특별시 관악구 신림1동",
    "explain": true
}' | jq

{
  "detail": {
    "custom_analyzer": true,
    "charfilters": [],
    "tokenizer": {
      "name": "custom_tokenizer",
      "tokens": [
        {
          "token": "서울특별시",
          "start_offset": 0,
          "end_offset": 5,
          "type": "word",
          "position": 0,
          "bytes": "[ec 84 9c ec 9a b8 ed 8a b9 eb b3 84 ec 8b 9c]",
          "leftPOS": "NNG(General Noun)",
          "morphemes": null,
          "posType": "MORPHEME",
          "positionLength": 1,
          "reading": null,
          "rightPOS": "NNG(General Noun)",
          "termFrequency": 1
        },
        {
          "token": "관악구",
          "start_offset": 6,
          "end_offset": 9,
          "type": "word",
          "position": 1,
          "positionLength": 2,
          "bytes": "[ea b4 80 ec 95 85 ea b5 ac]",
          "leftPOS": "NNP(Proper Noun)",
          "morphemes": "관악/NNP(Proper Noun)+구/NNG(General Noun)",
          "posType": "COMPOUND",
          "reading": null,
          "rightPOS": "NNP(Proper Noun)",
          "termFrequency": 1
        },
        {
          "token": "관악",
          "start_offset": 6,
          "end_offset": 8,
          "type": "word",
          "position": 1,
          "bytes": "[ea b4 80 ec 95 85]",
          "leftPOS": "NNP(Proper Noun)",
          "morphemes": null,
          "posType": "MORPHEME",
          "positionLength": 1,
          "reading": null,
          "rightPOS": "NNP(Proper Noun)",
          "termFrequency": 1
        },
        {
          "token": "구",
          "start_offset": 8,
          "end_offset": 9,
          "type": "word",
          "position": 2,
          "bytes": "[ea b5 ac]",
          "leftPOS": "NNG(General Noun)",
          "morphemes": null,
          "posType": "MORPHEME",
          "positionLength": 1,
          "reading": null,
          "rightPOS": "NNG(General Noun)",
          "termFrequency": 1
        },
        {
          "token": "신림",
          "start_offset": 10,
          "end_offset": 12,
          "type": "word",
          "position": 3,
          "bytes": "[ec 8b a0 eb a6 bc]",
          "leftPOS": "NNG(General Noun)",
          "morphemes": null,
          "posType": "MORPHEME",
          "positionLength": 1,
          "reading": null,
          "rightPOS": "NNG(General Noun)",
          "termFrequency": 1
        },
        {
          "token": "1",
          "start_offset": 12,
          "end_offset": 13,
          "type": "word",
          "position": 4,
          "bytes": "[31]",
          "leftPOS": "SN(Number)",
          "morphemes": null,
          "posType": "MORPHEME",
          "positionLength": 1,
          "reading": null,
          "rightPOS": "SN(Number)",
          "termFrequency": 1
        },
        {
          "token": "동",
          "start_offset": 13,
          "end_offset": 14,
          "type": "word",
          "position": 5,
          "bytes": "[eb 8f 99]",
          "leftPOS": "NNBC(Dependent noun)",
          "morphemes": null,
          "posType": "MORPHEME",
          "positionLength": 1,
          "reading": null,
          "rightPOS": "NNBC(Dependent noun)",
          "termFrequency": 1
        }
      ]
    },
    "tokenfilters": [
      {
        "name": "do_synonyms",
        "tokens": [
          {
            "token": "서울특별시",
            "start_offset": 0,
            "end_offset": 5,
            "type": "word",
            "position": 0,
            "bytes": "[ec 84 9c ec 9a b8 ed 8a b9 eb b3 84 ec 8b 9c]",
            "leftPOS": "NNG(General Noun)",
            "morphemes": null,
            "posType": "MORPHEME",
            "positionLength": 1,
            "reading": null,
            "rightPOS": "NNG(General Noun)",
            "termFrequency": 1
          },
          {
            "token": "서울",
            "start_offset": 0,
            "end_offset": 5,
            "type": "SYNONYM",
            "position": 0,
            "bytes": "[ec 84 9c ec 9a b8]",
            "leftPOS": null,
            "morphemes": null,
            "posType": null,
            "positionLength": 1,
            "reading": null,
            "rightPOS": null,
            "termFrequency": 1
          },
          {
            "token": "관악구",
            "start_offset": 6,
            "end_offset": 9,
            "type": "word",
            "position": 1,
            "positionLength": 2,
            "bytes": "[ea b4 80 ec 95 85 ea b5 ac]",
            "leftPOS": "NNP(Proper Noun)",
            "morphemes": "관악/NNP(Proper Noun)+구/NNG(General Noun)",
            "posType": "COMPOUND",
            "reading": null,
            "rightPOS": "NNP(Proper Noun)",
            "termFrequency": 1
          },
          {
            "token": "관악",
            "start_offset": 6,
            "end_offset": 8,
            "type": "word",
            "position": 1,
            "bytes": "[ea b4 80 ec 95 85]",
            "leftPOS": "NNP(Proper Noun)",
            "morphemes": null,
            "posType": "MORPHEME",
            "positionLength": 1,
            "reading": null,
            "rightPOS": "NNP(Proper Noun)",
            "termFrequency": 1
          },
          {
            "token": "구",
            "start_offset": 8,
            "end_offset": 9,
            "type": "word",
            "position": 2,
            "bytes": "[ea b5 ac]",
            "leftPOS": "NNG(General Noun)",
            "morphemes": null,
            "posType": "MORPHEME",
            "positionLength": 1,
            "reading": null,
            "rightPOS": "NNG(General Noun)",
            "termFrequency": 1
          },
          {
            "token": "신림",
            "start_offset": 10,
            "end_offset": 12,
            "type": "word",
            "position": 3,
            "bytes": "[ec 8b a0 eb a6 bc]",
            "leftPOS": "NNG(General Noun)",
            "morphemes": null,
            "posType": "MORPHEME",
            "positionLength": 1,
            "reading": null,
            "rightPOS": "NNG(General Noun)",
            "termFrequency": 1
          },
          {
            "token": "1",
            "start_offset": 12,
            "end_offset": 13,
            "type": "word",
            "position": 4,
            "bytes": "[31]",
            "leftPOS": "SN(Number)",
            "morphemes": null,
            "posType": "MORPHEME",
            "positionLength": 1,
            "reading": null,
            "rightPOS": "SN(Number)",
            "termFrequency": 1
          },
          {
            "token": "동",
            "start_offset": 13,
            "end_offset": 14,
            "type": "word",
            "position": 5,
            "bytes": "[eb 8f 99]",
            "leftPOS": "NNBC(Dependent noun)",
            "morphemes": null,
            "posType": "MORPHEME",
            "positionLength": 1,
            "reading": null,
            "rightPOS": "NNBC(Dependent noun)",
            "termFrequency": 1
          }
        ]
      }
    ]
  }
}
```

## termvector api

- 위 analyze api 처럼 query 를 직접 analyze 해보는 방법도 있지만, 기존에 색인된 document 를 확인하고 싶을때, `_termvectors` api 를 이용할 수 있다.
- url 의 query 부분을 보면 fields 에 `,`로 여러 필드를 요청할 수 있다.
- 아래 예시의 `QsVqqXcBX30-OeLufub3` 는 document 의 `_id` 이다. 

```
curl 'http://es.host.address:9200/hb_search/_termvectors/QsVqqXcBX30-OeLufub3?fields=bj_addr.analyzed' | jq
  
{
  "_index": "hb_search",
  "_type": "_doc",
  "_id": "QsVqqXcBX30-OeLufub3",
  "_version": 1,
  "found": true,
  "took": 0,
  "term_vectors": {
    "bj_addr.analyzed": {
      "field_statistics": {
        "sum_doc_freq": 229626,
        "doc_count": 21725,
        "sum_ttf": 232031
      },
      "terms": {
        "리": {
          "term_freq": 1,
          "tokens": [
            {
              "position": 6,
              "start_offset": 18,
              "end_offset": 19
            }
          ]
        },
        "시": {
          "term_freq": 1,
          "tokens": [
            {
              "position": 2,
              "start_offset": 10,
              "end_offset": 11
            }
          ]
        },
        "월령": {
          "term_freq": 1,
          "tokens": [
            {
              "position": 5,
              "start_offset": 16,
              "end_offset": 18
            }
          ]
        },
        "월령리": {
          "term_freq": 1,
          "tokens": [
            {
              "position": 5,
              "start_offset": 16,
              "end_offset": 19
            }
          ]
        },
        "읍": {
          "term_freq": 1,
          "tokens": [
            {
              "position": 4,
              "start_offset": 14,
              "end_offset": 15
            }
          ]
        },
        "제주": {
          "term_freq": 2,
          "tokens": [
            {
              "position": 0,
              "start_offset": 0,
              "end_offset": 7
            },
            {
              "position": 1,
              "start_offset": 8,
              "end_offset": 10
            }
          ]
        },
        "제주도": {
          "term_freq": 2,
          "tokens": [
            {
              "position": 0,
              "start_offset": 0,
              "end_offset": 7
            },
            {
              "position": 1,
              "start_offset": 8,
              "end_offset": 10
            }
          ]
        },
        "제주시": {
          "term_freq": 1,
          "tokens": [
            {
              "position": 1,
              "start_offset": 8,
              "end_offset": 11
            }
          ]
        },
        "제주특별자치도": {
          "term_freq": 2,
          "tokens": [
            {
              "position": 0,
              "start_offset": 0,
              "end_offset": 7
            },
            {
              "position": 1,
              "start_offset": 8,
              "end_offset": 10
            }
          ]
        },
        "한림": {
          "term_freq": 1,
          "tokens": [
            {
              "position": 3,
              "start_offset": 12,
              "end_offset": 14
            }
          ]
        },
        "한림읍": {
          "term_freq": 1,
          "tokens": [
            {
              "position": 3,
              "start_offset": 12,
              "end_offset": 15
            }
          ]
        }
      }
    }
  }
}
```

## index: analysis setting 관련

- `analysis` 필드는 `settings.index` 하위에 들어가고, `tokenizer`, `analyzer`, `filter`, `char_filter` 를 적용할 수 있다. 각각에 대한 자세한 내용은 위에서 언급한 김종민님의 도큐먼트와 es 공식 document 확인하는것이 좋다.
- 이해한 바로는, 결국 `analyzer` 를 정의하기 위함인데, `analyzer` 는 `char_filter`, `tokenizer`, `filter` 과정의 통칭이다.
- `char_filter -> tokenizer -> filter` 순으로 적용된다.

- 출처: [김종민님의 document](https://esbook.kimjmin.net/06-text-analysis/6.2-text-analysis)

> Elasticsearch는 문자열 필드가 저장될 때 데이터에서 검색어 토큰을 저장하기 위해 여러 단계의 처리 과정을 거칩니다. 이 전체 과정을 텍스트 분석(Text Analysis) 이라고 하고 이 과정을 처리하는 기능을 애널라이저(Analyzer) 라고 합니다. Elasticsearch의 애널라이저는 0~3개의 캐릭터 필터(Character Filter)와 1개의 토크나이저(Tokenizer), 그리고 0~n개의 토큰 필터(Token Filter)로 이루어집니다.

- `filter` 는 list 로 정의 가능한데, list 에 담긴 순서대로 적용된다. (`filter` 는 순서가 있다.)
- `mappings` 필드에는, index 에 정의한 analyzer 를 적용할 필드를 명시했다. `fields` 를 정의해서, `analyzed` 라는 필드를 추가하고, raw data(`hj_addr`) 는 유지한체, `hj_addr.analyzed` 라는 필드를 정의해서, `custom_analyzer` 를 적용했다.
- user_dictionary 와 synonyms_path 는 `$ES_PATH_CONF` 의 상대경로로 지정하는데, rpm 으로 설치한 es 는 `/etc/sysconfig/elasticsearch` 에 명시되어있었다.

```
# Elasticsearch configuration directory
ES_PATH_CONF=/etc/elasticsearch
```

```json
{
    "settings": {
        "index":{
            "analysis": {
                "tokenizer": {
                    "custom_tokenizer": {
                        "type": "nori_tokenizer",
                        "decompound_mode": "mixed",
                        "user_dictionary": "userdict_hb_search.txt"
                    }
                },
                "analyzer": {
                    "custom_analyzer": {
                        "type": "custom",
                        "tokenizer": "custom_tokenizer",
                        "filter": ["do_synonyms"]
                    }
                },
                "filter": {
                    "do_synonyms": {
                        "type": "synonym",
                        "synonyms_path": "synonyms_hb_search.txt"
                    }
                }
            }
        }
    },
    "mappings": {
        "properties": {
            "hj_addr": {
                "type": "text",
                "fields": {
                    "analyzed": {
                        "type": "text",
                        "analyzer": "custom_analyzer"
                    }
                }
            },
            "bj_addr": {
                "type": "text",
                "fields": {
                    "analyzed": {
                        "type": "text",
                        "analyzer": "custom_analyzer"
                    }
                }
            }
        }
    }
}
```

# references

- [김종민님의 document](https://esbook.kimjmin.net/06-text-analysis/6.2-text-analysis)
- [행정동/법정동 데이터](https://www.mois.go.kr/frt/bbs/type001/commonSelectBoardArticle.do?bbsId=BBSMSTR_000000000052&nttId=82856)
- [지하철 노선 관련 한글 형태소분석 글](http://kimjmin.net/2019/08/2019-08-how-to-analyze-korean/)
- [nori tokenizer 사용시 발생한 issue](https://github.com/elastic/elasticsearch/issues/37751)