# Rank Feature 

https://www.elastic.co/guide/en/elasticsearch/reference/7.17/rank-feature.html

https://www.elastic.co/guide/en/elasticsearch/reference/7.17/query-dsl-rank-feature-query.html#rank-feature-query-saturation

***

- 스코어와 부정적인 상관관계, 긍정적인 상관관계가 있는 Rank Feature 는 이를 선언해야한다.
  - 예를 들면 Website Searching 의 page rank 같은 경우는 긍정적인 상관관계를 줄 수 있고, url_length 같은 경우는 부정적인 상관관계를 줄 수 있다. 
  - 즉 숫자를 색인해서 rank feature 를 기반으로 문서를 부스팅 할 수 있다.
  - 이 말은 Scoring formula 를 변경시킨다. 스코어를 감소시키는 방향으로. 증가시키는 방향으로도.
  - `positive_score_impact=false`  스코어링과 부정적인 상관관계가 있다면 이렇게 써야한다. (기본 값은 trure)
- Rank feature 필드는 검색이나 aggregate 할 수 없다. 할려면 멀티 필드로 써야함.
- Bool query 의 should 절에서 사용된다.
- Rank Feature Field 와 Rank Feature Query 이렇게 있다. 
- 이 점수를 수학적인 공식을 통해서 값을 매긴다. 
  - Saturation 
  - Logarithm 
  - Sigmoid 
  - Linear

#### Example Rank Feature mapping 

```http request
PUT /test
{
  "mappings": {
    "properties": {
      "pagerank": {
        "type": "rank_feature"
      },
      "url_length": {
        "type": "rank_feature",
        "positive_score_impact": false
      },
      "topics": {
        "type": "rank_features"
      }
    }
  }
}
```

#### 문서 색인 
```http request
PUT /test/_doc/1?refresh
{
  "url": "https://en.wikipedia.org/wiki/2016_Summer_Olympics",
  "content": "Rio 2016",
  "pagerank": 50.3,
  "url_length": 42,
  "topics": {
    "sports": 50,
    "brazil": 30
  }
}

PUT /test/_doc/2?refresh
{
  "url": "https://en.wikipedia.org/wiki/2016_Brazilian_Grand_Prix",
  "content": "Formula One motor race held on 13 November 2016",
  "pagerank": 50.3,
  "url_length": 47,
  "topics": {
    "sports": 35,
    "formula one": 65,
    "brazil": 20
  }
}

PUT /test/_doc/3?refresh
{
  "url": "https://en.wikipedia.org/wiki/Deadpool_(film)",
  "content": "Deadpool is a 2016 American superhero film",
  "pagerank": 50.3,
  "url_length": 37,
  "topics": {
    "movies": 60,
    "super hero": 65
  }
}
```

- 3가지 문서를 색인
- `?refresh` 를 통해서 즉시 색인. 

#### Example Query 
```http request
GET /test/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "content": "2016"
          }
        }
      ],
      "should": [
        {
          "rank_feature": {
            "field": "pagerank"
          }
        },
        {
          "rank_feature": {
            "field": "url_length",
            "boost": 0.1
          }
        },
        {
          "rank_feature": {
            "field": "topics.sports",
            "boost": 0.4
          }
        }
      ]
    }
  }
}
```

- field: Required 
- boost: optional
  - relevance score 를 증가시키거나 감소시킴.
  - `relevance score` 는 내가 아는 그 스코어가 맞음. 검색했을 때 이 스코어를 바탕으로 정렬됨. (`_score` 라는 메타데이터)
