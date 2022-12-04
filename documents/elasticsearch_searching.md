# Elasticsearch Searching

이전에 엘라스틱서치에서 기본적인 개념을 배웠다면 여기서는 어떻게 검색을 하는지에 대해서 정리했다.

관계형 DB 에서 SQL 을 배워서 데이터를 조회하는 거랑 같은 개념이라고 생각하면 된다.

어떻게 검색을 하는지 알고 이후에 좀 더 깊게 공부하고 싶다면 내부 동작원리나 알고리즘을 공부해보면 될 것 같다.

엘라스틱서치는 정형,비정형,텍스트,숫자 등 데이터를 저장해놓으면 인덱싱 되고 이후에는 쿼리를 통해서 검색할 수 있다.

거기에 더해서 스코어링 알고리즘을 적용해서 연관성이 높은 결과에 대한 제어가 가능하므로 대용량 데이터를 대상으로 빠르고 정확한 검색이 가능하다.

여기서는 다음과 같은 것들에 대해서 정리해봤다.
- 쿼리 컨택스트와 필터 컨택스트의 차이점을 보자.
- 쿼리 스트링과 쿼리 DSL 의 차이점을 보자.
- 찾는 문서의 연관성을 계산해주는 스코어링 기본 알고리즘을 보자.
- 엘라스틱서치에서 자주 사용하는 쿼리를 본격적으로 살펴보자.
  - 매치 쿼리
  - 용어 쿼리
  - 멀티 매치 쿼리
  - 범위 쿼리
  - 논리 쿼리
  - 패턴 쿼리

***

## 쿼리 컨택스트와 필터 컨택스트 

엘라스틱서치의 검색은 크게 두 가지로 구별된다.

쿼리 컨택스트와 필터 컨택스트.

쿼리 컨택스트는 쿼리에 대한 유사도를 바탕으로 높은 유사도를 보이는 결과를 먼저 보여주는 형식이다.

필터 컨택스트는 유사도 계산을 하지 않고 찾고자 하는 문서가 일치하는지 아닌지를 기준으로 검색을 하는 방식이다.

예시로 알면 더 쉬운데 쿼리 컨택스트는 문서 본문에 "엘라스틱서치" 가 있는 것들 중에서 제일 연관성이 높은 걸 찾는 방식이라면 

필터 컨택스트는 문서 제목이 정확히 "엘라스틱서치" 여야 하는 문서만을 찾는 방식이다. 

엘라스틱서치에서 쿼리 컨택스트는 스코어 결과를 제공하고 필터 컨택스트는 예/아니오 결과만을 제공한다.

둘의 차이를 좀 보자면 필터 컨택스트의 경우에 스코어링 계산을 하지 않으므로 더 빠르고 

딱 특정한 형식으로만 쿼리가 올 것이니까 매번 결과 업데이트가 필요하지 않다면 캐싱을 적극적으로 이용할 수 있다.

하지만 쿼리 컨택스트는 쿼리가 매번 다를 가능성이 있으니까 캐시가 적합하지 않을 수 있다.

엘라스틱서치는 기본적으로 힙 메모리의 10% 를 캐시에 이용하고 있는데 빠른 검색이 필요하다면 필터 컨택스트를 적극적으로 이용해보자.

다음은 쿼리 컨택스트를 실행하는 예시이다.

```json
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "match": {
      "category": "clothing"
    }
  }
}
```

- _search 는 검색 쿼리 API 를 사용하겠다는 뜻이다.
- match 는 전문 검색을 이용하겠다는 뜻으로 쿼리 컨택스트를 이용한다.
- 즉 category 필드에 역인덱스 테이블에서 clothing 이 있는 것들을 찾을 것이다.
- 결과는 스코어링을 통해서 제일 유사도가 높은게 먼저 보일 것이다.

결과는 어떤식으로 나올까? 

```json
...
"hits": {
  "total": {
    "value": 3927,
    "relation": "eq"      
  },
  "max_score": 0.2947583,
  "hits": [
    {
      "_index": "kibana_sample_data_ecommerce",
      "_type": "_doc",
      "_id": "5JAFLKAJFLAJSFFLKAJF",
      "_score": 0.2947583
      ...
    }      
  ]
  ...
}
```
- hits.total 을 보면 총 문서로 3927 개를 찾았다는 걸 알 수 있다.
- hists.hits 에는 찾은 문서가 모두 스코어를 기반으로 정렬되어있다.
- _score 는 유사도를 나타내는 값이다. 

