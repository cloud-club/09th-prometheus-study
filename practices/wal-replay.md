| 메트릭 | 의미 |
| --- | --- |
| `prometheus_tsdb_wal_segments_current` | 현재 활성 세그먼트 번호 |
| `prometheus_tsdb_checkpoint_creations_total` | checkpoint 생성 누적 횟수 |
| `prometheus_tsdb_checkpoint_deletions_total` | 오래된 checkpoint 삭제 횟수 |
| `prometheus_tsdb_wal_truncations_total` | WAL truncation 발생 횟수 |
| `prometheus_tsdb_head_truncations_total` | Head truncation 횟수 (WAL truncation과 연동 확인) |

![](https://raw.githubusercontent.com/hvvup/Github-User-Content/main/cc-prom-study/wal/checkpoint.png)

| ![](https://raw.githubusercontent.com/hvvup/Github-User-Content/main/cc-prom-study/wal/head_truncations.png) | ![](https://raw.githubusercontent.com/hvvup/Github-User-Content/main/cc-prom-study/wal/wal_truncations.png) | ![](https://raw.githubusercontent.com/hvvup/Github-User-Content/main/cc-prom-study/wal/tsdb_checkpoint.png) |
|-|-|-|
|`prometheus_tsdb_head_truncations_total`|`prometheus_tsdb_wal_truncations_total`|`prometheus_tsdb_checkpoint_creations_total`|


## 첫 번째 재시작 (02:08:32)

```
Replaying WAL, this may take a while
WAL checkpoint loaded                          ← checkpoint 먼저 읽음
WAL segment loaded  segment=13 maxSegment=17   ← 13부터 시작
WAL segment loaded  segment=14 maxSegment=17
WAL segment loaded  segment=15 maxSegment=17
WAL segment loaded  segment=16 maxSegment=17
WAL segment loaded  segment=17 maxSegment=17
WAL replay completed  total_replay_duration=39.758ms
```

이후 바로 truncation이 연달아 발생:

```
02:08:38  Creating checkpoint  from_segment=13 to_segment=15   ← 재시작 직후 밀린 truncation 처리
02:17:33  Creating checkpoint  from_segment=16 to_segment=17
02:27:33  Creating checkpoint  from_segment=18 to_segment=19
```

---

## 두 번째 재시작 (02:31:52)

```
WAL checkpoint loaded                          ← checkpoint 먼저 읽음
WAL segment loaded  segment=20 maxSegment=23   ← 직전 checkpoint 마지막 번호 +1 부터
WAL segment loaded  segment=21 maxSegment=23
WAL segment loaded  segment=22 maxSegment=23
WAL segment loaded  segment=23 maxSegment=23
WAL replay completed  total_replay_duration=45.812ms
```

그리고 강제 종료였기 때문에 이 경고가 찍혔다:

```
WARN  A lockfile from a previous execution already existed. It was replaced
```

`docker kill`로 죽었기 때문에 lock 파일이 정리 안 된 채로 남아 있었던 것.

---

## 정리

| 이론 | 실제 로그 |
| --- | --- |
| checkpoint를 먼저 읽는다 | `WAL checkpoint loaded` 가 segment보다 먼저 출력 |
| checkpoint.X의 X+1부터 이어서 읽는다 | 1차: checkpoint 이후 segment=13부터 시작, 2차: segment=20부터 시작 |
| WAL truncation은 Head truncation 직후 발생 | `Head GC completed` → `Creating checkpoint` 순서로 연달아 출력 |
| checkpoint는 항상 최신 1개만 유지 | from=13~15 checkpoint 생성 후, from=16~17 생성 시 이전 것 교체 |
| 강제 종료 후 데이터 복원 | replay 완료 후 `TSDB started` — 유실 없이 정상 기동 |

<br>

두 번의 replay 시간을 비교

```
1차 재시작: total_replay_duration = 39.758ms  (segment 13~17, 5개)
2차 재시작: total_replay_duration = 45.812ms  (segment 20~23, 4개)
```

checkpoint가 있기 때문에 전체 WAL을 처음부터 읽지 않고 필요한 구간만 읽는다. checkpoint 없이 처음부터 읽었다면 시간이 훨씬 길었을 것

## 02:32:30 checkpoint

```
02:31:52  WAL segment loaded  segment=20 maxSegment=23   ← replay 시작점
02:31:52  WAL replay completed

02:32:30  Creating checkpoint  from_segment=20 to_segment=21  ← 재시작 직후 바로 truncation
02:32:30  WAL checkpoint complete  first=20 last=21
```

재시작 전 마지막 checkpoint는 `from=18 to=19`였다. 즉 재시작 시점의 디스크 상태는:

```
checkpoint.000019/   ← 직전 checkpoint
000020
000021
000022
000023
```

replay는 checkpoint.000019를 읽은 뒤 segment=20부터 시작했고, 재시작 후 약 **38초** 만에 segment 20~21을 대상으로 새 checkpoint를 즉시 만들었다.

이건 "재시작 후 밀린 truncation을 즉시 처리"하는 패턴이다. 강제 종료 직전에 truncation이 실행될 타이밍이었는데 죽어버렸고, 재시작 후 조건이 충족되자마자 바로 실행된 것이다.

---

### Replay 관련 전체 타임라인

| 시각 | 이벤트 |
| --- | --- |
| 02:31:52.307 | `Replaying WAL` 시작 |
| 02:31:52.315 | `WAL checkpoint loaded` (checkpoint.000019 읽음) |
| 02:31:52.324~352 | segment 20→21→22→23 순서대로 로드 |
| 02:31:52.352 | `WAL replay completed` — 총 45.8ms |
| 02:31:52.354 | `TSDB started` |
| **02:32:30** | **즉시 checkpoint 생성 (20→21), 디스크 정리** |

replay 자체는 45ms로 끝났고, 그 직후 정상 운영 사이클로 복귀