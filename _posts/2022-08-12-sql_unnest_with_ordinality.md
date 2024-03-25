---
layout: post
title: SQL Unnest with ordinality 를 활용한 쿼리 단순화
date: 2022-08-12 13:00:00 +0900
categories: [Wiki]
tags: [sql]
toc: true
comments: true
---

# Introduction

요근래 업무상 RDB 를 많이 사용하고있고, SQL 을 작성해야 될 일이 많아졌다. SQL 을 실행할 수 있는 runtime 에 상당히 긴 SQL 을 실행요청을 하면 Query 를 header 에 넣어서 요청을 보내는지 `431 Request Header Fields Too Large` 에러가 발생한다는 얘기를 들었고, 문제가 되는 쿼리를 어떻게하면 짧고 간결하게 만들어서 이 문제를 발생하지 않도록 할 수 있을까 생각하게 되었다. 그 결과 `UNNEST WITH ORDINALITY` 라는 구문을 알게되어 사용해봤다.

**주의: Postgres 에는 9.4 부터 적용되었다.**

# Contents 

모든 쿼리를 공개할 수는 없지만, CASE WHEN 구문을 이용해서 각 row 에 동적인 범위를 바탕으로 특정한 값을 할당하려고 하는 문제였다. 아래의 예시를 보면 [0,10], [33,40], [51,77] 에 해당하는 index 에 idx 라는 컬럼 값으로 각각 0, 1, 2 를 대응한다.

```sql
SELECT 
    CASE 
        WHEN index >= 0 AND index <= 10 AND mod(index, 3) = 0 THEN 0
        WHEN index >= 33 AND index <= 40 AND mod(index, 3) = 0 THEN 1
        WHEN index >= 51 AND index <= 77 AND mod(index, 3) = 0 THEN 2
(...)
        ELSE -1
    END as idx
FROM tbl_a WHERE index >= 0 AND index <= 77 AND idx <> -1
```

이 경우에 `BETWEEN` 구문을 사용하고, `mod` 부분을 공통부분으로 빼내어 더 줄일 수는 있겠지만, 범위의 숫자가 늘어날수록 Query 의 길이는 계속해서 늘어날것이라 확장성(scalability)이 좋진 않다.

```sql
-- BETWEEN 과 mod 공통화로 줄여본 Query
SELECT 
    CASE 
        WHEN index BETWEEN 0 AND 10 THEN 0
        WHEN index BETWEEN 33 AND 40 THEN 1
        WHEN index BETWEEN 51 AND 77 THEN 2
(...)
        ELSE -1
    END as idx
FROM tbl_a 
WHERE index BETWEEN 0 AND 77 
AND mod(index, 3) = 0
AND idx <> -1
```

그래서 아래와 같은 표를 만들어보면 어떨까 생각했다.

| idx | start | end | 
| --- | ---   | --- | 
| 0   | 0     | 10  |
| 1   | 33    | 40  |
| 2   | 51    | 77  |

그 방법으로 검색을 좀 해보다가, `unnest with ordinality` 라는 구문을 알게되었고 결국 아래와 같이 구현했다.
이 구문을 사용하게되면 `ARRAY` 를 두번만 선언하면 되고, 이 길이 역시 많이 길어질수는 있지만, `CASE WHEN` 을 하나하나 작성하는 것 보다는 상수배 절약되는 효과가 있다고 생각한다. 

```sql
-- ARRAY 의 선언 외에는 크게 복잡한 부분이 없다.
WITH tmp AS (
    SELECT zip(ARRAY[0, 33, 51], ARRAY[10, 40, 77]) fi
),
indices as (
    SELECT f.idx -1 AS idx, f.st, f.ed FROM tmp, unnest(fi) WITH ORDINALITY f(st, ed, idx)
)

SELECT i.idx AS idx FROM tbl_a, indices AS i
WHERE mod(index, 3) = 0
AND index BETWEEN i.st AND i.ed
```