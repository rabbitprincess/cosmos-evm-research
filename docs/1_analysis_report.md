# Cosmos EVM 분석

## 구조
    cosmos evm 은 2가지 계층으로 이루어짐
        1. 합의 계층 : 블록 생성, pbft 합의, 네트워크 통신 담당
        2. 어플리케이션 계층 : Cosmos SDK 및 EVM 기반 모듈 실행

    주요 컴포넌트 목록
        Server : StandAlone / InProcess 모드 실행
        Ante : 트랜잭션 사전 처리기
        PreCompiles : EVM 내에서 호출하는 코스모스 모듈
        Keeper : 모듈 간 상태 접근, 일반적으로 IAVL 기반 저장소 사용
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

    EVM 실행
        ctx 로 EVMConfig, TxConfig 생성
        vm.State 호환되는 자체 StateDB로 vm.EVM 생성
        evm.Create 또는 evm.Call 을 호출해 실행
        
    상태 커밋 및 이벤트 발생
        EVM 실행 결과를 stateDB 에서 commit
        수수료 처리 및 이벤트, receipt 반환

## go-ethereum (geth) 와의 비교
    State 저장소
        MPT ( Merkle Patricia Trie ) / IAVL ( Immutable AVL Tree )
    EVM 실행
        바이트코드 인터프리터는 동일

    Mempool 

## 추가 질문 및 답변

### 현재 cosmos-evm에서는 evm tx를 통해 module을 조작하기 위해 stateful precompiled contract를 지원하고 있습니다. 다만, 여기서 구조적 복잡함 및 결함으로 인해서 지속적으로 보안적 이슈가 나오고 있는데요, 어떤 디자인적 단점을 가지고 있고 이것을 개선할 수 있는 디자인이 있을까요?

    Cosmos EVM의 Precompiled Contract는 EVM의 PrecompiledContract 인터페이스와 호환되지만,
    Run 함수 내부에서 Keeper 를 통해 상태 트리를 초기화하고 관리하는 로직(RunSetup)을 포함하고 있습니다.
    이 설계는 다음과 같은 문제를 유발할 수 있습니다.

    1. 수수료 모델 불일치
    EVM의 핵심 원칙 중 하나는 모든 연산이 결정적(deterministic)이며, 예측 가능한 Gas 모델을 사용한다는 점에 있습니다.
    이는 사용자가 내는 수수료와 노드가 수행하는 연산 비율을 일치시켜 네트워크의 안정성을 유지하기 위함입니다.
    하지만 코스모스의 Precompile 은 이 특성과 충돌합니다.
    Precompile 모듈은 staking, slashing, distribution 등 복잡하고 상태 의존적인 로직을 포함하고 있습니다.
    문제는 이런 로직이 결정론적 실행을 보장하기 어렵다는 점에 있습니다.
    상태 접근 과정에서 동일한 로직이라도 IAVL 트리의 현재 상태에 따라 밸런스 계산 및 보정에 필요한 연산량이 크게 달라질 수 있어,
    사용자가 지불한 가스와 실제 검증인이 수행해야 하는 연산 비용 사이에 괴리가 발생합니다.
    이는 수수료 모델의 허점을 노린 DoS 공격에 취약해질 수 있습니다.

    2. 상태값 동기화 문제
    Precompile 은 Cosmos Keeper 가 서로 통신하며 각자의 상태를 갱신하는 방식으로 작동합니다.
    문제점은 이 Keeper 에 복잡한 EVM State 가 추가되면서 원자적 상태 일관성을 유지하는 것이 매우 까다로워진다는 점입니다.
    특히 통신 또는 내부 연산 문제로 트랜잭션 롤백이 필요한 경우 StateDB 를 포함한 모든 Keeper 가 처리된 모든 과정을 역순으로 되돌려야 하며, 그 과정에서 상태 불일치가 일어나기 쉽습니다.
    
    3. 성능 문제
    동일한 상태를 다루더라도 각 Keeper 마다 별도로 상태 조회 및 변경을 처리하기 떄문에, IAVL 접근이 불필요하게 중복됩니다.
    이는 StateDB 가 추가된 EVM 환경에서 더욱 치명적인 특성입니다. 여러 Keeper가 IAVL 트리에 동시에 접근하면서 발생하는 연산 부담과 가비지 컬렉션(GC) 부하가 누적되고,
    인터페이스 기반 접근으로 인해 캐싱 및 최적화에도 부정적인 영향을 미칩니다.
    특히 Staking Reward 지급과 같이 여러 계정의 상태를 반복적으로 갱신해야 하는 경우에는
    루프 기반 IAVL 접근이 대량으로 발생하여 심각한 성능 저하를 유발할 수 있습니다.


### cosmos-evm의 mempool 및 ante handler의 구조로 인해 evm tx의 UX가 실제 evm환경과는 다른 부분들이 있습니다. 어떤 부분들에서 다른지? UX를 개선할 수 있는 포인트들이 있을지?

    evm 의 처리 과정은 ~ 입니다
    그 반면에 mempool 과 ante handler 의 처리 과정은 아래와 같습니다.

    이 차이를 해결하기 위해서는 


### cosmos-evm은 cosmos-sdk위에서 개발되었기 때문에 기본적인 철학이 모두 sdk에 근간을 두고있습니다. 이로 인한 성능면에서의 아쉬움이 큰데요, tps를 어떻게 끌어올릴 수 있을지?

    tps 에 영향을 미치는 요소에 대해 




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