# Bulk API 

https://www.elastic.co/guide/en/elasticsearch/reference/7.17/docs-bulk.html#bulk-clients

***

## Index alias 가 뭔데

- alias 는 indices 또는 data stream 의 group 의 또다른 이름이다.
  - 이렇게 두 개가 있다.
  - 내가 원하는건 index alias 이고.
  - 아 spider 에서 쓰는게 스트림에서는 alias 로 적재를 하고, 배치에서는 버저닝을 해주고 alias 를 바꾸는 역할까지 하는거구나.

## op_type 이 뭔데?

- enum 값이다.
- Index 인지, create 인지.

## Data Streams 는 뭔데?

- 여러 인덱스에 걸쳐있는 append-only time series data  저장할 때 쓴다.
- 대신 요청을 보낼 때 single name 만 쓰는듯?
- Log 나 event metric 같은 것에 적합하다.

## Bulk Description

- index, create, delete, and update 가 하나의 request 에서 가능하다.
- Index 와 create 는 next line 있어야한다.
- Update 는 partial doc 이나 upsert, script 같은 것들을 할 수 있다.
- <taget> 을 request path 에 지정하면 _index 는 안써도 된다.
- action_meta_data 는 받는 노드에서만 parsing 된다.
  - 이게 뭔데
  - 작업을 해야하는 정보를 담은 메타데이터
    - Action 의 type
    - op_type 과 연결해서 씀.
  - 요청을 보낼 때 metadata 와 request body 데이터 이렇게 보낸다는 듯.
    - Request body 가 문서 데이터일거고
- 벌크 요청을 위해서 클라이언트 라이브러리를 쓰는 경우에는 buffering 을 가능하다면 줄이라고 하는데 이게 bulk size 를 줄이라는 것일까?
- 최적의 사이즈라는 건 없지만 HTTP 요청이 100mb 를 안넘도록 하자. Elasticsearch 는 이 크기로 제한시킨다.
- 물론 하나의 문서가 이 사이즈보다 크면 이것도 색인이 안됨.

## Optimistic concurrency control

- index 와 delete action 은 아마 `if_seq_no` 와 `if_primary_term` 파라미터를 포함할 것이다.
- 이 파라미터들은 operation 이 어떻게 실행되는 지를 결정해준다는 듯.
- Existing documents  에 있는 last modification 을 기반으로.
- Older version 의 문서가 newer version 의 문서를 덮어쓰지 않도록.
  - 비동기, 동시성 기반으로 문서가 create, update, delete 되니까.
  - 이걸 하지 않도록 모든 operation 에 sequence number 를 할당한다.
- Primary shard 에 의해서`_seq_no` 와 `_primary_term` 을 써서 추적한다.


## Versioning

- 버전을 순차적으로 적용하기 위한 것이겠지.
- 각각의 색인된 문서는 version number 를 가질 수 있다.
- index 와 delete action 을 할 때 version 이 따른다.
  - Version_type 도 지원함.
- Versioning 을 통해서 작업의 성공 실패를 결정한다.
- Internal versioning 과 external versioning 이렇게 있음.
  - Internal versioning 은 1부터 시작해서 각각의 update, delete 마다 증가한다.
  - External versioning 은 database system 같은 거라고 하고, 여기서 준 값으로 세팅한다고 하는듯.
  - External version type 을 쓸 때 색인 요청에 전달된 버전이 document 에 내제된 버전과 비교한다.
  - 문서의 버전이 더 크면 실패. 아니면 성공.
  - 실패나면 충돌이 발생한다.
  - 409 status 코드 발생
- External system 을 왜쓰는거지
  - 분산환경에서는 더 적합하다곤 하는듯. 
- Version 이 제공되지 않는다면 version check 는 하지 않는다.

## Sequence 와 version 의 차이는?

- sequence 는 해당 index 에서 발생한 operation 의 수.
  - 그래서 해당 인덱스에 문서를 색인할 때마다 이 번호는 증가하지만 versioning 은 증가하지 않는다.
- Version 은 문서가 업데이트된 수.
- 하나의 문서내에서 sequence 와 version 은 그렇게 큰 차이가 없는 것 같다.

## Routing

## Wait for active shards

- 샤드에 복사된 최소 숫자.
- 기본은 1 으로 primary 에 저장되면 응답이 옴. `ALL` 로 설정하는 방법도 있다. 

## Refresh 

- 변화를 visible 하게 볼 수 있돌고 하는 것 
- 기본은 false 이다. `wait for` 을 하면 기다림.  