다음은 필터 컨택스트를 이용해서 검색한 예제다.

```json
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "bool": {
      "filter": {
        "term": {
          "day_of_week": "Friday"
        } 
      }
    }
  }
}
```
- 필터 컨택스트와 쿼리 컨택스트 모두 _serach API 를 사용한다.
- 필터 컨택스트는 논리 쿼리 (bool) 내부의 filter 타입에 적용된다. 
- 여기서는 day_of_week 필드에 값이 Friday 인 도큐먼트를 찾아달라는 요청이다.
- 엘라스틱서치에서 논리 쿼리가 나온 이후부터는 필터 컨택스트는 모두 논리 쿼리에 포함되었다. 이를 이용해서 필터 컨택스트와 쿼리 컨택스트틑 같이 조합해서 쓰는 추세로 되었다.
- 이를 같이 쓰면 좋은게 쿼리 컨택스트 입장에서는 스코어링을 할 대상 자체를 줄여줄 수 있다.

***

## 쿼리 스트리와 쿼리 DSL

엘라스틱서치에서 쿼리를 사용하는 방법은 쿼리 스트링 (query string) 과 쿼리 DSL (Domain specific Language) 두 가지가 있다.

쿼리 DSL 은 엘라스틱서치에서만 사용하는 쿼리 전용 언어이고 JSON 기반으로 좀 더 직관적이다.

쿼리 스트링은 한 줄 정도의 간단한 쿼리에 사용하고 쿼리 DSL 이 이상의 좀 더 복잡한 쿼리를 날릴 때 사용한다.

### 쿼리 스트링

쿼리 스트링은 REST API 의 URI 주소에 쿼리문을 작성하는 방식으로 실행해볼 수 있다.

쿼리 스트링의 경우 조건이 복잡해지면 가독성이 좋지 않아서 오류를 범하기 쉬우므로 이 경우에는 쿼리 DSL 을 사용하도록 하자.

다음 예는 간단한 쿼리 스트링 예제이다.

```json
GET kibana_sample_data_ecommerce/_search?q=customer_full_name:Mary
```

- kibana_sample_data_ecommerce 인덱스에 필드가 customer_full_name 필드에 있는 값이 Mary 인 경우 조사하는 필드다. 
- 이 같은 쿼리는 curl 을 통해서 요청을ㄹ 날려 볼 수 있으므로 편할 수 있다.

### 쿼리 DSL 

쿼리 DSL 은 요청 본문에 JSON 형태로 쿼리를 작성하는 경우를 말한다.

다음 쿼리 DSL 의 예시를 보자.

```json
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "match":{
      "customer_full_name": "Mary"
    }
  }
}
```
- 이 쿼리는 지금까지 자주 사용했던 전문 쿼리다.

***

## 유사도 스코어 

쿼리 컨택스트는 엘라스틱서치에서 지원하는 다양한 스코어링 알고리즘을 사용할 수 있다.

기본적으로는 BM25 알고리즘을 사용해서 스코어를 계산한다. (스코어가 높을수록 찾고자하는 문서와 유사도가 높다는 뜻이다.) 

기본적인 스코어링 알고리즘을 이해하고 있다면 똑똑한 쿼리를 작성할 수 있을 것이고 인덱스를 효율적으로 관리할 수 있을 것이다.

쿼리를 요청하고 스코어가 어떤식으로 계산되는지 확인하고 싶다면 쿼리에 explain 옵션을 추가해보면 된다.

```json
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "match":{
      "customer_full_name": "Mary"
    }
  },
  "explain": true
}
```

### 스코어 알고리즘 BM25 이해하기

BM25 알고리즘 이전에는 TF-IDF (Term Frequency, Inverse Document Frequency) 알고리즘을 이용했었다.

BM25 알고리즘은 TF-IDF 에다가 추가로 문서 길이를 고려한 알고리즘이다. 

간단하게만 이 알고리즘을 설명한다면 검색어가 문서에서 중요한 용어인지 확인하고 얼마나 검색어가 자주 등장하는지를 측정하는 알고리즘이다.

BM25 는 IDF 와 TF 값을 알면 구할 수 있다. 하나씩 보자.


#### IDF 계산 

