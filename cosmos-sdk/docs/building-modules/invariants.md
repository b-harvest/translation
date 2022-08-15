# Invariants



불변(invariant)는 항상 true여야 하는 애플리케이션의 속성입니다. Cosmos SDK의 컨텍스트에서 `Invariant`는 특정 불변을 확인하는 기능입니다. 이러한 기능은 버그를 조기에 감지하고 잠재적인 결과(예: 체인 중단)를 제한하기 위해 조치를 취하는 데 유용합니다. 또한 시뮬레이션을 통해 버그를 감지하는 애플리케이션 개발 프로세스에서 유용합니다.



## Implementing `Invariant`s

`Invariant`는 모듈 내에서 특정 불변(invariant)를 확인하는 함수입니다. `Invariant` 모듈은 `Invariant` 타입을 따라야 합니다. 

```go
package types

import "fmt"

// Invariant는 특정 불변을 테스트하는 함수입니다.
// Invariant은 무슨 일이 일어났는지에 대한 설명 메시지와 불변이 깨졌는지 여부를 나타내는 불린(boolean)을 반환합니다.
// 그러면 시뮬레이터가 중지되고 로그를 출력합니다.
type Invariant func(ctx Context) (string, bool)

// Invariants defines a group of invariants
type Invariants []Invariant

// expected interface for registering invariants
type InvariantRegistry interface {
	RegisterRoute(moduleName, route string, invar Invariant)
}

// FormatInvariant returns a standardized invariant message.
func FormatInvariant(module, name, msg string) string {
	return fmt.Sprintf("%s: %s invariant\n%s\n", module, name, msg)
}

```



`string` 반환 값은 로그를 인쇄할 때 사용할 수 있는 불변 메시지이고 `bool` 반환 값은 불변 검사의 실제 결과입니다.

실제로 각 모듈은 모듈 폴더 내의 `keeper/invariants.go` 파일에 `Invariant`를 구현합니다. 표준은 다음 모델을 사용하여 불변들(invariants)의 논리적 그룹 당 하나의 `Invariant` 기능을 구현하는 것입니다.

```go
// Example for an Invariant that checks balance-related invariants

func BalanceInvariants(k Keeper) sdk.Invariant {
	return func(ctx sdk.Context) (string, bool) {
        // Implement checks for balance-related invariants
    }
}

```


또한 모듈 개발자는 일반적으로 모듈의 모든 `invariatn` 기능을 실행하는 `AllInvariants` 기능을 구현해야 합니다.

```go
// AllInvariants runs all invariants of the module.
// In this example, the module implements two Invariants: BalanceInvariants and DepositsInvariants

func AllInvariants(k Keeper) sdk.Invariant {

	return func(ctx sdk.Context) (string, bool) {
		res, stop := BalanceInvariants(k)(ctx)
		if stop {
			return res, stop
		}

		return DepositsInvariant(k)(ctx)
	}
}

```



마지막으로 모듈 개발자는 [`AppModule` 인터페이스](https://docs.cosmos.network/v0.46/building-modules/module-manager.html#appmodule)의 일부로 `RegisterInvariants` 메서드를 구현해야 합니다. 실제로 `module/module.go` 파일에 구현된 모듈의`RegisterInvariants` 메서드는 일반적으로 `keeper/invariants.go`파일에 구현된  `RegisterInvariants` 메서드에 대한 호출만 연기합니다. `RegisterInvariants` 메소드는 [`InvariantRegistry`](https://docs.cosmos.network/v0.46/building-modules/invariants.html#invariant-registry)에 각 `Invariant` 기능에 대한 경로를 등록합니다.

```go
func RegisterInvariants(ir sdk.InvariantRegistry, k Keeper) {
	ir.RegisterRoute(types.ModuleName, "module-accounts",
		ModuleAccountInvariants(k))
	ir.RegisterRoute(types.ModuleName, "nonnegative-power",
		NonNegativePowerInvariant(k))
	ir.RegisterRoute(types.ModuleName, "positive-delegation",
		PositiveDelegationInvariant(k))
	ir.RegisterRoute(types.ModuleName, "delegator-shares",
		DelegatorSharesInvariant(k))
}
```



자세한 내용은 [`staking` 모듈의 불변 구현](https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/x/staking/keeper/invariants.go) 을 참고하세요.



## Invariant Registry



`InvariantRegistry`는 애플리케이션의 모든 모듈의 `Invariant`가 등록되는 레지스트리입니다. **애플리케이션**당 하나의 `InvariantRegistry`만 있습니다. 즉, 모듈 개발자는 모듈을 빌드할 때 자신의 `InvariantRegistry`를 구현할 필요가 없습니다. **모든 모듈 개발자는 위 섹션에서 설명한 대로 `InvariantRegistry`에 모듈의 불변들을 등록해야 합니다.** 이 섹션의 나머지 부분에서는 `InvariantRegistry` 자체에 대한 자세한 정보를 제공하며 모듈 개발자와 직접적으로 관련된 내용은 포함하지 않습니다.

기본적으로 `InvariantRegistry` 는 Cosmos SDK에서 인터페이스로 정의됩니다.

```go
// expected interface for registering invariants
type InvariantRegistry interface {
	RegisterRoute(moduleName, route string, invar Invariant)
}
```



일반적으로 이 인터페이스는 특정 모듈의 `keeper`에서 구현됩니다. `InvariantRegistry`의 가장 많이 사용되는 구현은 `crisis` 모듈에서 찾을 수 있습니다.

```go
// RegisterRoute register the routes for each of the invariants
func (k *Keeper) RegisterRoute(moduleName, route string, invar sdk.Invariant) {
	invarRoute := types.NewInvarRoute(moduleName, route, invar)
	k.routes = append(k.routes, invarRoute)
}	
```



따라서 `InvariantRegistry`는 일반적으로 [애플리케이션의 생성자 함수](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#)에서 `crisis` 모듈의 `keeper`를 인스턴스화하여 인스턴스가 됩니다. 

`Invariant`는 [`message`](https://docs.cosmos.network/v0.46/building-modules/messages-and-queries.html)를 통해 수동으로 확인할 수 있지만 대부분 각 블록의 끝에서 자동으로 확인됩니다. 다음은 `crisis` 모듈의 예입니다.

```go
// check all registered invariants
func EndBlocker(ctx sdk.Context, k keeper.Keeper) {
	defer telemetry.ModuleMeasureSince(types.ModuleName, time.Now(), telemetry.MetricKeyEndBlocker)

	if k.InvCheckPeriod() == 0 || ctx.BlockHeight()%int64(k.InvCheckPeriod()) != 0 {
		// skip running the invariant check
		return
	}
	k.AssertInvariants(ctx)
}
```



두 경우 모두 `Invariant` 중 하나가 false를 반환하면 `InvariantRegistry`가 특수 로직을 트리거할 수 있습니다 (예: 애플리케이션을 패닉 상태로 만들고 로그에 `Invariant` 메시지를 출력함).



