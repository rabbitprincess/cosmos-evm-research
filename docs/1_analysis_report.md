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
    1. 트랜잭션 수신

    2. Mempool 검증
        - 기본 AnteHandler 검증 통과
        - ReCheckTx 단계에서 유효성 재확인

    3. 블록 포함
        Tendermint 합의로 블록에 포함

    4. EVM 실행 준비:
        Context 생성 (block height, timestamp 등)
        StateDB 초기화 (IAVL 스냅샷)

    5. EVM 실행

    6. 상태 커밋 및 이벤트 발생

    7. 결과 반환

## go-ethereum (geth) 와의 비교
    1. State 저장소
        MPT ( Merkle Patricia Trie ) / IAVL ( Immutable AVL Tree )
    2. Commit 전략
        Tx-by-Tx 단위 / Block 단위
    3. EVM 실행
        바이트코드 인터프리터는 동일

    4. Mempool 

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