# 목차 
- Magic Number + Offset 파싱
- mmap + Memory Footprint 관찰
- immutable + GC

err="open /prometheus/queries.active: permission denied"
panic: Unable to create mmap-ed active query log
처음 시작할때 권한 거부 문제가 발생했습니다.
프로메테우스 컨테이너 내부에서는 nobody라는 계정(UID 65534)이 작동합니다. 
그런데 현재 데이터를 저장하려는 폴더(/tmp/prom-data)의 권한은 ubuntu 계정으로 되어있기에 거부현상이 발생한 것입니다.
<img width="563" height="26" alt="image" src="https://github.com/user-attachments/assets/c07ecdde-7bc0-4f08-a2fe-73b0244ce647" />

- -R 옵션은 하위 폴더와 파일까지 싹 다 바꾸라는 뜻입니다.
- 65534는 프로메테우스의 'nobody' 계정 번호입니다.


## Magic Number + Offset 파싱
<img width="574" height="75" alt="image" src="https://github.com/user-attachments/assets/c51b8fb1-7421-4d2c-834a-a484d2098556" />

초기에는 파일 내용이 없는 것을 확인할 수 있습니다.
조금 있다 다시 확인을 해봤습니다.
<img width="521" height="91" alt="image" src="https://github.com/user-attachments/assets/da3bdcbc-c566-45d2-9865-47cecfdd50fe" />

- 000001 파일이 생겼고, 크기가 131072바이트(128KB)인 것을 확인할 수 있습니다. 
- 프로메테우스가 메모리에 있던 데이터를 디스크의 chunks_head로 Flush한 상태입니다.

<img width="602" height="39" alt="image" src="https://github.com/user-attachments/assets/c350b8e2-d975-45d3-ad13-d165e52f4a14" />

- 맨 앞의 00000000은 offset을 나타냅니다.
- 0130 bc91은 Magic Number를 나타냅니다.
- 그 뒤에오는 01은 version(chunk format을 나타냅니다.)
- 00 00 00은 padding을 나타냅니다.
- .0......: 오른쪽 끝은 이진 데이터를 사람이 읽을 수 있는 문자(ASCII)로 변환한 것인데, 바이너리 파일이라 대부분 점(.)으로 표시됩니다.

<img width="627" height="144" alt="image" src="https://github.com/user-attachments/assets/b0b84642-578a-4950-a09d-c9174c58cbba" />

- 0000 0000 0000 0001은 series_ref1을 나타내고 이 청크는 시스템이 관리하는 가장 첫 번째(ID =1)를 나타냅니다.
- 0000 019d 4732 659e은 시작 시간을 나타냅니다.
- 프로메테우스 쿼리는 mint ~ maxt사이에 있는 값이면 chunk를 가져옵니다.

## mmap + Memory Footprint 관찰
mpa을 먼저 확인해보겠습니다.
<img width="835" height="42" alt="image" src="https://github.com/user-attachments/assets/e81081a1-fd00-45ce-965e-424fa9a87eca" />

- f4faf6e00000-f4fafee00000: 가상 메모리 주소의 시작과 끝을 나타냅니다.
- r--s: 읽기 권한과 공유 권한은 있지만 쓰기 권한과 execute권한은 없습니다.
  - immutable을 상태이고, 데이터 파일이니깐 실행 권한은 당연히 없습니다.
- 00000000: 오프셋을 나타냅니다.
- 103:01: 데이터가 저장된 물리적 디스크의 식별 번호입니다.
- 263561: Inode 번호 입니다. 이름은 000001이지만 커널은 이 번호로 파일을 식별합니다.

메모리 사용량을 확인했습니다.

<img width="278" height="58" alt="image" src="https://github.com/user-attachments/assets/6012d058-844f-4d2b-b1aa-78ee9c490975" />
<img width="183" height="69" alt="image" src="https://github.com/user-attachments/assets/1fec5148-28d2-42d8-9e9b-39cd6fb26a76" />

- Vmsize: 약 1.6GB로 프로메테우스가 사용할 메모리 주소 공간을 OS에 선언한 총량을 나타냅니다. (mmap파일들이 있습니다.)
- VmRss: Resident Set Size로 실제 물리 메모리를 사용하는 양을 나타냅니다.
- VmRSS의 하락: 약 25MB 정도의 물리 메모리가 해제되었습니다. 이는 프로메테우스가 메모리의 Heap에 들고 있던 데이터를 Flush한 것을 확인할 수 있습니다. 
- VmSize의 하락: 가상 메모리 주소 공간까지 줄어들어, 프로메테우스가 필요 없는 메모리 매핑이나 힙 공간을 OS에 반납한 것을 확인할 수 있습니다.
  
<img width="788" height="214" alt="image" src="https://github.com/user-attachments/assets/2c1a34e1-ffc9-4375-8007-b1095f9fcaaa" />

- chunk의 생성과 삭제를 확인할 수 있습니다.


## immutable
<img width="619" height="161" alt="image" src="https://github.com/user-attachments/assets/e1abd282-ea88-4d21-8753-1fe3986260fe" />

- 아래에서 결과를 확인할 수 있습니다.

<img width="516" height="52" alt="image" src="https://github.com/user-attachments/assets/efb511d1-d65b-4f74-b82b-183b9ecb7791" />

- MMU에서 Page Table Entry에서 권한을 확인하고, write는 불가하기에 인터럽트가 발생합니다.
- 인터럽트 처리를 위해 커널이 개입을 하고, 해당 프로세스에게 signal을 보내면서 강제 종료됩니다.
- core dumped: 프로그램이 죽기전에 메모리를 기록했다는 뜻입니다.


### CRC 손상도 한번 확인했습니다.
<img width="364" height="104" alt="image" src="https://github.com/user-attachments/assets/6a6884ba-88ae-49d3-90e0-2faaebdf6b41" />

- 데이터를 수정했습니다.
<img width="1306" height="40" alt="image" src="https://github.com/user-attachments/assets/70b5ce53-c1eb-4ccd-a47e-8bdb28fed97e" />

- checksum의 expected와 actual이 달라서 에러가 발생한 것을 확인할 수 있습니다.

<img width="749" height="63" alt="image" src="https://github.com/user-attachments/assets/d2bc4e9a-e7ee-4b40-a7b6-ba4226f0acd5" />

- 복구 메커니즘이 작동하는 것 또한 확인할 수 있습니다.
<img width="630" height="33" alt="image" src="https://github.com/user-attachments/assets/e456f11b-e93e-4590-93e3-8334741d17fb" />

- Durability를 보장한다는 것을 확인할 수 있었습니다.

## Garbage Collection
<img width="1181" height="40" alt="image" src="https://github.com/user-attachments/assets/a86f4b30-d41b-466c-9a9c-1bdaba28810e" />

- 1분마다 GC가 발생하는 것을 확인할 수 있습니다.
- truncateMemory를 통해 RSS 감소의 원인을 확인할 수 있습니다.

