# 이벤트

`Events`는 애플리케이션 실행에 대한 정보를 포함하는 객체입니다. 그들은 주로 블록 익스플로러 및 지갑과 같은 서비스 제공자가 다양한 메시지의 실행을 추적하고 트랜잭션을 색인하는 데 사용합니다.

## 전제 조건 읽기

- [Cosmos SDK 응용 프로그램 분석](../basics/app-anatomy.md)
- [이벤트에 대한 텐더민트 문서](https://docs.tendermint.com/master/spec/abci/abci.html#events)

## 유형 이벤트

이벤트는 ABCI `Event` 유형의 별칭으로 Cosmos SDK에서 구현되며
`{eventType}.{attributeKey}={attributeValue}` 형식을 취합니다.

+++ https://github.com/tendermint/tendermint/blob/v0.35.4/proto/tendermint/abci/types.proto#L273-L279

이벤트에는 다음이 포함됩니다.

- `type`: 높은 수준에서 이벤트를 분류하기 위한 `type` 입니다. 예를 들어 Cosmos SDK는 `"message"` 유형(type)을 사용하여 `Msg`별로 이벤트를 필터링합니다.
- `attributes` 목록: 분류된 이벤트에 대한 자세한 정보를 제공하는 키-값 쌍입니다. 예를 들어 `"message"` 유형의 경우 `message.action={some_action}`, `message.module={some_module}` 또는 `message.sender={some_sender}`를 사용하여 키-값 쌍으로 이벤트를 필터링할 수 있습니다.

::: 팁
속성 값을 문자열로 구문 분석하려면 각 속성 값 주위에 ```(작은 따옴표)를 추가해야 합니다.
:::

*Typed Events*는 Cosmos SDK에서 사용하는 Protobuf로 정의된 [메시지](../architecture/adr-032-typed-events.md)입니다.
이벤트를 내보내고 쿼리합니다. 그것들은 **모듈 단위**로 `event.proto` 파일에 정의됩니다.
모듈의 Protobuf [`Msg` 서비스](../building-modules/msg-services.md)에서 트리거됩니다.
[`EventManager`](#eventmanager)를 사용하여 `proto.Message`로 읽습니다.

또한 각 모듈은 `spec/xx_events.md` 아래에 이벤트를 문서화합니다.

마지막으로 이벤트는 다음 ABCI 메시지의 응답으로 기본 합의 엔진으로 반환됩니다.

- [`BeginBlock`](./baseapp.md#beginblock)
- [`EndBlock`](./baseapp.md#endblock)
- [`CheckTx`](./baseapp.md#checktx)
- [`DeliverTx`](./baseapp.md#delivertx)

### 예

다음 예제에서는 Cosmos SDK를 사용하여 이벤트를 쿼리하는 방법을 보여줍니다.

| 이벤트                                           | 설명                                                                                                                                                   |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `tx.height=23`                                   | 높이 23에서 모든 트랜잭션 쿼리                                                                                                                         |
| `message.action='/cosmos.bank.v1beta1.Msg/send'` | x/bank `Send` [Service `Msg`](../building-modules/msg-services.md)가 포함된 모든 트랜잭션을 쿼리합니다. 값 주위의 `'`에 주의하십시오.                  |
| `message.action='send'`                          | x/bank `Send` [legacy `Msg`](../building-modules/msg-services.md#legacy-amino-msgs)가 포함된 모든 트랜잭션을 쿼리합니다. 값 주위의 `'`에 주의하십시오. |
| `message.module='bank'`                          | x/bank 모듈의 메시지가 포함된 모든 트랜잭션을 쿼리합니다. 값 주위의 `'`에 주의하십시오.                                                                |
| `create_validator.validator='cosmosval1...'`     | x/staking 관련 이벤트입니다. [x/staking SPEC](../../../cosmos-sdk/x/staking/spec/07_events.md)를 참조하십시오.                                         |

## 이벤트 매니저

Cosmos SDK 애플리케이션에서 이벤트는 `EventManager`라는 추상화에 의해 관리됩니다.
내부적으로 `EventManager`는 트랜잭션의 전체 실행 흐름에 대한 이벤트 목록 또는 `BeginBlock`/`EndBlock`을 추적합니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/types/events.go#L17-L25

`EventManager`는 이벤트를 관리하는 데 유용한 메소드 세트와 함께 제공됩니다.
모듈 및 애플리케이션 개발자가 가장 많이 사용하는 방법은 `EventManager`에서 추적하는 이벤트인 `EmitTypedEvent`입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/types/events.go#L50-L59

모듈 개발자는 각 메시지 `Handler` 및 `BeginBlock`/`Endlock` 핸들러를 `EventManager#EmitTypedEvent`를 통해 이벤트 방출을 처리해야 합니다.
`EventManager`는 다음을 통해 액세스됩니다.
[`Context`](./context.md), 여기서 Event는 이미 등록되어 있어야하고 다음과 같이 내보내야 합니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/x/group/keeper/msg_server.go#L89-L92

모듈의 `handler` 함수는 `message`별로 방출된 이벤트를 분리하기 위해 새로운 `EventManager`를 `context`로 설정해야 합니다.

```go
func NewHandler(keeper Keeper) sdk.Handler {
    return func(ctx sdk.Context, msg sdk.Msg) (*sdk.Result, error) {
        ctx = ctx.WithEventManager(sdk.NewEventManager())
        switch msg := msg.(type) {
```

일반적으로 이벤트를 구현하고 모듈에서 `EventManager`를 사용하는 방법에 대한 자세한 내용은 [`Msg` 서비스](../building-modules/msg-services.md) 개념 문서를 참조하세요.

## 이벤트 구독

Tendermint의 [Websocket](https://docs.tendermint.com/master/tendermint-core/subscription.html#subscribe-to-events-via-websocket)을 사용하여 `subscribe` RPC 메서드를 호출하여 이벤트를 구독할 수 있습니다. :

```json
{
  "jsonrpc": "2.0",
  "method": "subscribe",
  "id": "0",
  "params": {
    "query": "tm.event='eventCategory' AND eventType.eventAttribute='attributeValue'"
  }
}
```

구독할 수 있는 주요 `eventCategory`는 다음과 같습니다.

- `NewBlock`: `BeginBlock` 및 `EndBlock` 동안 트리거된 이벤트를 포함합니다.
- `Tx`: `DeliverTx`(즉, 트랜잭션 처리) 중에 트리거된 이벤트를 포함합니다.
- `ValidatorSetUpdates`: 블록에 대한 유효성 검사기 세트 업데이트를 포함합니다.

이러한 이벤트는 블록이 커밋된 후 `상태` 패키지에서 트리거됩니다.
이벤트 카테고리의 전체 목록은 [Tendermint Go 문서](https://pkg.go.dev/github.com/tendermint/tendermint/types#pkg-constants)에서 확인할 수 있습니다.

`query`의 `type` 및 `attribute` 값을 사용하면 찾고 있는 특정 이벤트를 필터링할 수 있습니다. 예를 들어, `Mint` 트랜잭션은 `EventMint` 유형의 이벤트를 트리거하고 `Id`와 `Owner`를 `속성`으로 가집니다(`NFT` 모듈의 [`events.proto` 파일에 정의됨] (https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/proto/cosmos/nft/v1beta1/event.proto#L14-L19)).

이 이벤트에 대한 구독은 다음과 같이 수행됩니다.

```json
{
  "jsonrpc": "2.0",
  "method": "subscribe",
  "id": "0",
  "params": {
    "query": "tm.event='Tx' AND mint.owner='ownerAddress'"
  }
}
```

여기서 `ownerAddress`는 [`AccAddress`](../basics/accounts.md#addresses) 형식을 따르는 주소입니다.

## 이벤트(지원 중단됨)

이전에 Cosmos SDK는 `types/events.go`에 정의된 이벤트 방출을 지원했습니다. 이벤트 유형 및 이벤트 속성을 정의하는 것은 모듈 개발자의 책임입니다. `spec/XX_events.md` 파일을 제외하고 이러한 이벤트 유형 및 속성은 불행히도 쉽게 검색할 수 없습니다.

이것이 이 메소드가 더 이상 사용되지 않고 \_[Typed Events](#typed-events)로 대체된 이유입니다.

이벤트를 정의하는 이전 방식에 대한 자세한 내용은 [이전 SDK 문서](https://docs.cosmos.network/v0.45/core/events.html#events-2)를 참조하세요.

## 다음

Cosmos SDK [telemetry](./telemetry.md)에 대해 알아보기
