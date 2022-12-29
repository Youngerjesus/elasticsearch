# Elasticsearch Queue

https://opster.com/guides/elasticsearch/glossary/elasticsearch-queue/

***

## Overview

- queue 라는 용어는 tread pool 의 context 에서 사용된다.
- ES 의 각각의 node 는 다양한 thread pool 을 가진다. 다양한 타입의 요청을 다루기 위해서.
- Queue 는 node 의 사이즈별로 초기 제한이 있지만 바꿀 수 있다.

## What it is used for

- 큐는 pending 된 request 를 위해서 사용된다.
  - 이 큐는 thread pool 과 관련이 있다.
- 큐가 가득차면 이제 request 는 reject 되는 거임.
- Search 요청이 차기 시작한다면 search thread pool queue 에 담김.
- 모니터링 예시도 보여주면 좋을듯.

#### monitoring thread pool 

````http request
GET /_cat/thread_pool?v
````

#### Get details about each thread pool 

````http request
GET /_nodes/thread_pool
````

## Common problems
- 큐가 가득찼을 때 발생하는 Exception
  - `EsRejectedExecutionException`

## Thread pool 

https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-threadpool.html

- 여러가지 스레드 풀이 있다.
  - keep alive 특성은 thread pool 이 scaling 특성일 때 유지되는 것이고 스레드의 생존 시간을 말한다.
    - 이외에도 core, max 프로퍼티가 있다. 
- generic 
  - Thread pool type is `scaling`
  - For generic operations (for example, background node discovery).

- search
  - For count/search/suggest operations
  - Thread pool type is `fixed` with a size of int`((# of allocated processors * 3) / 2) + 1`, and queue_size of 1000.

- search_throttled
  - search request 가 ES 가 처리할 수 있는 rate 를 넘었을 때 작동하는 Thread pool
  - search request 가 throttled 될 때 작동하는 스레드풀.
    - search thread pool 이 다찼다고 자동으로 이 스레드풀로 fall back 되지 않는다.
    - dynamic throttling mechanism 에 따라서 throttling 되었다고 판단되면 이 스레드풀을 이용한다.

- search_coordination
  - For lightweight search-related coordination operations
  - Thread pool type is `fixed` with a size of a max of min(5, (# of allocated processors) / 2), and queue_size of 1000.

- get
  - For get operations
  - Thread pool type is fixed with a size of # of allocated processors, queue_size of 1000.

- analyze
  - For analyze requests.
  - Thread pool type is fixed with a size of 1, queue size of 16

- write
  - For single-document index/delete/update and bulk requests
  - Thread pool type is fixed with a size of # of allocated processors, queue_size of 10000. The maximum size for this pool is 1 + # of allocated processors.

- snapshot
  - For snapshot/restore operations
  - Thread pool type is scaling with a keep-alive of 5m and a max of min(5, (# of allocated processors) / 2).

- snapshot_meta
  - For snapshot repository metadata read operations.
  - Thread pool type is scaling with a keep-alive of 5m and a max of min(50, (# of allocated processors* 3)).

- warmer
  - For segment warm-up operations. Thread pool type is scaling with a keep-alive of 5m and a max of min(5, (# of allocated processors) / 2).

- refresh
  - For refresh operations. Thread pool type is scaling with a keep-alive of 5m and a max of min(10, (# of allocated processors) / 2).

- fetch_shard_started
  - For listing shard states. Thread pool type is scaling with keep-alive of 5m and a default maximum size of 2 * # of allocated processors.

- fetch_shard_store 
  - For listing shard stores. Thread pool type is scaling with keep-alive of 5m and a default maximum size of 2 * # of allocated processors.

- flush
  - For flush and translog fsync operations. Thread pool type is scaling with a keep-alive of 5m and a default maximum size of min(5, (# of allocated processors) / 2).

- force_merge
  - For force merge operations. Thread pool type is fixed with a size of max(1, (# of allocated processors) / 8) and an unbounded queue size.

- management
  - For cluster management. Thread pool type is scaling with a keep-alive of 5m and a default maximum size of 5.

- system_read
  - For read operations on system indices. Thread pool type is fixed with a default maximum size of min(5, (# of allocated processors) / 2).

- system_write
  - For write operations on system indices. Thread pool type is fixed with a default maximum size of min(5, (# of allocated processors) / 2).

- system_critical_read
  - For critical read operations on system indices. Thread pool type is fixed with a default maximum size of min(5, (# of allocated processors) / 2).

- system_critical_write
  - For critical write operations on system indices. Thread pool type is fixed with a default maximum size of min(5, (# of allocated processors) / 2).

- watcher
  - For critical write operations on system indices. Thread pool type is fixed with a default maximum size of min(5, (# of allocated processors) / 2).

