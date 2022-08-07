# 컨텍스트(Context)

`컨텍스트`는 응용프로그램의 현재 상태를 함수 간 전달을 하기 위해 사용되는 데이터 구조입니다. 이는 `gasMeter`, `block height`, `consensus parameters` 등과 같은 유용한 객체와 정보를 포함하여 분기된 저장소(모든 상태의 안전한 분기)에 대한 접근을 제공합니다. {synopsis}

## 선행 학습 자료(Pre-requisites Readings)

* [Anatomy of a Cosmos SDK Application](../basics/app-anatomy.md) {prereq}
* [Lifecycle of a Transaction](../basics/tx-lifecycle.md) {prereq}

## 컨텍스트 정의

코스모스 SDK의 `컨텍스트`는 Go의 표준 라이브러리 [`context`](https://pkg.go.dev/context)를 포함하며, 코스모스 SDK에서 사용되는 다양한 타입이 추가된 커스텀 데이터 구조입니다. `컨텍스트`는 트랜젝션 처리에서 모듈이 [`multistore`](./store.md#multistore)에 포함된 각각의 [store](./store.md#base-layer-kvstores)에 쉽게 접근하고, 블록 헤더와 가스 미터 같은 트랜잭션에 필요한 정보를 검색하기 위해 반드시 필요합니다.  

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/types/context.go#L17-L42

* **Base Context:** Go [Context](https://pkg.go.dev/context)를 기반으로 합니다. 더 자세한 설명은 아래 [Go Context Package](#go-context-package) 섹션에서 확인할 수 있습니다.
* **Multistore:** 모든 응용의 `BaseApp`은 `Context`가 생성되면 제공되는 [`CommitMultiStore`](./store.md#multistore)를 가지고 있습니다. 고유한 `StoreKey`를 사용하여 `KVStore()`와 `TransientStore()` 메서드를 호출하여 모듈에서 각각의 [`KVStore`](./store.md#base-layer-kvstores) 가져올 수 있습니다.
* **Header:** [header](https://docs.tendermint.com/master/spec/core/data_structures.html#header)는 블록체인의 타입을 나타냅니다. 블록체인 높이(Height), 현재 블록의 제안자(Proposer)같은 블록체인 상태에 대한 중요한 정보를 전달합니다.
* **Header Hash:** `abci.RequestBeginBlock`에서 얻어지는 현재 블록 헤더의 해쉬값.
* **Chain ID:** 해당 블록체인의 고유 식별 번호.
* **Transaction Bytes:** 컨텍스트를 사용하여 처리된 `[]byte` 타입의 트랜잭션. 모든 트랜잭션은 그 [수명주기](../basics/tx-lifecycle.md)에 따라 코스모스 SDK의 다양한 부분과 합의 엔진 (예. 텐더민트)에서 처리되며, 이 중 일부는 트랜잭션 타입과 상관없이 처리됩니다. 그래서 트랜젝션은 [Amino](./encoding.md) 같은 [encoding format](./encoding.md)을 사용하여 일반적인 `[]byte` 형태로 마샬링(marshalling) 됩니다.
* **Logger:** 텐더민트 라이브러리에서 전달된 `logger`. 로그에 대해 자세히 알려면 [여기](https://docs.tendermint.com/master/nodes/logging.html)를 참고하세요. 모듈은 해당 모듈에서만 사용하는 자체  로거 생성을 위해 이 메서드를 호출합니다.
* **VoteInfo:** 검증자(validator)의 이름과 해당 블록 서명 여부를 포함하고 있는 ABCI 타입 [`VoteInfo`](https://docs.tendermint.com/master/spec/abci/abci.html#voteinfo)의 리스트(list)
* **가스 계량기(Gas Meters):** [`gasMeter`](../basics/gas-fees.md#main-gas-meter)는 해당 컨텍스트를 사용하여 현재까지 수행된 트랜젝션 가스를, [`blockGasMeter`](../basics/gas-fees.md#block-gas-meter)는 전체 블록에 대한 가스를 나타냅니다. 사용자는 얼마나 많은 수수료를 자신의 트랜젝션 수행에 지불할지 지정하고, 가스 계량기는 지금까지 트랙잭션과 블록에 사용된 [가스](../basics/gas-fees.md)를 추적합니다. 가스 계량기가 표시하는 가스가 없으면 수행은 중단됩니다..
* **CheckTx Mode:** 트랜잭션이 `CheckTx` 또는 `DeliverTx` 모드에서 처리되어야 하는지를 나타내는 불(boolean) 값.
* **Min Gas Price:** 노드가 트랜잭션을 블록에 포함하기 위해 가져가는 최소 [가스](../basics/gas-fees.md) 가격. 이 가격은 각각의 노드가 개별적으로 설정하는 로컬(local) 값으로 **다음에 상태 변화로 이어지는 어떤 함수에도 사용하면 안됩니다.** 
* **Consensus Params:** ABCI 타입 [Consensus Parameters](https://docs.tendermint.com/master/spec/abci/apps.html#consensus-parameters)로 블록의 최대 가스 같은 블록체인에 대한 특정 제한을 지정합니다.
* **Priority:** 트랜잭션 우선순위 `CheckTx`에만 관련 있습니다.
* **Event Manager:** 이벤트 관리자는 `Context`를 접근하여 어떤 호출자든 [`Events`](./events.md)를 내보낼(emit) 수 있게 해줍니다. 모둘은 다양한 `Types`과 `Attributes`을 사용하거나 `types/`에 미리 정의된 공통 정의를 이용해 모듈에 특화된 `Events`를 정의할 수 있습니다. 클라이언트는 이러한 `Events`를 구독(subscribe)을 하거나 쿼리할 수 있습니다. `Events`는 `DeliverTx`, `BeginBlock`, 그리고 `EndBlock`에서 수집되며, Tendermint로 돌아가 인덱싱 됩니다. 예시: 

```go
ctx.EventManager().EmitEvent(sdk.NewEvent(
    sdk.EventTypeMessage,
    sdk.NewAttribute(sdk.AttributeKeyModule, types.AttributeValueCategory)),
)
```

## Go Context Package

기본 `Context`는 [Golang Context Package](https://pkg.go.dev/context)에 정의되어 있다. `Context`는 API와 프로세스 전반에 걸쳐 요청된 범위의 데이터를 전달하는 변하지 않는 데이터 구조입니다. 컨텍스트는 고루틴(goroutine)에 사용하도록 동시성이 가능하게 설계되었습니다.

컨텍스트는 **불변**입니다; 절대 수정되서는 안되지만 `With` 함수를 이용해 부모로 부터 자식 컨텍스트를 만들 수 있습니다. 예시:

```go
childCtx = parentCtx.WithBlockHeader(header)
```

[Golang Context Package](https://pkg.go.dev/context) 문서에서는 개발자가 프로세스의 첫번째 인자로 컨텍스트 `ctx`를 명시적으로 전달하도록 가르치고 있습니다.

## 저장소 분기하기(Store branching)

`Context`는 `MultiStore`를 포함하며, 이는 분기 기능과 `CacheMultiStore`(왕복 시간 단축을 위한 쿼리 캐싱)를 사용한 캐싱 기능을 가집니다.
각각의 `KVStore`는 안전하며 격리된 임시(ephemeral) 저장소로 분기됩니다. 프로세스들은 해당 `CacheMultiStore`에 자유롭게 쓸 수 있습니다. 상태 변화 시퀀스가 문제 없이 수행되면, 해당 분기 저장소는 마지막 시퀀스에 근본(underlying) 저장소로 커밋(commit)되거나 잘못이 있는 경우 무시됩니다. 컨텍스트의 사용 패턴은 다음과 같습니다:

1. 프로세스는 부모 프로세스로 부터 프로세스를 수행하는데 필요한 정보를 가지는 컨텍스트 `ctx`를 받습니다
2. `ctx.ms`는 [multistore](./store.md#multistore)에서 **분기된 저장소**입니다. 이는 해당 프로세스가 원본 `ctx.ms` 수정 없이 상태를 변경할 수 있게 만들어주며, 어떤 실행 지점으로 원상복구될 때 근본 멀티저장소를 보호하는데 유용합니다.
3. 프로세스는 실행하면서 `ctx`를 읽고 쓸 수 있습니다. 필요하면 하위 프로세스를 호출하여 `ctx`를 전달할 수 있습니다 
4. 하위 프로세스가 반환되면 결과값이 성공인지이 실패인지 검사합니다. 실패시 아무것도 하지 않으며 분기된 `ctx`는 폐기됩니다. 성공시 `CacheMultiStore`는 `Write()`를 통해 원본 `ctx.ms`로 커밋될 수 있습니다.

아래 코드는 [`baseapp`](./baseapp.md)의 [`runTx`](./baseapp.md#runtx-antehandler-runmsgs-posthandler) 함수 예시입니다:

```go
runMsgCtx, msCache := app.cacheTxContext(ctx, txBytes)
result = app.runMsgs(runMsgCtx, msgs, mode)
result.GasWanted = gasWanted
if mode != runTxModeDeliver {
  return result
}
if result.IsOK() {
  msCache.Write()
}
```

프로세스는 다음과 같습니다:

1. 트랜젝션에 있는 메시지를 수행하는 `runMsgs` 호출 전에 `app.cacheTxContext()`를 사용하여 컨텍스트와 멀티스토어 분기와 캐시를 가져옵니다.
2. `runMsgCtx`는 컨텍스트가 분기된 저장소이며 `runMsgs`에서 결과를 반환하기 위해 사용됩니다.
3. 만약 프로세스가 [`checkTxMode`](./baseapp.md#checktx)이면, 변화를 저장할 필요가 없으니 바로 반환합니다
4. 만약 프로세스가 [`deliverTxMode`](./baseapp.md#delivertx)모드이고 모든 메시지 수행에 대한 결과가 성공적으로 반환되면, 분기된 멀티저장소는 원본에 다시 기록됩니다.

## 다음{hide}

[node client](./node.md)에 대해 알아보기 {hide}