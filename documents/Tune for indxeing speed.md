# Tune for indexing speed 

https://www.elastic.co/guide/en/elasticsearch/reference/7.10/tune-for-indexing-speed.html#_disable_swapping_2

***

## Use bulk requests

- 최적의 벌크 사이즈를 찾기 위해선 benchmarking 을 해봐라.
- 하나의 샤드의 퍼포먼스를 체크.
- 도큐먼트의 개수를 올리면서 100 -> 200 이렇게.
- 벌크 요청을 할 떄 memory pressure 를 생각해라.
  - 벌크 요청 하나당 MB 단위라면 문제가 생긴다. 스왑을 쓰거나 out-of-memory error 를 낼 듯. 


## User multiple workers/threads to send data Elasticsearch

- 하나의 스레드로 벌크 요청을 보내는 건 ES 의 최대 색인 속도가 아닐 것.
- 그래서 요청을 보내는 스레드는 여러개야한다.
- 그리고 각각의 요청마다 fsync 를 피해야한다고 한다. fsync 가 뭐지?
  - Operation system 에 있는 기능.
  - 이걸 true 라고 하면 디스크에 바로 반영한다. 그래서 시스템 장애가 전원이 나가도 잃는 데이터는 없다. `Durability` 를 높이기 위한 것.
  - Write Operations 이 느리다. 대신에.
  - 이 값은 index level or shard level 에서 적용이 가능하다.  
    - `index.translog.durability`
    - 기본 값은 `request` 로 설정됨.
      - 모든 색인에 대한 요청을 fsync 로 처리한다. Primary 에 쓰면 응답을 줌.
    - 다른 값으로 async 가 있다.
      - sync_interval 을 주기로 fsync 와 commit 을 한다.
    - `index.shard.translog.durability`
- 색인을 할 때 `TOO_MANY_REQUESTS` response code 를 받을 수 있도록 해야한다.
  - 이건 ES 에서 색인 요청을 따라가지 못한다는 응답임.
  - 현재 ES 의 부하가 spike 쳤다. 이런 뜻.
- 이 경우에는 잠시 색인 작업을 멈췄다가 다시 재개해야한다.
  - 재개할 땐 `randomized exponential backoff` 을 사용해야한다.
  - Retry 하기 전 delay 시간을 지수적으로 증가시키라는 것.
  - ES Client Library 에서 `exponential_backoff_retry` 이 기능이 있는듯.
  - Retry 의 최대 수와 최대 딜레이 시간을 argument 로 줘야한다.
  - 그러면 retry object 를 주고, 이걸 이용해서 retry 하면 된다. 이런 뜻.


## Unset of increase the refresh interval

- refresh 라는 기능을 쓰는 것은 indexing speed 에 영향을 미친다.
  - Refresh 는 색인을 반영해서 검색에 결과가 나오도록 하는 기능.
- 엘라스틱 서치는 주기적으로 색인을 refresh 한다. 매초마다. 그리고 이건 검색 요청을 받은 색인에 대해서만 이런 작업을 함. 마지막 30초 전에 받은 요청에 대해서만.
  - 이 기본 설정은 검색을 거의 안하는 경우에는 색인에 최적화를 할 수 있는 설정이다. 그래서 색인 속도를 빠르게 하는데 영향을 준다.
    - 검색이 없으면 refresh 를 안할거고, 검색이 있으면 하도록 하니까.
  - 즉 검색을 하면 1초마다 refresh 가 되는것임.
- 문서가 색인되는 시간과, 색인된게 보이는 시간 사이의 간격을 줄 수 있는 경우에는 `indexing.refresh_interval` 을 올리면 된다. 그러면 색인 속도가 빨라진다.


## Disable swapping

- 운영체제에서 java process 를 swap out 하는 걸 막는 것을 말한다.
- Swap 을 하는 이유가 `file system cache` 때문이라는데 정확히 이게 뭐지.
  - 가능한한 더 많은 `file system cache` 를 위해서 많은 메모리를 쓴다.
    - 여분의 남는 메모리가 있다면 `file system cache` 로 더 쓴다. 그런 거.
  - `File system cache` 는 최근에 엑세스한 파일에 대한 캐시를 말한다. 자주 접근한 파일 시스템이 있다면 그것에 대한 성능을 높이기 위해서. 메모리에 저장되는 영역임.
  - File system cache 와 page cache 의 차이는 뭔데?
    - 비슷한 점은 많은데 page cache 는 자주 접근한 `virtual memory` 영역을 캐싱해두는 것.
    - File system cache 도 메모리가 부족하면 swap 영역으로 내려가는데 이게 page cache 가 될수도 있겠네.  
    - 둘의 차이는 저장하는 데이터 타입이 다르다.
  - 그렇다면 OS 에서 관리하는 메모리의 종류는 뭔데?
    - Physical Memory
      - 어플리케이션이나 시스템 프로세스의 메모리 
    - Virtual Memory
      - HDD 와 SDD 와 같은 공간. 일시적인 공간임. Physical memory 를 다 사용하면 안쓰는 데이터는 Virtual memory 로 이동한다.
    - Swap Space
      - 이것도 virtual memory 와 같은 HDD 와 SDD 같은 공간, 일시적인 공간, physical memory 에서 안쓰는 영역을 옮긴 공간이다.
      - 다만 다른점이 있다.
        - 목적
          - Virtual memory 는 physical memory 를 효과적으로 쓰기 위해서 사용하는 공간. 프로그램을 쓰면 모든 프로그램 명령들을 메모리에 올리지 않도록. 필요하면 올리도록.
          - Swap space 는 physical memory 가 부족할 때 메모리에 있는 안쓰는 데이터를 옮기는 공간이다.
        - 사이즈
          - Virtual memory 가 더 크다. Swap 은 그에 반해 작다.
        - 성능
           - Virtual memory 가 더 느리다. 스왑은 HDD 나 SDD 에 적재되지만 그것보단 더 빠르다.
    - Page Cache
    - File system cache