IDF 는 전체 문서들 내에서 자주 발생하는 용어일수록 중요하지 않다고 판단해 가중치를 매기는 값이다.

생각해보면 당연하다. The, a 와 같은 용어들은 인덱스 내 모든 문서들에서 발생할 것이고 이는 중요하지 않다.

엘라스틱서치와 같은 용어는 모든 문서내에서 발생하지 않을 것이므로 이는 가중치를 높게 줘야한다.

idf 를 계산하는 식은 다음과 같다.

`idf, computed as log(1 + (N - n + 0.5) / (n + 0.5))`

- 즉 N 과 n 만 알면 된다. 
- N 은 전체 도큐먼트의 개수이다.
- n 의 설명은 검색했던 용어가 몇 개의 도큐먼트에서 발생했는지를 나타내는 값이다.
  - 즉 n 은 작을수록 가중치가 높다.

#### TF 계산

TF 는 문서내에서 용어가 얼마나 자주 등장했는가 를 나타내는 지표에다가 추가로 문서 길이를 고려한 값이다.

문서 내에서 용어가 자주 등장하면 그 용어가 문서 내에서 중요하다는 뜻이다.

그리고 문서 길이 같은 경우는 짧은 문서에서 용어가 많이 등장했는지와 긴 문서에서 용어가 많이 등장했는지를 고려한 값이라고 생각해보면 된다.

당연히 짧은 문서에서 많이 나온게 더 가중치가 높아야한다.

TF 를 계산하는 식은 다음과 같다.

`tf, computed as freq / (freq + k1 * (1 - b + b * dl / avgdl)) from;`

- 즉 여기서는 freq, k1, b, dl, avgdl 을 알면된다.
- freq 는 용어가 문서내에서 등장하는 빈도를 나타낸 값이다.
- k1 과 b 는 정규화를 하기 위한 상수이다.
- dl 과 avgdl 은 같이 이해하면 좋은데 dl / avgdl 은 짧은 문서일수록 높은 가중치를 주는 값이다.
  - dl 은 필드 값의 길이를 나타내고
  - avgdl 은 문서들 내에서 이 필드 값의 길이를 평균낸 것이다.
  - 즉 평균대비 얼마나 차지하고 있는지 나타낸 값이다.
  - 필드 값의 길이의 측정은 토큰의 개수로 측정한다. 토큰의 개수가 많다 라는 것은 그만큼 긴 문장이었다는 뜻이기 떄문이다.

이렇게 TF 와 IDF 를 계산하고 TF * IDF * boost 를 계산한 값이 스코어링이 된다.
- boost 는 고정 값으로 2.2 를 나타낸다.

***

## 쿼리

엘라스틱서치는 검색을 위해서 쿼리를 지원하는데 크게 리프 쿼리 와 복합 쿼리로 나눌 수 있다.

리프 쿼리는 특정 필드에서 용어를 찾는 것으로 크게 다음과 같은 것이 있다.
- 매치 쿼리
- 용어 쿼리
- 범위 쿼리

복합 쿼리는 쿼리를 조합해서 사용하는 쿼리로 대표적으로는 논리 쿼리등이 있다.

이 책에서 다루지 않는 쿼리는 온라인 문서를 참고하자.

### 전문 쿼리와 용어 수준 쿼리

전문 쿼리는 전문 검색을 위해서 사용되며 필드의 타입이 text 여야 한다.

용어 수준 쿼리는 필드의 타입이 keyword 여야 하고 정확한 결과를 얻기 위해서 사용한다.

전문 쿼리부터 예시로 보자.

다음과 같이 인덱스에 문서를 저장한다고 가정해보자.

```json
PUT qindex/_doc/1
{
  "contents": "I love elastic search"
}
```

- 이 경우 역인덱스로 [i, love, elastic, search] 가 담기게 된다.

그럼 다음과 같이 전문 쿼리로 검색을 해본다고 가정해보자.

```json
GET qindex/_search
{
  "query": {
    "match": {
      "contents": "elastic world"
    }
  }
}
```

검색을 하는 경우도 분석기에 의해서 토큰으로 분석되어서 검색된다. [elastic, world]

그리고 문서에 있는 인덱스들고 비교해서 스코어링을 통해서 결과가 나오는 것이다.

전문 쿼리는 일반적으로 블로그와 같이 텍스트가 많은 것들에서 사용되는 방식이다. 

