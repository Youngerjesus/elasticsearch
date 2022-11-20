# 엘라스틱 서치에 대한 궁금증들 

## 엘라스틱서치 매핑은 인덱스를 생성해주는 건가

- 인덱스에 도큐먼트를 새로 추가하면 자동으로 매핑이 생성된다.
- 매핑은 `mappings.properties` 에 저장되서 볼 수 있다.
- 매핑은 다이나믹 매핑과 명시적 매핑이 있다.
- 도큐먼트에 들어온 정보를 바탕으로 매핑을 매기고, 예로 매핑 타입 중 text 와 keyword 타입을 바탕으로 Inverted Index 를 생성한다.

***

## 엘라스틱서치에서 매핑 타입에 따라서 다양한 검색이 가능한건가? 

일단 타입은 이렇게 있다. 

- 문자열: text, keyword
- 숫자: long, double
- 날짜: date
- 불리언: Boolean
- Object 와 Nested
- 위치 정보 - Geo
- 기타 필드 타입으로 (IP, Range, Binary) 가 있다. 

- 날짜를 통해서 날짜 검색, 불리언을 통해서 불리언 검색 같은게 되는건가?
  - ㅇㅇ 숫자나 날짜 같은 경우는 Range 쿼리가 된다. (이외에도 다양한 검색이 있다.)
  - Range 쿼리는 `range: { <필드명>: { <파라미터>: 값} }` 으로 검색이 가능하다. 
  - 쿼리 파라미터는 4개가 있다. `Gte, gt, lte, lt`
  - 예시는 이렇다.

#### request 
```json
GET phones/_search
{
  "query": {
    "range": {
      "price": {
        "gte": 700,
        "lt": 900
      }
    }
  }
}
```

#### Response 
```
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "phones",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 1.0,
        "_source" : {
          "model" : "Samsung GalaxyS 6",
          "price" : 795,
          "date" : "2015-03-15"
        }
      },
      {
        "_index" : "phones",
        "_type" : "_doc",
        "_id" : "3",
        "_score" : 1.0,
        "_source" : {
          "model" : "Samsung GalaxyS 7",
          "price" : 859,
          "date" : "2016-02-21"
        }
      }
    ]
  }
}
```

***

## 엘라스틱 서치에 다양한 검색은 뭐가 있을까? 

https://esbook.kimjmin.net/05-search/5.1-query-dsl

### 풀텍스트 쿼리 (Full Text Query)

- 텍스트를 기반으로 검색하는 걸 풀텍스트 쿼리라고 하는 건가? 
  - ㄴㄴ 풀텍스트는 정확도 (relevancy) 를 기반으로 하는 검색을 말함.
- `match_all`
  - `GET my_index/_search` 
  - 별다른 조건 없이 해당 인덱스의 모든 도큐먼트를 검색하는 쿼리. 검색할 때 쿼리를 넣지 않으면 자동으로 `match_all` 을 적용한다.

- `match` 
  - 풀텍스트 검색에 사용되는 가장 일반적인 쿼리.
  - 다음 예제는 math 쿼리를 이용해서 `my_index` 의 `message` 필드에 `dog` 이 포함되어 있는 모든 문서를 검색하는 것. 

```json
GET my_index/_search
{
  "query": {
    "match": {
      "message": "dog"
    }
  }
}
```

  - 검색할 때 여러개의 검색어를 집어넣게되면 기본적으로 `OR` 조건으로 검색된다. 즉 하나라도 포함된 모든 문서를 탐색하게 된다.
  - 다음 예제는 `quick` 과 `dog` 중 하나의 단어라도 포함된 모든 문서가 탐색된다. 

```json
GET my_index/_search
{
  "query": {
    "match": {
      "message": "quick dog" 
    }
  }
}
```

  - `AND` 조건으로 검색할려면 이렇게 해야한다. 