- Swap 을 하면 Unused memory 가 스왑영역으로 내려감.
- JVM 의 Heap 영역과, 실행되고 있는 페이지가 디스크로 내려갈 수 있다.
- 이걸 피해야한다고 함. GC 가 ms 면 될걸 몇 분씩 걸릴 수도 있어서.
- Stability 를 위해서.
  - 응답이 느려지고 연결이 끊기는 것처럼 보일 수 있다.
- 스왑을 안하도록 하는 방법에는 3가지가 있다.
  - 선호되는 옵션으로는 완전히 스왑을 금지시키는 것.
  - K8s 에서는 swap 을 안쓰도록 설정됨.
  - Linux 환경에 배포하는 경우에는 스왑을 안쓰도록 설정을 해줘야함.
  - https://www.elastic.co/guide/en/elasticsearch/reference/7.10/setup-configuration-memory.html


## Give memory to the filesystem cache

- 메모리의 절반을 filesystem cache 로 줄 수도 있다.
- Linux 환경에선 `sysctl -w vm.max_map_count=262144` 을 주면 된다.
  - 뜻 자체는 Process 가 사용하는 메모리 맵의 최대 수를 증가시키는 것.
  - 이게 file 과 device 를 memory 에 매핑하는 map 인데 기본값이 너무 낮음.
  - 262144 는 쿠버네티스 환경에서 추천하는 값. 물론 최적의 값은 환경마다 다름.
  - 영구적으로 넣을려면 `/etc/sysctl.conf` 에 반영하면 된다.

- K8s 환경에선 yaml 파일로 작성하면 된다. Sysctl 을 이용해서.
- `file system cache` 는 `vm.max_map_count` 가 연관이 있음.
  - Elastic search 는 `mmapfs` 라는 디렉토리를 사용한다.
  - 이게 file 과 memory 를 매핑하는 map 인데 기본값이 너무 낮음.


## Use auto-generated ids

- ES 에서 문서를 색인할 때 명백한 id 를 주는 경우에는 check 한다. 이 문서가 존재하는지.
  - 이 과정 자체가 비싸니까 auto-generated ids 전략을 이용해서 이 검사를 skip 하도록 하자.
- 이 전략을 쓰면 같은 문서를 색인할 떄마다 매번 새로운 문서가 생겨나는건가?
  - 아님. 처음 만들 떄만 auto-generated id 를 쓰는거고, 그 이후 재색인을 할 때는 update 가 됨.
    - 만들 땐 hash function 을 쓴다.  
  - Auto-generated id 로 만들었다면, 이미 존재하는 문서가 있다는 뜻이고, 이를 통해서 매번 확인하는 과정을 거치지 않아도됨.
    - 이걸 어떻게 아는거지?
      - 컨텐츠가 똑같다면 해쉬값이 똑같을 것이니 알듯. 
  - 몰라도 업데이트는 할 수 있긴함. Unique 한 칼럼이 색인되어있다면 그걸로 문서를 찾아서 업데이트 하면 되니까.  


## Use faster hardware

- SSD > HDD
- Local storage > remote storage (such as NFS, SMB)
- Local Storage > AWS Elastic Block Storage


## Indexing buffer size

- heaving indexing 을 하고 있다면 `indices.memory.index_buffer_size` 를 튜닝해라. 이 값을
- Indexing buffer 가 뭔데?
  - 새로 색인된 문서가 저장될 때 쓰는 버퍼.
  - 이 버퍼 속에 문서가 있고 이게 segment 가 된다.
  - 모든 데이터 노드마다 설정해야한다.
  - 기본 값은 JVM Heap 의 10%.
  - 노드안에 있는 모든 샤드간에 공유된다.
  - 노드안에 샤드가 있다. 맞나? ㅇㅇ 하나의 노드안에 여러개의 샤드가 있을 수 있음. 


## Use cross-cluster replication to prevent searching from stealing resources from indexing

- 색인과 검색이 리소스를 놓고 경쟁하지 않도록 하는 것.
- 두 개의 클러스터를 둔다. 그리고 cross-cluster replication 을 한다.
- Datacenter outage 에도 불구하고 검색을 할 수 있다. 

## 그래서 Spider 에 적용할 수 있는 옵션들은?

- 최적의 bulk request
- randomized exponential backoff
- `index.translog.durability=async`
- `indexing.refresh_interval=60`
- `vm.max_map_count=262144`
