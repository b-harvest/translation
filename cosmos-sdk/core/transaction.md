# 트랜잭션(Transactions)

`트랜잭션` 트랜잭션은 최종 사용자가 애플리케이션의 상태 변화를 위해 생성한 객체입니다 {synopsis}

## 선행 학습 자료(Pre-requisite Readings)

* [코스모스 SDK 애플리케이션 분석](../basics/app-anatomy.md) {prereq}

## 트랜잭션(Transactions)

트랜잭션은 [컨텍스트](./context.md)와 [`sdk.Msg`s](../building-modules/messages-and-queries.md)의 메타데이터(metadata)로 구성되어 있으며, 모듈의 Protobuf [`Msg` service](../building-modules/msg-services.md)를 통해 모듈에 상태 변화를 일으킵니다 

사용자가 애플리케이션에 상호작용을 통해 상태 변화를 일으키고자 할때 (예를 들어 코인 전송) 트랜잭션을 생성합니다. 각 트랜잭션의 `sdk.Msg`는 네트워크에 전송하기 전에 해당 계정과 연결된 개인키로 서명을 해야합니다. 이후 트랜잭션은 블록에 포함되어, 합의 과정에 따라 네트워크에서 검증 및 승인됩니다. 트랜잭션의 수명 주기에 대해 자세히 알고 싶으면 [여기](../basics/tx-lifecycle.md)를 클릭하세요


## 타입 정의(Type Definition)

트랜잭션 객체는 코스모스 SDK 타입의 `Tx` 인터페이스로 구현됩니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/types/tx_msg.go#L38-L46

해당 인터페이스는 다음과 같은 메서드를 가집니다:

* **GetMsgs:** 해당 트랜잭션에 포함된 `sdk.Msg` 를 리스트를 반환합니다. 하나의 트랜잭션은 모듈에 정의된 하나 또는 여러개의 메시지를 가질 수 있습니다.

