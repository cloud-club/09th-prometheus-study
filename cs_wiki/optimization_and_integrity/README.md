# 09th-prometheus-study
About computer arch & optimization

# 목차
- Memory Hierarchy
- Garbage Collection
- Review: Writing a Time Series Database from Scratch (개인적 리뷰)


## Memory Hierarchy
<img width="643" height="486" alt="image" src="https://github.com/user-attachments/assets/4f1f267a-a8e2-42c7-a14b-06aa45943246" />

### CPU 레지스터/L1 ~ L3 캐시
<img width="682" height="308" alt="image" src="https://github.com/user-attachments/assets/e50f7b70-741e-4038-a9c1-6d878a8b1416" />

- CPU 캐시에서 RAM에 저장하기전에 Gorilla 압축을 통해 데이터 압축이 일어납니다.
- L1 캐시가 CPU 코어와 가깝기에 압축이 일어나고, L2, L3 캐시에 압축된 것이 저장됩니다.
- 훨씬 많은 데이터를 압축해서 저장하기에 캐시 적중률을 높입니다.


### 주기억장치(RAM)
<img width="678" height="433" alt="image" src="https://github.com/user-attachments/assets/e4e033b5-bd2e-454a-890a-4ed8d64a129c" />

- Append-Only 구조입니다.
- 인덱스 캐시는 라벨셋 문자열({job="api", instance="10.0.0.1"})을 정수 해시로 바꿔서 해당 Active Chunk의 메모리 주소를 바로 리턴합니다.


### 보조기억장치(SSD/HDD)
- 데이터의 영구 저장을 담당하는 역할을 수행합니다.
- WAL: RAM의 휘발성을 보완하는 역할을 수행합니다.
  - 메모리에 쓰는 동시에, 디스크에도 순차적 쓰기가 발생합니다. (순차적 쓰기는 물리적으로 가장 빠른 쓰기 방식입니다.)
- Persistent Block: 특정 기준을 충족하면 데이터를 묶어서 디스크에 씁니다.
  - 이때 데이터를 한번 정렬하고 압축하여, 나중에 읽기 시 디스크 I/O를 최소화하도록 합니다.

### 순차적 쓰기
<img width="696" height="320" alt="image" src="https://github.com/user-attachments/assets/8835b8e1-5deb-4102-895f-7d6f713dbfd1" />

- HDD의 경우 헤드가 매번 새 위치를 찾아 이동하는 Seek time과 플래터가 해당 섹터까지 회전해오는 Rotational latency 가 중첩되기 때문입니다.
<img width="665" height="337" alt="image" src="https://github.com/user-attachments/assets/e3ec9aa3-626c-4fe4-a71f-375e8edcb612" />

- SSD의 경우 NAND Block 구조를 가지는데, 덮어쓰시가 불가능합니다.
- Page단위로 수정을 할 수 없고, Block단위로 수정을 해야합니다.


## Garbage Collection
### Series Garbage Collection
- 프로메테우스는 stripeSeries라는 구조체로 메모리에 띄워진 인덱스 참조를 관리하는데, 데이터가 없어도 라벨은 남아있는 상황이 발생합니다. (Series Churn 상태)
- Compaction(기본적으로 2시간)이 일어날때 삭제가 발생하게 됩니다.

### Block Garabe Collection
1. Retention 삭제 (물리적)
  - Retention이 만료되면 os.RemoveAll()로 디렉토리를 삭제합니다.
  - 이때 block자체는 immutable이므로 lock경합 또한 피할 수 있습니다.
<img width="691" height="455" alt="image" src="https://github.com/user-attachments/assets/06269418-d547-4fc8-8749-197fd4ae10dc" />


2. Tombstone 삭제(논리적)
   - tombstone을 통해 삭제 대상이라고 마킹만 하고, compaction 단계시 마킹된 것만 제외하고 새로운 block을 구성하게 됩니다.
  


