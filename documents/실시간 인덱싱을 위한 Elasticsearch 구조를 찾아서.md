# 실시간 인덱싱을 위한 Elasticsearch 구조를 찾아서. 

https://techblog.woowahan.com/7425/

***

- 이것도 성능 개선 이야기

- 환경은 다음과 같다.
  - 데이터 (가게 수) 약 100만개
  - 데이터 변경 빈도 수천 건. (초당)


- 개편된 구조
  - 미리 어떻게 되었는지 결과를 말해준 것.
  - 기존 인덱스를 필드의 변경 빈도에 따라 두 인덱스로 분리.
  - 검색 시에는 두 인덱스에 검색 쿼리를 요청하고 결과를 조회한다.


## 문제

- 가게 변경 이벤트의 폭증
  - 이로인해서 검색 timeout 발생
  - 가게 변경 이벤트 처리의 지연.


## 저장하는 문서

- 가게 하나의 문서를 ES 에 저장한다.
- 그리고 메뉴 같은 건 nested 필드로 저장된다.
  - Nested object 는 쓰기 증폭을 유발한다.
  - Parent object 와 nested object 는 모두 새로 쓰이니까.
  - 그리고 이건 query cache 를 무효화한다.


## 어떻게 해결했는가?

- 색인을 나눴다.
  - 자주 변경되는 정보 (가게 정보, 배달 예상 시간과 같은) 와 메뉴 정보 (잘 변경되지 않는 정보)

- 색인을 나누면 검색은 어떻게 되는데?
  - 이게 문제임. 기존과 동일한 검색 결과를 얻어내는 방법이 중요.
  - 검색 성능은 괜찮은지.


## 검색 요청의 일반적 구조

- Query Step
  - 원하는 조건에 맞는 문서만 필터링 하는 단계
  - 개념 상 ES 에 query phase 에 해당

- Decision Step
  - 필터링 된 문서를 바탕으로 정렬 및 페이징을 진행해서 최종 문서 집합을 결정.

- Fetch Step
  - 최종 문서의 필드를 가져오고 highlighting 을 진행해서 최종 결과를 만드는 과정
- 개념상 Decision Step + Fetch Step 이 ES 에 fetch phase 에 해당됨.


## 그래서 검색 요청은 어떻게 되는데?

- 기존 인덱스가 가게 인덱스, 메뉴 인덱스로 분리되었기 떄문에 query step 을 분리해야함.
  - 1. Shop query
  - 2. Menu query
  - 3. Shop decision
  - 4. Menu decision
  - 5. Shop fetch
  - 6. Menu fetch

- 근데 이 경우에 filter 를 shop 과 menu 에 있는 필드를 합쳐서 검색하는게 불가능해짐. 
- 그래서 shop 에 있는 정보 중 일부가 menu 에 넘어감.
  - 비정규화를 했다. 이런 뜻.
- 그리고 실제로 검색 과정에선 더 간소화가 됨.
  - 정렬에 Menu field 가 사용되지는 않음.
    - 그래서 Menu decision 은 제거됨.
  - 같은 인덱스에 요청하는 연속 step 은 합칠 수 있다.
     - 그래서 shop decision + shop fetch 를 합칠 수 있음.
- 정리하자면 이를 통해서 인덱스는 나눠졌지만, 검색 결과는 동일하게 맞출 수 있었음.


## 그렇다면 검색 성능은?

- Menu query step 에서 latency 가 3배 이상 걸림.
  - ES 에 저장되는 메뉴 문서는 메뉴 하나에 해당함.
  - 모든 메뉴를 가져오도록 모든 샤드에 요청됨. 그리고 이걸 집계 (aggregation) 하는 과정에서 걸리는 시간.
    - Coodrinating Node 가 data node 들에게 요청을 보내고 그 결과들을 집계함.
    - Minimize Coodinate 를 못해서 그럼. Coordinating node 가 병목이됨.
    - 이 과정을 하지 않도록 해야함.
    - 쿼리 시간에 집계하지말고, 색인에 그걸 반영해두자.
      - 메뉴를 가계단위로 묶어놓는거지. Nested document 를 이용해서.
    - ES 의 Document Modeling 을 따르면 검색을 더 간단하게 할 수 있다고함.

- 검색 step 을 바꿈.
  - Menu query -> Shop query, decision fetch -> Menu fetch
  - 왜냐하면 여기서는 대부분 필터링이 Menu 에서 걸러지기 떄문에, 초반에 많은 걸 거르는게 더 나으니까.

- Sharing 의 사이즈를 줄이고 개수를 늘리는 방식으로 개선
  - 하나의 샤드에 많은 데이터가 들어가면 그만큼 느려질 수 있기 때문에.



