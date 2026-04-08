# Week5 - Persistent Block: Configuration Tuning & 성능 측정 분석

> **Reference**: Ganesh Vernekar (Prometheus maintainer), *"TSDB format" — Persistent Block and its Index* (Part 4 of the TSDB blog series)
>
> 본 문서에서 인용한 구문은 모두 위 블로그의 본문이며, `prometheus/tsdb/` 소스 코드 분석은 이 글을 근거로 진행되었습니다.

Persistent Block의 구조를 이해한 뒤, `prometheus/tsdb/` 내부에서 **튜닝 가능한 configuration**과 그 효과를 측정할 수 있는 **기존 벤치마크 테스트**를 매핑했습니다.

## 1. 튜닝 가능한 Configuration

> *"A block consists of 4 parts: meta.json (file): the metadata of the block. chunks (directory): contains the raw chunks without any metadata about the chunks. index (file): the index of this block. tombstones (file): deletion markers to exclude samples when querying the block."*
>
> — Ganesh Vernekar, "TSDB format" blog Part 4

블로그에서 설명하는 4가지 구성요소(`meta.json`, `chunks/`, `index`, `tombstones`)와 그 생성 과정에 영향을 주는 항목들을 `tsdb/db.go`의 `Options` 구조체 (`DefaultOptions()` 참고)에서 추려냈습니다:

| Configuration | 기본값 | 의미 | 영향 영역 |
|---------------|--------|------|-----------|
| `WALSegmentSize` | `128 MiB` (`wlog.DefaultSegmentSize`) | WAL 세그먼트 파일 크기 | WAL write throughput, fsync 빈도, replay 시간 |
| `WALCompression` | `None` | WAL 레코드 압축 (Snappy/Zstd) | 디스크 사용량, CPU, write latency |
| `MaxBlockChunkSegmentSize` | `512 MiB` (`chunks.DefaultChunkSegmentSize`) | Persistent block의 chunks 파일 크기 상한 | 블록 I/O 패턴, mmap 효율 |
| `RetentionDuration` | `15d` | 데이터 보존 기간 | 디스크 사용량, 블록 개수 |
| `MaxBytes` | `0` (off) | 보존 데이터 최대 크기 | 디스크 압박 시 truncation 빈도 |
| `MinBlockDuration` | 2h (`DefaultBlockDuration`) | Head → Persistent block으로 cut되는 최소 범위 | 블록 개수, compaction 빈도 |
| `MaxBlockDuration` | `MinBlockDuration` | Compacted block 최대 범위 | level별 블록 크기, 쿼리 성능 |
| `StripeSize` | `DefaultStripeSize` (16384) | Series hash map 스트라이프 수 | Append 동시성 (lock contention), 메모리 |
| `HeadChunksWriteBufferSize` | `chunks.DefaultWriteBufferSize` (4 MiB) | Head chunks mapper 쓰기 버퍼 | mmap chunk flush 빈도, write latency |
| `HeadChunksWriteQueueSize` | `chunks.DefaultWriteQueueSize` | 비동기 chunk write 큐 깊이 | 쓰기 스파이크 흡수 |
| `SamplesPerChunk` | `DefaultSamplesPerChunk` (120) | chunk당 목표 샘플 수 | chunk 크기, 압축률, 쿼리 시 디코딩 비용 |
| `WALReplayConcurrency` | `GOMAXPROCS` | WAL replay 시 사용 CPU 수 | 재시작 시간 |
| `OutOfOrderTimeWindow` | `0` | OOO 샘플 허용 윈도우 | OOO write 처리, WBL 부하 |
| `OutOfOrderCapMax` | `DefaultOutOfOrderCapMax` (32) | OOO chunk 최대 샘플 수 | OOO 메모리, mmap 빈도 |
| `EnableOverlappingCompaction` | `true` | 겹치는 블록의 vertical compaction 허용 | OOO 블록 쿼리 효율 |
| `IsolationDisabled` | `false` | append/read isolation 끄기 | append throughput |
| `EnableMemorySnapshotOnShutdown` | `false` | 종료 시 head 메모리 스냅샷 | 재시작 속도 |
| `UseUncachedIO` | `false` | 페이지 캐시 우회 (Direct I/O) | 큰 I/O 시 cache pollution 방지 |
| `EnableSharding` | `false` | 쿼리 sharding 활성화 | 분산 쿼리 성능 |
| `BlockReloadInterval` | `1m` | 블록 reload 주기 | 새 블록 가시화 latency |

