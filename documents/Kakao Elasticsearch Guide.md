# Kakao Elasticsearch Guide

## 클러스터 생성을 할 때 고려해야할 사항

### 클러스터 용도를 먼저 정해야한다.

Ex) 유저 생성 피드 저장 및 검색용, 애플리케이션 로그 분석용

### 인덱스에 대해서도 나름 정리를 해야한다.

- 인덱스 패턴에 대해서

- Primary 기준 일 별 생성 데이터 크기

- 인덱스 유지 기간
  - ex) elasticsearch-2019.11.25 / 100GB(daily) / 15일, filebeat-* / 50G(daily) / 30일
  - 이걸 바탕으로 클러스터에 필요한 데이터 노드를 산정한다.
  - 계산식
    - Elasticsearch-* 인덱스 총 보관 용량 = 100GB * 15 * 2 (레플리카 1 기준) = 3TB
    - Filebeat-* 인덱스 총 보관 용량 = 50GB * 30 * 2 (레플리카 1 기준) = 3TB
    - 7.8 TB = 6TB + 6TB * 0.3 (버퍼 30%)
    - 1.6 TB = 7.8TB / 데이터 노드 의 수 (5 로 가정)
    - 데이터 노드 당 1.9 TB = 1.6 TB  + OS 영역 

- 초당 인입되는 Document 개수 (예상치) 에 대해서도 정리해야한다.
  - ex) 5,000 docs/s

- ES 버전에 대해서도 정해야한다.


### Elasticsearch 는 검색 엔진과 분석 엔진으로 크게 사용할 수 있다.

- 검색 엔진
  - 검색 엔진으로 ES 를 쓸 땐. 응답속도가 제일 중요한 요소이다.
  - 그래서 고려할 요소로는 데이터 규모, 실행할 쿼리, 원하는 응답속도 상한선, 검색 부하량이 가장 많은 시간대, 캐시 가능 유무 등을 확인해야한다.

- 분석 엔진
  - 최근 ES 는 분석 엔진으로도 많이 쓰고 있다.
  - 어플리케이션에서 발생한 로그를 바탕으로 로그를 집계하거나, 로그 속에 있는 데이터들을 집계하거나 등에 이용
  - 분석 엔진의 ES 의 경우 일반적으로 데이터의 규모가 검색 엔진보다 크다.
  - 고려할 요소로는 다음과 같다.
    - 1일당 색인 데이터 크기와 보관 기간
    - 1초당 색인량
    - 검색 부하량과 가장 사용량이 많은 시간대집계에 사용할 쿼리
    - 핫/웜 구조 사용 유무 (데이터를 tier 에 맞춰서 관리하는 것.)
      - 자주 사용하는 데이터는 SSD 와 같은 곳에 저장하고, 접근을 잘 안하는 데이터는 비교적 저렴하고 용량이 큰 SATA 노드에 저장하는 것.
      - 핫 웜 구조에서는 쿼리를 좀 더 정교하게 써야한다.
        - 예로 고객의 나이별 구매 수량을 하루씩 기간으로 쪼개서 날리는 게 아니라 하나의 쿼리로 날리면 ES 가 먹통이 될 수 있다. (행업) 잘 쓰지 않는 티어의 데이터까지 접근하는 쿼리가 포함된 경우 쪼개서 날려야할듯

## ES 성능 개선

- 쿼리 튜닝
- 인덱스 보관 기간 검토 (분석 엔진의 경우)
- 인덱스 저장 단위 변경 (분석 엔진의 경우)
- Primary/Replica 샤드 설정
- Elasticsearch 설정 변경 (queue, cache 등)
- 슬로우 로그 조회

### 색인 성능 개선

- 인덱스 매핑 정의
  - 스키마를 어떻게 할 건지 정하는 것.
  - 하나의 인덱스에는 하나의 인덱스 매핑만 적용가능.
  - 나이 같은 타입은 long 타입보다는 생년월일을 저장하는게 낫다. 계속 값이 변할거기 때문에.

- 인덱스 매핑으로 Dynamic mapping 을 쓰지 않는게 좋다. 
  - 문자열을 저장하면 text 타입과 field 타입 둘 다 생성되기 떄문에.
  - 문자열 같은 경우는 text 로 쓸 건지 keyword 로 쓸 건지 확실하게 정해야한다.

### 검색 기능 개선

