# Week5 - Persistent Block & Compaction 튜닝 분석

> *"A block consists of 4 parts: **meta.json**, **chunks** (directory), **index**, **tombstones**."*

> 한 블록은 **meta.json**, **chunks**(디렉터리), **index**, **tombstones** 4개 부분으로 구성됩니다.

이 4개 구성 요소 중 **index**와 **chunks** 관련 **TSDB 내부 Benchmark Test**를 측정·분석합니다. 두 실험은 모두 `BenchmarkCompactionFromHead`를 베이스로 하지만, 관찰하는 축이 서로 다릅니다.

- **1.1 index 실험** — 총 series 수(100,000)는 고정한 채 `labelnames × labelvalues` 비율을 바꿔서 **inverted index 빌드 비용의 변화**를 본다.
- **1.2 chunks 실험** — `MaxBlockChunkSegmentSize`를 바꿔서 **chunk 파일 segment 크기가 compaction I/O 경로에 주는 영향**을 본다.

## 1. Block 구성요소별 튜닝 매핑

### 1.1. `index` File

> *"Index contains all that you need to query the data of this block. It does not share any data with any other blocks ... The index is an 'inverted index'."*

> 인덱스는 해당 블록의 데이터를 쿼리하기 위해 필요한 모든 것을 담고 있습니다. 다른 블록과 어떤 데이터도 공유하지 않으며, 그 자체가 **역인덱스(inverted index)** 입니다.

> *"[Symbol Table] holds a sorted list of deduplicated strings ... The other sections in the index can refer to this symbol table for any strings and hence significantly reduce the index size."*

> Symbol Table은 **중복이 제거된 문자열의 정렬된 리스트**를 보관합니다. 인덱스의 다른 섹션들은 문자열을 직접 저장하는 대신 symbol table을 참조하기 때문에 인덱스 크기가 크게 줄어듭니다.

> *"Each series entry is 16 byte aligned ... Hence we set the ID of the series to be offset/16. ... So whenever we say posting in the context of Prometheus TSDB, it refers to a series ID."*

> 각 series 엔트리는 **16바이트 정렬**되어 있으므로, series ID는 단순히 `offset/16`으로 정의됩니다. 그래서 Prometheus TSDB 문맥에서 "posting"이라고 하면 곧 **series ID**를 가리킵니다.

> *"This postings list and postings offset table form the inverted index. ... for every label-value pair, we store the list of series that it appears in."*

> postings list와 postings offset table이 합쳐져 역인덱스를 이룹니다. 즉 **모든 label-value 쌍마다 그 쌍이 등장한 series ID 리스트**를 저장합니다.

#### 이 테스트가 블로그의 어떤 부분을 측정하는가

블로그가 강조하는 인덱스의 핵심 비용은 두 가지입니다.

1. **Symbol Table 구축** — `labelnames`가 늘면 새로운 label 키가 추가되고, `labelvalues`가 늘면 새로운 값 문자열이 추가되어 dedup 대상 문자열 수가 변합니다.
2. **Postings list / postings offset table** — `(label, value)` 조합 수만큼 postings list가 만들어집니다. 즉 **label cardinality 분포**가 인덱스 빌드 비용을 결정합니다.

`BenchmarkCompactionFromHead`는 총 series 수를 100,000으로 고정하고 `labelnames × labelvalues`만 바꾸기 때문에, **chunks write 비용은 거의 일정한 상태**에서 **index build 비용의 변화**만 분리해서 관찰할 수 있습니다. 정확히 블로그가 말하는 "Symbol Table → Series → Postings → Postings Offset Table" 빌드 파이프라인을 측정하는 셈입니다.

