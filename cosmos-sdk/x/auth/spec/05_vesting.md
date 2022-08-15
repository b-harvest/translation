# Vesting



- Vesting
  - [Intro and Requirements](https://docs.cosmos.network/v0.46/modules/auth/05_vesting.html#intro-and-requirements)
  - [Note](https://docs.cosmos.network/v0.46/modules/auth/05_vesting.html#note)
  - Vesting Account Types
    - [BaseVestingAccount](https://docs.cosmos.network/v0.46/modules/auth/05_vesting.html#basevestingaccount)
    - [ContinuousVestingAccount](https://docs.cosmos.network/v0.46/modules/auth/05_vesting.html#continuousvestingaccount)
    - [DelayedVestingAccount](https://docs.cosmos.network/v0.46/modules/auth/05_vesting.html#delayedvestingaccount)
    - [Period](https://docs.cosmos.network/v0.46/modules/auth/05_vesting.html#period)
    - [PeriodicVestingAccount](https://docs.cosmos.network/v0.46/modules/auth/05_vesting.html#periodicvestingaccount)
    - [PermanentLockedAccount](https://docs.cosmos.network/v0.46/modules/auth/05_vesting.html#permanentlockedaccount)
  - Vesting Account Specification
    - Determining Vesting & Vested Amounts
      - [Continuously Vesting Accounts](https://docs.cosmos.network/v0.46/modules/auth/05_vesting.html#continuously-vesting-accounts)
    - Periodic Vesting Accounts
      - [Delayed/Discrete Vesting Accounts](https://docs.cosmos.network/v0.46/modules/auth/05_vesting.html#delayeddiscrete-vesting-accounts)
    - Transferring/Sending
      - [Keepers/Handlers](https://docs.cosmos.network/v0.46/modules/auth/05_vesting.html#keepershandlers)
    - Delegating
      - [Keepers/Handlers](https://docs.cosmos.network/v0.46/modules/auth/05_vesting.html#keepershandlers-1)
    - Undelegating
      - [Keepers/Handlers](https://docs.cosmos.network/v0.46/modules/auth/05_vesting.html#keepershandlers-2)
  - [Keepers & Handlers](https://docs.cosmos.network/v0.46/modules/auth/05_vesting.html#keepers--handlers)
  - [Genesis Initialization](https://docs.cosmos.network/v0.46/modules/auth/05_vesting.html#genesis-initialization)
  - Examples
    - [Simple](https://docs.cosmos.network/v0.46/modules/auth/05_vesting.html#simple)
    - [Slashing](https://docs.cosmos.network/v0.46/modules/auth/05_vesting.html#slashing)
    - [Periodic Vesting](https://docs.cosmos.network/v0.46/modules/auth/05_vesting.html#periodic-vesting)
  - [Glossary](https://docs.cosmos.network/v0.46/modules/auth/05_vesting.html#glossary)



## Intro and Requirements

이 사양은 Cosmos Hub에서 사용하는 베스팅 계정 구현을 정의합니다. 이 베스팅 계정에 대한 요구 사항은 시작 잔액 `X`와 베스팅 종료 시간 `ET`로 생성 중에 초기화되어야 한다는 것입니다. 베스팅 계정은 베스팅 시작 시간 `ST`와 많은 베스팅 기간 `P`로 초기화될 수 있습니다. 베스팅 시작 시간이 포함된 경우 시작 시간에 도달할 때까지 베스팅 기간이 시작되지 않습니다. 베스팅 기간이 포함된 경우 베스팅은 지정된 기간 동안 발생합니다.

모든 베스팅 계정에 대해 베스팅 계정의 소유자는 검증인에게 위임 및 위임을 취소할 수 있지만 해당 코인이 베스팅될 때까지 다른 계정으로 코인을 전송할 수 없습니다. 이 사양은 네 가지 종류의 베스팅을 허용합니다.

- `ET`에 도달하면 모든 코인이 수령되는 지연 베스팅 (Delayed vesting)
- `ST`에서 베스팅을 시작하고 `ET`에 도달할 때까지 시간에 대해 선형적으로 코인이 수령되는 연속적인 베스팅 (Continuous vesting)
-  `ST`에 베스팅되기 시작하여 기간 수와 기간별 베스팅 금액에 따라 주기적으로 코인이 수령되는 주기적 베스팅 (Periodic vesting)
  - 기간 수, 기간 당 길이 및 기간 당 금액을 구성할 수 있습니다. 주기적인 베스팅 계정은 코인이 시차를 둔 트랜치로 출시될 수 있다는 점에서 연속적인 베스팅 계정과 구별됩니다. 예를 들어, 주기적인 베스팅 계정은 코인이 분기별, 연간 또는 시간이 지남에 따라 토큰의 다른 기능을 통해 릴리스되는 베스팅 계약에 사용될 수 있습니다.

- 영구적으로 잠긴 베스팅 (Permanent locked vesting)
  - 코인이 영원히 잠겨 있습니다. 이 계정의 코인은 잠겨 있는 동안에도 위임 및 거버넌스 투표에 계속 사용할 수 있습니다.




## Note

베스팅 계정은 일부 베스팅 및 비-베스팅 코인으로 초기화할 수 있습니다. 비-베스팅 코인은 즉시 양도할 수 있습니다. DelayedVesting 및 ContinuousVesting 계정은 제네시스 이후 일반 메시지로 생성할 수 있습니다. 다른 유형의 베스팅 계정은 생성 시 또는 수동 네트워크 업그레이드의 일부로 생성해야 합니다. 현재 사양은 *unconditional* 베스팅만 허용합니다(즉, 'ET'에 도달하고 코인이 베스팅되지 않을 가능성이 없음).



## Vesting Account Types

```go
// VestingAccount는 모든 베스팅 계정 유형이 구현해야 하는 인터페이스를 정의합니다.
type VestingAccount interface {
  Account

  GetVestedCoins(Time)  Coins
  GetVestingCoins(Time) Coins

  // TrackDelegation은 베스팅 계정에서 위임할 때 필요한 내부 베스팅 회계를 수행합니다. 
  // 현재 블록 시간이 수락되면, 원래 베스팅 잔고에 해당 명칭이 존재하는 모든 코인의 위임 양과 계정 잔고를 확정합니다.
  TrackDelegation(Time, Coins, Coins)

  // TrackUndelegation은 베스팅 계정이 위임 취소를 수행할 때 필요한 내부 베스팅 회계를 수행합니다.
  TrackUndelegation(Coins)

  GetStartTime() int64
  GetEndTime()   int64
}

```





### BaseVestingAccount

```go
// BaseVestingAccount implements the VestingAccount interface. It contains all
// the necessary fields needed for any vesting account implementation.
message BaseVestingAccount {
  option (gogoproto.goproto_getters)  = false;
  option (gogoproto.goproto_stringer) = false;

  cosmos.auth.v1beta1.BaseAccount base_account       = 1 [(gogoproto.embed) = true];
  repeated cosmos.base.v1beta1.Coin original_vesting = 2
      [(gogoproto.nullable) = false, (gogoproto.castrepeated) = "github.com/cosmos/cosmos-sdk/types.Coins"];
  repeated cosmos.base.v1beta1.Coin delegated_free = 3
      [(gogoproto.nullable) = false, (gogoproto.castrepeated) = "github.com/cosmos/cosmos-sdk/types.Coins"];
  repeated cosmos.base.v1beta1.Coin delegated_vesting = 4
      [(gogoproto.nullable) = false, (gogoproto.castrepeated) = "github.com/cosmos/cosmos-sdk/types.Coins"];
  int64 end_time = 5;
}
```





### ContinuousVestingAccount

```go
// ContinuousVestingAccount implements the VestingAccount interface. It
// continuously vests by unlocking coins linearly with respect to time.
message ContinuousVestingAccount {
  option (gogoproto.goproto_getters)  = false;
  option (gogoproto.goproto_stringer) = false;

  BaseVestingAccount base_vesting_account = 1 [(gogoproto.embed) = true];
  int64              start_time           = 2;
}
```



### DelayedVestingAccount

```go
// DelayedVestingAccount implements the VestingAccount interface. It vests all
// coins after a specific time, but non prior. In other words, it keeps them
// locked until a specified time.
message DelayedVestingAccount {
  option (gogoproto.goproto_getters)  = false;
  option (gogoproto.goproto_stringer) = false;

  BaseVestingAccount base_vesting_account = 1 [(gogoproto.embed) = true];
}
```



### Period

```go
// Period defines a length of time and amount of coins that will vest.
message Period {
  option (gogoproto.goproto_stringer) = false;

  int64    length                          = 1;
  repeated cosmos.base.v1beta1.Coin amount = 2
      [(gogoproto.nullable) = false, (gogoproto.castrepeated) = "github.com/cosmos/cosmos-sdk/types.Coins"];
}
```

```go
// Stores all vesting periods passed as part of a PeriodicVestingAccount
type Periods []Period
	
```





### PeriodicVestingAccount

```go
// PeriodicVestingAccount implements the VestingAccount interface. It
// periodically vests by unlocking coins during each specified period.
message PeriodicVestingAccount {
  option (gogoproto.goproto_getters)  = false;
  option (gogoproto.goproto_stringer) = false;

  BaseVestingAccount base_vesting_account = 1 [(gogoproto.embed) = true];
  int64              start_time           = 2;
  repeated Period    vesting_periods      = 3 [(gogoproto.nullable) = false];
}
```



In order to facilitate less ad-hoc type checking and assertions and to support flexibility in account balance usage, the existing `x/bank` `ViewKeeper` interface is updated to contain the following:

임시 타입 체킹과 assertions을 줄이고 계정 잔액 사용의 유연성을 지원하기 위해 기존 `x/bank` `ViewKeeper` 인터페이스가 다음을 포함하도록 업데이트되었습니다.

```go
type ViewKeeper interface {
  // ...

  // Calculates the total locked account balance.
  LockedCoins(ctx sdk.Context, addr sdk.AccAddress) sdk.Coins

  // Calculates the total spendable balance that can be sent to other accounts.
  SpendableCoins(ctx sdk.Context, addr sdk.AccAddress) sdk.Coins
}

```





### PermanentLockedAccount

```go
// PermanentLockedAccount implements the VestingAccount interface. It does
// not ever release coins, locking them indefinitely. Coins in this account can
// still be used for delegating and for governance votes even while locked.
//
// Since: cosmos-sdk 0.43
message PermanentLockedAccount {
  option (gogoproto.goproto_getters)  = false;
  option (gogoproto.goproto_stringer) = false;

  BaseVestingAccount base_vesting_account = 1 [(gogoproto.embed) = true];
}
```





## Vesting Account Specification

베스팅 계정이 주어지면 진행 작업에서 다음을 정의합니다.

- `OV`: 원래 베스팅 (Original Vesting) 코인 양입니다. 일정한 값입니다.
- `V`: 아직 *베스팅* 중인 `OV` 코인의 수입니다. `OV`, `StartTime` 및 `EndTime`에 의해 파생됩니다. 이 값은 블록 단위가 아니라 요청 시 계산됩니다.
- `V'`: *베스트*(잠금 해제)된 `OV` 코인의 수입니다. 이 값은 블록 단위가 아니라 요청 시 계산됩니다.
- `DV`: 위임된 *베스팅* (Delegated Vesting) 코인의 수입니다. 변수 값입니다. 베스팅 계정에 직접 저장 및 수정됩니다.
- `DF`: 위임된 *베스트*(잠금 해제, DelegatedFree) 코인의 수입니다. 변수 값입니다. 베스팅 계정에 직접 저장 및 수정됩니다.
- `BC`: `OV` 코인에서 이전된 코인을 뺀 수(음수 또는 위임된). 내장된 기본 계정의 잔액으로 간주됩니다. 베스팅 계정에 직접 저장 및 수정됩니다.



### Determining Vesting & Vested Amounts

이러한 값은 필수 블록 단위가 아니라 요청 시 계산된다는 점에 유의해야 합니다(예: `BeginBlocker` 또는 `EndBlocker`).



#### Continuously Vesting Accounts

주어진 블록 시간 `T` 동안 베스팅된 코인의 양을 결정하기 위해 다음이 수행됩니다.

1. Compute `X := T - StartTime`
2. Compute `Y := EndTime - StartTime`
3. Compute `V' := OV * (X / Y)`
4. Compute `V := OV - V'`

따라서 수령된 코인의 총 금액은 `V'`이고 나머지 금액인 `V`는 *베스팅* 중 입니다.

```
func (cva ContinuousVestingAccount) GetVestedCoins(t Time) Coins {
    if t <= cva.StartTime {
        // We must handle the case where the start time for a vesting account has
        // been set into the future or when the start of the chain is not exactly
        // known.
        return ZeroCoins
    } else if t >= cva.EndTime {
        return cva.OriginalVesting
    }

    x := t - cva.StartTime
    y := cva.EndTime - cva.StartTime

    return cva.OriginalVesting * (x / y)
}

func (cva ContinuousVestingAccount) GetVestingCoins(t Time) Coins {
    return cva.OriginalVesting - cva.GetVestedCoins(t)
}

```





### Periodic Vesting Accounts

주기적인 베스팅 계정은 주어진 블록 시간 `T` 에 대해 각 기간 동안 릴리즈된 코인을 계산해야 합니다. `GetVestedCoins`를 호출할 때 여러 기간이 경과했을 수 있으므로 해당 기간이 `T` 이후가 될 때까지 각 기간을 반복해야 합니다.

1. Set `CT := StartTime`
2. Set `V' := 0`



각 기간 P에 대해서:

1. Compute `X := T - CT`

2. IF

   ```
   X >= P.Length
   ```
   
   1. Compute `V' += P.Amount`
   2. Compute `CT += P.Length`
   3. ELSE break
   
3. Compute `V := OV - V'`

```go
func (pva PeriodicVestingAccount) GetVestedCoins(t Time) Coins {
  if t < pva.StartTime {
    return ZeroCoins
  }
  ct := pva.StartTime // The start of the vesting schedule
  vested := 0
  periods = pva.GetPeriods()
  for _, period  := range periods {
    if t - ct < period.Length {
      break
    }
    vested += period.Amount
    ct += period.Length // increment ct to the start of the next vesting period
  }
  return vested
}

func (pva PeriodicVestingAccount) GetVestingCoins(t Time) Coins {
    return pva.OriginalVesting - cva.GetVestedCoins(t)
}

```



#### Delayed/Discrete Vesting Accounts

지연된 베스팅 계정은 특정 시간까지 전체 금액만 베스팅된 다음 모든 코인이 베스팅(잠금 해제)되기 때문에 추론하기 쉽습니다. 여기에는 계정에 처음에 있을 수 있는 잠금 해제된 코인이 포함되지 않습니다.

```
func (dva DelayedVestingAccount) GetVestedCoins(t Time) Coins {
    if t >= dva.EndTime {
        return dva.OriginalVesting
    }

    return ZeroCoins
}

func (dva DelayedVestingAccount) GetVestingCoins(t Time) Coins {
    return dva.OriginalVesting - dva.GetVestedCoins(t)
}

```



### Transferring/Sending

주어진 시간에 베스팅 계정은 `min((BC + DV) - V, BC)`를 이체할 수 있습니다.

즉, 베스팅 계정은 기본 계정 잔고, 기본 계정 잔고와 현재 위임된 베스팅 코인 수의 합에서 지금까지 베스팅된 코인 수를 뺀 금액 중 작은 것을 이체할 수 있다.

그러나 계정 잔액이 `x/bank` 모듈을 통해 추적되고 전체 계정 잔액을 로드하지 않으려는 경우 대신 `max(V - DV, 0)`로 정의할 수 있는 잠긴 잔액을 결정할 수 있습니다. `, 그리고 그것으로부터 지출 가능한 잔액을 추론합니다.

```go
func (va VestingAccount) LockedCoins(t Time) Coins {
   return max(va.GetVestingCoins(t) - va.DelegatedVesting, 0)
}
	
```



그러면 `x/bank` `ViewKeeper`가 API를 제공하여 모든 계정에 대해 잠긴 코인과 사용 가능한 코인을 결정할 수 있습니다.

```go
func (k Keeper) LockedCoins(ctx Context, addr AccAddress) Coins {
    acc := k.GetAccount(ctx, addr)
    if acc != nil {
        if acc.IsVesting() {
            return acc.LockedCoins(ctx.BlockTime())
        }
    }

    // non-vesting accounts do not have any locked coins
    return NewCoins()
}

```





#### Keepers/Handlers

해당 'x/bank' 키퍼는 해당 계정이 베스팅 계정인지 여부에 따라 코인 전송을 적절하게 처리해야 합니다.

```go
func (k Keeper) SendCoins(ctx Context, from Account, to Account, amount Coins) {
    bc := k.GetBalances(ctx, from)
    v := k.LockedCoins(ctx, from)

    spendable := bc - v
    newCoins := spendable - amount
    assert(newCoins >= 0)

    from.SetBalance(newCoins)
    to.AddBalance(amount)

    // save balances...
}

```





### Delegating

For a vesting account attempting to delegate `D` coins, the following is performed:

`D` 코인을 위임하려는 베스팅 계정의 경우 다음이 수행됩니다.

1. Verify `BC >= D > 0`
2. Compute `X := min(max(V - DV, 0), D)` (portion of `D` that is vesting)
3. Compute `Y := D - X` (portion of `D` that is free)
4. Set `DV += X`
5. Set `DF += Y`

```go
func (va VestingAccount) TrackDelegation(t Time, balance Coins, amount Coins) {
    assert(balance <= amount)
    x := min(max(va.GetVestingCoins(t) - va.DelegatedVesting, 0), amount)
    y := amount - x

    va.DelegatedVesting += x
    va.DelegatedFree += y
}

```



**Note** `TrackDelegation` only modifies the `DelegatedVesting` and `DelegatedFree` fields, so upstream callers MUST modify the `Coins` field by subtracting `amount`.

**참고** `TrackDelegation`은 `DelegatedVesting` 및 `DelegatedFree` 필드만 수정하므로 업스트림 호출자는 `amount`를 빼서 `Coins` 필드를 수정해야 합니다.



#### Keepers/Handlers

```go
func DelegateCoins(t Time, from Account, amount Coins) {
    if isVesting(from) {
        from.TrackDelegation(t, amount)
    } else {
        from.SetBalance(sc - amount)
    }

    // save account...
}

```



### Undelegating

'D' 코인의 위임을 취소하려는 베스팅 계정의 경우 다음이 수행됩니다. 참고: 위임/위임 취소 논리의 반올림 문제로 인해 `DV < D` 및 `(DV + DF) < D`가 가능할 수 있습니다.

1. Verify `D > 0`
2. Compute `X := min(DF, D)` (portion of `D` that should become free, prioritizing free coins)
3. Compute `Y := min(DV, D - X)` (portion of `D` that should remain vesting)
4. Set `DF -= X`
5. Set `DV -= Y`

```go
func (cva ContinuousVestingAccount) TrackUndelegation(amount Coins) {
    x := min(cva.DelegatedFree, amount)
    y := amount - x

    cva.DelegatedFree -= x
    cva.DelegatedVesting -= y
}

```



**참고** `TrackUnDelegation`은 `DelegatedVesting` 및 `DelegatedFree` 필드만 수정하므로 업스트림 호출자는 `amount`를 추가하여 `Coins` 필드를 수정해야 합니다.

**참고**: 위임이 삭감되면 모든 코인이 베스팅된 후에도 지속적인 베스팅 계정은 초과 `DV` 금액으로 끝납니다. 무료 코인을 위임 해제하는 것이 우선이기 때문입니다.

**참고**: 위임 취소(본드 환불) 금액은 위임 취소가 본드 환불을 자르는 방식으로 인해 위임된 베스팅(본드) 금액을 초과할 수 있으며, 위임되지 않은 토큰이 있을 경우 검증인의 환율(tokens/shares)이 약간 증가할 수 있습니다. 



#### Keepers/Handlers

```go
func UndelegateCoins(to Account, amount Coins) {
    if isVesting(to) {
        if to.DelegatedFree + to.DelegatedVesting >= amount {
            to.TrackUndelegation(amount)
            // save account ...
        }
    } else {
        AddBalance(to, amount)
        // save account...
    }
}

```





## Keepers & Handlers

`VestingAccount` 구현은 `x/auth`에 있습니다. 그러나 잠재적으로 베스팅 코인을 활용하려는 모듈의 키퍼(예: `x/staking`에 스테이킹)는 `SendCoins` 및 `SubtractCoins`에 반대되는 `x/bank` 키퍼에 대한 명시적 메서드 (e.g. `DelegateCoins`)를 호출해야 합니다.

또한 베스팅 계정은 다른 사용자로부터 받은 모든 코인을 사용할 수 있어야 합니다. 따라서 은행 모듈의 'MsgSend' 핸들러는 베스팅 계정이 잠금 해제된 코인 금액을 초과하는 금액을 보내려고 하면 오류가 발생해야 합니다.

전체 구현 세부 사항은 위의 사양을 참조하십시오.



## Genesis Initialization

베스팅 계정과 비-베스팅 계정을 모두 초기화하기 위해 `GenesisAccount` 구조체에는 `Vesting`, `StartTime` 및 `EndTime`과 같은 새 필드가 포함됩니다. `BaseAccount` 타입 또는 비-베스팅 타입인 계정에는 `Vesting = false`가 있습니다. 제네시스 초기화 로직(예: `initFromGenesisState`)은 이러한 필드를 기반으로 적절한 계정을 구문 분석하고 반환해야 합니다.



```go
type GenesisAccount struct {
    // ...

    // vesting account fields
    OriginalVesting  sdk.Coins `json:"original_vesting"`
    DelegatedFree    sdk.Coins `json:"delegated_free"`
    DelegatedVesting sdk.Coins `json:"delegated_vesting"`
    StartTime        int64     `json:"start_time"`
    EndTime          int64     `json:"end_time"`
}

func ToAccount(gacc GenesisAccount) Account {
    bacc := NewBaseAccount(gacc)

    if gacc.OriginalVesting > 0 {
        if ga.StartTime != 0 && ga.EndTime != 0 {
            // return a continuous vesting account
        } else if ga.EndTime != 0 {
            // return a delayed vesting account
        } else {
            // invalid genesis vesting account provided
            panic()
        }
    }

    return bacc
}

```





## Examples

### Simple

10개의 베스팅 코인이 있는 지속적인 베스팅 계정이 주어집니다.

```
OV = 10
DF = 0
DV = 0
BC = 10
V = 10
V' = 0
```



1. 즉시 1코인을 받습니다.

   ```
   BC = 11
   ```

   

2. 시간이 흘러 2코인 수령합니다.

   ```
   V = 8 
   V' = 2
   ```

   

3. 검증자 A에게 4개의 코인을 위임합니다.

   ```
   DV = 4 
   BC = 7
   ```

   

4. 3개의 코인을 보냅니다

   ```
   BC = 4
   ```

   

5. 시간이 흘러 2개의 코인을 더 수령합니다.

   ```
   V = 6 
   V' = 4
   ```

   

6. 2코인을 보냅니다. 이 시점에서 계정은 추가 코인이 베스팅 차거나 추가 코인을 받을 때까지 더 이상 보낼 수 없습니다. 그러나 여전히 위임할 수 있습니다.

   ```
   BC = 2
   ```
   
   



### Slashing

간단한 예와 동일한 초기 시작 조건.

1. 시간이 흘러 5코인 수령

   ```
   V = 5 
   V' = 5
   ```

   

2. 검증자 A에게 5개의 코인을 위임합니다.

   ```
   DV = 5 
   BC = 5
   ```

   

3. 검증자 B에게 5개의 코인을 위임합니다.

   ```
   DF = 5 
   BC = 0
   ```

   

4. 검증인 A가 50% 감소하여 A에 대한 위임은 이제 2.5코인의 가치가 있습니다.

   

5. 검증인 A로부터 위임 취소(코인 2.5개)

   ```
   DF = 5 - 2.5 = 2.5 
   BC = 0 + 2.5 = 2.5
   ```

   

6. 검증자 B로부터 위임을 취소합니다(5코인). 이 시점에서 계정은 더 많은 코인을 받거나 더 많은 코인이 베스팅 찰 때까지 2.5개의 코인만 보낼 수 있습니다. 그러나 여전히 위임할 수 있습니다.

   ```
   DV = 5 - 2.5 = 2.5
   DF = 2.5 - 2.5 = 0 
   BC = 2.5 + 5 = 7.5
   ```

   

   과도한 양의 'DV'가 어떻게 생겼는지 주목하세요.



### Periodic Vesting

100개의 토큰이 1년에 걸쳐 출시될 예정인 베스팅 계정이 생성되며, 각 분기마다 토큰의 1/4이 베스팅됩니다. 베스팅 일정은 다음과 같습니다.

```
Periods:
- amount: 25stake, length: 7884000
- amount: 25stake, length: 7884000
- amount: 25stake, length: 7884000
- amount: 25stake, length: 7884000
```

```
OV = 100
DF = 0
DV = 0
BC = 100
V = 100
V' = 0

```



1. 즉시 1코인을 받습니다.

   ```
   BC = 101
   ```

   

2. 베스팅 기간 1 경과, 25코인 수령

   ```
   V = 75 
   V' = 25
   ```

   

3. 베스팅 기간 2 동안, 5개의 코인이 전송되고 5개의 코인이 위임됩니다.

   ```
   DV = 5 
   BC = 91
   ```

   

4. 베스팅 기간 2 경과, 25코인 수령

   ```
   V = 50 
   V' = 50
   ```

   



## Glossary

- **OriginalVesting**: 초기에 베스팅 계정의 일부인 코인의 양(단위당)입니다. 이 코인은 제네시스에 설정되어 있습니다.
- **StartTime**: 베스팅 계정이 베스팅을 시작하는 BFT 시간입니다.
- **EndTime**: 베스팅 계정이 완전히 베스팅되는 BFT 시간입니다.
- **DelegatedFree**: 위임 시점에 완전히 귀속된 베스팅 계정에서 위임된 추적된 코인(금액당) 금액입니다.
- **DelegatedVesting**: 위임 당시 베스팅 계정에서 위임된 추적된 코인의 양(단위당)입니다.
- **ContinuousVestingAccount**: 시간에 따라 선형적으로 코인을 베스팅하는 베스팅 계정 구현.
- **DelayedVestingAccount**: 주어진 시간에 모든 코인을 완전히 베스팅하는 베스팅 계정 구현.
- **PeriodicVestingAccount**: 맞춤형 베스팅 일정에 따라 코인을 베스팅하는 베스팅 계정 구현.
- **PermanentLockedAccount**: 코인을 방출하지 않고 무기한으로 잠급니다. 이 계정의 코인은 잠겨 있는 동안에도 위임 및 거버넌스 투표에 계속 사용할 수 있습니다.