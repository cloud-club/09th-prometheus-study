# 목차
- WAL 순차 쓰기 I/O 관찰
- Compression 관찰
- Checksum 손상 탐지
- Checkpoint GC 관찰

## 세팅
- aws위에 Docker로 구축했습니다.
- 메모리의 데이터를 디스크의 block으로 전환하는 'duration time'은 5분으로 세팅했습니다.
- 디스크에 저장된 블록을 얼마나 보관할지 결정하는 'retention time'은 10분으로 세팅했습니다.


## WAL 순차 쓰기 I/O 관찰
<img width="1440" height="549" alt="image" src="https://github.com/user-attachments/assets/16f40219-b078-4c77-92f8-6f29c3295690" />

- 프로메테우스 프로세스가 1초당 평균적으로 약 1.6ms ~ 2.4ms의 CPU 시간을 사용하는 것을 알 수 있습니다.
- 주기적으로 평균치를 상회하는 그래프 모양이 나오는 곳이 있습니다.
  - 5분마다 duration time이 적용된것이 나타난 것입니다.
- 평균 적으로 증가한 부분보다 그래프가 더 증가한 곳은 stress-ng의 반응이 나타난 것입니다.
  - 현재 지표는 프로메테우스 프로세스만을 가리키고 있습니다만,
  - CPU 스케줄링 경합: OS가 stress-ng를 위해 CPU를 제공하면, Context Switching에 의해 해당 프로세스 부하 역시 더 늘어난다는 것을 확인할 수 있습니다.
- 10분단위로 retention이 발생하는데, 삭제는(os.RemoveAll) CPU에 많은 영향을 주지 않는다는 것을 다시 한번 확인할 수 있었습니다.

## Compression 관찰
- WAL의 상태를 통해 압축한 상태와 압축하지 않은 상태를 비교해봤습니다.
<img width="749" height="102" alt="image" src="https://github.com/user-attachments/assets/21b138b3-5c8f-46c9-baf0-dabc267dbf92" />

시간이 지남에 따라 압축 폴더는 천천히 증가하지만, 압축하지 않은 폴더는 압축한 폴더보다 빠르게 증가하는 것을 확인할 수 있습니다.

<img width="336" height="77" alt="image" src="https://github.com/user-attachments/assets/56661cef-1c47-4ea2-99f9-fc41fde47314" />

<img width="304" height="74" alt="image" src="https://github.com/user-attachments/assets/eef77d64-2ae0-4465-912e-170af7e9c0cf" />

완전히 역전한 모습도 확인할 수 있었습니다.

### Snappy 알고리즘 
- Snappy는 압축률은 조금 낮더라도 CPU를 아주 적게 쓰면서 빠르게 압축과 해제를 반복합니다.
- Snappy는 숫자나 타임스탬프가 반복되는 패턴(Byte sequence)을 찾아내서 작업을 효율적으로 수행합니다.


## Checksum 손상 탐지
- 먼저 WAL 세그먼트를 확인해봤습니다.
<img width="610" height="151" alt="image" src="https://github.com/user-attachments/assets/ddfe2a6a-4bd5-49f5-9afd-0217359c7dc7" />

- checkpoint.00000029 폴더
  - WAL 세그먼트 0~29번을 순회하며, Head에 아직 살아있는 시리즈 레코드와 아직 Compaction 안 된 샘플만 추려서 만든 필터링된 WAL 입니다.
  - 이 체크포인트가 완성된 후 원본 세그먼트 0~29는 삭제됩니다.


그리고 가장 최근 파일을 한번 열어보겠습니다. (xxd를 통해 16진수로 변환해서 확인을 합니다.)
<img width="622" height="359" alt="image" src="https://github.com/user-attachments/assets/f4a31f21-3b85-4314-8aea-b56406a34297" />

<img width="508" height="119" alt="image" src="https://github.com/user-attachments/assets/1989120d-5cc6-4319-9dc9-b55c5a91bfcf" />

- r+b 모드: Read + Write를 Binary 모드로 열겠다는 뜻입니다.
- f.seek(500): 파일의 맨 처음부터 500바이트 지점으로 커서를 이동한다는 뜻입니다.

- 해당 파일에 변경을 하겠습니다.
<img width="1156" height="109" alt="image" src="https://github.com/user-attachments/assets/8fcb047c-0dcd-4784-aa19-1770f58cf122" />

- unexpected checksum 8ca0c029, expected 7e1f3ebc: 기존의 checksum결과와 다른 값이 나왔다는 뜻입니다.
- Encountered WAL read error, attempting repair: 자동 복구를 시작하겠다고 알리고 있습니다.
- Deleting all segments newer than corrupted segment: 30번 파일이 깨졌기에 그 뒤로 나온 31번 32번 등... 다 날려서 데이터 정합성을 보장합니다.

<img width="851" height="51" alt="image" src="https://github.com/user-attachments/assets/9c742c23-f3f4-44be-acb3-4e6db2d0f90f" />

- 이후 복구가 성공적이었다는 것을 확인할 수 있습니다.

## Checkpoint GC 관찰
- 메모리 GC (Head GC started)
  - 프로메테우스는 최근 데이터를 메모리에 들고있다가, 메모리에 있던 데이터를 HDD로 이동시키는 것을 말합니다.
  - 이동이 완료된 후 Head GC를 통해 메모리를 정리하게 됩니다.

- 디스크 GC (Creating checkpoint & Deleting segments)
  - WAL 파일은 임시 기록 입니다. 이미 디스크 블록으로 저장이 끝났다면, 이 임시 기록들은 용량만 차지하기에 정리가 필요합니다.
  - checkpoint를 만들어 현재 상태를 요약한 뒤, 그보다 오래된 WAL 파일들을 삭제(Truncate)합니다.

 <img width="2068" height="380" alt="image" src="https://github.com/user-attachments/assets/5aee2d35-df99-4b58-a8ca-59d62c8d9c30" />

- Compaction: Block 생성
  - write block started → write block completed
  - block 파일로 디스크에 저장된 것을 확인할 수 있습니다.
- Head GC
  - Head GC started → Head GC completed
  - 메모리에 데이터를 지우는 것을 확인할 수 있습니다.
- Checkpoint 생성
  - Creating checkpoint → WAL checkpoint complete
  - WAL(Write Ahead Log) 파일 중, 이미 디스크 블록으로 변환된 segment 30과 31을 정리하고 있습니다.
  - 체크포인트에 저장하는 것은 '시계열 메타데이터'와 'last timestamp 데이터'입니다.




