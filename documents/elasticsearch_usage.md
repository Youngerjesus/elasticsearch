# Elasticsearch 의 사용법 

- 여기서는 kibana -> management -> devtool 기준으로 설명. 

## 시스템 상태확인 

```http request
GET _cat 
```

- 노드, 샤드, 템플릿 정보들에 대해서 알 수 있다.

## 클러스터 내부 인덱스 목록 

```http request
GET _cat/indices?v
```

## 인덱스 CRUD 

- 인덱스는 용량이 있으므로 날짜나 시간에 따라서 생성되는 경우에는 구별할 수 있도록 해야겠다.  

#### 인덱스 생성 

```http request
PUT index1 // index1 이라는 인덱스 추가. 
```

#### 인덱스 조회 

```http request
GET index1 // index1 이라는 인덱스 검색 
```

- `mappings` 를 보면 인덱스의 매핑 정보를 볼 수 있다. 
  - 매핑을 따로 지정하지 않으면 데이터를 보고 알아서 특정 타입으로 매핑 짓고 색인한다. 
  - 자동 형변환 정도는 알자. 
    - 숫자 필드에 문자열이 들어오면 숫자로 형변환 할려고한다. 
    - 정수 필드에 소수가 들어오면 소숫점이 사라진다.

#### 인덱스 삭제 

```http request
DELETE index1 // index1 이라는 인덱스 삭제
```

## 도큐먼트 CRUD 

#### 인덱스에 도큐먼트 생성

```http request
PUT index1/_doc/1 // index1 이라는 인덱스에 1번 문서로 색인.
{
    "name": "mike", 
    "age": 30,
    "gender": "male"
}
```

- 문서 아이디를 넣어줘야한다. 

#### 도큐먼트 불러오기 

```
GET index1/_doc/1 // index1 이라는 인덱스에 1번 문서 가지고 옴.
```

## 벌크 API 

```http request
POST _bulk
{ "index": { "_index": "index2", "_id": 4 } } // index2 에 documentId: 4 인 문서 생성. 문서의 name 은 park 임
{ "name": "park", "age": 30, "gender": "female"} 
{ "index": { "_index": "index2", "_id": 5 } } // index2 에 documentId: 5 인 문서 생성. 문서의 name 은 jung 임
{ "name": "jung", "age": 50, "gender": "female"} 
```

- 벌크는 생성/수정/삭제만 지원한다. 
- 지울려면 `{ "delete": { "_index": "index2", "_id": 4}}` 이런식으로 해야한다.
- 갱신을 할려면  `{ "update": { "_index": "index2"}}` 후에 `{ "doc": { "field2": "value2" }}` 하면 된다.

## 텍스트 타입을 토큰으로 분석 

```http request
POST _analyze
{
    "analyzer": "standard", 
    "text": ""we offer solutions""
}
```

## 텍스트 타입을 가진 text_index 생성 

```http request
PUT test_index
{
  "mappings": {
    "properties": {
      "contents": { "type": "text" } 
    }
  }
}
```

## 텍스트 타입을 가진 도큐먼트 생성 및 검색 

```http request
PUT test_index
{
  "mappings": {
    "properties": {
      "contents": { "type": "keyword" }
    }
  }
}
```

- 색인에 매핑 정보는 이렇게 넣는다. 

```http request
PUT text_index/_doc/1
{
  "contents": "beautiful of day"
}
```

```http request
GET text_index/_search
{
  "query": {
      "match": {
        "contents": "day"
      }
  }
}
```

- `match` 는 전문 검색을 할 수 있는 API 이다. 

## 멀티 필드 지정 및 검색 

```http request
PUT multifield_index
{
  "mappings": {
    "properties": {
      "message": { "type": "text" }, 
      "contents": {
        "type": "text", 
        "fields": {
          "keyword": { "type": "keyword" }
        }
      }
    }
  }
} 
```

- message 및 content 를 색인 필드로 지정했고, contents 는 멀티필드이다. content 자체는 text 로 색인되고, fields 는 

```http request
GET multifield_index/_search
{
  "query": {
    "match": {
      "contents": "day"
    }
  }
}
```

```http request
GET multifield_index/_search
{
  "query": {
    "term": {
      "contents.keyword": "beautiful of day"
    }
  }
}
```

- match 와 term 을 통해서 찾을 수 있다. 
  - term 은 입력한 쿼리와 색인된 필드의 정확한 일치를 말한다.
    - 색인된 필드가 텀이 여러개라면 여러 쿼리로 검색이 되겠지. 
  - match 는 입력된 쿼리를 분석해서 색인될 필드와 매칭을 한다. 
    - `beautiful day` 라고 쿼리를 입력하면 `beautiful` 과 `day` 로 나눠서 검색함.

## 인덱스 템플릿 확인 

```http request
GET _index_template
```

- 전체 인덱스 템플릿을 확인하는 도구.

```http request
GET _index_template/lim-*
GET _index_template/lim-history 
```

- 특정 인덱스 템플릿만을 확인하는 방법