구글이나 네이버에서 검색을 이용하는 방식이 이와 같다고 생각해보면 된다.

전문 쿼리의 종류로는 매치 쿼리 (match query), 매치 프레이즈 쿼리 (match phrase query), 멀티 매치 쿼리 (multi-match query), 쿼리 스트링 쿼리 (query string query) 가 있다.

다음으로 용어 수준의 쿼리를 보자.

```json
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "term": {
      "customer_full_name.keyword": "Mary Bailey"
    }
  }
}
```
- term 을 이용해 용어 쿼리를 작성할 수 있다. 

용어 수준의 쿼리는 전문 수준의 쿼리와 비슷하지만 인덱스로 만들 때 분석기 (Analyzer) 를 거치지 않는다는 특징이 있다.

즉 그러므로 Tech 를 문서에 저장하면 Tech 그대로 저장한다. 

찾을 때도 마찬가지로 분석기를 거쳐서 토큰화되지 않는다. 그러므로 tech 라고 검색하면 조회가 되지 않고 Tech 라고 조회해야 검색이 된다.

용어 수준의 쿼리는 정확해야 조회가 되므로 숫자, 날짜, 범주형과 같은 데이터를 정확히 검색할 때 사용하면 좋다.

용어 수준 쿼리에는 용어 쿼리 (term query), 용어들 쿼리 (terms query), 퍼지 쿼리 (fuzzy query) 가 있다.

### 매치 쿼리

매치 쿼리는 대표적인 전문 쿼리다.

전문 쿼리의 가장 기본이 되는 쿼리로 전체 텍스트 중에서 특정 용어를 검색할 때 사용하는 쿼리로 생각하면 된다.

하나의 용어를 검색하는 매치 쿼리는 이전에 많이 보았으므로 복수 개의 용어를 검색하는 매치 쿼리를 보자.

```json
GET kibana_sample_data_ecommerce/_search
{
  "_source": ["customer_full_name"],
  "query": {
    "match": {
      "customer_full_name": "mary bailey"
    } 
  }
}
```
- _source 파라미터를 통해서 응답으로 가져오는 필드는 customer_full_name 만 가져온다. (효율적이다.)
- 쿼리로 날리는 mary bailey 는 분석기를 통해서 [mary, bailey] 로 토큰화 된다.
- 이렇게 여러개로 토큰화 되는 쿼리는 OR 로 문서를 검색한다. 즉 둘 중 하나만 있어도 검색이 되고 둘 다 있으면 스코어가 높게 나온다.

매치 쿼리로 AND 조건을 날려서 찾을려고 하면 어떻게 하면 될까? 

operator 파라미터를 주면 된다.

```json
GET kibana_sample_data_ecommerce/_search
{
  "_source": ["customer_full_name"],
  "query": {
    "match": {
      "customer_full_name": {
        "operator": "and",
        "query": "mary bailey" 
      }
    },
  }
}
```
- operator 를 명시하지 않으면 기본값이 세팅되고 기본값은 OR 이다.

### 매치 프레이즈 쿼리

매치 쿼리와 마찬가지로 전문 쿼리의 한 종류다.

메치 프레이즈 쿼리는 구 (phase) 를 검색할 때 용이하다.

구는 2 개 이상의 단어로 이뤄진 단어이다.

즉 빨간색 바지, 17인치 텔레비전 같은 것들을 말한다. 

구의 또 다른 특징으로는 순서가 맞아야 한다는 점이다. 빨간색 바지와 바지 빨간색은 의미상으로 완전히 다르다.

자 그럼 이제 엘라스틱서치에서 구를 검색하는 방법을 보자.

```json
GET kibana_sample_data_ecommerce/_search
{
  "_source": ["customer_full_name"],
  "query": {
    "match_phase": {
      "customer_full_name": "mary bailey"
    } 
  }
}
```

- mary bailey 가 분석기를 통해서 [mary, bailey] 로 토큰화 되는 것은 동일하다.
- 하지만 문서에서 이 두개가 동시에 있어야 하며 순서도 mary bailey 로 일치하는 것들을 찾는다.
- 즉 mary tony bailey 같은 것들은 찾지 않는다.

**매치 프레이즈 쿼리는 많은 리소스를 필요로 하기 때문에 자주 사용하지 않는 것이 좋다.**
- 궁금한 부분이네.

