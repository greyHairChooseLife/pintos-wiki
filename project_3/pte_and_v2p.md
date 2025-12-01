> [!nt]
>
> - VPN stands for Virtual Page Number
> - PFN stands for Physical Frame Number


## 1. 주소 변환 개요 (VA to PA)

x86-64 가상 주소(Virtual Address, 64비트 중 하위 48비트 사용)는 다음과 같이 쪼개져서 해석됩니다. 질문하신 **Offset**은 PTE 안에 있는 것이 아니라, **가상 주소 자체의 하위 비트**에 존재합니다.

*   **Virtual Address (48 bits):**
    *   **PML4 Index (9 bits):** Level 4 테이블 인덱스
    *   **PDP Index (9 bits):** Level 3 테이블 인덱스
    *   **PD Index (9 bits):** Level 2 테이블 인덱스
    *   **PT Index (9 bits):** Level 1 테이블 인덱스 (여기서 최종 PTE를 찾음)
    *   **Page Offset (12 bits):** 4KB 페이지 내에서의 위치

PTE는 "이 페이지가 물리 메모리 어디(Base Address)에 있는가"를 알려주고, **Offset**은 그 시작점에서 얼마나 떨어져 있는지를 나타내므로 변환 없이 그대로 물리 주소의 하위 비트로 붙습니다.


## 2. Page Table Entry (PTE) 구조 (64 bits)

PTE는 64비트(8바이트) 크기이며, 각 비트는 다음과 같은 역할을 합니다.

| 비트 위치 | 이름 | 설명 |
| :--- | :--- | :--- |
| **0 (P)** | **Present** | 1이면 물리 메모리에 존재함. 0이면 Page Fault 발생 (Swap 처리 등). |
| **1 (R/W)** | **Read/Write** | 1이면 쓰기 가능, 0이면 읽기 전용. |
| **2 (U/S)** | **User/Supervisor** | 1이면 유저 모드 접근 가능, 0이면 커널만 접근 가능. |
| **3 (PWT)** | Write-Through | 캐시 정책 (Write-Through vs Write-Back). |
| **4 (PCD)** | Cache Disable | 1이면 해당 페이지 캐싱 안 함 (MMIO 등에 사용). |
| **5 (A)** | **Accessed** | CPU가 이 페이지를 읽거나 썼을 때 1로 세팅 (OS의 페이지 교체 알고리즘에 사용). |
| **6 (D)** | **Dirty** | 페이지에 쓰기가 발생했을 때 1로 세팅 (Swap out 시 디스크 저장 필요 여부 판단). |
| **7 (PAT)** | Page Attribute Table | 메모리 타입 설정 (캐싱 등). |
| **8 (G)** | Global | TLB 플러시 시 유지됨 (커널 영역 등 자주 쓰이는 페이지). |
| **9-11** | Ignored | OS가 임의로 사용 가능. |
| **12-51** | **Physical Address** | 실제 물리 페이지 프레임 번호 (PFN). 하위 12비트는 0으로 가정(4KB 정렬). |
| **52-62** | Ignored | OS가 임의로 사용 가능. |
| **63 (XD)** | **Execute Disable** | 1이면 해당 페이지에서 코드 실행 불가 (NX bit, 보안 기능). |

- 참고: VPN(가상 주소 인덱스)는 36비트지만, 물리 주소 공간은 아키텍처상 최대 52비트까지 지원하므로 PFN은 40비트가 할당됨


## 3. 요약 및 흐름

1.  CPU가 가상 주소에 접근합니다.
2.  MMU(Memory Management Unit)가 가상 주소를 9비트씩 잘라 4단계 테이블을 타고 내려갑니다.
3.  마지막 Page Table에서 **PTE**를 읽습니다.
4.  **PTE의 P 비트**를 확인하여 메모리에 없으면 Page Fault를 일으킵니다.
    - **P == 0:** 해당 페이지가 물리 메모리에 존재하지 않음(Swap out 되었거나 아직 할당되지 않음).
5.  **R/W, U/S, XD 비트**를 확인하여 권한 위반 시 Protection Fault를 일으킵니다.
    * 현재 실행 모드(CPL)와 접근 유형(Read/Write/Execute)이 PTE의 설정(R/W, U/S, NX)과 일치하는지 검사합니다.
6.  권한이 맞다면, **PTE의 Physical Address(12~51비트)** 를 가져옵니다.
7.  가상 주소의 **Offset(0~11비트)** 을 그대로 가져와 물리 주소 뒤에 붙여 최종 물리 주소를 완성합니다.


## C 언어 구조체 예시 (Linux 커널 스타일)

이해를 돕기 위해 비트 필드로 표현한 가상의 구조체입니다.

````c
typedef struct {
    uint64_t present        : 1;  // [0]
    uint64_t rw             : 1;  // [1]
    uint64_t user_supervisor: 1;  // [2]
    uint64_t pwt            : 1;  // [3]
    uint64_t pcd            : 1;  // [4]
    uint64_t accessed       : 1;  // [5]
    uint64_t dirty          : 1;  // [6]
    uint64_t pat            : 1;  // [7]
    uint64_t global         : 1;  // [8]
    uint64_t ignored_1      : 3;  // [9-11]
    uint64_t pfn            : 40; // [12-51] Physical Frame Number
    uint64_t ignored_2      : 11; // [52-62]
    uint64_t nx             : 1;  // [63] No-Execute
} pte_t;
````