## 인덱스 템플릿 생성 

```http request
PUT _index_template/test_template
{
  "index_patterns": ["test_*"], 
  "priority": 1
  "template": {
    "settings": {
      "number_of_shards": 3, 
      "number_of_replica": 1 
    },
    mappings: {
      "properties": {
        "name": { "type": "text" }, 
        "age": { "type": "short" }, 
        "gender": { "type": "keyword" }
      }
    }
  }
}
```

- index_pattern 을 통해서 새로 만들어지는 인덱스가 있다면 해당 템플릿이 적용된다. 
- priority 는 매칭되는 인덱스 템플릿이 많다면 적용되는 우선순위를 말한다. 
- template 은 새로 생성되는 인덱스에 settings.mappings 와 같은 것들이 적용된다. 

## 다이나믹 템플릿 적용 

```http request
PUT dynamic_index1
{
  "mappings": {
    "dynamic_templates": [
      {
        "my_string_field": {
          "match_mapping_type": "string", 
          "mapping": {"type": "keyword"}
        }
      }
    ]
  }
}
```

## 분석기와 필터를 통한 텍스트 토큰화 

```http request
POST _analyze 
{
  "tokenizer": "standard", 
  "filter": ["uppercase"], 
  "text": "The 10 most loving dog breads"
}
```

## 쿼리 컨택스트 실행 

- 엘라스틱 서치의 검색은 쿼리 컨택스트를 통한 검색과 필터 컨택스트를 이용한 검색 이렇게 있다. 

```http request
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "match": {
      "category": "clothing"
    }
  }
}
```

- 쿼리 컨택스트는 스코어링을 통해서 유사도가 제일 높은 것들을 노출시키겠다는 소리임.


## 필터 컨택스트 검색 

```http request
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

## 쿼리 스트링을 이용한 쿼리 검색 

```http request
GET kibana_sample_data_ecommerce/_search?q=customer_full_name:Mary
```

- 쿼리를 작성하는 방법은 쿼리 스트링을 이용하는 방법과 쿼리 DSL 을 이용하는 방법 이렇게 있다. 
  - 간단한 건 쿼리 스트링, 복잡한 쿼리는 쿼리 DSL 을 이용한다고 생각하면 된다. 

## 쿼리 DSL 을 이용한 쿼리 검색 

```http request
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "match":{
      "customer_full_name": "Mary"
    }
  }
}
```

- term, match, match_phase 등이 쿼리 DSL 을 이용한 검색이다. 

## 매치 쿼리를 이용한 예시 

```http request
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

- `_source` 파라미터를 통해서 응답으로 가져오는 필드는 customer_full_name 만 가져온다.
- 쿼리로 날리는 mary bailey 는 분석기를 통해서 [mary, bailey] 로 토큰화 된다.

```http request
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

- `operator` 를 명시하지 않으면 OR 로 검색된다. 

## 매치 프레이즈 쿼리 

```http request
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

## 용어들 쿼리

```http request
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

## 멀티 매치 쿼리 

```http request
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

## 와일드카드 쿼리 요청하기 

```http request
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

## 멀티 필드 매치 가중치 적용  

```http request
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

## 범위 쿼리 

```http request
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

- 문자열/키워드 데이터는 범위 쿼리가 안된다.  
- 날짜/숫자/IP 데이터는 범위 쿼리가 가능하다. 

#### 범위 쿼리 예시 2 

```http request
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


#### 범위 타입 문서

```http request
PUT range_test_index/_doc/1
{
  "test_date": {
    "gte": "2021-01-21",
    "lt": "2021-01-25"
  }
}
```

#### 범위 타입 검색 

```http request
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

## 논리 쿼리 

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

```http request
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
```

#### must_not 타입 검색 

```http request
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
```

#### should 타입 검색 

```http request
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
```

#### should 타입 검색 (2) (여러개의 should OR 효과) 

```http request
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
```

#### filter 타입 검색 

```http request
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
```

#### 와일드 카드 쿼리 

```http request
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "wildcard": {
      "customer_full_name.keyword": "M?r*" 
    }
  }
}
```

#### 정규식 쿼리 

먼저 . 기호를 알아보자.

- . 은 하나의 문자를 의미하고 어떤 문자가 와도 상관없다는 기호다.
+ 기호는 앞 문자와 같은 문자가 하나 이상 반복된다고 판단될 때 사용하는 기호다.
* 는 앞 문자와 같은 문자가 0 개 이상 반복된다고 판단될 때 사용하는 기호다.
- ? 기호는 앞문자와 같은 문자가 0 번 혹은 한 번 나타나면 매칭되었다고 판단한다.
- b? 의 경우 b 나 공백에 매칭된다.
- () 기호는 그룹핑을 위해서 사용하는 기호다.
- [] 기호는 특정 범위 매칭에 사용하는 기호다.

```http request
GET kibana_sample_data_ecommerce/_search
{
  "query": {
    "regexp": {
      "customer_first_name.keyword": "Mar."
    }
  }
```