### 용어들 쿼리

용어들 쿼리는 여러 용어들을 한번에 검색하게 해주는 방법이다.

용어 쿼리와 마찬가지로 정확히 일치하는 값만 찾아오므로 이를 신경쓰자.

다음 예시는 키워드 필드 day_of_week 를 이용해서 용어들 쿼리를 하는 예시다.

```json
GET kibana_sample_data_ecommerce/_search
{
  "_source": ["day_of_week"],
  "query": {
    "terms": {
      "day_of_week": ["Monday", "Sunday"]
    }
  }
}
```
- terms 를 통해서 용어들 쿼리를 할 수 있다.
- Monday 또는 Sunday 를 가지고 있는 문서를 찾아서 준다.

### 멀티 매치 쿼리

지금까지의 쿼리는 어떤 필드에 있는지 알고 있었다.

하지만 우리가 찾는 검색이 어떤 필드에 있는지 명확하게 알지 못해서 여러 필드에서 검색해야 할 수도 있다.

이를 위해 멀티 매치 쿼리가 존재한다.

멀티 매치 쿼리는 전문 검색 쿼리의 일종으로 텍스트 타입으로 매핑된 필드에서 사용할 수 있다. 

```json
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "multi_match": {
      "query": "mary",
      "fields": [
        "customer_full_name",
        "customer_first_name",
        "customer_last_name"
      ]
    }
  }
}
```
- multi_match 를 통해서 멀티 매치 쿼리를 이용할 수 있다.
- 쿼리 값인 mary 는 여러개의 필드에서 검색된다.
  - customer_full_name
  - customer_first_name
  - customer_last_name
- 이 경우 모든 필드에서 검색을 하고나서 스코어링을 한 후 가장 가중치가 높은 것이 먼저 출력된다.
- 만약 특정한 패턴을 가지고 있다면 다음과 같이 와일드 카드를 사용해도 된다.

```json
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "multi_match": {
      "query": "mary",
      "fields": "customer_*_name"
    }
  }
}
```

멀티 매치 쿼리에서 여러 개의 필드중에 가중치를 매기는 것도 가능하다.

당연하겠지만 제목에 등장하는 용어와 문서 본문에 등장하는 용어는 중요도가 다르다.

다음 예시는 customer_full_name 에 2 배의 가중치를 준 에시이다. 

```json
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "multi_match": {
      "query": "mary",
      "fields": [
        "customer_full_name^2",
        "customer_first_name",
        "customer_last_name"
      ]
    }
  }
}
```
- ^ 를 붙이고 숫자를 주면 그 만큼의 가중치를 줄 수 있다. 이를 통해서 customer_full_name 은 두 배의 스코어를 가진다.

### 범위 쿼리

이번에는 문자열에서는 되지 않는 특정 숫자나 날짜 IP 에서만 가능한 범위 쿼리를 보자.

범위 쿼리는 말 그대로 범위 안에 포함된 데이터를 검색할 때 사용한다.

다음예는 kibana_sample_data_flights 인덱스에서 timestamp 필드를 이용해 날짜/시간 범위 쿼리의 요청 예시다.

```json
GET kibana_sample_data_flights/_search
{
  "quety": {
    "range": {
      "timestamp": {
        "gte": "2020-12-15",
        "lt": "2020-12-16"
      }
    }
  }
}
```
- range 를 통해서 범위 쿼리를 날릴 수 있다.
- timestamp 필드에서 2020-12-15 00:00:00 ~ 2020-12-15 23:59:59 안에 있는 데이터를 찾는다.
- gte 와 le 는 검색 범위를 지정하는 파라미터다.
- gte 는 >= 를 나타내며
- lt 는 < 를 나타낸다.
- lte 는 <= 를 나타내며
- ge 는 > 를 나타낸다.
- 날짜/시간 데이터의 경우 포맷이 잘 맞아야 검색이 가능하다. 여기서는 yyyy-mm-dd 인데 실제 인덱스에 저장된 데이터가 yyyy/mm/dd 인 경우 parse_exception 이 발생한다.

날짜 시간 의 경우 현재 시점을 기준으로 계산하는 쿼리가 있을 수 있다. 

다음 예시는 현재 시점 기준으로 한 달 전까지의 모든 데이터를 가져오는 쿼리다.

