# Tune for search speed

https://www.elastic.co/guide/en/elasticsearch/reference/7.10/tune-for-search-speed.html#_use_index_sorting_to_speed_up_conjunctions

***

## Give memory to the filesystem cache.
- Tune for indexing speed 에서 소개한 것과 이유 동일.

## Use faster hardware

- Tune for indexing speed 에서 소개한 것과 이유 동일.
- CPU 집약적인 어플리케이션이라면 이것도 고려해야한다고 한다.
  - Elasticsearch 에서 cpu 기반의 작업이 있는가?
    - Indexing
      - Data 가 parsing 되고 검색가능한 형태로 format 을 변화시키는 건 cpu 의 일이다.
      - 이 작업은 특히 큰 데이터를 다룰 때 많이 일어남.
    - Searching
      - 많은 색인이 있는 경우에 검색하는 작업도 cpu 집약적인 일임.
      - 관련 있는 Results 를 찾는 일이니까. Inverted index 를 뒤져보면서 Filter 도 해야함.

    - Aggregation Operations
      - Calculating averages or sum 과 같은 것들은 cpu 를 쓰겠지.

    - Machine learning tasks
      - Machine learning task 를 위한 기능을 제공해준다는 듯.
      - Training model 과 making prediction
      - 이런 작업을은 cpu 집약적인 일들.


## Document modeling

- Tune for indexing speed 에서 소개한 것과 이유 동일.


## Search as few fields as possible

- 많은 필드를 이용해서 query_string 과 multi_match 검색을 하면 더 느려진다.
- query_string 검색은 뭔데?
  - 주어진 query string 을 이용해서 검색하는 것.
  - Strict syntax 가 존재함
  - Parser 를 이용한다.
    - Parsing 되면 terms 와 operations 으로 분리된다.
    - Term 은 single word 임.
    - 예시) `title:(quick OR brown)`
    - 이렇게 쪼개진 text 는 각각 독립적으로 분석됨.
    - 이게 느린 이유겠다.

#### query_string 예시

```http request
GET /_search
{
  "query": {
    "query_string": {
      "query": "(new york city) OR (big apple)",
      "default_field": "content"
    }
  }
}
```
  
- multi_match query 검색은 뭔데?
  - Match query 를 이용하는 방식이고 multi-field 가 허용됨.
  - 이것도 여러 필드를 통해서 각각 검색하는 것이니까 느리지 않을까.
  
#### Match query 얘시 
```http request
GET /_search
{
  "query": {
    "match": {
      "message": {
        "query": "this is a test"
      }
    }
  }
}
```
  
####  Multi-match query 예시 
```http request
GET /_search
{
  "query": {
    "multi_match" : {
      "query":    "this is a test", 
      "fields": [ "subject", "message" ] 
    }
  }
}
```

- 아. 여러 필드를 통해서 검색하고 싶을 때는 여러 필드를 copy 에서 하나의 필드로 만들어라. 색인 시점에. 
  - 이게 copy-to 를 이용하는 방식임.

#### copy-to 예시 

```http request
PUT movies
{
  "mappings": {
    "properties": {
      "name_and_plot": {
        "type": "text"
      },
      "name": {
        "type": "text",
        "copy_to": "name_and_plot"
      },
      "plot": {
        "type": "text",
        "copy_to": "name_and_plot"
      }
    }
  }
}
```  



## Pre-index data

- 여기서는 예시로 바로 보여준다.
  - Price 필드가 있고, 이걸 range aggregation 을 한다고 생각해보자.
  - 이 경우 aggregation 을 더 빠르게 하기 위해서 ranges 를 미리 색인해둔다. Term 형식으로.

#### Pre-index data 예시 

```http request
PUT index
{
  "mappings": {
    "properties": {
      "price_range": {
        "type": "keyword"
      }
    }
  }
}

PUT index/_doc/1
{
  "designation": "spoon",
  "price": 13,
  "price_range": "10-100"
}
```

- `price-range` 를 keyword 타입으로 미리 색인해둔 것. 



## Consider mapping identifiers as keyword

- 모든 숫자 필드를 numeric 필드로 매핑할 필요는 없다. 이런 이야기.
- Numeric 필드를 keyword 로 매핑하는 것도 고려해봐라.
  - Range 쿼리를 안쓴다면.
  - 아니면 `multi-field` 로 고려해봐도 괜찮음.
- Keyword 타입이 numeric 타입보다 검색이 더 빠름.


## Avoid scripts

- scripts 와 scripted fields 를 피해라.
- Script 는 elasticsearch 의 index 구조와 최적화를 사용하지 않아서 search 의 속도가 느리다.
  - Scripts 는 Indexed 한 데이터를 변형시키는데 사용할 수도 있음.

