# 생각 정리

### 1. 실행 모드
    inprocess 시 프로세스 내부 통신, 통신 오버헤드 거의 없음
    standalone 은 프로세스 간 통신 ( 거의 안쓸듯 )

### 2. EVM 실행 옵션
    1. Permissioned / Restricted EVM
    2. EVM Extensions
    3. Single Token Representation v2 & ERC-20 Module
    4. EIP-1559 Fee Market Mechanism
    5. JSON-RPC Server
    6. EIP-712 Signing
    7. Custom Improvement Proposals (Opcodes)

### 3. EVM 에 IAVL을 쓰는하는 이유?
    텐더민트 체인과 호환되니까, 근데 실행 레이어까지 호환성이 필수일까?
    Cosmos의 핵심은 합의 + IBC, 실행 레이어는 제약 없음
    무조건적인 MPT 대체 주장보다는, 둘의 장단점을 잘 비교해야 함

### 4. IAVL 의 문제점?
    SLOAD 느릴 가능성 (MPT 처럼 별도 캐시 없으면 병목)
    getproof 등 표준 호환성 부족
    EVM은 MPT 기반으로 최적화 및 발전 중 (ZK, Verkle Tree)
    MPT 대비 성능 열세일 가능성 존재 (측정 필요)

### 5. MPT 로 바꿨을때 생길 수 있는 이슈들
    MPT State tree 를 DB 에 추가로 만들어야 함 -> 상태값 이중화
    IAVL에는 블록·TX만, Account/Contract/Storage/Governance는 MPT로 전환한다면?
    합의 레이어만 IAVL, 실행 레이어는 MPT로 전환 가능성 검토
    Keeper 및 AppHash, Governance 쪽 확인 필요

### 6. 추가 아이디어
    MPT 도입 시 Lazy Proof / Flat Storage 적용 가능성 (리서치 필요)

### 7. 다른 체인 사례
    sei → 병렬 EVM