* **ValidateBasic:** ABCI 메시지 [`CheckTx`](./baseapp.md#checktx), [`DeliverTx`](./baseapp.md#delivertx)에서 사용되는 가볍고 [상태없는](../basics/tx-lifecycle.md#types-of-checks) 트랜잭션의 유효성을 확인합니다. 예를 들어 [`auth`](https://github.com/cosmos/cosmos-sdk/tree/main/x/auth) 모듈의  `ValidateBasic`은 올바른 수의 서명자가 서명했는지 확인하고, 사용자가 지정한 최대 수수료(fee)를 초과하지 않았는지 검사합니다. 이 함수는 메시지의 기본 검사를 수행하는 `sdk.Msg` [`ValidateBasic`](../basics/tx-lifecycle.md#ValidateBasic) 와는 다릅니다. [`runTx`](./baseapp.md#runtx) 에서 [`auth`](https://github.com/cosmos/cosmos-sdk/tree/main/x/auth/spec) 모듈에서 생성된 트랜잭션을 확인할 때 먼저 각 메시지의 `ValidateBasic` 을 확인하고 , 해당 트랜잭션의 `ValidateBasic`을 호출하기 위해 `auth`모듈 AnteHandler 를 수행합니다.

`Tx`는 트랜잭션 생성을 위해 중간 타입인 `Tx`를 직접 다루는 것 보단 `TxBuilder` 인터페이스를 이용하는 것이 좋습니다. 이에 대해서는 [아래](#transaction-generation)에서 확인할 수 있다.


### 트랜잭션 서명(Signing Transactions)

트랜잭션에 포함된 모든 메시지는 `GetSigners`에 특정된 주소로 서명되야 합니다. 코스모스 SDK에서는 현재 두가지 트랜잭션 서명 방법을 지원합니다.


#### `SIGN_MODE_DIRECT` (권장됨)

가장 많이 사용되는 `Tx` 인터페이스는 `SIGN_MODE_DIRECT`를 사용하는 Protobuf `Tx` 메시지입니다. 

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/proto/cosmos/tx/v1beta1/tx.proto#L13-L26

Protobuf 일렬화(serialization)는 오직 하나의 결과만 보장하지 않기 떄문에  코스모스 SDK는 추가적인 `TxRaw` 타입을 사용하여 서명된 트랜잭션을 고정된 바이트로 나타냅니다. 어떤 사용자든 유효한 `body`와 `auth_info`를 생성하고 이를 protobuf를 통해 일렬화할 수 있습니다. `TxRaw`를 사용하여 `body`와 `auth_info`를 각각 `body_bytes`와 `auth_info_bytes`로 정확한 바이너리로 나타냅니다. `SignDoc`은 트랜잭션의 모든 서명자가 서명한 문서입니다. ([ADR-027](../architecture/adr-027-deterministic-protobuf-serialization.md)를 사용해 결정적 일렬화됩니다):

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/proto/cosmos/tx/v1beta1/tx.proto#L48-L65

모든 서명자에 의해 서명되면 `body_bytes`, `auth_info_bytes` 그리고 `signatures`는 `TxRaw` 타입으로 일렬화되어 네트워크에 전송됩니다.


#### `SIGN_MODE_LEGACY_AMINO_JSON`

`Tx` 인터페이스의 레거시 구현으로는 `x/auth`의 `StdTx` 구조체가 있습니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/x/auth/migrations/legacytx/stdtx.go#L82-L92

`StdSignDoc`은 모든 서명자에 의해 서명된 문서입니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/x/auth/migrations/legacytx/stdsign.go#L38-L52

이는 Amino JSON을 사용하여 바이트로 인코딩됩니다. 모든 서명은 `StdTx`로 합쳐지고 Amino JSON을 사용하여 일렬화되어 네트워크에 전송됩니다. 

#### 기타 서명 모드

코스모스 SDK는 특정한 용도로 몇가지 다른 서명 모드를 제공합니다. 


#### `SIGN_MODE_DIRECT_AUX`

`SIGN_MODE_DIRECT_AUX`는 여러명의 서명자를 가지는 트랜잭션을 위해 코스모스 SDK v0.46 에서 지원하는 서명 모드입니다. `SIGN_MODE_DIRECT`에서 각각의 서명자가 `TxBody`와 `AuthInfo` (다른 모든 서명자의 계정 시퀀스, 공개키와 모드 정보 같은 서명자 정보를 포함함)에 서명해아 하는데 반해 , `SIGN_MODE_DIRECT_AUX`는 N-1의 서명자가 `TxBody`와 _본인_ 서명 정보만 제공하면 됩니다. 또한 각 보조 서명자 (즉 `SIGN_MODE_DIRECT_AUX` 모드를 사용하는 서명자) 수수료를 초과해서 서명할 필요가 없습니다

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/proto/cosmos/tx/v1beta1/tx.proto#L67-L93

이는 다중 서명자 트랜잭션에서 사용됩니다. 서명자 한명이 모든 서명을 수집하여 전송하고 수수료를 지불하는 경우, 다른 서명자들은 트랜잭션 body만 신경쓰면 됩니다. 이는 다중 서명 UX를 더 좋게 만들어 줍니다. 만약 Alice, Bob 그리고 Charlie 가 3명이 다중 서명하는 경우, Alice와 Bob은 `SIGN_MODE_DIRECT_AUX` 모드로 `TxBody` 와 각자의 서명자 정보만 서명하고 (`SIGN_MODE_DIRECT`에서 처럼 모든 서명 정보를 수집하는 과정이 필요 없습니다), 각자의 SignDoc에 별도의 수수료를 붙이지 않습니다. Charlie는 Alice과 Bob서명을 모아 수수료를 붙여 최종 트랜잭션을 만들 수 있습니다. 최종 수수료 제공자(예시에서 Charlie)는 지정 수수료 이상으로 사인해야 하기 떄문에 `SIGN_MODE_DIRECT` 또는  `SIGN_MODE_LEGACY_AMINO_JSON` 모드를 사용해야 합니다.

구체적 사례는 [트랜잭션 팁](./tips.md)에 구현되어 있습니다: 팁을 주는 사람은 `SIGN_MODE_DIRECT_AUX` 사용하여 트랜잭션 수수료 없이 특정 팁을 트랜잭션에 명시하고, 수수료 납부자는 팁 주는 사람의 요청된 `TxBody`에 포함된 수수료를 받아 지불하여 해당 트랜잭션을 전송합니다. 


#### `SIGN_MODE_TEXTUAL`

`SIGN_MODE_TEXTUAL`는 하드웨어 지갑 사용하여 서명시 더 나은 경험을 제공하기 위한 신규 모드로 현재 구현 중에 있습니다. 자세한 사항은 [ADR-050](https://github.com/cosmos/cosmos-sdk/pull/10701)을 참고하세요.


## 트랜잭션 프로세스(Transaction Process)

최종사용자가 트랜잭션을 보내는 프로세스는 다음과 같습니다 

* 트랜잭션에 포함시킬 메시지 선택
* 코스모스 SDK `TxBuilder`를 사용하여 트랜잭션 생성
* 사용가능한 인터페이스 중 하나를 사용하여 트랜잭션 전송

다음 문단에서 각각의 동작을 순서대로 설명합니다

### 메시지(Messages)

::: 팁
Module `sdk.Msg`와 Tendermin와 어플리케이션 레이어 상호작용을 정의하는 [ABCI Messages](https://docs.tendermint.com/master/spec/abci/abci.html#messages) 와 혼돈해서는 안됩니다
:::

**Messages** (또는 `sdk.Msg`s)는 모듈 안에서 상태 변화를 가져오는 모듈-특화된 객체입니다. 모듈 개발자는 개발하는 모듈에 메시지를 정의하고 Protobuf [`Msg` service](../building-modules/msg-services.md)를 위한 메서드를 추가하고, 해당 메시지 처리를 위한 `MsgServer`를 구현합니다.

각 `sdk.Msg`는 단 하나의 Protobuf [`Msg` service](../building-modules/msg-services.md) RPC와 연관되며, 각 모듈의 `tx.proto` 파일에 정의됩니다. SDK의 app router는 자동으로 모든 `sdk.Msg`를 관련된 RPC로 연결합니다. Protobuf 는 각 모듈의 `Msg` 서비스를 위한 `MsgServer` 인터페이스를 생성합니다. 그리고 모듈 개발자는 해당 인터페이스를 구현해야 합니다.

이러한 설계는 모듈 개발자에게 더 많은 책임을 부여하며, 애플리케이션 개발자는 공통된 기능인 상태 변이 로직을 반복적으로 구현하지 않고 재사용 할 수 있게 해줍니다.

Protobuf `Msg` 서비스와  `MsgServer` 구현에 대해 자세히 알고 싶으면 [여기](../building-modules/msg-services.md)를 클릭하세요.

메시지는 상태 변화 로직을 위한 정보를 담고 있는 반면, 트랜잭션의 다른 메타데이터와 관련된 정보는 `TxBuilder` 와 `Context`에 저장됩니다.


### 트랜잭션 생성(Transaction Generation)

`TxBuilder` 인터페이스는 트랜잭션 생성을 위해 사용자가 자유롭게 설정할 수 있는 밀접한 데이터를 포함합니다. 

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/client/tx_config.go#L33-L50

* `Msg`는 해당 트랜잭션에 포함된 [메시지](#messages) 배열.
* `GasLimit`, 사용자가 얼마나 많은 가스를 지불할지 계산하는 법을 결정하는 옵션.
* `Memo`, 트랜잭션에 담기는 메모.
* `FeeAmount`, 수수료로 제공할 수 있는 최대 수량.
* `TimeoutHeight`, 해당 트랜잭션이 유효한 최대 블록 높이.
* `Signatures`, 해당 트랜잭션의 모든 서명자들의 서명.

현재 두 개의 서명 모드가 있으므로, `TxBuilder`도 2가지 구현이 있습니다:

* [wrapper](https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/x/auth/tx/builder.go#L18-L34) `SIGN_MODE_DIRECT` 모드를 위한 트랜잭션 생성시
* [StdTxBuilder](https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/x/auth/migrations/legacytx/stdtx_builder.go#L15-L21) `SIGN_MODE_LEGACY_AMINO_JSON` 사용시

하지만 최종 사용자는 두 개의 `TxBuilder` 구현체를 알 필요 없이 `TxConfig` 인터페이스를 사용하는 것이 좋습니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/client/tx_config.go#L22-L31

`TxConfig`는 트랜잭션 관리를 위해 앱 전체에서 사용되는 설정입니다. 해당 설정에는 `SIGN_MODE_DIRECT` 또는 `SIGN_MODE_LEGACY_AMINO_JSON` 모드로 각 트랜잭션을 서명하는 정보를 가지고 있습니다. `txBuilder := txConfig.NewTxBuilder()`를 호출하면 적절한 서명 모드로 새로운 `TxBuilder`가 생성됩니다.

`TxBuilder`가 앞에 설정된 여러 설정 함수(setter)로 채워지면 `TxConfig`는 `SIGN_MODE_DIRECT` 또는 `SIGN_MODE_LEGACY_AMINO_JSON`를 사용하여 바이트 인코딩을 처리합니다. `TxEncoder()` 메소드를 사용하여 트랜잭션을 생성하고 인코딩하는 방법에 대한 의사 코드는 다음과 같습니다.


```go
txBuilder := txConfig.NewTxBuilder()
txBuilder.SetMsgs(...) // and other setters on txBuilder

bz, err := txConfig.TxEncoder()(txBuilder.GetTx())
// bz are bytes to be broadcasted over the network
```

### 트랜잭션 전송(Broadcasting the Transaction)

트랜잭션 바이트가 생성되면 현재 이를 전송하는 방법은 세 가지가 있습니다.

#### CLI

어플리케이션 개발자는 [명령줄 인터페이스](../core/cli.md), [gRPC, REST 인터페이스](../core/grpc_rest.md)를 만들어 응용 프로그램에 접근하며, 이는 보통 응용 프로그램 `./cmd` 폴더에 존재합니다. 이러한 인터페이스를 통해 사용자는 명령줄을 통해 응용 프로그램과 상호 작용할 수 있습니다.

[명령줄 인터페이스](../building-modules/module-interfaces.md#cli) 의 경우 모듈 개발자는 하위 명령어을 만들어 최상위 트랜잭션 명령어 `TxCmd` 아래 추가합니다. CLI 명령은 실제로 트랜잭션 처리의 모든 단계를 하나의 간단한 명령(메시지 생성, 트랜잭션 생성 및 브로드캐스트)으로 묶습니다. 구체적인 예는 [노드와 상호 작용](../run-node/interact-node.md) 섹션을 참조하세오. CLI를 사용하여 트랜잭션 생성 예는 다음과 같습니다.

```bash
simd tx send $MY_VALIDATOR_ADDRESS $RECIPIENT 1000stake
```

#### gRPC

[gRPC](https://grpc.io)는 코스모스 SDK의 RPC 계층의 주요 구성 요소입니다. 주요 사용법은 모듈의 [`질의` 서비스](../building-modules)에서 확인할 있습니다. 그러나 모듈과 관련 없이 사용할 수 있는 일부 gRPC 서비스도 코스모스 SDK에서 제공되며, 그 중 하나가 `Tx` 서비스입니다. 

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/proto/cosmos/tx/v1beta1/service.proto

`Tx` 서비스는 트랜잭션 시뮬레이션 또는 트랜잭션 질의와 같은 몇 가지 유틸리티 기능과 트랜잭션을 전송하는 기능을 제공합니다.

트랜잭션을 전송하고 시뮬레이션하는 예제는 [여기](../run-node/txs.md#programmatically-with-go) 있습니다.

#### REST

각 gRPC 메서드에는 [gRPC-gateway](https://github.com/grpc-ecosystem/grpc-gateway)를 사용하여 생성된 REST 엔드포인트가 있습니다. 따라서 gRPC를 사용하는 대신 HTTP를 사용하여 `POST /cosmos/tx/v1beta1/txs` 엔드포인트에서 동일한 트랜잭션을 전송할 수 있습니다.

예시는 [여기](../run-node/txs.md#using-rest)서 확인할 수 있습니다

#### Tendermint RPC

위에 제시된 세 가지 방법은 [여기](https://docs.tendermint.com/master/rpc/#/Tx) 문서화 되어 있는 Tendermint RPC `/broadcast_tx_{async,sync,commit}` 엔드포인트 보다 실제로 더 추상화되어 있습니다. 원하는 경우 Tendermint RPC 엔드포인트를 직접 사용하여 트랜잭션을 전송할 수 있습니다.

## 다음 {hide}

[컨텍스트](./context.md) 배우기 {hide}