```json
GET my_index/_search
{
  "query": {
    "match": {
      "message": {
        "query": "quick dog",
        "operator": "and"
      }
    }
  }
}
```

- `match_phrase`
  - AND 조건 말고 구문과 공백을 포함해서 정확히 일치하는 검색을 할려면 이 검색을 이용하면 된다. 
  - 예시로 `lazy dog` 라는 쿼리가 정확히 매칭 되는 걸 보고 싶은 경우. (이건 quick AND dog 가 아니다.)

```json
GET my_index/_search
{
  "query": {
    "match_phrase": {
      "message": "lazy dog"
    }
  }
}
```

- `query string`
  - 루씬의 검색 문법을 본문 검색에 이용하고 싶을 때 쓴다. 
  - 예시로 보면 쉽다. `message` 필드에 `lazy` 와 `jumping` 을 모두 포함하거나 또는 `quick dog` 구문을 포함하는 도큐먼트를 검색하고 싶다면 이렇게 쓰자.
  - `match_phase` 처럼 구문 검색을 할 때는 검색할 구문을 쌍따옴표 `\"` 안에 넣으면 된다.

```json
GET my_index/_search
{
  "query": {
    "query_string": {
      "default_field": "message",
      "query": "(jumping AND lazy) OR \"quick dog\""
    }
  }
}
```

### Bool 복합 쿼리 (Bool Query)

### 정확도 (Relevancy)

- 정확도에서 TF/IDF 와 BM25 가 나온다. (스코어링 기반의 검색)

### Bool (Should)

### 정확값 쿼리 (Exact Value Query)

- Bool: Filter 검색
  - 지금까지 본 풀텍스트 검색은 스코어 기반으로 정확도 (relevancy) 가 높은 결과부터 가져온다.
  - 여기서는 정확도를 고려하는 풀텍스트 검색 외에도 검색 조건의 `참/거짓 여부만 판별` 해서 결과를 가져올 수 있다. 
  - 풀텍스트와 상반되는 이 특성을 정확값 (Exact Value) 라고 부른다. 즉 정확히 일치하는지를 따진다. 
  - Exact Value 에는 `term` 과 `range` 와 같은 쿼리들이 이 부분에 속한다. 
  - 스코어를 계산하지 않기 때문에 보통 `bool` 쿼리의 `filter` 내부에서 사용하게된다. 

다음 3가지 쿼리를 비교해보자.

```json 
GET my_index/_search
{
  "query": {
    "match": {
      "message": "fox"
    }
  }
}
```

```json
GET my_index/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "message": "fox"
          }
        },
        {
          "match": {
            "message": "quick"
          }
        }
      ]
    }
  }
}
```

```json
GET my_index/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "message": "fox"
          }
        }
      ],
      "filter": [
        {
          "match": {
            "message": "quick"
          }
        }
      ]
    }
  }
}
```

- 첫 번쨰 쿼리가 풀텍스트 검색 
- 두 번째 검색은 bool 쿼리를 이용하지만 filter 는 이용하지 않는다. 
  - `bool` 쿼리의 `must` 쿼리가 참인 도큐먼트를 검색한다. 
  - `bool` 쿼리의 `filter` 는 쿼리가 참인 도큐먼트를 검색하지만 스코어링은 계산하지 않는다. `must` 보다 검색 속도가 빠르고 캐싱이 가능하다는 장점이 있다.
  - (filter 이외의 bool 쿼리는 스코어링을 이용한다.)
  - (세 번째 검색을 해도 스코어 기반으로 나열되기는 한다. 물론)
- 세 번째 검색은 bool, filter 쿼리를 이용한 것. 

filter 내부에서 `must_not` 과 다른 bool 쿼리를 이용할려면 이렇게 하면 된다.

```json
GET my_index/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "message": "fox"
          }
        }
      ],
      "filter": [
        {
          "bool": {
            "must_not": [
              {
                "match": {
                  "message": "dog"
                }
              }
            ]
          }
        }
      ]
    }
  }
}
```