- ES 에서 날릴 수 있는 쿼리의 종류로는 Query DSL 에 정리되어있다.
  - Query and filter context
  - Match All Query
  - Full text queries
  - Term-level queries
  - Compound queries
  - Joining queries
  - Geo queries
  - Specialized queries
  - Span queries
  - Minimum should match
  - Multi Tem Query rewrite

  - (요거 정리해놓는 것도 좋을듯.)


- 여기서 크게 나누면 Full text queries 와 Term level queries 로 나뉜다.
  - 문서의 유사성을 바탕으로 검색할건지, 정확히 일치하는 걸 기반으로 검색할건지.
  - **이러한 개념을 Query Context 와 Filter Context 로 나뉜다.**
  - Query Context 는 쿼리가 문서와 얼마나 관련이 있는지로 나뉜다. (Full Text Queries)
  - Filter Context 는 이 검색 쿼리가 문서에 일치하는가? (Term level 의 query)  
    - 이건 캐싱이 된다. 문서들의 내용이 Heap 에 저장됨.
    - Bool 쿼리의 filter or must_not 항목에 Filter Context 가 들어갈 때
    - constant_score 쿼리의 filter 항목에 filter context 가 들어갈 때
    - Filter aggregation 항목의 쿼리에 filter context 가 들어갈 때
    - Term 쿼리는 캐시가 되는가? 아님. 그래서 scoring 기반의 쿼리가 아니면 Filter Context 를 이용하도록 해야한다
      - 레퍼런스에서 이렇게 적혀있음.
        - Term queries and queries used outside of a filter context are not eligible for caching.
    - 이때의 캐시를 query cache 라고 한다. 그리고 쿼리 캐시가 되기 위한 제약도 있다.
      - 쿼리의 대상이 되는 인덱스가 클러스터 내의 전체 인덱스 크기의 3% 이상일 때
      - 쿼리의 대상이 되는 인덱스 내 샤드를 구성하는 세그먼트가 10,000 개의 문서 이상을 가질 때

- 캐싱이 잘 되고 있는지 확인하려면 이걸 검색해보면 된다. http://localhost:9200/_cat/nodes?v&h=name,qcm,qce&s=name:asc%27
  - Eviction 이 일어난다면 캐시 공간이 부족하다는 뜻으로 좀 더 늘려줘도된다.
  - 기본적으로 쿼리 캐시의 공간은 Heap 의 10% 이다.

- Full Text Queries 의 뜻 자체는 analyzing 한 토큰들을 검색하는 것을 말한다.

- 쿼리 캐시 (Query Cache) 에 대해서.
  - 노드에 저장되는 요소임.
  - 노드에 샤드가 여러개 있으니까 샤드에 공유된다.


## ES 팀에서 기본적으로 모니터링 하는 요소

- JVM Heap 사용률 
  - 건강한 JVM 의 경우에는 GC 가 이렇다. 
    - Heap 그래프가 톱니 패턴임. (30% ~ 70% 를 왓다갓다하는.) 이것보다 높은 Heap 사용량은 GC 가 Heap 이 차는 걸 못따라가서 그렇다. 
    - Young GC is processed quickly (within 50 ms)
    - Young GC is not frequently executed (about 10 seconds)
    - Old GC is processed quickly (within 1 seconds.)
    - Old GC is not frequently executed (about 10 minutes or more)
  
  - Heap 사용량이 너무 높다면 이유는 많을 수 있다. 크게 이렇다고 하는듯. 
    - Oversharding 
      - 클러스터가 너무 많은 샤드의 metadata 상태를 관리해야하는게 부담이다. 샤드의 수가 많아질수록 오버헤드는 더 심해진다. 
      - 이상적인 샤드는 a few GB ~ a few ten of GB 이다.
    - Large aggregation sizes
    - Excessive bulk size 
    - Mapping issues
    - Heap size incorrectly set
    - JVM New ratio incorrectly set 

  - 번외로 기본적으로 RAM 의 50% 가 가장 밸런스 있는 값이라고 한다.
  - 캐시로 쓰일 메모리와 Heap 으로 쓰일 공간의 밸런스.

- 샤드 개수 초과
- 클러스터 노드 장애
- 클러스터 상태 변경

- 적절한 샤드 개수
  - 여기서 말하는 샤드는 프라이머리 샤드 수를 말한다.
  - 샤드당 20GB ~ 40GB
  - 노드개수의 배수 또는 약수

- (여기에 있는거 정리가 필요할 듯)


