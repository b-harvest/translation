<!--
order: 0
-->

# List of Modules

다음은 각각의 문서와 함께 Cosmos SDK 애플리케이션에서 사용할 수 있는 일부 프로덕션 등급 모듈입니다.

* [Auth](auth/spec/README.md) - Cosmos SDK 애플리케이션에 대한 계정 및 트랜잭션 인증.
* [Authz](authz/spec/README.md) - 계정이 다른 계정을 대신하여 작업을 수행할 수 있는 권한 부여
* [Bank](bank/spec/README.md) - 토큰 전송 기능.
* [Capability](capability/spec/README.md) - 개체 기능 구현.
* [Crisis](crisis/spec/README.md) - 특정 상황에서 블록체인 정지(예: invariant가 손상된 경우).
* [Distribution](distribution/spec/README.md) - 수수료 분배 및 스테이킹 토큰 제공 분배.
* [Epoching](epoching/spec/README.md) - 모듈이 특정 블록 높이에서 실행할 메시지를 큐에 넣을 수 있도록 합니다.
* [Evidence](evidence/spec/README.md) - 이중 서명, 부정 행위 등에 대한 증거 처리
* [Feegrant](feegrant/spec/README.md) - 거래 실행에 대한 수수료 수당을 부여합니다.
* [Governance](gov/spec/README.md) - 온체인 제안 및 투표.
* [Mint](mint/spec/README.md) - 스테이킹 토큰의 새로운 단위 생성.
* [Params](params/spec/README.md) - 전역에서 사용 가능한 매개변수 저장소.
* [Slashing](slashing/spec/README.md) - 검증인 처벌 메커니즘.
* [Staking](staking/spec/README.md) - 공개 블록체인을 위한 지분 증명(Proof of Stake) 계층
* [Upgrade](upgrade/spec/README.md) - 소프트웨어 업그레이드 처리 및 조정.
* [Nft](nft/spec/README.md) - [ADR43](https://docs.cosmos.network/master/architecture/adr-043-nft-module.html)을 기반으로 구현된 NFT 모듈.

모듈 빌드 프로세스에 대해 자세히 알아보려면 [모듈 빌드 참조 문서](../docs/building-modules/README.md)를 방문하세요.

## IBC

SDK용 IBC 모듈이 [자체 저장소](https://github.com/cosmos/ibc-go)로 이동되었습니다.

## CosmWasm

CosmWasm 모듈은 스마트 계약을 가능하게 하며, [자체 저장소](https://github.com/CosmWasm/cosmwasm) 및 [문서 사이트](https://docs.cosmwasm.com/docs/1.0)가 있습니다.