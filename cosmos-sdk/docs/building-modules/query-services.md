
# 쿼리 서비스(Query Services)

Protobuf 쿼리 서비스는 [`쿼리`](./messages-and-queries.md#queries)를 수행합니다. 쿼리 서비스는 특정 모듈에 정의하며 해당 모듈에서만 `쿼리`를 합니다. `BaseApp`'의 [`쿼리` 메서드](../core/baseapp.md#query)를 통해 호출합니다. {synopsis}

## 선행학습(Pre-requisite Readings)

* [Module Manager](./module-manager.md) {prereq}
* [Messages and Queries](./messages-and-queries.md) {prereq}

## `Querier` 타입

 코스모스 SDK에 정의된 `querier` 타입은 [gRPC Services](#grpc-service)를 위해 더 이상 사용되지 않습니다. 이는 `querier` 함수의 일반적인 구조를 명시합니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/types/queryable.go#L9

하나씩 살펴봅시다:

* [`컨텍스트`](../core/context.md)는 `쿼리`를 처리하기 위한 최신 상태에서 분기한 모든 필요한 정보를 가지고 있습니다. 해당 상태를 접근하기 위해 보통 [`keeper`](./keeper.md)를 사용합니다.
* `path`는 쿼리 타입을 가지며, `쿼리` 인자를 포함할 수 있는 `문자열` 배열입니다. 더 많은 정보는 [`queries`](./messages-and-queries.md#queries) 에서 확인할 수 있습니다.
* `req`는 인자가 너무 커서 `path`에 담을 수 없을때 인자를 가져오는데 주로 사용됩니다. 이는 `req`의 `Data` 필드가 사용됩니다.
* 결과값은 응용의 [`codec`](../core/encoding.md)을 사용하여 `[]byte` 형태로 마샬링되어 `BaseApp`에 반환됩니다. 

## 모듈 쿼리 서비스 구현

### gRPC 서비스

Protobuf `쿼리` 서비스를 정의할 때, 각 모듈의 모든 서비스 메서드를 포함하는 `QueryServer` 인터페이스가 생성됩니다:

```go
type QueryServer interface {
	QueryBalance(context.Context, *QueryBalanceParams) (*types.Coin, error)
	QueryAllBalances(context.Context, *QueryAllBalancesParams) (*QueryAllBalancesResponse, error)
}
```

사용자 지정 쿼리 메서드는 모듈의 Keeper를 사용해 구현해야 하며, 이는 보통 `./keeper/grpc_query.go`에 존재합니다. 이 메서드들의 처음 인자는 제네릭 `context.Context`이며, 쿼리 메서드는 저장소를 읽어오기 위해 `sdk.Context` 인스턴스가 필요합니다. 따라서 코스모스 SDK는 `sdk.UnwrapSDKContext` 함수를 제공하여 제공된 `context.Context`로부터 `sdk.Context`를 가져옵니다.

여기 bank 모듈의 구현 예시가 있습니다:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/x/bank/keeper/grpc_query.go

### 레거시(Legacy) Queriers

모듈의 레거시 `querier`는 일반적으로 모듈 폴더 아래 `./keeper/querier.go` 파일에 구현되어 있습니다. [모듈 관리자](./module-manager.md) 는 `NewQuerier()` 메서드를 사용하여 모듈의 `querier`를 [응용의 `queryRouter`](../core/baseapp.md#query-routing)에 추가합니다. 모듈 관리자의 `NewQuerier()` 메서드는 `keeper/querier.go`의 `NewQuerier()`를 호출하며 이는 다음과 같습니다:

```go
func NewQuerier(keeper Keeper) sdk.Querier {
	return func(ctx sdk.Context, path []string, req abci.RequestQuery) ([]byte, error) {
		switch path[0] {
		case QueryType1:
			return queryType1(ctx, path[1:], req, keeper)

		case QueryType2:
			return queryType2(ctx, path[1:], req, keeper)

		default:
			return nil, sdkerrors.Wrapf(sdkerrors.ErrUnknownRequest, "unknown %s query endpoint: %s", types.ModuleName, path[0])
		}
	}
}
```

수신한 `쿼리`의 타입에 따라 해당되는 `querier` 함수를 호출합니다. [쿼리 수명주기](../basics/query-lifecycle.md)의 이 시점에서 `path`의 첫 번째 요소(`path[0]`)는 쿼리 타입을 나타냅니다. `path`의 이어지는 요소들은 비어있거나 쿼리를 처리하는데 필요한 인자를 포함할 수 있습니다.

`querier` 함수 자체는 매우 간단합니다. [`keeper`](./keeper.md)를 통해 상태 값을 가지고와 [`codec`](../core/encoding.md)을 사용해 마샬링을 하고 처리된 `[]byte` 결과를 반환합니다.

`querier`에 대해 자세히 알고싶으면 뱅크 모듈의 [`querier` 함수 구현 예제](https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/x/gov/keeper/querier.go)를 참고하세요

## 다음 {hide}

[`BeginBlocker` and `EndBlocker`](./beginblock-endblock.md)에 대해 알아보기 {hide}