# 방향성
분석 / 개선안 챕터 작성

# 생각들

- 실행시 inprocess / standalone 모드
    - inprocess 시 사실상 내부 통신, 오버헤드 x
    - standalone 은 프로세스 간 통신 ( 거의 안쓸듯 )

- EVM 실행 옵션 있음
    - Permissioned/Restricted EVM
    - EVM Extensions
    - Single Token Representation v2 & ERC-20 Module
    - EIP-1559 Fee Market Mechanism
    - JSON-RPC Server
    - EIP-712 Signing
    - Custom Improvement Proposals (Opcodes)

- EVM 사용 시 MPT 가 아닌 IAVL 사용하는데, 몇가지 문제점?
    - SLOAD 느릴듯 (MPT 처럼 따로 캐싱이 없으면 병목)
    - getproof 등 표준 호환성 못지키는 부분 발생
    - EVM 은 MPT 를 기반으로 최적화 및 발전중 ( ZK, Verkle Tree )
    - MPT 에 비해 기본적으로 느릴수도? ( 측정 필요 )

- 그럼에도 IAVL 을 하는 이유가 있나?
    - 텐더민트 체인과 호환되니까, 근데 실행 레이어까지 호환성이 필수일까?
    - Cosmos의 핵심은 합의 + IBC인데, 이부분은 합의쪽에 가까움
    - 무작정 MPT 대체하려고 하기보다는, 둘의 장단점을 잘 비교해야 함

- MPT 로 바꿨을때 생길 수 있는 이슈들
    - MPT State tree 를 DB 에 추가로 만들어야 함 -> 상태값 이중화
    - IAVL 은 블록, TX 만 넣고 / Account, Contract, Storage, Governence 는 MPT 로 바꿔버리면?
    - 합의 레이어만 IAVL 이고, 실행은 완전히 MPT 에 맡기면 되지 않을까?
    - Keeper 를 MPT 로 바꿀 수 있는지 연구 필요
    - AppHash 를 자유롭게 넣을 수 있는지 확인 필요

- 추가로 MPT 넣으면 lazy proof / flat Storage 개념 넣을 수 있을수도 ( 리서치 필요 )

- 다른 체인들
    - sei -> 병렬 EVM