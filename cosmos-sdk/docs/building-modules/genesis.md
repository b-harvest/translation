# Module Genesis

모듈은 일반적으로 상태의 하위 집합을 처리하므로 관련 제네시스 파일의 하위 집합과 초기화(initialize), 확인(verify) 및 내보내기(export) 메서드를 정의해야 합니다.



## Type Definition

주어진 모듈에서 정의된 제네시스 상태의 하위 집합은 일반적으로 `genesis.proto` 파일에 정의됩니다 (protobuf 메시지를 정의하는 방법에 대한 [추가 정보](https://docs.cosmos.network/v0.46/core/encoding.html#gogoproto) ). 모듈의 제네시스 상태 하위 집합을 정의하는 구조체는 일반적으로 `GenesisState`라고 하며 제네시스 프로세스 중에 초기화해야 하는 모든 모듈 관련 값을 포함합니다.

`auth` 모듈에서 `GenesisState` protobuf 메시지 정의의 예를 참조하세요.

```go
syntax = "proto3";
package cosmos.auth.v1beta1;

import "google/protobuf/any.proto";
import "gogoproto/gogo.proto";
import "cosmos/auth/v1beta1/auth.proto";

option go_package = "6";

// GenesisState defines the auth module's genesis state.
message GenesisState {
  // params defines all the paramaters of the module.
  Params params = 1 [(gogoproto.nullable) = false];

  // accounts are the accounts present at genesis.
  repeated google.protobuf.Any accounts = 2;
}

```



다음으로 모듈 개발자가 Cosmos SDK 응용 프로그램에서 모듈을 사용하기 위해 구현해야 하는 주요 제네시스 관련 방법을 제시합니다.





### `DefaultGenesis`

`DefaultGenesis()` 메소드는 각 매개변수에 대한 기본값으로 `GenesisState`에 대한 생성자 함수를 호출하는 간단한 메소드입니다. `auth` 모듈의 예를 참조하세요.

```go
// DefaultGenesis는 기본 생성 상태를 auth 모듈의 raw 바이트를 반환합니다.
func (AppModuleBasic) DefaultGenesis(cdc codec.JSONCodec) json.RawMessage {
	return cdc.MustMarshalJSON(types.DefaultGenesisState())
}
```



### `ValidateGenesis`

제공된 `genesisState`가 올바른지 확인하기 위해 `ValidateGenesis(data GenesisState)` 메소드가 호출됩니다. `GenesisState`에 나열된 각 매개변수에 대해 유효성 검사를 수행해야 합니다. `auth` 모듈의 예를 참조하세요.

```go
// ValidateGenesis는 실패한 유효성 검사 기준에 대해 오류를 반환하는 인증 데이터의 기본 유효성 검사를 수행합니다.
func ValidateGenesis(data GenesisState) error {
	if err := data.Params.Validate(); err != nil {
		return err
	}

	genAccs, err := UnpackAccounts(data.Accounts)
	if err != nil {
		return err
	}

	return ValidateGenAccounts(genAccs)
}
```



## Other Genesis Methods

모듈 개발자는 `GenesisState`와 직접 관련된 방법 외에 [`AppModuleGenesis` 인터페이스](https://docs.cosmos.network/v0.46/building-modules/module-manager.html#appmodulegenesis)의 일부로 두 가지 다른 방법을 구현할 것으로 예상됩니다. (모듈이 genesis 내 상태의 하위 집합을 초기화해야 하는 경우에만). 



### `InitGenesis`

`InitGenesis` 메소드는 애플리케이션이 처음 시작될 때 [`InitChain`](https://docs.cosmos.network/v0.46/core/baseapp.html#initchain) 동안 실행됩니다. `GenesisState`가 주어지면 `GenesisState` 내의 각 매개변수에 대해 모듈의 [`keeper`](https://docs.cosmos.network/v0.46/building-modules/keeper.html) setter 함수를 사용하여 모듈에서 관리하는 상태의 하위 집합을 초기화합니다. 

애플리케이션의 [모듈 관리자](https://docs.cosmos.network/v0.46/building-modules/module-manager.html#manager)는 각 애플리케이션 모듈의 `InitGenesis`메소드를 순서대로 호출하는 역할을 합니다.  이 순서는 [애플리케이션의 생성자 함수](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#constructor-)에서 호출되는 모듈 관리자의 `SetOrderGenesisMethod`를 통해 애플리케이션 개발자가 설정합니다.

`auth` 모듈의 `InitGenesis` 예를 참조하세요.

```go
// InitGenesis - 제네시스 데이터에서 상태 정보를 읽어 스토어를 초기화
//
// 계약: FeeCollectionKeeper의 오래된 코인은 제네시스 포트 스크립트를 통해 새 수수료 징수자 계정으로 전송해야 합니다.
func (ak AccountKeeper) InitGenesis(ctx sdk.Context, data types.GenesisState) {
	ak.SetParams(ctx, data.Params)

	accounts, err := types.UnpackAccounts(data.Accounts)
	if err != nil {
		panic(err)
	}
	accounts = types.SanitizeGenesisAccounts(accounts)

	for _, a := range accounts {
		acc := ak.NewAccount(ctx, a)
		ak.SetAccount(ctx, acc)
	}

	ak.GetModuleAccount(ctx, types.FeeCollectorName)
}
```



### `ExportGenesis`

`ExportGenesis` 메서드는 상태를 내보낼 때마다 실행됩니다. 모듈에서 관리하는 상태 하위 집합의 알려진 최신 버전을 가져와서 새 `GenesisState`를 만듭니다. 이것은 주로 하드포크를 통해 체인을 업그레이드 해야 할 때 사용됩니다.

`auth` 모듈에서 `ExportGenesis`의 예를 참조하십시오.

```go
// ExportGenesis returns a GenesisState for a given context and keeper
func (ak AccountKeeper) ExportGenesis(ctx sdk.Context) *types.GenesisState {
	params := ak.GetParams(ctx)

	var genAccounts types.GenesisAccounts
	ak.IterateAccounts(ctx, func(account types.AccountI) bool {
		genAccount := account.(types.GenesisAccount)
		genAccounts = append(genAccounts, genAccount)
		return false
	})

	return types.NewGenesisState(params, genAccounts)
}
```