```json
GET kibana_sample_data_flights/_search
{
  "quety": {
    "range": {
      "timestamp": {
        "gte": "now-1M",
      }
    }
  }
}
```
- M 은 Month 를 나타낸다.
- m 은 minute 를 타나낸다.
- y 는 year 를 나타낸다. 
- w 는 weak 를 나타낸다.
- H or h 는 hour 를 나타낸다.
- d 는 day 를 나타낸다.
- s 는 seconds 를 나타낸다.

#### 범위 데이터 타입 검색

이전에 엘라스틱서치에서 지원하는 여러가지 데이터 타입을 배웠을 때 범위 데이터가 있었다. (integer_range 등)

날짜/시간 타입의 범위 인덱싱에 문서를 넣을 땐 다음과 같이 하면 된다.

```json
PUT range_test_index/_doc/1
{
  "test_date": {
    "gte": "2021-01-21",
    "lt": "2021-01-25"
  }
}
```
- 범위 유형의 데이터는 gte 나 lt 와 같은 범위를 지정해서 넣어야 한다.

범위 유형의 데이터는 다음과 같이 검색을 하면 된다.

```json
GET range_test_index/_search
{
  "query": {
    "range": {
      "test_date": {
        "gte": "2021-01-21",
        "lt": "2021-01-25",
        "relation": "within"
      }
    }
  }
}
```
- 여기서 특별히 보이는 부붕은 relation 인데 크게 3 가지의 값이 있다.
  - intersects (Default): 범위 중에 한 영역만 걸쳐도 문서를 찾는다.
  - within: 도큐먼트의 범위 데이터가 쿼리 범위 내에 있어야 한다.
  - contains: 도큐먼트의 범위 데이터가 쿼리 범위 데이터를 포함해야한다.

### 논리 쿼리

논리 쿼리는 복합 쿼리로 앞에서 배웠던 쿼리들을 조합할 수 있다.

논리 쿼리는 쿼리를 조합할 수 있도록 4 개의 타입을 지원한다.

|타입|설명|
|:---|:---|
|must| 쿼리를 실행해서 참인 문서를 찾는다.|
|must| 복수로 이용할 경우 AND 연산을 의미한다.|
|must_not| 쿼리를 실행해서 거짓인 도큐먼트를 찾는다.|
|must_not| 다른 타입과 같이 사용할 경우 도큐먼트를 제외하는 역할을 한다.|
|should| 단독으로 사용할 경우 실행이 참인 도큐먼트를 찾는다. (= must)|
|should| 복수의 쿼리로 이용할 경우 OR 연산을 한다.|
|should| 다른 타입과 같이 사용할 경우 스코어에만 활용된다.|
|filter| 쿼리를 실행해서 필터/컨택스트를 수행한다.|

예시로 보자. 

#### must 타입

가장 기본적인 must 타입의 사용 예이다.

````json
GET kibana_sample_ecommerce/_search
{
  "query": {
    "bool": {
      "must": {
        "match": {
          "customer_first_name": "mary"
        }
      }
    }
  }
}
````
- customer_first_name 에 mary 인 값을 찾는다.
- match 는 전문 검색이라는 뜻이다. 

must 타입을 복수 개 사용하면 AND 연산을 효과를 얻을 수 있다.

````json
GET kibana_sample_ecommerce/_search
{
  "query": {
    "bool": {
      "must": [
        {"term": {"day_of_week": "Sunday"}},
        {"match": {"customer_full_name": "mary"}}
      ]
    }
  }
}
````
- 여기서는 용어 검색과 전문 검색을 합쳐서 AND 로 검색하는 예시다. 
- 다양한 타입의 검색을 must 로 묶었다.

#### must_not 타입 

가장 기본적인 must_not 타입을 보자.

````json
GET kibana_sample_ecommerce/_search
{
  "query": {
    "bool": {
      "must_not": {
        "match": {
          "customer_first_name": "mary"
        }
      }
    }
  }
}
````
- customer_first_name 이 mary 가 아닌 문서들을 찾는다.

must_not 은 다른 타입과 함께 사용하면 효과적이다.

````json
GET kibana_sample_ecommerce/_search
{
  "query": {
    "bool": {
      "must": {
        "match": {
          "customer_first_name": "mary"
        }
      },
      "must_not": {
        "term": {"customer_last_name": "bailey"}
      }
    }
  }
}
````
- customer_last_name 이 bailey 가 아닌 애들 중에서 customer_first_name 이 mary 인 애를 찾는다.

