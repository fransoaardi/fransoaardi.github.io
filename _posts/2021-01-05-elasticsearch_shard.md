---
layout: post
title: elasticsearch shard limit
date: 2021-01-05 01:50:00 +0900
categories: [Wiki]
tags: [elk, aws]
toc: true
comments: true
---

# introduction
- AWS 에서 제공하는 elasticsearch service(opendistro) 를 사용하고있다.
- index template 을 정의해두고, daily index 를 생성하며 사용하고있다.
- **로그가 집계가 되고 있지 않은것 같아서, 인덱스를 확인해보니 오늘 날짜 인덱스가 생성되지 않았다.**
```
{
  "error" : {
    "root_cause" : [
      {
        "type" : "validation_exception",
        "reason" : "Validation Failed: 1: this action would add [10] total shards, but this cluster currently has [994]/[1000] maximum shards open;"
      }
    ],
    "type" : "validation_exception",
    "reason" : "Validation Failed: 1: this action would add [10] total shards, but this cluster currently has [994]/[1000] maximum shards open;"
  },
  "status" : 400
}
```

# progress
- Dev Tools 의 Console 기능을 활용해서 아래와 같이 query 했고, max shard limit 을 확인했다.

```
$ GET /_cluster/settings?include_defaults&flat_settings

(...)
"cluster.max_shards_per_node" : "1000"
```

- cluster 구성을 단일 node 로 해놨었고, 그 결과 max 1000 shard 까지 생성될 수 있었다.
- index template 에 따로 shard 갯수 관련 setting 을 설정하지 않았어서, index settings 를 확인했다.

```
$ GET _template/index-name-here

(...)
"number_of_shards" : "5",
"number_of_replicas" : "1"
```

- elasticsearch 6.x 에서는 shard 기본값이 5, 7.x 에서는 1 이라고 하는데, opendistro 는 기본 5 로 잡혀있었다.
    - ec2 에 직접 설치한 es 에서는 1로 동작하고있었다.

- shard 정보를 봤을때, replicas 는 1, shard 는 5로 동작하여, shard 가 0~4 까지 assign 되었고, `prirep(primary/replica 구분)` 은 p/r 이 번갈아가며 등장했다. 1개의 daily index 를 위해 (1+replicas) * shard = 10 개의 shard 가 생성된것을 확인할 수 있다.

```
$ GET /_cat/shards?v

index                                                   shard prirep state
index-here-2021.01.03                                       2     p      STARTED
index-here-2021.01.03                                       2     r      STARTED
index-here-2021.01.03                                       1     p      STARTED
index-here-2021.01.03                                       1     r      STARTED
index-here-2021.01.03                                       4     r      STARTED
index-here-2021.01.03                                       4     p      STARTED
index-here-2021.01.03                                       3     p      STARTED
index-here-2021.01.03                                       3     r      STARTED
index-here-2021.01.03                                       0     p      STARTED
index-here-2021.01.03                                       0     r      STARTED
(...)
```

- cluster 구성 node 갯수를 늘리기 전까지는 아래와 같이 `UNASSIGNED` 된 replica 가 있는데, replica 는 primary 와 동일한 node 에 shard 할당하지 않는다. 하지만 replicas 가 1로 세팅되어, assign 시도는 하지만, assign 되지 못한 상태로 남아있다. (단일 node cluster 로 구성한, 약식 구성의 경우에는 replicas 는 0으로 세팅하는게 타당해보인다)

```
index                              shard prirep state
index-here-one-log-2020.12.23         0     p      STARTED
index-here-one-log-2020.12.23         0     r      UNASSIGNED                               
metricbeat-7.6.2-2020.10.18-000013    0     p      STARTED
metricbeat-7.6.2-2020.10.18-000013    0     r      UNASSIGNED
(...)
```

- 이미 생성된 indices 는 30일 이후에 ilm 에 의해 index 삭제가 진행되며 shard 가 정리되길 기대하기로 했다.
- 앞으로 생성될 indices 가 적용될 _template 설정에 아래 부분을 추가해서 적용해서, shard 가 1 index 에 2개(primary:1, replica:1) 씩 설정될 것이다.

```
PUT _template/index-here

(...)
"settings": {
    "index.number_of_shards": 1,
    "index.number_of_replicas": 1,
    (...)
}
```

- 앞으로 열릴 shard 갯수는 아래의 수식으로 표현 될 수 있다.

```
30일 x 인덱스 종류(a) x shard(b) x (1+repli(c)) <= 1,000 * node 갯수(d)

a: 앞으로 늘어날 예정, 일단은 넉넉하다
b: 1
c: 1 (replica:1), 총 2개의 shard(primary:1 + replica:1) 가 생김
d: 1 이었는데, 3으로 늘림
```

- a 는 시간이 지날수록 늘어날 것이고, b 와 c 는 reliability & availability 를 위해 조정할 수 있을텐데 AWS 의 managed service 를 이용하는 이상 d 를 편하게 늘릴 수 있어서 크게 신경쓰진 않기로했다. 

# further readings
- index 구성에 있어, 적당한 shard 의 사이즈는 분명 튜닝포인트가 될 만하다.
- 우선은 daily index 를 구성하는 document 수, document size 가 전혀 위협적이지 않아, 1개의 shard 로 충분할 것이고, 많은 shard 유지는 타당하지 않아보였다.

- 아래와 같이, 다양한 API 를 제공한다 (`_cat` 은 `v` param 으로 verbose 한 결과를 얻을 수 있다)

```
# index 의 shard 할당 확인과 shard 현황을 확인해 볼 수 있는 api

$ GET /index-here-*/_stats?level=shards
$ GET /_cat/shards/index-here-*?v
```

- 위의 unassigned 된, allocation 이 실패한 이유를, 아래와 같이 Dev Console 에 query 해서 확인해볼 수 있다.
  - `"explanation" : "the shard cannot be allocated to the same node on which a copy of the shard already exists`

```
$ GET /_cluster/allocation/explain

{
  "index" : "index-here-log-2020.12.23",
  "shard" : 0,
  "primary" : false,
  "current_state" : "unassigned",
  "unassigned_info" : {
    "reason" : "INDEX_CREATED",
    "at" : "2020-12-31T05:23:44.827Z",
    "last_allocation_status" : "no_attempt"
  },
  "can_allocate" : "no",
  "allocate_explanation" : "cannot allocate because allocation is not permitted to any of the nodes",
  "node_allocation_decisions" : [
    {
      "node_id" : "OenW3BKCQgy5R1lQbz72aQ",
      "node_name" : "node-name-here",
      "transport_address" : "123.45.123.12:9300",
      "node_attributes" : {
        "ml.machine_memory" : "8362352640",
        "xpack.installed" : "true",
        "ml.max_open_jobs" : "20"
      },
      "node_decision" : "no",
      "weight_ranking" : 1,
      "deciders" : [
        {
          "decider" : "same_shard",
          "decision" : "NO",
          "explanation" : "the shard cannot be allocated to the same node on which a copy of the shard already exists [[index-here-log-2020.12.23][0], node[OenW3BKCQgy5R1lQbz72aQ], [P], s[STARTED], a[id=p-DkN6gJQue0iFJXu-F5dA]]"
        }
      ]
    }
  ]
}
```

# references
- elastic 의 에반젤리스트인 김종민님의 블로그 중 shard 관련 부분
    - https://esbook.kimjmin.net/03-cluster/3.2-index-and-shards