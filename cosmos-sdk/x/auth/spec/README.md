# `auth`

## Abstract

이 문서는 Cosmos SDK의 인증 모듈을 지정합니다.

SDK 자체는 이러한 세부 사항에 애그노스틱하기 때문에 auth 모듈은 애플리케이션에 대한 기본 트랜잭션 및 계정 타입을 지정하는 역할을 합니다. 여기에는 모든 기본 트랜잭션 유효성 검사(시그니처, 논스, 보조 필드들)가 수행되는 미들웨어가 포함되어 있으며 다른 모듈이 계정을 읽고, 쓰고, 수정할 수 있도록 하는 계정 키퍼(account keeper)를 노출합니다.

이 모듈은 코스모스 허브에서 사용됩니다.

## Contents

1. **[Concepts](01_concepts.md)**
   * [Gas & Fees](01_concepts.md#gas-&-fees)
2. **[State](02_state.md)**
   * [Accounts](02_state.md#accounts)
3. **[AnteHandlers](03_antehandlers.md)**
4. **[Keepers](04_keepers.md)**
   * [Account Keeper](04_keepers.md#account-keeper)
5. **[Vesting](05_vesting.md)**
   * [Intro and Requirements](05_vesting.md#intro-and-requirements)
   * [Vesting Account Types](05_vesting.md#vesting-account-types)
   * [Vesting Account Specification](05_vesting.md#vesting-account-specification)
   * [Keepers & Handlers](05_vesting.md#keepers-&-handlers)
   * [Genesis Initialization](05_vesting.md#genesis-initialization)
   * [Examples](05_vesting.md#examples)
   * [Glossary](05_vesting.md#glossary)
6. **[Parameters](06_params.md)**
7. **[Client](07_client.md)**
   * **[Auth](07_client.md#auth)**
      * [CLI](07_client.md#cli)
      * [gRPC](07_client.md#grpc)
      * [REST](07_client.md#rest)
   * **[Vesting](07_client.md#vesting)**
      * [CLI](07_client.md#vesting#cli)