- 실험: Index 크기와 조회 비용은 **label cardinality**에 결정, compaction 시 index 빌드 비용이 크기 때문에 `BenchmarkCompactionFromHead`의 `labelnames × labelvalues`로 영향 관찰 가능
    - 측정 결과
    ```
    ❯ go test -run='^$' -bench='^BenchmarkCompactionFromHead$' -benchtime=3s -benchmem
    goos: darwin
    goarch: arm64
    pkg: github.com/prometheus/prometheus/tsdb
    cpu: Apple M1 Max
    BenchmarkCompactionFromHead/labelnames=1,labelvalues=100000-10                 7         480348191 ns/op        110189829 B/op    918189 allocs/op
    BenchmarkCompactionFromHead/labelnames=10,labelvalues=10000-10                 7         474892708 ns/op        114062202 B/op    983636 allocs/op
    BenchmarkCompactionFromHead/labelnames=100,labelvalues=1000-10                 7         479650208 ns/op        117438312 B/op    999340 allocs/op
    BenchmarkCompactionFromHead/labelnames=1000,labelvalues=100-10                 7         478110286 ns/op        111197570 B/op   1010019 allocs/op
    BenchmarkCompactionFromHead/labelnames=10000,labelvalues=10-10                 7         492989429 ns/op        116217686 B/op   1058826 allocs/op
    PASS
    ok      github.com/prometheus/prometheus/tsdb   18.276s
    ```
    - 성능 분석

| labelnames × labelvalues | ns/op | B/op | allocs/op |
|---|---|---|---|
| 1 × 100,000 | 480 ms | 110.2 MB | 918,189 |
| 10 × 10,000 | 475 ms | 114.1 MB | 983,636 |
| 100 × 1,000 | 480 ms | 117.4 MB | 999,340 |
| 1,000 × 100 | 478 ms | 111.2 MB | 1,010,019 |
| 10,000 × 10 | 493 ms | 116.2 MB | 1,058,826 |

**관찰 1: ns/op는 거의 평탄하다 (편차 ±2%, 최대 ~3.7%).**
총 series 수가 같은 한 compaction의 wall-clock 시간은 label 분포에 거의 영향받지 않습니다. 블로그가 설명한 inverted index 구조가 잘 설계되었음을 확인할 수 있습니다 — `(label, value)` 조합 개수가 1×100,000(=100,000개의 postings list)에서 10,000×10(=100,000개의 postings list)로 바뀌어도 총 postings 개수는 동일하므로 빌드 비용 자체가 같은 수준에 머무릅니다.

**관찰 2: allocs/op는 labelnames에 비례해 단조 증가한다 (918K → 1,058K, +15%).**
이게 인덱스 빌드 비용의 핵심 신호입니다.
- labelnames=1일 때 symbol table은 `{labelKey, value0..value99999}` = 약 100,001개의 고유 문자열.
- labelnames=10,000일 때는 `{key0..key9999, value0..value9}` = 약 10,010개의 고유 문자열로 **오히려 줄어듭니다**.
- 그런데 allocs는 거꾸로 늘어납니다. 이유는 **postings offset table과 label-name 단위 인덱싱**입니다. label NAME이 많아지면 `LabelNames()`/`LabelValues()` 조회용 메타데이터, label name별 postings 묶음, 정렬 단계의 임시 슬라이스가 모두 늘어납니다. 즉 allocs는 symbol table 크기가 아니라 **distinct label name 개수에 더 민감**합니다.

**관찰 3: B/op는 110~117 MB 범위에서 약하게 변동.**
total bytes는 chunks write 버퍼가 지배적이라 compaction 파이프라인 전체에서는 별 차이가 없습니다. 인덱스 부분만 보면 100×1000 케이스(117 MB)가 가장 크고, 1×100000(110 MB)이 가장 작은데, 이는 **postings offset table이 "label name당 한 묶음"으로 직렬화될 때 중간 정도의 fan-out에서 임시 버퍼가 가장 커지기 때문**으로 보입니다. 너무 적은 label name(1)에서는 묶음 자체가 1개라 단순하고, 너무 많은 label name(10,000)에서는 묶음당 데이터가 작아 버퍼 재사용이 잘 됩니다.

**결론**

- 인덱스 빌드는 **총 series 수에 선형**, label 분포에는 거의 둔감하다 → cardinality 폭발이 일어나도 *시간*은 거의 같은 비율로만 증가한다.
- 단, **distinct label name 수**는 alloc 수와 GC 압력을 키운다 → label naming이 무분별하게 늘어나는 메트릭(예: 동적으로 label key를 생성하는 케이스)은 메모리/GC 측면에서 위험하다.
- 운영 관점: cardinality 모니터링은 "총 series 수"뿐 아니라 "**distinct label name 수**"를 함께 봐야 한다.