- keyword 검색 
  - 문자열 데이터 타입이 keyword 라면 정확한 검색이 가능하다. 
  - 아래의 쿼리는 `message` 필드 값이 `Brown fox brown dog` 문자열과 공백, 대소문자까지 정확히 일치하는 데이터만을 결과로 리턴한다. 

```json
GET my_index/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "match": {
            "message.keyword": "Brown fox brown dog"
          }
        }
      ]
    }
  }
}
```

- keyword 타입으로 저장된 필드는 스코어를 계산하지 않는다. 정확한 여부만을 따진다. (그래서 빠른 것.)
- 스코어를 계산하지 않기 때문에 `keyword` 검색을 할 때는 `filter` 구문안에 넣도록 하자.
- (keyword 와 range 쿼리는 스코어 계산이 필요하지 않아서 filter 안에 넣어서 실행하는 것이 좋다.)
- (keyword 타입이라면 이렇게 검색하고 text 타입이라면 bool: Filter 만써서 검색하라. 이런게 아닐까.)


### 범위 쿼리 (Range Query)

*** 

## 엘라스틱 서치에서 색인은 어떻게 되는 것인가? 

- 엘라스틱 서치에선 `text` 타입이라는 게 있는데 예시로는 문장들을 생각해보면 된다.
  - 이런 `text` 타입은 Analyzer 에 의해서 Token 으로 분석된다. 이렇게 만들어진 Token 은 필터 작업을 거쳐서 인덱싱 되고 **이를 역인덱싱 (Inverted Indexing) 이라고 부른다.** 
  - 역인덱스 (Inverted Index) 에 의해서 저장된 토큰을 용어 (Term) 이라고 부른다.
  - 정리하자면 Raw Data -> Token -> Filtering -> Term (in Inverted Index) 라고 생각하면 될려나.
    - *필터링을 거쳤다는 건 뭘까? Token 중 등록이 안될수도 있다. 이런건가?*
      - 토큰을 가공하는 행위를 말한다. 모두 소문자로 변경하는 필터가 있으면 대문자로 매핑된 Term 을 소문자 Term 으로 옮기거나, 불용어는 제거하거나 등

- 예시로 보자. 
  - 문장: `We offer solutions for enterprise search, observability, and security that are built on a single, flexible technology stack that can be deployed anywhere`
  - Analyzer 를 거친 후 Token 은 이렇게 됨
  
```text
we
offer 
solutions
for
enterprise
...
```

- 역인덱스는 이런식임. 

![](./images/inverted%20Index.png)
- RDBMS 에서의 검색은 LIKE 쿼리 일텐데 이건 모든 ROW 를 읽으면서 해당 Text 필드에 원하는 문자열이 있는지 검색하는 구조다.
- 행이 늘어날 수록 굉장히 느릴 것. 
- ES 에서는 Term 과 매칭되는 Document 가 배열에 추가되는 형태라서 크게 느려지지 않는다. 

- 다시 색인의 과정을 보자.
  - 참고: https://esbook.kimjmin.net/06-text-analysis/6.1-indexing-data
- 문자열 (text) 필드가 들어올 때 검색어 토큰을 저장하기 위한 전체 과정을 **텍스트 분석 (Text Analysis)** 라고 부른다.
  - 이게 Inverted Index 를 만든다고 생각하면 된다.
  - 이 과정을 처리하는게 **애널라이저 (Analyzer)** 라고 보면 된다.
  - `keyword` 타입의 문자열은 이런 분석하는 과정을 겪지 않는다. 

![](./images/text%20analysis.png)

