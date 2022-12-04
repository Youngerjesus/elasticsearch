# Lucene 의 검색 

## 여러가지 검색 기능 

- 기본 검색 질의 (텀 기반)
- 범위 검색
- 접두어 검색 
- 와일드카드 검색 
- 구문 질의 
- 퍼지 텀 검색 질의 
- BooleanQuery 를 기반으로 하는 메타 질의 

## 루씬의 핵심 검색 API 컴포넌트 

- IndexSearcher 
  - 검색을 담당하는 부분. search() 메소드를 바탕으로 검색 기능을 수행한다. 
- Query 
  - 질의를 위한 쿼리. IndexSearcher.search() 메소드의 인자로 전달해준다. 
- QueryParser
  - 사람이 직접 입력한 텍스트를 Query 객체로 변환해주는 역할 
- TopDocs
  - 검색 결과의 문서 집합들. 스코어를 바탕으로 내림차순으로 정렬되어 있음. 
- ScoreDocs
  - TopDocs 클래스에 담긴 검색 결과 하나를 나타내는 클래스

## TopDocs 결과 활용 

- `TopDocs search(Query query, int n)`
  - 가장 기본적인 검색 메소드로 n 개의 결과만을 가지고 온다. 
- `TopDocs search(Query query, Filter filter, int n)`
  - filter 를 줘서 검색 대상의 범위를 제한한다. 
- `TopDocs search(Query query, Filter filter, int n, Sort sort)`
  - 2번과 같고 추가로 Sort 를 바탕으로 대상을 정렬할 기준을 준다.
- `void search(Query query, Collector results)`
  - 문서를 조회하는 방법을 새로 구현하는 경우 
- `void search(Query query, Filter filter, Collector results)`
  - 4번에다가 필터까지 적용한느 부분
