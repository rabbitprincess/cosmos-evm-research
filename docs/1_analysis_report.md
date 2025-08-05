# Cosmos EVM 분석

## 구조
    cosmos evm 은 2가지 계층으로 이루어짐
        1. 합의 계층 : 블록 생성, pbft 합의, 네트워크 통신 담당
        2. 어플리케이션 계층 : Cosmos SDK 및 EVM 기반 모듈 실행

    주요 컴포넌트 목록
        Server : StandAlone / InProcess 모드 실행
        Ante : 트랜잭션 사전 처리기
        PreCompiles : EVM 내에서 호출하는 코스모스 모듈
        Keeper : 모듈 간 상태 접근, IAVL 기반 저장소 사용
        Cosmos SDK
            erc20 : erc20 토큰 실행 기능 제공
            vm : EVM 호환성 제공, Keeper 를 geth 의 StateDB 로 래핑
            feemarket : eip-1559 호환 모드 제공
            presisebank : 이더리움 소수점 정밀도 제공
            ibc : 체인 간 상호운용성

## 트랜잭션 처리 Flow
    트랜잭션 수신
        DeliverTx
        keeper.EthereumTx
        keeper.ApplyTransaction

    EVM 실행:
        ctx 로 EVMConfig, TxConfig 생성
        vm.State 호환되는 자체 StateDB로 vm.EVM 생성
        evm.Create 또는 evm.Call 을 호출해 실행
        
    상태 커밋 및 이벤트 발생
        EVM 실행 결과를 stateDB 에서 commit
        수수료 처리 및 이벤트 발생

## go-ethereum (geth) 와의 비교
    State 저장소
        MPT ( Merkle Patricia Trie ) / IAVL ( Immutable AVL Tree )
    EVM 실행
        바이트코드 인터프리터는 동일

    Mempool 

## Cosmos EVM 개선안

### 의존성 리팩토링
    현재 evmd 와 evm 모듈이 서로를 require 하고 있어, 순환 참조 발생
    이는 go 의 빌드 및 테스트 속도를 저하시키며, gopls, gofmt 등의 포맷팅 도구에서도 문제 발생 가능.
    또한 빌드 시 의도치 않은 패키지 순환참조 오류 발생 가능.

    이 문제를 해결하기 위해, 아래 두 방법 고려 가능:
    1. 단방향 의존성으로 변경 (권장)
        evm -> evmd 에의 의존성 삭제하고, 양방향 구조를 evmd -> evm 으로 단순화
    2. 모듈 구조 단순화
        evmd 의 go.mod 를 삭제하고 단일 모듈로 test, ante, config, cmd 등의 패키지를 evm 으로 공통화해 단일 모듈로 관리

### IVAL MPT 전환
    현재 Cosmos EVM은 IAVL을 사용하고 있지만, MPT로 전환하는 것이 성능 및 호환성 측면에서 유리할 수 있음

    tx index 단위로 처리 -> 근데 index 는 안날라오는데 어떻게 조회?
    그냥 블록 단위로 처리하면 안될까?
    state transition