# Durability (내구성)

## **1. ACID란?**

데이터베이스 시스템에서 Transaction의 안전성을 보장하기 위한 4가지 핵심 속성입니다.

| 속성 | 의미 | 핵심 질문 |
|------|------|-----------|
| **Atomicity** (원자성) | Transaction은 전부 실행되거나 전부 취소되거나 (All or Nothing) | "중간에 멈추면 어떡하지?" |
| **Consistency** (일관성) | Transaction 전후로 데이터베이스는 항상 유효한 상태를 유지 | "규칙이 깨지진 않나?" |
| **Isolation** (격리성) | 동시에 실행되는 Transaction들이 서로 간섭하지 않음 | "다른 작업이 끼어들면?" |
| **Durability** (내구성) | Commit된 Transaction의 결과는 영구적으로 보존됨 | "저장했는데 날아가면?" |

## **2. Durability란?**

**Commit된 Transaction의 결과는 시스템 장애가 발생하더라도 영구적으로 보존되어야 한다**는 속성입니다.

예를 들어 비행기 좌석 예약 시스템에서 "예약 완료"라는 응답을 받았다면, 그 직후 시스템이 crash 되더라도 해당 좌석은 예약된 상태로 남아 있어야 합니다.

### 2.1 Durability가 견뎌야 하는 3가지 장애 수준

| 수준 | 장애 원인 | 예시 |
|------|-----------|------|
| **Transaction Level** | Transaction 자체의 실패 | 데이터 입력 오류, 타임아웃, 잔액 부족 등 |
| **System Level** | 휘발성 메모리(RAM)의 손실 | 시스템 crash, 정전, OOM |
| **Media Level** | 비휘발성 저장소의 손실 | 디스크 고장, SSD 손상 |

### 2.2 각 수준별 보장 메커니즘

#### Transaction Level — Serializability

Transaction 수준의 장애(취소, 제약 조건 위반 등)는 **직렬화 가능성(Serializability)** 으로 해결합니다. Commit된 Transaction의 결과는 메인 메모리에 유지되고, Commit되지 않은 Transaction의 변경 사항은 다른 Transaction과 분리되어 있으므로 안전하게 폐기(Undo)할 수 있습니다.

- 기존 값을 덮어쓰는 **in-place update는 지양**하고, 변경 이력을 추적하는 방식(로깅, 타임스탬프 등)을 사용합니다.

#### System Level — WAL (Write-Ahead Log)

시스템 장애(Crash)에 대비하는 핵심 메커니즘은 **WAL**입니다.

```
 [ 클라이언트 요청 ]
        ↓
[ 메모리에 변경 반영 ]  ←→  [ WAL에 순차 기록 (비휘발성 저장소, Disk) ]
        ↓
    [ Commit ]
```

1. 변경 사항을 **메모리에서 디스크로 동기화하기 전에**, 먼저 로그를 비휘발성 저장소에 기록
2. 장애 발생 시 로그를 재실행(Redo)하여 Commit된 Transaction 복구
3. 미완료 Transaction은 로그를 통해 Undo하여 데이터베이스 상태에 영향을 주지 않음

> WAL에 대한 자세한 내용은 [write-ahead-log.md](write-ahead-log.md) 참고

#### Media Level — Replication & Stable Storage

디스크 자체가 손상되는 경우에 대비하여, **안정 저장소(Stable Storage)** 를 통해 보장합니다.

- **디스크 미러링 (RAID)**: 동일 데이터를 여러 디스크에 복제
- **백업 & 복구**: 오프라인 복사본을 통한 재구성
- **체크포인트 + 증분 덤프**: 전체 로그를 처음부터 재실행하지 않고, 체크포인트 이후의 로그만 재실행

> 체크포인트에 대한 자세한 내용은 [checkpointing-log_truncation-compaction.md](checkpointing-log_truncation-compaction.md) 참고

### 2.3 분산 데이터베이스에서의 Durability

분산 환경에서는 단일 노드만으로 Commit 여부를 결정할 수 없습니다. Transaction에 사용된 리소스가 여러 노드에 분산되어 있기 때문입니다.

- **2PC (Two-Phase Commit)**: 모든 참여 노드가 Commit에 동의해야만 최종 Commit
- **ARIES 알고리즘**: 분산 환경에서의 복구 및 격리를 보장하는 알고리즘 계열

## **3. Prometheus TSDB에서의 Durability**

Prometheus TSDB는 전통적인 RDBMS가 아니므로 ACID Transaction을 지원하지 않지만, **시계열 데이터의 Durability**을 보장하기 위해 동일한 원리를 적용합니다.

### 3.1 장애 수준별 대응

| 장애 수준 | RDBMS | Prometheus TSDB |
|-----------|-------|-----------------|
| Transaction | Rollback + Serializability | 해당 없음 (Transaction 개념 없음, append-only) |
| System | WAL → Redo | WAL → Replay |
| Media | RAID, 백업, Replication | Remote Write, Thanos/Cortex 등 외부 장기 저장소 |

### 3.2 WAL을 통한 System Level Durability

Prometheus는 수집된 메트릭을 **Head Block(메모리)** 에 저장하면서 동시에 **WAL에 기록**합니다.

```
  [ scrape된 메트릭 ]
         ↓
[ Head Block (Memory) ]  ←→  [ WAL 세그먼트 파일 (Disk) ]
         ↓                              ↓
 [ Persistent Block ]        [ Checkpoint + Truncate ]
```