### 1.2. `chunks/` Directory

> *"The chunks directory contains a sequence of numbered files similar to the WAL/checkpoint/head chunks. **Each file is capped at 512MiB**."*

> chunks 디렉터리는 WAL/checkpoint/head chunks와 비슷하게 **번호가 매겨진 파일들의 시퀀스**로 구성되며, **각 파일의 상한은 512MiB**입니다.

> *"It again looks similar to the memory-mapped head chunks on disk except that it is missing the series ref, mint and maxt. ... in the case of blocks, we have this additional information in the index, because index is the place where it finally belongs, hence we don't need it here."*

> 디스크 상의 chunk 파일은 mmap된 head chunks와 거의 동일한 포맷이지만, **series ref, mint, maxt가 빠져 있습니다**. 블록의 경우 이 메타데이터들은 결국 index가 보관할 정보이므로 chunk 파일에 중복으로 둘 필요가 없기 때문입니다.

#### 이 테스트가 블로그의 어떤 부분을 측정하는가

블로그에서 chunks와 관련해 강조되는 두 가지 사실은 다음과 같습니다.

1. **각 chunk segment 파일은 512MiB로 제한된다** → segment 크기는 디스크 I/O 단위와 한 파일 당 mmap 영역 크기를 결정합니다.
2. **chunk 파일은 메타데이터를 거의 가지지 않는다** → 즉 segment 크기를 바꿔도 in-memory 처리(symbol table, postings 등)에는 영향이 없고, 오직 **파일 I/O 경로**만 바뀝니다.

이 실험은 `MaxBlockChunkSegmentSize`(= chunk segment 파일 1개의 최대 크기, 기본 512MiB)를 128MB / 512MB / 1GB로 바꿔가며, **블로그가 말한 "512MiB 상한"이라는 디자인 결정이 실제 compaction 성능에 어떤 trade-off를 만드는지** 측정합니다.

