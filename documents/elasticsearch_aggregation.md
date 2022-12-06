# Elasticsearch Aggregation 

## Intro 

- 집계는 SQL 로 치면 `GROUP_BY` 절을 통해서 데이터를 그룹핑하고 통계를 낼 떄 사용한다. 
  - 예컨대 날짜 별로 묶거나 카테고리 별로 묶을 때. 
- 고성능 검색 엔진 기능 뿐 아니라 고성능 집계 엔진으로의 역할도 할 수 있다. 이건 그리고 키바나와 쓰일 떄 더욱 효과적이라고 함. 
  - 키바나의 데이터 시각화 기능은 집계를 기반으로 하는 것이니까. 집계를 더 잘 알면 키바나도 잘 알지 않을까. 그런 것. 

## 집계의 요청 - 응답 형태 

- 엘라스틱 서치의 집계는 크게 두 가지다. 
  - 메트릭 집계: 통계나 계산을 위한 집계
  - 버킷 집계: 도큐먼트 그룹핑을 위한 집계 

#### 집계를 위한 기본 형태 

```http request
GET <인덱스>/_search 
{
  "aggs": {
    "my_aggs": {
      "agg_type": {...}
    }
  }
}
```

- aggs 는 집계 기능을 이용하겠다는 뜻. 
- my_aggs 는 사용자가 정한 집계 이름이다. 
- agg_type 은 집계 타입을 말한다. 

#### 응답 형태 

```json
{
    ...
    "hits": {
      "total": {...}
    }, 
    "aggregations": {
      "my_aggs": {
        "value": {...}
      }
    }
}
```

- value 가 실제 집계 결과이다. 

## 메트릭 집계 

- 메트릭 집계는 필드의 최소/최대/합/평균/중간값 등을 계산/통계 할 때 사용한다.
- 필드의 타입에 따라서 집계 기능에 제한이 있다.
  - 텍스트의 경우에는 합계/평균 기능이 안됨. 

| 메트릭 집계 | 설명 |
|:--:|:---:|
| avg | 평균값을 계산 |
| min | 최솟값을 계산 |
| max | 최댓값을 계산 |
| sum | 합계를 계산 |
| percentiles | 필드의 백분위값을 계산 |
| stats | min, max, sum. avg. count 등을 한번에 보기 가능 |
| cardinality | 필드의 유니크한 값 개수를 보여준다. |
| geo-centroid | 필드 내부의 위치 정보의 중심점을 계산한다. |


#### 집계 예시 

```http request
GET kibana_sample_data/_search 
{
  "size": 0, 
  "aggs": {
    "stats_aggs": {
      "avg": {
        "field": "products.base_price"
      }
    }
  }
}
```

- 평균 (avg) 집계를 이용한 것. 
- 집계 필드로는 `products.base_price` 를 이용함.
- 응답을 볼 떄 `stats_aggs` 쪽을 보면 된다. (여기서 한 단계 더 depth 를 들어가면 `stats_aggs.value` 를 보면 됨.)

#### 백분위를 구하는 집계 


```http request
GET kibana_sample_data/_search 
{
  "size": 0, 
  "aggs": {
    "stats_aggs": {
      "percentiles": {
        "field": "products.base_price",
        "percents": [25, 50]
      }
    }
  }
}
```

#### 필드의 유니크한 값 개수 확인하기 

```http request
GET kibana_sample_data/_search 
{
  "size": 0, 
  "aggs": {
    "stats_aggs": {
      "cardinality": {
        "field": "day_of_week",
        "precision_threshold": 100
      }
    }
  }
}
```

- `day_of_week` 필드의 유니크한 데이터 개수를 본다.
- `precision_threshold` 는 정확도 수치라고 보면 된다. 정확도가 올라갈 수록 리소스를 더 많이 쓴다고 보면 됨.
  - 기본 값은 3000 이며 40000 까지 넣을 수 있따. 
- 카디널리티는 매우 적은 메모리로 집합의 원소를 측정할 수 있는 `HyperLogLog++` 알고리즘을 쓴다. (레디스에서 본 듯.)
  - 집합 내 중복되지 않은 개수를 세기 위한 알고리즘이다. 
  - HyperLogLOg 를 병렬 처리를 통해 개선된 버전이 HyperLogLog++ 임. 
  - 특징으로는 완벽히 정확한 값을 내지는 않지만 오차 범위는 5% 미만이다. 그리고 정확도를 직접 지정 하는 것도 가능하다. 
  - 대용량 데이터에서 유용하다는 듯. 

#### 용어 집계 요청

```http request
GET kibana_sample_data/_search 
{
  "size": 0, 
  "aggs": {
    "card_aggs": {
      "term": {
        "field": "day_of_week"
      }
    }
  }
}
```

- 용어 별 집계로 유니크한 값들도 볼 수 있다. 개수와 함께.

### 검색 결과 내에서의 집계 

- 검색 쿼리와 함께 집계를 사용하는 기능. 
- 예시로 `day_of_week` 의 필드 값이 `Monday` 인 문서만 집계하는 것. (여기선 메트릭 집계. )

```http request
GET kibana_sample_data/_search 
{
  "size": 0, 
  "query": {
    "term": {
      "day_of_week": "Monday"
    }
  }
},
{
  "size": 0, 
  "aggs": {
    "query_aggs": {
      "sum": {
        "field": "products.base_price"
      }
    }
  }
}
```
