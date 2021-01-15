---
layout: post
title: bash-string-manipulation
date: 2021-01-06 14:13:00 +0900
categories: [Wiki]
tags: [bash, linux]
toc: true
---

#!/bin/bash

# ./context.sh 886035-DODUJUDEFS 2
sid=$1
sseq=$2

# data='{ "_source": ["context"], "query": { "bool": { "must": [{ "terms": { "sessionId": ["VAR_SID"] } }], "filter": [{ "range": { "datetime": { "gte": "now-1d", "lt": "now" } } }, { "term": { "issued": "server" } }] } } }'
#data='{ "_source": ["context"], "query": { "bool": { "filter": [ {"range": {"datetime": {"gte": "now-1d", "lt": "now" }}}, {"term": {"issued": "server"}}, {"term": {"sessionId": "VAR_SID"}}, {"term": {"sessionSeq": VAR_SSEQ}} ] } } }'
#data=$(echo "$data" | sed "s/VAR_SID/$sid/" | sed "s/VAR_SSEQ/$sseq/")
data='{ "_source": ["context"], "query": { "bool": { "filter": [ {"range": {"datetime": {"gte": "now-1d", "lt": "now" }}}, {"term": {"issued": "server"}}, {"term": {"sessionId": "'"$sid"'"}}, {"term": {"sessionSeq": "'"$sseq"'"}} ] } } }'

resp=$(curl 'http://abc.def.com:9200/_search' -XPOST -H 'Content-Type: application/json' -d ''"$data"'' | jq '.hits.hits[0]._source')
echo $resp

if [ -z "{$resp}" ]; then echo "no result"; else echo "${resp}"; fi