## ES 의 nested 타입  

- nested 타입은 object 타입의 특수한 버전이다.
  - 오브젝트의 array 를 허용한다. 그리고 각각의 오브젝트를 검색할 수 있도록 도와줌.
  - 그냥 오브젝트 어레이 타입을 색인하면 계층 구조가 깨진다. 그래서 부정확한 검색이 가능할 수 있음.

- Nested 타입은 어레이 속의 object 를 hidden document 로 취급해서 nested document 를 독립적으로 검색할 수 있게 도와줌.
  - 이떄는 nested query 를 써야한다.

- 예시로 보면 제일 나을듯.
- 내부적으로 parent object 와 netsted object 로 구분된다.
  - Document 안에 있는 nested object 는 원본의 parent object 와 구분되서 저장되서 검색할 수 있게됨.
  - 그리고 이것들은 연속된 위치에 저장되어서 서로간의 연관관게를 유지한다.
  - 따라서 nested object 와 parent object 는 모두 새로 쓰인다.

- Object array 를 포함할 수 있음. 그리고 이 배열의 요소들이 검색이 되야함.
- Nested Type 보다 flatten 타입이 더 나은 경우가 있음.
- Nested type 으로 매핑을 지정해서 색인하면 배열의 경우 multi value field 로 색인된다.
- 참고
  - https://www.elastic.co/guide/en/elasticsearch/reference/7.10/nested.html#nested-arrays-flattening-objects


## ES 의 flatten 타입

- 기본적으로 오브젝트의 subfield 들은 각각 매핑되고 색인된다.
- Flatten 타입을 쓰면 오브젝트를 하나의 필드로 매핑해서 검색할 수 있다.
  - 색인은 leaf value 들을 함.

- 이것도 예시를 보면 더 이해하기 쉬울듯.
- 이 타입은 오브젝트가 크고, 유니크한 키들을 알 수 없을 때 유용함.
- Mapping explosion 을 막아준다고함.
  - 너무 많은 distinct 매핑 필드를 가지는 것을 막음.
  - 메모리 부족을 유발함.
  - 색인에 너무 많은 필드를 가지는 것은 안좋다고함.
  - 매번 문서의 새로운 필드가 들어와서 다이나믹 매핑이 된다면, 매핑이 커지는 문제가 생긴다.
  - JVM Heap size 의 문제를 일으킴. Mapping 이 너무 커져서.
- https://www.elastic.co/guide/en/elasticsearch/reference/7.10/flattened.html


## ES 의 query cache

- filter context 에 사용된 쿼리의 결과는 node query cache 에 저장된다.
- 노드별로 query cache 가 있다.
- LRU eviction 정책을 이용함.
- Term queries and queries 는 query cache 에 사용되지 않는다.
- Total heap space 의 10%, 최대 10000 개의 쿼리를 가질 수 있다.
- Segment 기반 캐싱도 가능하다.
  - Segment 에 10000 개의 문서가 있고, 세그먼트가 shard 의 total document 의 3% 를 넘는 경우. 
  - 이 경우 세그먼트의 병합은 캐시를 무효화한다.


## ES 의 fetch phase 란?

- 참고
  - https://www.elastic.co/kr/blog/found-elasticsearch-top-down#then-fetch


## ES 의 Coordinating Node 와 Data Node 는?

- Coordinating Node
  - 요청을 처음 받는 노드.
  - 요청을 받으면 data node 에 scatter 한다. 그래서 scatter phase 라고 함.
  - 그리고 요청을 돌려받으면 reduce result 해서 client 에 돌려준다. 이걸 gather phase 라고함.

- Data node
  - Coordinating node 에 요청을 받으면 요청을 처리하고 돌려줌.


## ES 의 Document Modeling

- join 은 피하도록.
- Denormalizing documents 를 통해서 성능을 개선할 수 있다.
- https://www.elastic.co/guide/en/elasticsearch/reference/7.10/tune-for-search-speed.html#search-as-few-fields-as-possible 


## ES 의 “copy_to” 는 뭔데?

- multiple fields 의 value 를 copy 해서 group field 로 만드는 것.
- 그리고 하나의 필드로 검색이 가능함.
- Multiple field 를 통해서 검색을 하고 있다면, 이걸 이용해서 검색 성능을 높일 수 있다.
- 예제 보면 더 쉬울듯.
- Copy 된 field 는 term 분석 과정을 거치지 않는다. 그래서 더 빠름.
  - Text 필드인데, 이걸 안거친다면 빠르겠지.
- https://www.elastic.co/guide/en/elasticsearch/reference/7.10/copy-to.html
