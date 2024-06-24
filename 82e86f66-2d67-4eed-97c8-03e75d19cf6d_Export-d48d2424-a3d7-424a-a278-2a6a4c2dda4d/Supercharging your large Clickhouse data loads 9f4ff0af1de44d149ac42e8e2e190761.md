# Supercharging your large Clickhouse data loads

### Link

- [Supercharging your large ClickHouse data loads - Performance and resource usage factors](https://clickhouse.com/blog/supercharge-your-clickhouse-data-loads-part1)
- [Supercharging your large ClickHouse data loads - Tuning a large data load for speed](https://clickhouse.com/blog/supercharge-your-clickhouse-data-loads-part2)
- [Supercharging your large ClickHouse data loads - Making a large data load resilient](https://clickhouse.com/blog/supercharge-your-clickhouse-data-loads-part3)
- [Python Integration with ClickHouse Connect | ClickHouse Docs](https://clickhouse.com/docs/en/integrations/python)
- https://github.com/long2ice/asynch
- https://github.com/ClickHouse/clickhouse-connect
- [Python Integration with ClickHouse Connect | ClickHouse Docs](https://clickhouse.com/docs/en/integrations/python#multithreaded-multiprocess-and-asyncevent-driven-use-cases)

### Table of Contents

# Performance and resource usage factors

ClickHouse는 빠르고 자원 효율적이도록 설계되었다. 사용자가 허용하면 ClickHouse는 실행되는 하드웨어를 이론적 한계까지 활용하고 데이터를 엄청나게 빠르게 로드할 수 있다. 또는 대용량 데이터 로드의 리소스 사용량을 줄일 수도 있다. 
첫 번째 파트에서는 ClickHouse의 기본 데이터 삽입 매커니즘과 리소스 사용량 및 및 성능 제어를 위한 세 가지 주요 요소를 설명하여 기초를 다진다.
두 번째 파트에서는 레이스 트랙에서 대용량 데이터 로드의 속도를 최대로 조정하는 방법을 살펴본다.
마지막 파트에서는 네트워크 중단과 같은 일시적인 문제에 대해 대용량 데이터 로드를 견고하고 탄력적으로 만드는 방법에 대해 설명한다.

## Data insert mechanics

다음 다이어그램은 MergeTree 엔진 제품군의 ClickHouse 테이블에 데이터를 삽입하는 일반적인 매커니즘을 스케치한 것이다. 서버는 일부 데이터 부분(예: 삽입 쿼리에서)을 수신하고, ① 수신된 데이터에서 (적어도) 하나의 인메모리 삽입 블록(파티셔닝 키당)을 형성한ㄴ다. 블록의 데이터가 정렬되고 테이블 엔진별 최적화가 적용된다. 그런 다음 데이터를 압축하고 ② 새로운 데이터 부분의 형태로 데이터베이스 스토리지에 기록한다.

![Untitled](Supercharging%20your%20large%20Clickhouse%20data%20loads%209f4ff0af1de44d149ac42e8e2e190761/Untitled.png)

서버가 아닌 클라이언트가 삽입 블록을 형성하는 경우도 있다.

ClickHouse의 데이터 삽입 매커니즘의 성능과 리소스 사용량에 영향을 미치는 주요 요인은 크게 세 가지이다.

- 삽입 블록 크기
- 삽입 병렬 처리
- 하드웨어 크기

## Insert block size

### Impact on performance and resource usage

![Untitled](Supercharging%20your%20large%20Clickhouse%20data%20loads%209f4ff0af1de44d149ac42e8e2e190761/Untitled%201.png)

삽입 블록 크기(Insert block size)는 ClickHouse 서버의 디스크 파일 I/O 사용량과 메모리 사용량에 모두 영향을 준다. 삽입 블록이 클수록 더 많은 메모리를 사용하지만 초기 파트(Initial block)를 더 많이 생성하고 더 적게 생성한다. 많은 양의 데이터를 로드하기 위해 ClickHouse가 생성해야 하는 파트가 적을수록 필요한 디스크 파일 I/O 및 자동 백그라운드 병합이 줄어든다.

삽입 블록 크기를 구성하는 방법은 데이터 수집 방식에 따라 달라진다. ClickHouse 서버가 직접 가져오거나 아니면 외부 클라이언트가 삽입하거나

## Configuration when data is pulled by ClickHouse servers

통합 테이블 엔진 또는 테이블 함수와 함께 `INSERT INTO SELECT` 쿼리를 사용하는 경우, ClickHouse 서버 자체에서 데이터를 가져온다. 

![Untitled](Supercharging%20your%20large%20Clickhouse%20data%20loads%209f4ff0af1de44d149ac42e8e2e190761/Untitled%202.png)

데이터가 완전히 로드될 때까지 서버는 루프를 다음 루프를 실행한다.

> ① 데이터의 다음 부분을 가져와 구문 분석하고 이로부터 인메모리 블록(파티션닝 키당 하나)을 형성한다.

② 이 블록을 스토리지의 새 부분에 기록한다.

다시 ①로 이동한다.
> 

①에서 부분 크기는 삽입 블록 크기에 따라 달라지며, 두 가지 설정으로 제어할 수 있다.

- `min_insert_block_size_rows` (default : 1048545 million rows)
- `min_insert_block_size_bytes` (default: 256 MB)

삽입 블록에 지정된 수의 행이 수집되거나 구성된 데이터 양에 도달하면 (어느 쪽이 먼저 발생하든) 블록이 새 부분에 기록되는 트리거가 실행된다. 삽입 루프는 ① 단계에서 계속된다.

`min_insert_block_size_bytes` 값은 압축되지 않은 메모리 내 블록 크기(압축된 온 디스크 파트 크기가 아닌)를 나타낸다. 또한, ClickHouse는 데이터를 행(Row) 블록 단위로 스트리밍하고 처리하므로 생성된 블록과 파트가 구성된 행 또는 바이트 수를 정확하게 포함하는 경우는 드물다. 따라서 이러한 설정은 최소 임계값을 지정한다.

## Configuration when data is pushed by clients

클라이언트 또는 클라이언트 라이브러리에서 사용하는 데이터 전송 형식 및 인터페이스에 따라 인메모리 데이터 블록은 ClickHouse 서버 또는 클라이언트 자체에 의해 형성된다. 이에 따라 블록 크기를 제어할 수 있는 위치와 방법이 결정된다. 이는 또한 동기식 또는 비동기식 삽입이 사용되는지 여부에 따라 달라진다.

## Synchronous inserts

**When the server forms the blocks**

클라이언트가 일부 데이터를 가져와 동기식 삽입 쿼리와 함께 Native 형식이 아닌 형식으로 전송하면(예: JDBC 드라이버가 삽입에 RowBinary를 사용하는 경우), 서버는 삽입 쿼리의 데이터를 구문 분석하여 (최소한) 하나의 인메모리 블록(파티셔닝 키당)을 형성하고, 이 블록은 파트 형태로 스토리지에 기록된다.

![Untitled](Supercharging%20your%20large%20Clickhouse%20data%20loads%209f4ff0af1de44d149ac42e8e2e190761/Untitled%203.png)

삽입 쿼리의 행 개수는 자동적으로 블록 크기를 조절한다. 하지만, 블록(파티셔닝 키당)의 최대 크기는 `max_insert_block_size` 행 설정으로 지정할 수 있다. 삽입 쿼리의 데이터의 구성된 단일 블록에 `max_insert_block_size` 행 (기본값: 100만 행)이 초과되면 서버는 블록과 파트를 각각 추가로 생성한다.

생성되고 병합되는 파트의 수를 최소화하려면 일반적으로 클라이언트 측에서 데이터를 버퍼링하고 데이터를 일괄적으로 삽입하여 작은 삽입을 많이 보내는 대신 적은 수의 큰 삽입을 보내는 것이 좋다.

**When the client forms the blocks**

ClickHouse의 command-line 클라이언트 (clickhouse-client)와 Go, Python, C++ 클라이언트 등 일부 프로그래밍 언어별 라이브러리는 클라이언트 측에서 삽입 블록을 형성하고 Native 형식으로 ClickHouse 서버로 전송하며, 이 서버는 블록을 스토리지에 직접 기록한다.

예를 들어, 다음은 일부 데이터를 삽입하는 데 ClickHouse 명령줄 클라이언트를 사용하는 경우를 보여준다.

```
./clickhouse client --host ...  --password …  \
 --input_format_parallel_parsing 0 \
 --max_insert_block_size 2000000 \
 --query "INSERT INTO t FORMAT CSV" < data.csv
```

그런 다음 클라이언트가 자체적으로 데이터를 구문 분석하고 데이터에서 인메모리 블록을 생성한 후 네이티브 ClickHouse 프로토콜을 통해 네이티브 형식으로 서버로 전송하면 서버가 블록을 스토리지에 일부로 기록한다.

![Untitled](Supercharging%20your%20large%20Clickhouse%20data%20loads%209f4ff0af1de44d149ac42e8e2e190761/Untitled%204.png)

클라이언트 측 인메모리 블록 크기(행 개수)는 `max_insert_block_size`(기본값: 1048545 million, 1조 485억 4500만) 옵션으로 제어할 수 있다.

위의 명령줄 호출 예제에서는 병렬 구문 분석을 비활성화했다. 그렇지 않으면 clickhouse-client는 `max_insert_block_size` 설정을 무시하고 병렬 구문 분석의 결과인 여러 블록을 하나의 삽입 블록으로 나눈다.

클라이언트 측 `max_insert_block_size` 설정은 clickhouse-client에만 적용된다. 유사한 설정은 클라이언트 라이브러리의 문서와 설정을 확인해야 한다.

### **Asynchronous data inserts**

클라이언트 측 일괄 처리(client-size batching)와 함께 비동기 삽입을 사용할 수도 있다. 비동기 데이터 삽입을 사용하면 사용되는 클라이언트, 형식 및 프로토콜에 관계없이 항상 서버에서 블록을 형성한다.

![Untitled](Supercharging%20your%20large%20Clickhouse%20data%20loads%209f4ff0af1de44d149ac42e8e2e190761/Untitled%205.png)

비동기 삽입을 사용하면 수신된 삽입 쿼리의 데이터를 먼저 인메모리 버퍼에 넣고(①,②,③ 참조), ④ 구성 설정에 따라 버퍼가 플러시되면(예를 들어, 특정 양의 데이터가 수집되면), 서버는 버퍼의 데이터를 파싱하여 인메모리 블록을 형성하고, ⑤ 이를 파트 형태로 스토리지에 기록한다. 단일 블록에 `max_insert_block_size`보다 많은 행이 포함되면 서버는 각각 추가 블록과 파트를 생성한다.

### **More parts = more background part merges**

구성된 삽입 블록 크기가 작을수록 대규모 데이터 로드를 위해 더 많은 초기 파트가 생성되고 데이터 수집과 동시에 더 많은 백그라운드 파트 병합이 실행된다. 이로 인해 리소스 경합(CPU 및 메모리)이 발생할 수 있으며 수집이 완료된 후 정상적인 수의 파트에 도달하는 데 추가 시간이 필요할 수 있다.

ClickHouse는 압축 크기가 최대 150GB에 도달할 때까지 계속해서 파트를 더 큰 파트로 병합한다. 이 다이어그램은 ClickHouse 서버가 파트를 병합하는 방법을 보여준다.

![Untitled](Supercharging%20your%20large%20Clickhouse%20data%20loads%209f4ff0af1de44d149ac42e8e2e190761/Untitled%206.png)

단일 ClickHouse 서버는 여러 백그라운드 병합 스레드를 사용하여 동시 파트 병합을 실행한다. 각 스레드는 아래 루프를 실행한다.

```
① 다음에 병합할 부분을 결정하고 해당 부분을 블록으로 메모리에 로드한다.

② 메모리에 로드된 블록을 더 큰 블록으로 병합한다.

③ 병합된 블록을 디스크의 새 파트에 기록한다.

다시 ①로 이동
```

ClickHouse는 병합할 전체 파트를 한 번에 메모리에 로드하지 않는다. 여러 가지 요인에 따라 (병합 속도를 희생하는 대신) 메모리 소비를 줄이기 위해 소위 수직 병합은 한 번에 로드 하지 않고 블록 단위로 파트를 로드하고 병합한다. 또한 CPU 코어 수와 RAM 크기를 늘리면 백그라운드 병합 처리량이 증가한다.

더 큰 파트로 병합된 파트는 비활성으로 표시되고 구성 가능한 시간 후에 최종적으로 삭제된다. 시간이 지나면 병합된 파트의 트리가 만들어진다. 최종적으로 병합 트리 테이블이라는 이름이 붙는다.

![Untitled](Supercharging%20your%20large%20Clickhouse%20data%20loads%209f4ff0af1de44d149ac42e8e2e190761/Untitled%207.png)

각 파트는 해당 파트로 이어지는 병합 횟수를 나타내는 특정 레벨에 속한다.

## Insert parallelism

### Impact on performance and resource usage

![Untitled](Supercharging%20your%20large%20Clickhouse%20data%20loads%209f4ff0af1de44d149ac42e8e2e190761/Untitled%208.png)

ClickHouse 서버는 데이터를 병렬로 처리하고 삽입할 수 있다. 삽입 병렬 처리 수준은 ClickHouse 서버의 수집 처리량과 메모리 사용량에 영향을 준다. 데이터를 병렬로 로드하고 처리하면 더 많은 메인 메모리가 필요하지만 데이터가 더 빨리 처리되므로 수집 처리량이 증가한다.

삽입 병렬 처리 수준을 구성하는 방법은 다시 데이터 수집 방식에 따라 달라진다.

### Configuration when data is pulled by ClickHouse servers

s3, URL, hdfs와 같은 일부 통합 테이블 함수는 glob 패턴을 통해 로드할 파일 이름 집합을 지정할 수 있다. glob 패턴이 여러 기존 파일과 일치하는 경우, ClickHouse는 서버별로 병렬로 실행되는 삽입 스레드를 활용하여 이러한 파일 전체 및 파일 내에서 읽기를 병렬화하고 데이터를 테이블에 병렬로 삽입할 수 있다.

![Untitled](Supercharging%20your%20large%20Clickhouse%20data%20loads%209f4ff0af1de44d149ac42e8e2e190761/Untitled%209.png)

모든 파일의 데이터가 모두 처리될 때까지 각 삽입 스레드는 아래 루프를 실행한다.

```
① 처리되지 않은 파일 데이터의 다음 부분을 가져와서 인메모리 데이터 블록을 생성한다.

② 이 블록을 스토리지의 새 파트에 기록한다.

다시 ①로 이동
```

이러한 병렬 삽입 스레드의 수는 `max_insert_threads` 설정으로 구성할 수 있다. 기본값은 OSS의 경우 1, ClickHouse Cloud의 경우 4이다.

파일 수가 많으면 여러 개의 삽입 스레드를 통한 병렬 처리가 잘 작동한다. 사용 가능한 CPU 코어와 네트워크 대역폭(병렬 파일 다운로드용)을 모두 완전히 포화시킬 수 있다. 테이블에 몇 개의 대용량 파일만 로드되는 시나리오에서 ClickHouse는 자동으로 높은 수준의 데이터 처리 병렬성을 설정하고 대용량 파일 내에서 더 많은 범위를 병렬로 읽기(다운로드)하기 위해 삽입 스레드당 추가 읽기 스레드를 생성하여 네트워크 대역폭 사용을 최적화한다. 고급 읽기의 경우, `max_download_threads`와 `max_download_buffer_size`에 영향을 받는다. 이 매커니즘은 현재 s3, URL 테이블 함수에 대해 구현되어 있다. 또한 병렬 읽기에 너무 작은 파일의 경우 처리량을 늘리기 위해 ClickHouse는 이러한 파일을 비동기식으로 미리 읽어 데이터를 자동으로 프리페치(prefetch)한다.

### Configuration when data is pushed by clients

ClickHouse 서버는 삽입 쿼리를 동시에 수신하고 실행할 수 있다. 클라이언트는 병렬 클라이언트 측 스레드를 실행하여 이를 활용할 수 있다.

각 스레드는 아래 루프를 실행한다.

```
① 다음 데이터 부분을 가져와서 INSERT를 생성한다.

② INSERT를 ClickHouse로 전송하고 INSERT가 성공했다는 확인을 기다린다.

다시 ①로 이동
```

## Hardware size

**Impace on performance**

사용 가능한 CPU 코어의 수와 RAM의 크기는 

- 초기 부품의 크기
- 가능한 삽입 병렬 처리 수준
- 백그라운드 파트 병합 처리량

영향을 주고, 결과적으로 전체 수집 처리량에 영향을 준다.