#### should 타입 

should 타입 하나만 사용하면 must 와 동일하다. 

````json
GET kibana_sample_ecommerce/_search
{
  "query": {
    "bool": {
      "should": {
        "match": {
          "customer_first_name": "mary"
        }
      }
    }
  }
}
````

should 를 여러개 사용하면 OR 연산 효과를 낼 수 있다.

````json
GET kibana_sample_ecommerce/_search
{
  "query": {
    "bool": {
      "should": [
        {"match": {"customer_first_name": "mary"}},
        {"term": {"day_of_week": "Sunday"}}
      ]
    }
  }
}
````

- 여기서는 day_of_week 이 Sunday 이거나 customer_first_name 이 mary 인 애를 찾는다.

#### filter 타입

filter 는 must 와 같은 동작을 하지만 필터 컨택스트 역할을 하기 떄문에 유사도 스코어에 영향을 미치지 않는다.

즉 스코어 계산을 하지 않고 0.0 만 낸다.

다음은 filter 예시이다.

````json
GET kibana_sample_ecommerce/_search
{
  "query": {
    "bool": {
      "filter": {
        "range": {
          "product.base_price": {
            "gte": 30,
            "lte": 60
          }
        }
      }
    }
  }
}
````

- filter 타입과 range 쿼리를 이용했다.

필터 타입을 이용하면 스코어링을 할 도큐먼트 개수를 줄일 수 있어서 유용하다.

### 패턴 검색

검색하려는 검색어가 길거나 정확히 알지 못하고 대략적으로 아는 경우에는 패턴을 이용해서 검색할 수 있다.

패턴 검색은 와일드카드 쿼리와 정규표현식을 이용하는 정규식 쿼리 두 가지 방법이 존재한다.

다만 두 쿼리 모두 용어 수준 쿼리에 해당하므로 분석기에 분리된 용어를 찾기 위한 쿼리이다.

#### 와일드카드 쿼리

와일드카드는 용어를 검색할 때 * 와 ? 를 이용한다.

* 는 공백까지 포함해서 글자 수 상관없이 매칭할 수 있고 

? 는 정확히 한 용어만 매칭하는게 가능하다.

와일드카드 패턴 검색을 이용할 땐 첫 글자에 * 와 ? 를 쓰지 않도록 하자. 매우 많은 경우의 수를 만든다.

와일드카드 패턴에서 `aabbcd` 를 가진 문서를 찾는다고 해보자.

다음과 같은 패턴은 성공한다.
- `*`
- `aabbcd*`
- `a?bb???`
- `a????cd`

다음과 같은 패턴은 실패한다.
- `b*`
- `a*e`
- `a?`
- `aabbbcd?`

와일드카드 패턴 검색은 다음과 같이 호출하면 된다.

```json
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "wildcard": {
      "customer_full_name.keyword": "M?r*" 
    }
  }
}
```
- 이렇게 검색하면 Mary, Mar, Mora dory 와 같은 용어를 포함한 문서가 매칭될 것이다.

#### 정규식 쿼리 

이번에는 정규식을 이용해 쿼리를 날리는 방식을 살펴보자.

여기서도 * 와 ? 기호가 나오는데 와일드카드와는 다소 다르다.

먼저 `.` 기호를 알아보자.

`.` 은 하나의 문자를 의미하고 어떤 문자가 와도 상관없다는 기호다.

`+` 기호는 앞 문자와 같은 문자가 하나 이상 반복된다고 판단될 때 사용하는 기호다.

`*` 는 앞 문자와 같은 문자가 0 개 이상 반복된다고 판단될 때 사용하는 기호다.

`?` 기호는 앞문자와 같은 문자가 0 번 혹은 한 번 나타나면 매칭되었다고 판단한다.
- `b?` 의 경우 b 나 공백에 매칭된다.

`()` 기호는 그룹핑을 위해서 사용하는 기호다.

`[]` 기호는 특정 범위 매칭에 사용하는 기호다. 

이번에는 정규식을 이용해서 쿼리를 날려보자.

```json
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "regexp": {
      "customer_first_name.keyword": "Mar."
    }
  }
}
```
- 이 검색의 경우 Mar 다음에 하나의 문자가 오면 매칭되는 검색이다. Mary 같은것이 검색될 것이다.
