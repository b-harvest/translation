<!--
order: 4
-->

# `Msg` Services

Protobuf `Msg` 서비스는 [messages](./messages-and-queries.md#messages)를 처리합니다. Protobuf `Msg` 서비스 정의된 모듈에 따라 구분되며, 해당 모듈에서만 처리됩니다. 메시지 서비스는 `BaseApp`의 [`DeliverTx`](../core/baseapp.md#delivertx)에서 호출됩니다. {synopsis}

## Pre-requisite Readings

* [Module Manager](./module-manager.md) {prereq}
* [Messages and Queries](./messages-and-queries.md) {prereq}

## 모듈 `Msg` 서비스 구현

각 모듈은 요청(`sdk.Msg`로 구현됨)을 처리하고 응답을 반환하는 
Protobuf `Msg` 서비스를 정의해야 합니다.

더 자세한 설명은 [ADR 031](../architecture/adr-031-msg-service.md) 설명되어 있으며, 이러한 방식은 서버와 클라이언트 코드를 생성하고 반환 타입을 명확하게 특정할 수 있는 장점이 있습니다.

Protobuf는 `Msg` 서비스 정의를 참고하여 `MsgServer` 인터페이스를 생성합니다. 모듈 개발자는 각 `sdk.Msg`를 받아 일어나는 상태 변화 로직을 작성하여 해당 인터페이스를 구현합니다. 예제는 두 개의 `sdk.Msg` 를 가지는 `x/bank` 모듈을 위해 생성된 `MsgServer` 인터페이스 입니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/x/bank/types/tx.pb.go#L288-L294

가능하면 기존의 모듈 [`Keeper`](keeper.md)에 `MsgServer`를 구현해야 하며, 그렇지 않은 경우 `Keeper`를 포함하는 `msgServer` 구조체를 만들 수 있습니다. 이는 일반적으로 `./keeper/msg_server.go`에 있습니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/x/bank/keeper/msg_server.go#L14-L16

`msgServer` 메서드는 `sdk.UnwrapSDKContext`을 사용해서 `context.Context` 인자로 부터 `sdk.Context`를 가져올 수 있습니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/x/bank/keeper/msg_server.go#L27-L27

`sdk.Msg` 처리는 보통 다음 3단계를 거칩니다:

### 검증(Validation)

`msgServer` 메서드가 실행되기 전에 메시지의 [`ValidateBasic()`](../basics/tx-lifecycle.md#ValidateBasic) 메서드가 먼저 호출됩니다. `msg.ValidateBasic()`는 단지 기본 검사만 수행하기 때문에 이 단계에서 `message`가 유효한지 확인하기 위해 다른 모든 검사를 수행해야 합니다. (*stateful*과 *stateless* 모두). `msgServer` 메서드에서 검사 수행시 가스가 더 들 수 있으며, 서명자는 이를 위해 가스를 충전해야 합니다. 예를 들어 `transfer` 메시지를 위한 `msgServer` 메서드는 전송하기 전에 보내는 계정이 충분한 자금이 있는지 확인해야 합니다. 

상태 값들을 분리된 함수의 인수로 전달하여 모든 유효성 검사를 수행하는 것을 추전합니다. 이러 방식의 구현은 테스트를 단순화합니다. 예상하듯 더 많은 유효성 검사 함수들은 더 많은 가스를 부과합니다. 예시:

```go
ValidateMsgA(msg MsgA, now Time, gm GasMeter) error {
	if now.Before(msg.Expire) {
		return sdkerrrors.ErrInvalidRequest.Wrap("msg expired")
	}
	gm.ConsumeGas(1000, "signature verification")
	return signatureVerificaton(msg.Prover, msg.Data)
}
```

### 상태 변화(State Transition)

검증이 성공하면, `msgServer` 메서드는 [`keeper`](./keeper.md) 함수를 사용하여 상태에 접근하고 상태 변화를 수행합니다.

### 이벤트(Events)

함수를 반환하기 전에, `msgServer` 메서드는 일반적으로 `ctx`에 포함된  `EventManager`를 이용해 하나 또는 여러개의 [events](../core/events.md)를 내보냅니다. 새로운 `EmitTypedEvent` 함수를 사용해 protobuf-기반 이벤트 타입을 내보낼 수 있습니다:

```go
ctx.EventManager().EmitTypedEvent(
	&group.EventABC{Key1: Value1,  Key2, Value2})
```

또는 이전 `EmitEvent` 함수: 

```go
ctx.EventManager().EmitEvent(
	sdk.NewEvent(
		eventType,  // e.g. sdk.EventTypeMessage for a message, types.CustomEventType for a custom event defined in the module
		sdk.NewAttribute(key1, value1),
		sdk.NewAttribute(key2, value2),
	),
)
```

이러한 이벤트들은 기본 합의 엔진으로 전달되어 서비스 제공자가 응용과 관련된 서비스를 구현하는데 사용될 수 있습니다. 이벤트에 대해 더 많은 정보가 알고 싶으면 [여기](../core/events.md)를 클릭하세요.

`msgServer` 가 수행되면 `proto.Message` 응답과 `error`를 반환합니다. 반환되는 값은 `sdk.WrapServiceResult(ctx sdk.Context, res proto.Message, err error)`를 통해 `*sdk.Result` 또는 `error`로 래핑(wrapping)됩니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/baseapp/msg_service_router.go#L127

이 메서드는 `res` 매개변수를 protobuf으로 마샬링(marshalling)하고 `ctx.EventManager()`에 있는 이벤트를 `sdk.Result`에 첨부합니다. 


+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/proto/cosmos/base/abci/v1beta1/abci.proto#L88-L109

이 다이어그램은 Protobuf `Msg` 서비스의 일반적인 구조와 메시지가 모듈을 통해 전파되는 방식을 보여줍니다

![트랜잭션 흐름](../uml/svg/transaction_flow.svg)

## 측정(Telemetry)

메시지를 처리할 때 `msgServer` 메서드에서 새로운 [측정 지표](../core/telemetry.md)를 만들 수 있습니다.

아래는 `x/auth/vesting` 모듈의 예제입니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/x/auth/vesting/msg_server.go#L73-L85

## 다음 {hide}

[질의 서비스](./query-services.md)에 대해 알아보기 {hide}