- 엘라스틱 서치는 0-3 개의 **캐릭터 필터(Character Filter)** 와 **1개의 토크나이저** 그리고 **0-n 개의 토큰 필터 (Token Filter)** 이렇게 있다고 보면 된다. 
  - 캐릭터 필드는 전체 문장에서 특정 문자르 대치하거나 제거하는 과정을 담당한다.
  - 토크나이저는 문장에 속한 단어들을 텀 (Term) 단위로 분리하는 과정을 담당한다.
    - 예시로는 공백을 기준으로 분리하는 Whitespace 토크나이저를 생각하면 된다. 
    - 토크나이저는 하나밖에 쓰지 못한다. 
  - 토큰 필터는 Term 을 기반으로 가공하거나 제거, 병합하는 과정을 담당한다. 
    - 가치가 없는 단어 (= 불용어 (stopword)) 는 Inverted Index 에서 제거된다.
    - lowercase 토큰 필터를 통해서 대문자 단어를 소문자로 변경해서 병합해준다.
    - 영어권에서는 형태소 분석을 위해서 `snowball` 이라는 토큰 필터를 자주 쓰는데 이는 `~s, ìng` 와 같은 것들을 제거한다.
    - 토큰 필터는 여러 개를 쓸 수 있지만 단계별로 처리되니까 순서가 중요하다.
    - 필요하다면 `동의어` 를 추가해주기도 한다. `synonym` 토큰 필터가 있다면 **quick** 과 관련된 Term 이 있을 경우 **fast** 라는 동의어 Term 을 추가해준다.
      - 추가 예) Amazon 동의어 (= AWS) 이렇게 추가하는 것도 가능하다.
    - 영어권에서는 문법에 따라서 변형되는 형태가 많다. `~s or ~ness or ~ing or ~ed` 와 같이. 이런 변형되는 형태에 상관없이 검색이 가능해야 하므로 텍스트 데이터를 분석할 때 각 Term 에 붙은 기본 형태인 어간을 추출하는 과정을 진행해야한다. 이 과정을 **어간 추출** 또는 **형태소 분석** 이라고 부른다. 영어로는 **stemming** 이라고 한다.
      - 영어 형태소 분석 알고리즘은 Snowball 이 있고, 한글 형태소 분석기는 Nori 가 있다. 
#### 동의어 필터 예시 

![](./images/synonym%20토큰필터.png)

***

## Muffin 은 왜 Index 배치와 Document 배치가 구별되어 있는 것인가? 

- Document 배치는 Document 를 색인할 때 Filter 필드를 만들고 거기에다가 데이터를 넣어주기 위해서 그런건가? 
  - Filter 필드에 데이터가 들어가면 그걸로 인해서 자동으로 색인이 생성될 것.
    
- Index 배치는 Trie 를 구성하기 위한 것일 거임. 
  - Trie 안에는 대표적으로 Keyword Set, Keyword, Filter 필드가 있음.
    
- 실제 QA 에서 검색을 하면 Trie 안에서 Keyword 를 바탕으로 찾는다. 찾으면 여기에 있는 데이터 중 Keyword Set 과 Filter 를 이용한다.
- Trie 를 통해서 반응한 Keyword Set 을 찾고, 이를 통해서 후보 색인룰을 찾는다.
- (후보 색인룰 안에서 반응할 색인룰이 있다면 쿼리가 반응해도 된다는 뜻임.)
- 매칭된 색인룰이 있다면 Filter 를 통해서 문서를 찾는다. ES 검색 중에 Exact 검색을 하겠지. 
  - 아 여기서 ES 에서 만들어진 Inverted Index 를 이용하는구나. 
  - ES 를 저장소로만 쓰고 있다는 이유는 이거구나. 
  - 매칭된 색인룰 안에 어떤 컬렉션이 반응해야 하는지도 있는건가
- 만약에 이런 구조가 아니라 ES 를 그대로 쓰게 되면 어떻게 될까? 
  - 검색을 할 때 Exact 검색을 한다고 해도, 룰 기반의 검색이 어려워짐. 특정 필드를 통한 AND, OR 위주의 검색이 되거나 스코어링 기반의 검색을 위주로 쓰게 되지 않을까?  