## 2. 영역별 매핑: Configuration ↔ 기존 벤치마크 테스트

### 2.1 WAL (Write-Ahead Log)

블로그의 **WAL 섹션**과 직접 연결됩니다. WAL 세그먼트 크기와 압축이 핵심 튜닝 포인트입니다.

> *"When the Head block fills up with data ranging chunkRange*3/2 in time, we take the first chunkRange of data and convert into a persistent block. ... Here we call that chunkRange as blockRange in the context of blocks, and the first block cut from the Head spans 2h by default in Prometheus."*
>
> — Ganesh Vernekar, "TSDB format" blog Part 4 (Persistent Block)

| 벤치마크 | 위치 | 측정 가능 항목 |
|---------|------|----------------|
| `BenchmarkWAL_Log` | [wlog/wlog_test.go:544](prometheus/tsdb/wlog/wlog_test.go#L544) | 단일 레코드 write throughput (`compress=None/Snappy/Zstd` 케이스 내장) |
| `BenchmarkWAL_LogBatched` | [wlog/wlog_test.go:514](prometheus/tsdb/wlog/wlog_test.go#L514) | 1000개 batch write throughput, page flush 효율 |
| `BenchmarkWAL_HistogramEncoding` | [record/record_test.go:622](prometheus/tsdb/record/record_test.go#L622) | 히스토그램 record 인코딩 비용 |
| `BenchmarkLoadWLs` | [head_test.go:194](prometheus/tsdb/head_test.go#L194) | WAL replay 시간 — `batches × seriesPerBatch × samplesPerSeries`, OOO 비율, exemplar 비율을 파라미터화 |
| `BenchmarkLoadRealWLs` | [head_test.go:443](prometheus/tsdb/head_test.go#L443) | 실제 WAL 디렉터리 기반 replay |

**튜닝 실험 아이디어**
- `WALCompression`을 `None / Snappy / Zstd`로 바꿔가며 `BenchmarkWAL_Log`의 `ns/op`, `MB/s` 비교 → 현재 케이스가 이미 내장
- `New(...)`에 전달하는 segment size를 `NewSize(...)`로 바꿔 `pageSize` 배수 조건 하에서 다양한 크기로 실험 (64MB / 128MB / 256MB)
- `BenchmarkLoadWLs`에서 `WALReplayConcurrency`를 변경 (현재는 default GOMAXPROCS 사용)

### 2.2 Head Block (인메모리 → Persistent로 cut)

| 벤치마크 | 위치 | 측정 가능 항목 |
|---------|------|----------------|
| `BenchmarkHeadAppender_AppendCommit` | [head_bench_test.go:214](prometheus/tsdb/head_bench_test.go#L214) | append + commit throughput. `series=10/100`, `samples_per_append=1/5/100`, sample type별 (float, histogram, exemplar) |
| `BenchmarkHeadStripeSeriesCreate` | [head_bench_test.go:255](prometheus/tsdb/head_bench_test.go#L255) | series 생성 비용 (단일 스레드) |
| `BenchmarkHeadStripeSeriesCreateParallel` | [head_bench_test.go:270](prometheus/tsdb/head_bench_test.go#L270) | series 생성의 lock contention 측정 — `StripeSize` 튜닝 효과 직접 측정 가능 |
| `BenchmarkCreateSeries` | [head_test.go:103](prometheus/tsdb/head_test.go#L103) | 단순 series 생성 |
| `BenchmarkHead_Truncate` | [head_test.go:1311](prometheus/tsdb/head_test.go#L1311) | head truncation 비용 (block cut 직후) |
| `BenchmarkCuttingHeadHistogramChunks` | [head_test.go:6227](prometheus/tsdb/head_test.go#L6227) | 히스토그램 chunk cut 비용 |

**튜닝 실험 아이디어**
- `StripeSize`를 1024 / 4096 / 16384 / 65536으로 바꿔 `BenchmarkHeadStripeSeriesCreateParallel`로 contention 변화 측정
- `SamplesPerChunk`를 60 / 120 / 240으로 바꿔 `BenchmarkHeadAppender_AppendCommit` 비교 → chunk cut 빈도 변화
- `HeadChunksWriteBufferSize`/`HeadChunksWriteQueueSize`를 줄여서 mmap flush 압박 시뮬레이션

### 2.3 Persistent Block 생성 / Compaction

블로그에서 다룬 **persistent block의 chunks/index/tombstones 파일 생성 비용**을 직접 측정.

> *"A block on disk is a collection of chunks for a fixed time range consisting of its own index. It is a directory with multiple files inside it. Every block has a unique ID, which is a Universally Unique Lexicographically Sortable Identifier (ULID). ... A block has an interesting property that the samples in it are immutable. If you want to add more samples, or delete some, or update some, you have to rewrite the entire block with the required modifications and the new block has a new ID."*
>
> *"When the blocks get old, multiple blocks are compacted (or merged) to form a new bigger block while the old ones are deleted. So we have 2 ways of creating a block, from the Head and from existing blocks."*
>
> — Ganesh Vernekar, "TSDB format" blog Part 4

> *"The chunks directory contains a sequence of numbered files similar to the WAL/checkpoint/head chunks. Each file is capped at 512MiB."*
>
> → 이 512MiB가 바로 `MaxBlockChunkSegmentSize`의 기본값이며, [chunks/chunks.go](prometheus/tsdb/chunks/chunks.go)의 `DefaultChunkSegmentSize` 상수로 정의되어 있습니다.

| 벤치마크 | 위치 | 측정 가능 항목 |
|---------|------|----------------|
| `BenchmarkCompaction` | [compact_test.go:1175](prometheus/tsdb/compact_test.go#L1175) | normal/vertical compaction, 다양한 시간 범위, 10000 series |
| `BenchmarkCompactionFromHead` | [compact_test.go:1245](prometheus/tsdb/compact_test.go#L1245) | Head → Block 변환 비용. `labelnames × labelvalues = 100000` 조합 |
| `BenchmarkCompactionFromOOOHead` | [compact_test.go:1275](prometheus/tsdb/compact_test.go#L1275) | OOO Head → Block 변환 비용 |
| `BenchmarkOpenBlock` | [block_test.go:82](prometheus/tsdb/block_test.go#L82) | 디스크 블록 open 시간 (index 로드 포함) |

**튜닝 실험 아이디어**
- `MaxBlockChunkSegmentSize`를 128MB / 512MB / 1GB로 바꿔 `BenchmarkCompactionFromHead`의 chunks 파일 분할 횟수 vs 처리 속도 측정
- `MinBlockDuration`을 1h / 2h / 4h로 바꿔 cut 빈도와 한 번에 처리하는 데이터양 trade-off 측정
- vertical compaction (`EnableOverlappingCompaction`) on/off 비교

### 2.4 Index (Symbol Table / Series / Postings)

블로그의 **index 섹션 (Symbol Table → Series → Postings Offset Table)** 과 직접 연결됩니다.

> *"Index contains all that you need to query the data of this block. It does not share any data with any other blocks or external entity which makes it possible to read/query the block without any dependencies. The index is an 'inverted index' which is also widely used in indexing documents."*

> *"This section [Symbol Table] holds a sorted list of deduplicated strings which are found in label pairs of all the series in this block. ... The other sections in the index can refer to this symbol table for any strings and hence significantly reduce the index size."*

> *"Each series entry is 16 byte aligned, which means the byte offset at which the series starts is divisible by 16. Hence we set the ID of the series to be offset/16 ... So whenever we say posting in the context of Prometheus TSDB, it refers to a series ID."*

> *"This postings list and postings offset table form the inverted index. For indexing documents using an inverted index, for every word, we store a list of documents that it appears in. Similarly here, for every label-value pair, we store the list of series that it appears in."*
>
> — Ganesh Vernekar, "TSDB format" blog Part 4

| 벤치마크 | 위치 | 측정 가능 항목 |
|---------|------|----------------|
| `BenchmarkPostings_Stats` | [index/postings_test.go:906](prometheus/tsdb/index/postings_test.go#L906) | postings 통계 수집 |
| `BenchmarkMemPostings_ensureOrder` | [index/postings_test.go:72](prometheus/tsdb/index/postings_test.go#L72) | postings 정렬 비용 |
| `BenchmarkMemPostings_Delete` | [index/postings_test.go:1003](prometheus/tsdb/index/postings_test.go#L1003) | postings 삭제 비용 (tombstone과 연관) |
| `BenchmarkMemPostings_PostingsForLabelMatching` | [index/postings_test.go:1421](prometheus/tsdb/index/postings_test.go#L1421) | label matcher 기반 postings 조회 |
| `BenchmarkIntersect` | [index/postings_test.go:315](prometheus/tsdb/index/postings_test.go#L315) | 여러 postings list 교집합 (PromQL `{a="x", b="y"}` 쿼리 패턴) |
| `BenchmarkMerge` | [index/postings_test.go:373](prometheus/tsdb/index/postings_test.go#L373) | postings list 합집합 |
| `BenchmarkListPostings` | [index/postings_test.go:1382](prometheus/tsdb/index/postings_test.go#L1382) | 단일 list 순회 |
| `BenchmarkReader_ShardedPostings` | [index/index_test.go:507](prometheus/tsdb/index/index_test.go#L507) | sharded postings 읽기 — `EnableSharding` 효과 |
| `BenchmarkPostingStatsMaxHep` | [index/postingsstats_test.go:58](prometheus/tsdb/index/postingsstats_test.go#L58) | top-N postings 추출 |
| `BenchmarkLabelValuesWithMatchers` | [block_test.go:426](prometheus/tsdb/block_test.go#L426) | matcher 기반 label values 조회 |
| `BenchmarkLabelValues_SlowPath` | [label_values_bench_test.go:31](prometheus/tsdb/label_values_bench_test.go#L31) | slow path label values |
| `BenchmarkHeadLabelValuesWithMatchers` | [head_test.go:3620](prometheus/tsdb/head_test.go#L3620) | head 기준 label values |

**튜닝 실험 아이디어**
- `PostingsDecoderFactory`를 커스텀 구현으로 교체하여 디코딩 전략별 성능 비교
- `EnableSharding`을 켠/끈 상태로 `BenchmarkReader_ShardedPostings` 비교

### 2.5 Querier (블록 통합 쿼리)

| 벤치마크 | 위치 | 측정 가능 항목 |
|---------|------|----------------|
| `BenchmarkQuerier` | [querier_bench_test.go:34](prometheus/tsdb/querier_bench_test.go#L34) | 다양한 쿼리 패턴의 종합 성능 |
| `BenchmarkQuerierSelect` | [querier_bench_test.go:312](prometheus/tsdb/querier_bench_test.go#L312) | Select() 호출 |
| `BenchmarkQuerierSelectWithOutOfOrder` | [querier_bench_test.go:349](prometheus/tsdb/querier_bench_test.go#L349) | OOO 데이터 포함 쿼리 |
| `BenchmarkQueries` | [querier_test.go:3157](prometheus/tsdb/querier_test.go#L3157) | 다양한 쿼리 시나리오 |
| `BenchmarkQueryIterator` | [querier_test.go:2421](prometheus/tsdb/querier_test.go#L2421) | iterator 순회 비용 |
| `BenchmarkQuerySeek` | [querier_test.go:2484](prometheus/tsdb/querier_test.go#L2484) | seek 성능 |
| `BenchmarkSetMatcher` | [querier_test.go:2564](prometheus/tsdb/querier_test.go#L2564) | regex set matcher (`a=~"x|y|z"`) |
| `BenchmarkMergedSeriesSet` | [querier_test.go:2031](prometheus/tsdb/querier_test.go#L2031) | 여러 블록의 series merge |
| `BenchmarkHeadChunkQuerier` | [querier_test.go:3492](prometheus/tsdb/querier_test.go#L3492) | head chunk 쿼리 |
| `BenchmarkHeadQuerier` | [querier_test.go:3534](prometheus/tsdb/querier_test.go#L3534) | head 전체 쿼리 |

### 2.6 Chunks (인코딩/디코딩)

블로그의 **chunks 섹션** (raw chunk 포맷) 과 연결.

> *"It again looks similar to the memory-mapped head chunks on disk except that it is missing the series ref, mint and maxt. We needed this additional information for the Head chunks to recreate the in-memory index during startup. But in the case of blocks, we have this additional information in the index, because index is the place where it finally belongs, hence we don't need it here."*
>
> — Ganesh Vernekar, "TSDB format" blog Part 4
>
> → 즉 chunk 파일 자체는 가볍고, 메타데이터는 모두 index에 모여있습니다. 따라서 chunk 파일 크기 튜닝(`MaxBlockChunkSegmentSize`)은 mmap 단위에, index 튜닝은 쿼리 비용에 직접 영향을 줍니다.

| 벤치마크 | 위치 | 측정 가능 항목 |
|---------|------|----------------|
| `BenchmarkXORAppender` | [chunkenc/chunk_test.go:261](prometheus/tsdb/chunkenc/chunk_test.go#L261) | XOR(=Gorilla) 인코딩 (float chunk) |
| `BenchmarkXORIterator` | [chunkenc/chunk_test.go:257](prometheus/tsdb/chunkenc/chunk_test.go#L257) | XOR 디코딩 |
| `BenchmarkXorRead` | [chunkenc/xor_test.go:22](prometheus/tsdb/chunkenc/xor_test.go#L22) | XOR read |
| `BenchmarkAppendable` | [chunkenc/histogram_test.go:1727](prometheus/tsdb/chunkenc/histogram_test.go#L1727) | 히스토그램 chunk append 가능성 검사 |
| `BenchmarkChunkWriteQueue_addJob` | [chunks/chunk_write_queue_test.go:209](prometheus/tsdb/chunks/chunk_write_queue_test.go#L209) | chunk write queue enqueue 비용 — `HeadChunksWriteQueueSize` 영향 |

### 2.7 Isolation / Exemplars

| 벤치마크 | 위치 | 측정 가능 항목 |
|---------|------|----------------|
| `BenchmarkIsolation` | [isolation_test.go:82](prometheus/tsdb/isolation_test.go#L82) | isolation 오버헤드 — `IsolationDisabled` 효과 비교용 |
| `BenchmarkIsolationWithState` | [isolation_test.go:112](prometheus/tsdb/isolation_test.go#L112) | 상태 보존 isolation |
| `BenchmarkAddExemplar` | [exemplar_test.go:1049](prometheus/tsdb/exemplar_test.go#L1049) | exemplar 추가 — `MaxExemplars` 튜닝 시 비교 |
| `BenchmarkAddExemplar_OutOfOrder` | [exemplar_test.go:1080](prometheus/tsdb/exemplar_test.go#L1080) | OOO exemplar |
| `BenchmarkResizeExemplars` | [exemplar_test.go:1165](prometheus/tsdb/exemplar_test.go#L1165) | circular buffer 리사이즈 |

## 3. 추천 튜닝 시나리오 (블로그 챕터 ↔ 실험 매핑)

| 블로그 챕터 | 핵심 관찰 | 추천 실험 |
|------------|----------|-----------|
| **chunks 디렉터리** (512MiB cap) | 파일이 커질수록 mmap 단위가 커지지만 random access 영향 | `MaxBlockChunkSegmentSize` 변경 → `BenchmarkCompactionFromHead`, `BenchmarkOpenBlock` |
| **index → Symbol Table** | 중복 제거된 string 풀 → label cardinality에 민감 | `BenchmarkCompactionFromHead`에서 `labelnames × labelvalues` 케이스 비교 |
| **index → Series 섹션** | 16-byte aligned, ID = offset/16 | `BenchmarkLoadWLs`로 series 수 증가 시 메모리/시간 측정 |
| **index → Postings (inverted index)** | label-pair → series ID list. 쿼리 핵심 경로 | `BenchmarkIntersect`, `BenchmarkMerge`, `BenchmarkMemPostings_PostingsForLabelMatching` |
| **tombstones** | 블록 생성 후에도 수정되는 유일한 파일 | `BenchmarkMemPostings_Delete`로 삭제 작업 부하 측정 |

> *"Tombstones are deletion markers, i.e., they tell us what time ranges of which series to ignore during reads. This is the only file in the block which is created and modified after writing a block to store the delete requests."*
>
> — Ganesh Vernekar, "TSDB format" blog Part 4

| **meta.json + compaction level** | level이 올라갈수록 블록 크기 증가 | `BenchmarkCompaction`의 `ranges` 케이스로 level별 비용 |
| **WAL → Head replay** | 시작 시 가장 큰 비용 | `BenchmarkLoadWLs`에서 `WALReplayConcurrency`, `WALCompression` 변화 비교 |

## 4. 실행 예시

```bash
# 1) WAL 압축 방식별 throughput
cd prometheus/tsdb/wlog
go test -run='^$' -bench='^BenchmarkWAL_LogBatched$' -benchtime=5s -count=3

# 2) StripeSize 효과 (소스에서 DefaultStripeSize 또는 opts.StripeSize 변경 후)
cd prometheus/tsdb
go test -run='^$' -bench='^BenchmarkHeadStripeSeriesCreateParallel$' \
        -benchtime=3s -cpu=1,2,4,8 -count=3

# 3) Compaction 비용 (label cardinality 영향)
go test -run='^$' -bench='^BenchmarkCompactionFromHead$' -benchtime=3s -count=3

# 4) WAL replay (재시작 시간 시뮬레이션)
go test -run='^$' -bench='^BenchmarkLoadWLs$' -benchtime=1x -count=3 -timeout=30m

# 5) Index postings 쿼리 패턴
cd index
go test -run='^$' -bench='^BenchmarkIntersect$|^BenchmarkMerge$' -benchtime=3s -count=3
```

## 5. 결론: 블록 구조 ↔ 튜닝 포인트

블로그에서 본 persistent block의 4가지 구성 요소(`meta.json`, `chunks/`, `index`, `tombstones`)는 각각 다른 configuration에 의해 형태와 성능이 결정됩니다:

- **chunks/** ← `MaxBlockChunkSegmentSize`, `SamplesPerChunk`, chunk 인코딩 (`BenchmarkXOR*`)
- **index** ← label cardinality, `PostingsDecoderFactory`, `EnableSharding` (`BenchmarkPostings*`, `BenchmarkLabelValues*`)
- **tombstones** ← 삭제 패턴, compaction 주기 (`BenchmarkMemPostings_Delete`)
- **meta.json + 블록 자체** ← `MinBlockDuration`, `MaxBlockDuration`, `EnableOverlappingCompaction` (`BenchmarkCompaction*`)

또한 블록 *생성 이전* 단계인 Head/WAL의 튜닝(`StripeSize`, `WALCompression`, `HeadChunksWriteBufferSize`)이 결국 어떤 블록이 만들어지는지를 결정합니다. 위 벤치마크들은 모두 코드 수정 없이 **`Options` / `HeadOptions` 필드를 수정**하거나 **벤치마크 내부의 케이스 슬라이스를 확장**하는 것만으로 다양한 시나리오를 측정할 수 있도록 잘 설계되어 있습니다.
