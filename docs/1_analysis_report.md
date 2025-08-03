# Cosmos EVM 분석

## Cosmos EVM 구조

1. Server
Server 실행 cli
StandAlone 및 InProcess 모드 제공
InProcess 시 tendermint node 를 내부에 초기화해 evm 모듈과 인라인 통신 (abci inprocess 통신)

2. Cosmos SDK (x)

x/vm
내장 EVM 실행 엔진, 상태 저장소로 IAVL 트리 사용
Keeper 를 자체 StateDB로 말아 evm 의 StateDB interface 와 호환

x/feemarket

x/erc20

x/precisebank

x/ibc


3. Precompiles

Precompile 관리: 0x000... 주소에 매핑되는 Cosmos 모듈 함수 제공 가능


## Cosmos EVM 트랜잭션 처리 Flow

## go-ethereum (geth) 와의 비교
State 저장소 : MPT ( Merkle Patricia Trie ) / IAVL ( Immutable AVL Tree )
Commit 전략 : Tx-by-Tx 단위 
EVM 실행 : 바이트코드 인터프리터는 동일 / Precompile
Mempool : 

## Cosmos EVM 개선안

### 의존성 리팩토링
현재 evmd 와 evm 모듈이 서로를 require 하고 있어, 순환 참조가 발생하고 있습니다.
이는 go 의 빌드 및 테스트 속도를 저하시키며, gopls, gofmt 등의 포맷팅 도구에서도 문제를 일으킵니다.
또한 빌드 시 의도치 않은 패키지 순환 참조 오류가 발생할 수 있습니다.

이 문제를 해결하기 위해, 아래 두 방법을 고려할 수 있습니다:
1. 단방향 의존성으로 변경 (권장)
evm -> evmd 에의 의존성 삭제하고, 양방향 구조를 evmd -> evm 으로 단순화시킵니다.

2. 모듈 구조 단순화
evmd 의 go.mod 를 삭제하고 단일 모듈로 test, ante, config, cmd 등의 패키지를 evm 으로 공통화해 단일 모듈로 관리합니다.

### IVAL MPT 전환
현재 Cosmos EVM은 IAVL을 사용하고 있지만, MPT로 전환하는 것이 성능 및 호환성 측면에서 유리할 수 있습니다. MPT는 EVM의 최적화 및 발전에 기여하고 있으며, IAVL에 비해 더 나은 성능을 제공할 수 있습니다.

전환을 위해 다음과 같은 단계를 고려할 수 있습니다:
1. MP