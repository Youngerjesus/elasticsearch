# Refresh 

https://www.elastic.co/guide/en/elasticsearch/reference/7.17/docs-refresh.html
https://www.elastic.co/guide/en/elasticsearch/reference/7.17/indices-refresh.html

***

## Refresh 란

- index, update, delete, bulk api 등으로 색인에 변화를 줬을 때 변화가 search 시점에 보이는 걸 반영하는것.
- 기본적으로 3가지 옵션이 있다.
  - False (the default)
    - Refresh 를 하지 않는 것. 떄가 되면 자동으로 검색했을 때 변화를 볼 수 있다.
  - Wait for
    - 색인했을 때 변화가 되는 시점까지 기다렸다가 응답받는 것.
    - 이 때는 `index.refresh_interval` 을 기다려야된다.
  - Empty string or true
    - Primary shard 에 Refresh 를 하도록 하는 것. 즉시 refresh 를 하기 떄문에 빠르지만 그만큼 resource 를 쓴다.


## Choosing which setting to use

- 변화를 바로 봐야하는 게 아니라면 항상 refresh=false 인 기본 설정으로 하는 게 좋다.
- 하지만 변화를 볼 수 있도록 해야한다면 true 옵션으로 줄 것인지, wait for 로 할 것인지 설정해야한다. 여기에는 몇가지 고려 포인트가 있다.
  - `wait for` 로 기다리는 이유가 적절한 segment 크기를 만들기 위함이다.
  - 색인에 변화가 많을수록 wait for 이 더 효율적이다.
  - `true` 옵션은 색인을 만들 때 더 비효율적이다. 더 작은 semgent 를 만드니까. 색인시점에 더 많은 비용을 지불해서 그만큼 검색이 더 빠르다.
    - 나중에 segment merge 를 해야한다.
- 하나의 row 에서 Multiple request 로 `wait for` 옵션으로 날리지말고 하나의 배치 요청으로 `wait_for` 로 날려라.
- `index.refresh_interval` 을 -1 로 설정한다면 자동 refresh 를 금지한다. 그래서 `refresh=wait_for` 은 무기한으로 기다린다.
- `refresh=true` 는 다른 요청에도 영향을 미치지만 `refresh=wait for` 는 해당 요청에만 영향을 미친다.

- refresh=wait_for 은 refresh 를 강제로 실행할수도 있다.
  - `index.max_refresh_listeners` 의 기본 개수가 1000 인데 만약 리프레쉬를 기다리는 request 가 이 값을 넘는다면 refresh 를 강제로 요청한다.

### Refresh Usage

````http request
POST /my-index-000001/_refresh
````
    
### Index Setting 에서 refresh_interval 을 바꿔서 테스트 가능하다.

````http request
PUT /my-index-000001/_settings
{
  "index" : {
    "refresh_interval" : "10s"
  }
}
````