## Review: Writing a Time Series Database from Scratch
참고 링크가 있어서 읽다 보니 재밌어서 개인적으로 요약 해봤습니다.
- https://web.archive.org/web/20220205173824/https://fabxc.org/tsdb/

분산시스템 환경에서 모니터링 시스템이 겪는 한계를 어떻게 극복했는지 설계 과정에 관한 글입니다.

### 1. 기존 V2의 한계
프로메테우스는 원래부터 쿠버네티스처럼 변화무쌍한 환경을 모니터링하기 위해 태어났습니다.
기존 역동적인 것에 대한 처리 능력에, 일시적인(Transient) 서비스를 처리하는 능력을 업그레이드 하겠다고 하고 있습니다.
즉, 기존 방식(V2)은 한계가 왔으니, 변화하는 환경에서도 버틸 수 있는 완전히 새로운 스토리지 엔진(V3)을 만들겠다고 시작을 하고 있습니다.


- 읽기와 쓰기의 딜레마: 시계열 데이터는 여러 서버와 지표에서 동시다발적으로 들어오기 때문에 row단위로 일괄 작성(Batch)하는 것이 효율적입니다. 하지만 사용자가 데이터를 조회할 때는 특정 지표의 과거 데이터를 col 단위로 쭉 읽는 것이 효율적입니다.

- 시리즈 이탈 (Series Churn): k8s 환경에서는 컨테이너(Pod)가 계속 교체됩니다. 이때마다 기존 지표(Series)는 끊기고 완전히 새로운 지표들이 폭발적으로 생성되는데, 이를 시리즈 이탈이라고 합니다.

- V2의 실패: 기존 버전은 '하나의 지표(Series)당 하나의 파일'을 만드는 구조였습니다. 시리즈 이탈이 발생하자 파일 개수가 수백만 개로 폭증했고, 이로 인해 디스크 I/O가 마비되고, 쿼리 한 번에 메모리 부족(OOM)으로 서버가 죽어버리며, 오래된 데이터를 지우는 데만 몇 시간이 걸리는 참사가 발생했습니다.


### V3 아키텍처의 패러다임 전환
- Block 분할: 전체 데이터를 2시간과 같은 특정 시간 단위의 블록으로 쪼갰습니다. 각 블록은 독립적인 인덱스와 청크를 가진 완벽한 미니 데이터베이스로 동작합니다.

- 메모리 매핑 도입: 수백만 개의 쪼개진 파일을 소수의 큰 파일로 합치고, OS의 mmap 기능을 도입했습니다. Prometheus가 직접 메모리를 관리하는 대신, OS의 페이지 캐시에 메모리 관리를 전적으로 위임하여 메모리 부족(OOM) 현상을 예방하였습니다.

- 컴팩션과 초고속 삭제: 흩어진 작은 블록들은 시간이 지나면 큰 블록으로 병합(Compaction)되어 쿼리 효율을 증가시킵니다. 오래된 데이터를 지울 때 full scan을 할 필요 없이 해당 시간대의 블록 디렉터리 하나만 통째로 지우면 됩니다. (시간복잡도가 O(1)을 나타냅니다.)

### 인덱스 최적화: 검색 엔진의 원리 도입
- 역색인 방식을 활용 하였습니다.
- 교집합 탐색시 전체를 탐색하는 O(N^2)에서 교집합 부분만 찾도록 정렬을 수행해 O(N)의 복잡도를 나타내고 있습니다.


### 벤치마킹 결과
- 리소스: CPU와 메모리 사용량이 기존 대비 3~4배 감소했습니다.

- 디스크 I/O 최적화: 디스크 쓰기 작업이 97~99% 감소하여, 잦은 쓰기로 인해 SSD 수명이 깎이던 문제를 해결하였습니다.

- 안정성: 아무리 지표가 새로 생성되고 사라져도(Series Churn), 쿼리 응답 속도가 크게 느려지지 않고 대부분 유지됩니다.