- Scripts 가 뭔데? 
  - Update, delete, or return specific documents 를 할 수 있고 거기에다가 custom 한 로직을 실행시킬 수 있다.


## Search rounded dates

- date field 를 통해서 검색할 때 now 를 이용한다면 `query cache` 가 적용되지 않는다. 그래서 검색이 느림.
- 근데 date field 를 round 처리를 한다면 `query cache` 가 적용될 수 있다. 같은 시간대에서 검색한다면.
- 이것도 예시를 보여주면 좋을듯.
https://www.elastic.co/guide/en/elasticsearch/reference/7.10/tune-for-search-speed.html#map-ids-as-keyword

#### query cache 가 적용되지 않는 예 
```http request
PUT index/_doc/1
{
  "my_date": "2016-05-11T16:30:55.328Z"
}

GET index/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "range": {
          "my_date": {
            "gte": "now-1h",
            "lte": "now"
          }
        }
      }
    }
  }
}
```

#### query cache 를 적용시킨 예 

```http request
GET index/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "range": {
          "my_date": {
            "gte": "now-1h/m",
            "lte": "now/m"
          }
        }
      }
    }
  }
}
```

- 만약 현재 시간이 `16:31:29` 라면 range 시간은 `15:31:00` 와 `16:31:59` 이렇게 될 것이다. 
- 즉 현재의 `분` 동안 들어오는 요청은 캐시가 될 것. 즉 이 예시 기준으로는 `16:31:00 ~ 16:31:59` 까지는 캐시가 될 것이다.

## Force-merge read-only indices

- Force merge API 는 뭔데
  - Shard 에 있는 indices 를 merge 하는 것을 말하는듯.
  - Merge 는 뭔데?
    - Lucene 의 segment 를 합치는 것.
    - Segment 의 개수를 줄이는 용도, deleted document 의 공간을 해방시키는 용도로 사용한다.

  - 검색을 잘 안하는 짜잘짜잘한 인덱스의 경우에는 merge 를 하는게 좋다고 함. 그게 더 효율적인 구조.
  - Merge 는 자동으로 되는데 때때로 수동으로 하는게 유용하다고 함. (force merge 는 수동으로 하는거)
  
  - 색인이 다 써저야지 호출이 가능하다.
  - Force merge 는 앞으로 write 할 가능성이 없는 old indices 에 쓰라고 함. 계속 wrtie 하는 경우에는 background 로 처리되는 automatic merge 에 의존하는게 낫다.
    - 계속 write 하는 경우에 force merge 를 하면 큰 segment 가 만들어 질 것이고 이건 disk 사용이 엄청 커지고 메모리에 가져오는 것도 힘드니까 검색 속도가 느려진다. 
  - Merge 를 하면 segment 가 커진다. 이건 메모리에 올리기엔 적합하진 않음. 그리고 이 색인에 update 하는 것도 복잡하고.
    - Merge 하는 것도 리소스가 들어간다. 커지면 필요한 시점에 자동으로 merge 될건데 굳이 미리 할 필요는 없음.


## Warm up global ordinals

- global ordinals 에 대해서 먼저 알아보자.
  - Field 의 unique values 를 document id 에 매핑시켜주는 것.
    - 필드를 통해서 검색이 가능해짐. 
  - 목적 자체는 field 에 unique 한 value 가 많고 클 때, aggregation 과 search 를 빠르게 하기 위한 것.  
  - Ordinal mapping 을 shard-level 에 적용하기 위한 것이라고 생각하면 됨.
  - _source documents 라는 곳에 a separate data structure 로 저장됨. 그리고 효율적인 접근과 압축을 보장하고 자동적으로 만들어짐.


- Doc values 에 대해서 알아보자.
  - 색인 시간에 만들어지는 On-disk data structure
  - Data access pattern 을 가능하게 해준다는데 이게 무슨 뜻일까
  - 모든 필드에 쓸 수 있고, 기본적으로 활성화 되어있음.
  - Field 를 기반으로 sorting 과 aggregation 이 필요하지 않다면 disable 하는 것도 가능하다고 함.
  - 필드를 기반으로 Aggregations, sorting, scripting 을 효율적으로 만들어준다고 함. Column-oriented format 을 이용해서.
  - 대신에 색인 시간이 늘어나고, 디스크 공간을 더 많이쓴다.
  - 이런식으로 저장됨.
  - Inverted index 랑 비슷함. Term 과 documentId 가 매핑된게. 다만 inverted index 는 word 를 토큰으로 쪼개서 관리하는것.

#### Doc values 예제 