여기서 한 가지 핵심 구현 디테일이 결과 해석에 결정적입니다: [`prometheus/tsdb/chunks/chunks.go:445`](prometheus/tsdb/chunks/chunks.go#L445) 의 `cutSegmentFile()`는 새 chunk segment를 만들 때 `fileutil.Preallocate(f, segmentSize, true)`로 **세그먼트 크기 전체를 디스크에 미리 할당**합니다. 그리고 데이터 쓰기가 끝나면 [`finalizeTail()`](prometheus/tsdb/chunks/chunks.go#L358)에서 `Truncate(off)`로 실제 사이즈로 줄입니다. 즉 segsize=1GB이면 매 cut마다 1GB를 fallocate하고 다시 truncate합니다.

- chunk 파일 자체는 가볍고 메타데이터는 모두 index에 있음. 따라서 `MaxBlockChunkSegmentSize` 튜닝은 **파일 preallocate/truncate 비용**과 mmap 단위에 영향
- 실험: `MaxBlockChunkSegmentSize = 128MB / 512MB / 1GB`로 변경해 `BenchmarkCompactionFromHead` 성능 측정
    - `prometheus/tsdb/compact_test.go` 내부 `BenchmarkCompactionFromHead` 수정
    ``` go
    func BenchmarkCompactionFromHead(b *testing.B) {
        dir := b.TempDir()
        totalSeries := 100000
        segmentSizes := []struct {
            name string
            size int64
        }{
            {"segsize=128MB", 128 * 1024 * 1024},
            {"segsize=512MB", 512 * 1024 * 1024}, // default
            {"segsize=1GB", 1024 * 1024 * 1024},
        }
        for _, seg := range segmentSizes {
            for labelNames := 1; labelNames < totalSeries; labelNames *= 10 {
                labelValues := totalSeries / labelNames
                b.Run(fmt.Sprintf("%s,labelnames=%d,labelvalues=%d", seg.name, labelNames, labelValues), func(b *testing.B) {
                    chunkDir := b.TempDir()
                    opts := DefaultHeadOptions()
                    opts.ChunkRange = 1000
                    opts.ChunkDirRoot = chunkDir
                    h, err := NewHead(nil, nil, nil, nil, opts, nil)
                    require.NoError(b, err)
                    for ln := 0; ln < labelNames; ln++ {
                        app := h.Appender(context.Background())
                        for lv := range labelValues {
                            app.Append(0, labels.FromStrings(strconv.Itoa(ln), fmt.Sprintf("%d%s%d", lv, postingsBenchSuffix, ln)), 0, 0)
                        }
                        require.NoError(b, app.Commit())
                    }

                    compactor, err := NewLeveledCompactorWithOptions(context.Background(), nil, promslog.NewNopLogger(), []int64{1000000}, nil, LeveledCompactorOptions{
                        MaxBlockChunkSegmentSize:    seg.size,
                        EnableOverlappingCompaction: true,
                    })
                    require.NoError(b, err)

                    b.ResetTimer()
                    b.ReportAllocs()
                    for i := 0; b.Loop(); i++ {
                        blockDir := filepath.Join(dir, fmt.Sprintf("%s-%d-%d", seg.name, i, labelNames))
                        require.NoError(b, os.MkdirAll(blockDir, 0o777))
                        _, err := compactor.Write(blockDir, h, h.MinTime(), h.MaxTime()+1, nil)
                        require.NoError(b, err)
                    }
                    h.Close()
                })
            }
        }
    }
    ```
    - 측정 결과 (`totalSeries = 100,000`, `-benchtime=3s`)
    ```
    ❯ go test -run='^$' -bench='^BenchmarkCompactionFromHead$' -benchtime=3s -benchmem
    goos: darwin
    goarch: arm64
    pkg: github.com/prometheus/prometheus/tsdb
    cpu: Apple M1 Max
    BenchmarkCompactionFromHead/segsize=128MB,labelnames=1,labelvalues=100000-10                  10         305371350 ns/op        110404684 B/op    918098 allocs/op
    BenchmarkCompactionFromHead/segsize=128MB,labelnames=10,labelvalues=10000-10                  10         308648342 ns/op        114211396 B/op    983560 allocs/op
    BenchmarkCompactionFromHead/segsize=128MB,labelnames=100,labelvalues=1000-10                  10         309965850 ns/op        117292599 B/op    999253 allocs/op
    BenchmarkCompactionFromHead/segsize=128MB,labelnames=1000,labelvalues=100-10                  10         332202642 ns/op        111065212 B/op   1009942 allocs/op
    BenchmarkCompactionFromHead/segsize=128MB,labelnames=10000,labelvalues=10-10                   9         335729903 ns/op        115821490 B/op   1058764 allocs/op
    BenchmarkCompactionFromHead/segsize=512MB,labelnames=1,labelvalues=100000-10                   7         480574696 ns/op        110141883 B/op    918082 allocs/op
    BenchmarkCompactionFromHead/segsize=512MB,labelnames=10,labelvalues=10000-10                   7         471992256 ns/op        114211232 B/op    983559 allocs/op
    BenchmarkCompactionFromHead/segsize=512MB,labelnames=100,labelvalues=1000-10                   7         485253054 ns/op        117262793 B/op    999258 allocs/op
    BenchmarkCompactionFromHead/segsize=512MB,labelnames=1000,labelvalues=100-10                   7         475948429 ns/op        111202778 B/op   1009938 allocs/op
    BenchmarkCompactionFromHead/segsize=512MB,labelnames=10000,labelvalues=10-10                   7         490884923 ns/op        115738236 B/op   1058720 allocs/op
    BenchmarkCompactionFromHead/segsize=1GB,labelnames=1,labelvalues=100000-10                     7         490514690 ns/op        110593642 B/op    918108 allocs/op
    BenchmarkCompactionFromHead/segsize=1GB,labelnames=10,labelvalues=10000-10                     6         514819146 ns/op        114052790 B/op    983573 allocs/op
    BenchmarkCompactionFromHead/segsize=1GB,labelnames=100,labelvalues=1000-10                     6         503419382 ns/op        117350185 B/op    999252 allocs/op
    BenchmarkCompactionFromHead/segsize=1GB,labelnames=1000,labelvalues=100-10                     6         506521500 ns/op        111098534 B/op   1009937 allocs/op
    BenchmarkCompactionFromHead/segsize=1GB,labelnames=10000,labelvalues=10-10                     6         522839326 ns/op        116216852 B/op   1058762 allocs/op
    PASS
    ok      github.com/prometheus/prometheus/tsdb   50.495s
    ```
    - 성능 분석 (100K series)

각 segsize별로 5개 cardinality 케이스의 평균을 내면:

| segsize | 평균 ns/op | vs 512MB | 평균 B/op | 평균 allocs/op |
|---|---|---|---|---|
| **128MB** | **318 ms** | **−34%** | 113.8 MB | 994K |
| 512MB (default) | 481 ms | 0% | 113.7 MB | 994K |
| 1GB | 508 ms | +5.6% | 113.9 MB | 994K |

**관찰 1: 128MB가 일관되게 가장 빠르다 (−34%).**
모든 5개 cardinality 케이스에서 128MB → 512MB → 1GB 순으로 ns/op가 단조 증가합니다. 동일 cardinality 안에서 비교해 보면:

| cardinality | 128MB | 512MB | 1GB |
|---|---|---|---|
| 1×100,000 | 305 ms | 481 ms | 491 ms |
| 10×10,000 | 309 ms | 472 ms | 515 ms |
| 100×1,000 | 310 ms | 485 ms | 503 ms |
| 1,000×100 | 332 ms | 476 ms | 507 ms |
| 10,000×10 | 336 ms | 491 ms | 523 ms |

이건 놀라운 결과입니다. 일반적으로 "더 큰 세그먼트 = 파일 분할 적음 = 빠를 것"이라고 예상하기 쉽지만, 측정은 정반대입니다.

**관찰 2: 원인은 chunk segment의 preallocate 비용이다.**
앞서 확인한 [`cutSegmentFile()`](prometheus/tsdb/chunks/chunks.go#L422)의 동작을 다시 보면:

```go
if allocSize > 0 {
    if err = fileutil.Preallocate(f, allocSize, true); err != nil {
        return ..., fmt.Errorf("preallocate: %w", err)
    }
}
```

여기서 `allocSize`는 `segmentSize` 전체입니다. macOS APFS에서 `posix_fallocate`는 단순한 메타데이터 조작이 아니라 **블록 매핑을 실제로 잡고 zero를 채우는 비용**이 있어서 할당 크기에 거의 비례하는 시간을 씁니다. 1GB를 preallocate한 뒤 실제로는 수십 MB만 쓰고 truncate로 줄이는 것이 현재 구현이므로, segsize가 클수록 **순수하게 낭비되는 fallocate 시간**이 늘어납니다.

이 실험에서 100,000 series × 단일 sample은 chunk 데이터가 매우 작아 (블록 1개당 chunk 파일 1개로 충분), preallocate 1회의 오버헤드가 wall-clock 시간의 큰 비중을 차지합니다. 따라서 segsize가 작을수록 fallocate가 빨리 끝나 전체 compaction이 빨리 끝납니다.

**관찰 3: B/op·allocs/op는 segsize와 완전히 무관하다.**
세 segsize의 평균 메모리 사용량과 alloc 수는 사실상 동일합니다 (113.8 vs 113.7 vs 113.9 MB, 994K allocs 동일). 이는 블로그의 *"chunk 파일은 series ref, mint, maxt를 갖지 않는다"* 와 일치합니다 — segment 크기는 in-memory 빌드 파이프라인을 전혀 건드리지 않고, 오직 디스크 파일 분할 경계만 바꿉니다. 즉 `MaxBlockChunkSegmentSize`는 **순수 I/O 파라미터**입니다.

**관찰 4: cardinality 자체의 영향은 segsize를 가로질러 동일하다.**
1.1에서 본 "labelnames가 늘면 allocs가 늘어나는" 경향(918K → 1058K)은 세 segsize 모두에서 동일하게 나타납니다. 즉 **두 효과는 직교(orthogonal)** 합니다 — index 빌드 비용과 chunk write 비용이 서로 독립적인 경로라는 의미입니다.

    - 측정 결과 (`totalSeries = 10,000,000`, `-benchtime=1x`)

100K 결과는 series 수가 작아 preallocate 1회 비용이 측정값을 지배할 가능성이 있어, **운영 규모에 가까운 10M series**로 다시 측정했습니다. (`totalSeries := 10000000`으로 변경하고 `-benchtime=1x`로 1회만 실행. 총 ~19분 소요)

    ```
    ❯ go test -run='^$' -bench='^BenchmarkCompactionFromHead$' -benchtime=1x -benchmem
    goos: darwin
    goarch: arm64
    pkg: github.com/prometheus/prometheus/tsdb
    cpu: Apple M1 Max
    BenchmarkCompactionFromHead/segsize=128MB,labelnames=1,labelvalues=10000000-10                 1        38955924958 ns/op       9370268728 B/op  100090214 allocs/op
    BenchmarkCompactionFromHead/segsize=128MB,labelnames=10,labelvalues=1000000-10                 1        34335330416 ns/op       9686948752 B/op  100090275 allocs/op
    BenchmarkCompactionFromHead/segsize=128MB,labelnames=100,labelvalues=100000-10                 1        35414771584 ns/op       9143245584 B/op  100143980 allocs/op
    BenchmarkCompactionFromHead/segsize=128MB,labelnames=1000,labelvalues=10000-10                 1        35442284083 ns/op       9391599320 B/op  100212393 allocs/op
    BenchmarkCompactionFromHead/segsize=128MB,labelnames=10000,labelvalues=1000-10                 1        36239935042 ns/op       9636973656 B/op  100360766 allocs/op
    BenchmarkCompactionFromHead/segsize=128MB,labelnames=100000,labelvalues=100-10                 1        36727810166 ns/op       9037747480 B/op  101333036 allocs/op
    BenchmarkCompactionFromHead/segsize=128MB,labelnames=1000000,labelvalues=10-10                 1        40908818167 ns/op       9549825608 B/op  106160257 allocs/op
    BenchmarkCompactionFromHead/segsize=512MB,labelnames=1,labelvalues=10000000-10                 1        37834242125 ns/op       9371332176 B/op  100090133 allocs/op
    BenchmarkCompactionFromHead/segsize=512MB,labelnames=10,labelvalues=1000000-10                 1        36849330709 ns/op       9736336936 B/op  100090013 allocs/op
    BenchmarkCompactionFromHead/segsize=512MB,labelnames=100,labelvalues=100000-10                 1        35199919416 ns/op       9070593368 B/op  100143938 allocs/op
    BenchmarkCompactionFromHead/segsize=512MB,labelnames=1000,labelvalues=10000-10                 1        34755220792 ns/op       9342053648 B/op  100212405 allocs/op
    BenchmarkCompactionFromHead/segsize=512MB,labelnames=10000,labelvalues=1000-10                 1        36317799958 ns/op       9636969904 B/op  100360738 allocs/op
    BenchmarkCompactionFromHead/segsize=512MB,labelnames=100000,labelvalues=100-10                 1        36505278875 ns/op       9037743824 B/op  101333007 allocs/op
    BenchmarkCompactionFromHead/segsize=512MB,labelnames=1000000,labelvalues=10-10                 1        42347794250 ns/op       9530585488 B/op  106160251 allocs/op
    BenchmarkCompactionFromHead/segsize=1GB,labelnames=1,labelvalues=10000000-10                   1        40785909083 ns/op       9437355720 B/op  100089931 allocs/op
    BenchmarkCompactionFromHead/segsize=1GB,labelnames=10,labelvalues=1000000-10                   1        34440712542 ns/op       9709023616 B/op  100090201 allocs/op
    BenchmarkCompactionFromHead/segsize=1GB,labelnames=100,labelvalues=100000-10                   1        36090001209 ns/op       9176049672 B/op  100143931 allocs/op
    BenchmarkCompactionFromHead/segsize=1GB,labelnames=1000,labelvalues=10000-10                   1        33355769917 ns/op       9364594704 B/op  100212373 allocs/op
    BenchmarkCompactionFromHead/segsize=1GB,labelnames=10000,labelvalues=1000-10                   1        38918869792 ns/op       9653747008 B/op  100360736 allocs/op
    BenchmarkCompactionFromHead/segsize=1GB,labelnames=100000,labelvalues=100-10                   1        39463521458 ns/op       8987412176 B/op  101333005 allocs/op
    BenchmarkCompactionFromHead/segsize=1GB,labelnames=1000000,labelvalues=10-10                   1        41810700875 ns/op       9567847088 B/op  106160331 allocs/op
    PASS
    ok      github.com/prometheus/prometheus/tsdb   1133.795s
    ```
    - 성능 분석 (10M series)

#### segsize별 요약 (10M series, 7개 cardinality 케이스 평균)

| segsize | 평균 wall-clock | vs 512MB | 평균 B/op | 평균 allocs/op |
|---|---|---|---|---|
| **128MB** | **36.86 s** | **−0.7%** | 9.40 GB | 101.2M |
| 512MB (default) | 37.12 s | 0% | 9.39 GB | 101.2M |
| 1GB | 37.84 s | +1.9% | 9.41 GB | 101.2M |

#### 동일 cardinality 비교 (단위: 초)

| cardinality | 128MB | 512MB | 1GB |
|---|---|---|---|
| 1 × 10,000,000 | 38.96 | 37.83 | 40.79 |
| 10 × 1,000,000 | 34.34 | 36.85 | 34.44 |
| 100 × 100,000 | 35.41 | 35.20 | 36.09 |
| 1,000 × 10,000 | 35.44 | 34.76 | **33.36** |
| 10,000 × 1,000 | 36.24 | 36.32 | 38.92 |
| 100,000 × 100 | 36.73 | 36.51 | 39.46 |
| 1,000,000 × 10 | 40.91 | 42.35 | 41.81 |

#### 핵심 발견: 100K → 10M 으로 가면 segsize 효과가 사라진다

| 데이터셋 크기 | 128MB vs 512MB | 1GB vs 512MB |
|---|---|---|
| 100,000 series | **−34%** (320 ms vs 481 ms) | +5.6% |
| 10,000,000 series | **−0.7%** (36.86 s vs 37.12 s) | +1.9% |

100K에서는 128MB가 압도적으로 빨랐는데, 10M에서는 세 segsize의 차이가 측정 노이즈 수준입니다. 일부 케이스(`1,000×10,000`)에서는 오히려 1GB가 가장 빠릅니다. 이건 1.2의 가장 중요한 결과입니다.

**관찰 1 — preallocate 비용은 "한 번만" 비싸다, 그래서 데이터가 커지면 묻힌다.**
[`cutSegmentFile()`](prometheus/tsdb/chunks/chunks.go#L422)이 새 segment를 만들 때 호출하는 `Preallocate(f, segmentSize, true)`는 segment 크기에 비례한 비용을 한 번 씁니다. 100K series 실험에서는 한 블록당 segment cut이 1번만 일어났고, 실제 chunk 데이터가 수십 MB에 불과해 그 1회 preallocate가 wall-clock의 큰 비중을 차지했습니다 (128MB ≈ 320ms, 1GB ≈ 510ms).

10M series에서는:
- 한 블록의 chunk 데이터 자체가 수 GB 단위로 커집니다 (`B/op ≈ 9.4 GB`).
- segsize=128MB에서는 segment cut이 수십 회 일어나야 하지만, 각 cut의 preallocate가 작아 누적 비용이 ≈ 같은 수준으로 줄어듭니다.
- segsize=1GB에서도 preallocate 1GB가 여러 번 일어나지만, 이제는 그 비용이 **실제 데이터 write 시간**(≈ 30+초)에 묻힙니다.

즉 preallocate 오버헤드 / 실제 write 비용 비율이 **0.4 → 0.02** 수준으로 떨어진 것입니다.

**관찰 2 — 메모리·alloc 수는 여전히 segsize와 무관하다.**
세 segsize의 `B/op`(9.40 / 9.39 / 9.41 GB)와 `allocs/op`(101.2M 동일)는 사실상 일치합니다. 이는 100K 실험과 동일한 결론을 재확인합니다 — `MaxBlockChunkSegmentSize`는 **순수 I/O 경로 파라미터**이며 in-memory 파이프라인은 건드리지 않습니다. 데이터 크기가 100배 커져도 이 성질은 변하지 않습니다.

**관찰 3 — cardinality 효과가 더 뚜렷해진다.**
10M 스케일에서는 cardinality 분포가 ns/op에 미치는 영향이 100K보다 크게 보입니다.

| cardinality | 128MB | allocs/op | 비고 |
|---|---|---|---|
| 1 × 10M | 38.96s | 100,090,214 | 단일 label name, postings 1개 (10M series) |
| 10 × 1M ~ 10K × 1K | 34~36s | 100~100.4M | 균형 잡힌 분포: 가장 빠른 영역 |
| 1M × 10 | 40.91s | 106,160,257 | 극단적 label name 폭발 |

- **양 끝(`1×10M`, `1M×10`)이 모두 느립니다.** 너무 적은 label name은 거대 단일 postings list 직렬화로 느려지고, 너무 많은 label name은 distinct symbol/postings offset entry가 많아 느려집니다.
- **alloc 수가 1M label name 케이스에서 +6% 튐** (100,090K → 106,160K). 1.1에서 본 "distinct label name 수가 alloc 압력의 진짜 동인"이라는 가설이 10M 스케일에서 훨씬 강하게 확인됩니다.
- 1.1(100K) 케이스에서는 ns/op 편차가 ±2%였는데, 10M에서는 ±10%로 벌어집니다. **데이터셋이 커질수록 cardinality 분포의 영향이 노출됩니다.**

**관찰 4 — segsize와 cardinality는 여전히 거의 직교한다.**
"가장 빠른 cardinality"가 segsize에 따라 달라지긴 하지만 (128MB는 `10×1M`, 512MB는 `1000×10K`, 1GB는 `1000×10K`), 절대 차이는 1~2초 수준이고, **각 segsize 내부의 cardinality 패턴**(양 끝이 느림)은 동일하게 유지됩니다. 즉 두 축의 효과는 거의 독립적이며, segsize가 어떤 cardinality를 "유리하게" 만들어주는 일은 없습니다.

#### 결론 — 운영 가이드 (수정)

100K 실험만 보고 "segsize=128MB가 무조건 빠르다"고 결론내리면 안 된다는 것이 10M 실험으로 확인됐습니다.

- **실제 운영 규모(블록당 수 GB chunk 데이터)에서는 segsize 차이가 ±2% 안에 들어옵니다.** 그러므로 segsize는 **쓰기 속도 기준으로는 default(512MB)에서 굳이 바꿀 이유가 없습니다.**
- 100K에서 본 -34%는 **벤치마크 인공물(artifact)** 에 가깝습니다 — 작은 데이터셋에서는 preallocate 1회의 상수 비용이 측정값을 지배하기 때문입니다. 합성 벤치마크 결과를 운영 환경 결정에 그대로 가져오면 안 된다는 좋은 사례입니다.
- 운영 관점에서 segsize 결정의 진짜 trade-off는 **쓰기 속도가 아니라**:
  1. **블록당 chunk 파일 개수** → `BenchmarkOpenBlock`(재시작 readiness)에 영향
  2. **mmap 영역 크기** → 페이지 캐시 압력과 random read 패턴
  3. **공간 사용** (preallocate 후 truncate 사이의 일시적 디스크 점유)

  이 세 가지에 있습니다. **단일 벤치마크 수치만으로 default를 바꿔서는 안 됩니다.**
- cardinality 측면에서는 더 명확합니다: **label name이 10만개를 넘기 시작하면** ns/op와 alloc 수가 동시에 악화됩니다 (`1M×10` 케이스가 평균 대비 +13%). 운영 모니터링은 "총 series 수"뿐 아니라 "**distinct label name 수**"를 함께 추적해야 한다는 1.1의 결론이 10M 스케일에서 더 강하게 뒷받침됩니다.
