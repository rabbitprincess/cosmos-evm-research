# Cosmos EVM 분석

## 구조

### server
    실행 방식
        server CLI를 통해 서버 실행
    동작 모드
        StandAlone: 독립 실행형 EVM 노드
        InProcess: Tendermint 노드 내장 모드, 내부 초기화된 Tendermint 노드와 - EVM 모듈 간 ABCI in-process 통신

### Ante

### Precompiles
    Precompile 관리: 0x000... 주소에 매핑되는 Cosmos 모듈 함수 제공 가능

### Keeper

### Cosmos sdk (x)

#### x/vm
    내장 EVM 실행 엔진
    IAVL 트리 기반 상태 저장소
    Keeper를 StateDB 인터페이스로 래핑해 EVM 호환성 제공

#### x/feemarket

#### x/erc20

#### x/precisebank

#### x/ibc


## 트랜잭션 처리 Flow

### 트랜잭션 수신

### Mempool 검증:
- 기본 AnteHandler 검증 통과
- ReCheckTx 단계에서 유효성 재확인

### 블록 포함: Tendermint 합의로 블록에 포함

### EVM 실행 준비:
- Context 생성 (block height, timestamp 등)
- StateDB 초기화 (IAVL 스냅샷)

### EVM 실행

### 상태 커밋 및 이벤트 발생

### 결과 반환

## go-ethereum (geth) 와의 비교
### State 저장소
- MPT ( Merkle Patricia Trie ) / IAVL ( Immutable AVL Tree )
### Commit 전략
- Tx-by-Tx 단위 
### EVM 실행
- 바이트코드 인터프리터는 동일 / Precompile
### Mempool 

## Cosmos EVM 개선안

### 의존성 리팩토링
현재 evmd 와 evm 모듈이 서로를 require 하고 있어, 순환 참조 발생
-> go 의 빌드 및 테스트 속도를 저하시키며, gopls, gofmt 등의 포맷팅 도구에서도 문제 발생 가능.
또한 빌드 시 의도치 않은 패키지 순환 참조 오류 발생 가능.

이 문제를 해결하기 위해, 아래 두 방법을 고려할 수 있습니다:
1. 단방향 의존성으로 변경 (권장)
evm -> evmd 에의 의존성 삭제하고, 양방향 구조를 evmd -> evm 으로 단순화.
2. 모듈 구조 단순화
evmd 의 go.mod 를 삭제하고 단일 모듈로 test, ante, config, cmd 등의 패키지를 evm 으로 공통화해 단일 모듈로 관리.

### IVAL MPT 전환
현재 Cosmos EVM은 IAVL을 사용하고 있지만, MPT로 전환하는 것이 성능 및 호환성 측면에서 유리할 수 있습니다.