| price | doc Id |
|:-----:|:------:|
|  100  |   1    |
|  200  |   2    |
|  300  |   3    |
|  400  |   4    |
 | 500 |   5    |


- Ordinal mapping 이 뭔데?
  - 각각의 Term 에 incremental Integer or ordinal 을 매핑시키는 것. 사전순대로.
  - Field 의 Doc values 는 이 ordinal 에 저장된다. Original term 대신에. 그리고. Ordinals 와 term 을 변환해주는 별도의 look up structure 와 함께
  - Ordinal mapping 은 doc values 를 정렬한 거라고 봐도 되나?
    - Unique field 들을 document Ids 로 매핑한 것.
    - 비슷한점이 많다.

- _source documents 는 뭔데? 
  - 색인이 될 original json body 가 저장된다.
  - 기본적으로 색인이 되지 않음. 그래서 검색과 aggregatable 이 되지는 않음.
    - 근데 search result 에 포함시키는 것도 가능.
    - 이런 _source 필드 이외에도 엘라스틱서치는 document id, index, type, version 과 같은 메타정보들을 저장한다.

- Warm up global ordinals 는 jvm heap 에 저장되는 global ordinals 를 요청을 받기 전에 미리 구성하도록 하는 것.
  - 이건 `field data cache` 로 저장된다.
  - **자주 사용하는 필드 데이터를 미리 로딩해놔라 이런거.**
  - `eager_global_ordinals: true` 로 적용이 가능함.
  - 문서가 정형화된 구조를 가지고 있을 떄 사용할 수 있을듯.


## Warm up the filesystem cache

- elastic search 를 재시작할 때 filesystem cache 도 비어있을 것.
- `index.store.preload` 를 이용해서 운영체제에게 hot index 를 불러오도록 하는 것.
- File extension 을바탕으로 불러오는 듯.
  - 색인의 경우 이런식으로 `index.store.preload: [“nvd”, “dvd”]`


## Use index sorting to speed up conjunctions

- Index sorting 을 통해서 serach sorting request 를 빠르게 응답. 보낼 수 있다.

## Use preference to optimize cache utilization

- search performance 를 위한 여러 캐시가 있다.
  - Filesystem cache
  - Request cache
  - Query cache

- Request cache 란
  - Shard-level 의 캐시. (다른 애들은 Node-level 의 캐시)
  - 각각의 샤드에 있는 local result 를 캐시해두는 것.
  - 그래서 자주 사용된 검색 요청은 즉시 결과가 나갈 수 있는 것.
  - 캐시 무효화는 샤드가 refresh 될 때 invalidate 된다. (+ 데이터가 변한 것에만 해당.)

- Query cache 란
  - Filter context 로 사용된 쿼리의 결과들은 node query cache 로 저장된다. 이것도 빠르게 조회하기 위해서.
  - 노드별로 query cache 가 있다.
  - 기본적으로 10000 query 를 가질 수 있다. Heap 의 10% 정도.

- Request Cache 의 경우에는 shard-level 의 cache 니까 똑같은 요청이 여러번와도 다른 샤드엥게 갈 수 있다. 정확하게는 replica 에게 가는거지.
  - 이걸 원하지 않는다면 preference 를 쓰라고 하는듯.
  - 이걸 통한 속도 개선은 너무 작은 차이인거 같다.


## Replicas might help with throughput, but not always

- resiliency 와 throughput 을 위해선 replica 가 많이 도와줄 수 있다.

- 질문) 노드 하나에 하나의 샤드와 복제본이 (= replica) 다 있을까? ㄴㄴ 그러면 노드가 다운되면 큰일이니까 replica 는 다른 노드에 있는게 맞지.

- 하나의 노드에는 적은 양의 샤드가 있는게 효율이 좋다. 리소스는 공유되니까. (Filesystem cache)
- Replica 의 수를 결정하기 위한 요소는 다음과 같다.
  - num_nodes
  - num_primaries
  - Node 의 max_failures
  - 공식은 max(max_filures, ceil(num_nodes / num_primaries) - 1)
    - Ceil 은 크거나 같은 수 중에 가장 작은 수를 반환.
    - Math.ceil(0.95) -> 1
    - Match.ceil(4) -> 4


## 그래서 Spider 에는 어떻게 쓸 수 있는데?

- pre-index 패턴도 쓸 수 있을 것 같은데
  - QA 와 format 을 맞춰놔야겠다.
  - Keyword 테이블에도 설정을 해놔야할듯.

- 여러 필드를 하나로 모으는 copy_to 도 적용할 수 있을듯.

- Range 쿼리에서 numeric 필드가 필요하다면 multi field 로 고려해보는 것.
- Warm up the file system cache 를 적용해보는 것도 좋을듯. 