소스 코드(`wlog.go`)에서 확인할 수 있는 구현 디테일:

- **세그먼트**: 기본 **128MB** 단위 파일 (`DefaultSegmentSize`)
- **페이지**: 세그먼트 내부는 **32KB** 페이지 단위로 기록 (`pageSize`)
- **레코드 분할**: 큰 레코드는 페이지 경계를 넘어 분할(`recFirst` → `recMiddle` → `recLast`)되지만, **세그먼트 경계는 넘지 않음** — 이를 통해 세그먼트 단위의 안전한 truncation 보장
- **CRC32 Castagnoli 체크섬**: 레코드 헤더에 포함되어 데이터 무결성 검증
- **fsync**: 세그먼트 파일을 디스크에 동기화하여 OS 버퍼 캐시에만 머무르지 않도록 보장
- **torn write 처리**: 마지막 페이지가 불완전하게 기록된 경우 zero-fill로 패딩 후 정상 복구

```go
// wlog.go:172-181 — WL 구조체 핵심 주석
// Segments are written to in pages of 32KB, with records possibly split
// across page boundaries.
// Records are never split across segments to allow full segments to be
// safely truncated. It also ensures that torn writes never corrupt records
// beyond the most recent segment.
```

### 3.3 WAL Record 타입

WAL에 기록되는 레코드 타입(`record.go`)은 다음과 같습니다:

| Type ID | 이름 | 설명 |
|---------|------|------|
| 1 | Series | 시리즈 ID ↔ 라벨셋 매핑 |
| 2 | Samples | float 타임스탬프/값 쌍 |
| 3 | Tombstones | 삭제 마커 (시간 구간 기반) |
| 4 | Exemplars | trace 연결용 예시 샘플 |
| 5 | MmapMarkers | mmap 완료 표시 |
| 6 | Metadata | 시리즈 메타데이터 (타입, 단위, 설명) |
| 7-10 | Histogram 변형들 | 다양한 히스토그램 타입 |

### 3.4 Checkpoint을 통한 복구 시간 단축

WAL만으로는 재시작 시 **모든 세그먼트를 처음부터 replay** 해야 합니다. Checkpoint은 이 문제를 해결합니다.

`checkpoint.go`의 `Checkpoint()` 함수 동작:

1. WAL 세그먼트 `[from, to]` 범위를 읽음 (이전 checkpoint 포함)
2. **불필요한 데이터 필터링**:
   - `keep(id)` 함수로 이미 사라진 Series 드롭
   - `mint` 이전의 Samples, Tombstones, Exemplars 드롭
   - Metadata는 시리즈별 **최신 것만** 유지
3. **Atomic write**: `.tmp` 디렉터리에 먼저 기록 → `fsync` → `rename`으로 원자적 교체
4. 이후 해당 구간의 WAL 세그먼트를 `Truncate()`로 삭제

```go
// checkpoint.go:126-131 — atomic write 패턴
cpdir := checkpointDir(w.Dir(), to)
cpdirtmp := cpdir + ".tmp"
// ... .tmp에 기록 후 ...
if err := fileutil.Replace(cpdirtmp, cpdir); err != nil {
    return nil, fmt.Errorf("rename checkpoint directory: %w", err)
}
```

이 패턴은 RDBMS의 체크포인트와 동일한 원리입니다:
- **체크포인트 이전**: 이미 안전하게 저장 → WAL 삭제 가능
- **체크포인트 이후**: WAL replay 필요

### 3.5 Tombstone — 삭제의 Durability

Prometheus에서 데이터 삭제는 즉시 물리적 삭제가 아닌 **Tombstone(삭제 마커)** 방식으로 처리됩니다.

```go
// record.go — Tombstones 타입 (Type = 3)
// tombstones.Stone: 시리즈의 특정 시간 구간(Intervals)에 대한 삭제 마커
```

- 삭제 요청 시 WAL에 Tombstone 레코드를 기록
- 쿼리 시 Tombstone 구간의 데이터를 필터링하여 제외
- **Compaction** 단계에서 Tombstone에 해당하는 데이터를 실제로 물리 삭제

이 방식의 장점:
- 삭제 자체도 WAL에 기록되므로 **crash-safe**
- 삭제 연산이 빠름 (마커만 추가, O(1))
- Compaction 시점에 일괄 정리하여 I/O 효율적

## **4. 요약: Durability 보장 계층**

```
┌──────────────────────────────────────────────────┐
│              Media Level Durability              │
│       (RAID, Remote Write, Thanos, 백업 등)        │
├──────────────────────────────────────────────────┤
│             System Level Durability              │
│                                                  │
│  ┌──────────┐    ┌────────────┐    ┌──────────┐  │
│  │   WAL    │───→│ Checkpoint │───→│ Truncate │  │
│  │ (순차기록) │    │ (상태 압축)  │    │ (로그삭제) │   │
│  └──────────┘    └────────────┘    └──────────┘  │
│        ↓                                         │
│  ┌──────────┐    ┌────────────┐                  │
│  │  Replay  │    │ Compaction │ ← Tombstone 처리  │
│  │ (장애복구) │    │ (블록 병합)  │                  │
│  └──────────┘    └────────────┘                  │
├──────────────────────────────────────────────────┤
│           Transaction Level Durability           │
│     (Prometheus: append-only, Transaction 없음)   │
└──────────────────────────────────────────────────┘
```
