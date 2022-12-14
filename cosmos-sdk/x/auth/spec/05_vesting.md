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

??? ????????? Cosmos Hub?????? ???????????? ????????? ?????? ????????? ???????????????. ??? ????????? ????????? ?????? ?????? ????????? ?????? ?????? `X`??? ????????? ?????? ?????? `ET`??? ?????? ?????? ?????????????????? ????????? ????????????. ????????? ????????? ????????? ?????? ?????? `ST`??? ?????? ????????? ?????? `P`??? ???????????? ??? ????????????. ????????? ?????? ????????? ????????? ?????? ?????? ????????? ????????? ????????? ????????? ????????? ???????????? ????????????. ????????? ????????? ????????? ?????? ???????????? ????????? ?????? ?????? ???????????????.

?????? ????????? ????????? ?????? ????????? ????????? ???????????? ??????????????? ?????? ??? ????????? ????????? ??? ????????? ?????? ????????? ???????????? ????????? ?????? ???????????? ????????? ????????? ??? ????????????. ??? ????????? ??? ?????? ????????? ???????????? ???????????????.

- `ET`??? ???????????? ?????? ????????? ???????????? ?????? ????????? (Delayed vesting)
- `ST`?????? ???????????? ???????????? `ET`??? ????????? ????????? ????????? ?????? ??????????????? ????????? ???????????? ???????????? ????????? (Continuous vesting)
-  `ST`??? ??????????????? ???????????? ?????? ?????? ????????? ????????? ????????? ?????? ??????????????? ????????? ???????????? ????????? ????????? (Periodic vesting)
  - ?????? ???, ?????? ??? ?????? ??? ?????? ??? ????????? ????????? ??? ????????????. ???????????? ????????? ????????? ????????? ????????? ??? ???????????? ????????? ??? ????????? ????????? ???????????? ????????? ????????? ???????????????. ?????? ??????, ???????????? ????????? ????????? ????????? ?????????, ?????? ?????? ????????? ????????? ?????? ????????? ?????? ????????? ?????? ??????????????? ????????? ????????? ????????? ??? ????????????.

- ??????????????? ?????? ????????? (Permanent locked vesting)
  - ????????? ????????? ?????? ????????????. ??? ????????? ????????? ?????? ?????? ???????????? ?????? ??? ???????????? ????????? ?????? ????????? ??? ????????????.




## Note

????????? ????????? ?????? ????????? ??? ???-????????? ???????????? ???????????? ??? ????????????. ???-????????? ????????? ?????? ????????? ??? ????????????. DelayedVesting ??? ContinuousVesting ????????? ???????????? ?????? ?????? ???????????? ????????? ??? ????????????. ?????? ????????? ????????? ????????? ?????? ??? ?????? ?????? ???????????? ?????????????????? ????????? ???????????? ?????????. ?????? ????????? *unconditional* ???????????? ???????????????(???, 'ET'??? ???????????? ????????? ??????????????? ?????? ???????????? ??????).



## Vesting Account Types

```go
// VestingAccount??? ?????? ????????? ?????? ????????? ???????????? ?????? ?????????????????? ???????????????.
type VestingAccount interface {
  Account

  GetVestedCoins(Time)  Coins
  GetVestingCoins(Time) Coins

  // TrackDelegation??? ????????? ???????????? ????????? ??? ????????? ?????? ????????? ????????? ???????????????. 
  // ?????? ?????? ????????? ????????????, ?????? ????????? ????????? ?????? ????????? ???????????? ?????? ????????? ?????? ?????? ?????? ????????? ???????????????.
  TrackDelegation(Time, Coins, Coins)

  // TrackUndelegation??? ????????? ????????? ?????? ????????? ????????? ??? ????????? ?????? ????????? ????????? ???????????????.
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

?????? ?????? ????????? assertions??? ????????? ?????? ?????? ????????? ???????????? ???????????? ?????? ?????? `x/bank` `ViewKeeper` ?????????????????? ????????? ??????????????? ???????????????????????????.

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

????????? ????????? ???????????? ?????? ???????????? ????????? ???????????????.

- `OV`: ?????? ????????? (Original Vesting) ?????? ????????????. ????????? ????????????.
- `V`: ?????? *?????????* ?????? `OV` ????????? ????????????. `OV`, `StartTime` ??? `EndTime`??? ?????? ???????????????. ??? ?????? ?????? ????????? ????????? ?????? ??? ???????????????.
- `V'`: *?????????*(?????? ??????)??? `OV` ????????? ????????????. ??? ?????? ?????? ????????? ????????? ?????? ??? ???????????????.
- `DV`: ????????? *?????????* (Delegated Vesting) ????????? ????????????. ?????? ????????????. ????????? ????????? ?????? ?????? ??? ???????????????.
- `DF`: ????????? *?????????*(?????? ??????, DelegatedFree) ????????? ????????????. ?????? ????????????. ????????? ????????? ?????? ?????? ??? ???????????????.
- `BC`: `OV` ???????????? ????????? ????????? ??? ???(?????? ?????? ?????????). ????????? ?????? ????????? ???????????? ???????????????. ????????? ????????? ?????? ?????? ??? ???????????????.



### Determining Vesting & Vested Amounts

????????? ?????? ?????? ?????? ????????? ????????? ?????? ??? ??????????????? ?????? ???????????? ?????????(???: `BeginBlocker` ?????? `EndBlocker`).



#### Continuously Vesting Accounts

????????? ?????? ?????? `T` ?????? ???????????? ????????? ?????? ???????????? ?????? ????????? ???????????????.

1. Compute `X := T - StartTime`
2. Compute `Y := EndTime - StartTime`
3. Compute `V' := OV * (X / Y)`
4. Compute `V := OV - V'`

????????? ????????? ????????? ??? ????????? `V'`?????? ????????? ????????? `V`??? *?????????* ??? ?????????.

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

???????????? ????????? ????????? ????????? ?????? ?????? `T` ??? ?????? ??? ?????? ?????? ???????????? ????????? ???????????? ?????????. `GetVestedCoins`??? ????????? ??? ?????? ????????? ???????????? ??? ???????????? ?????? ????????? `T` ????????? ??? ????????? ??? ????????? ???????????? ?????????.

1. Set `CT := StartTime`
2. Set `V' := 0`



??? ?????? P??? ?????????:

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

????????? ????????? ????????? ?????? ???????????? ?????? ????????? ???????????? ?????? ?????? ????????? ?????????(?????? ??????)?????? ????????? ???????????? ????????????. ???????????? ????????? ????????? ?????? ??? ?????? ?????? ????????? ????????? ???????????? ????????????.

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

????????? ????????? ????????? ????????? `min((BC + DV) - V, BC)`??? ????????? ??? ????????????.

???, ????????? ????????? ?????? ?????? ??????, ?????? ?????? ????????? ?????? ????????? ????????? ?????? ?????? ????????? ???????????? ???????????? ?????? ?????? ??? ?????? ??? ?????? ?????? ????????? ??? ??????.

????????? ?????? ????????? `x/bank` ????????? ?????? ???????????? ?????? ?????? ????????? ???????????? ???????????? ?????? ?????? `max(V - DV, 0)`??? ????????? ??? ?????? ?????? ????????? ????????? ??? ????????????. `, ????????? ?????????????????? ?????? ????????? ????????? ???????????????.

```go
func (va VestingAccount) LockedCoins(t Time) Coins {
   return max(va.GetVestingCoins(t) - va.DelegatedVesting, 0)
}
	
```



????????? `x/bank` `ViewKeeper`??? API??? ???????????? ?????? ????????? ?????? ?????? ????????? ?????? ????????? ????????? ????????? ??? ????????????.

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

?????? 'x/bank' ????????? ?????? ????????? ????????? ???????????? ????????? ?????? ?????? ????????? ???????????? ???????????? ?????????.

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

`D` ????????? ??????????????? ????????? ????????? ?????? ????????? ???????????????.

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

**??????** `TrackDelegation`??? `DelegatedVesting` ??? `DelegatedFree` ????????? ??????????????? ???????????? ???????????? `amount`??? ?????? `Coins` ????????? ???????????? ?????????.



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

'D' ????????? ????????? ??????????????? ????????? ????????? ?????? ????????? ???????????????. ??????: ??????/?????? ?????? ????????? ????????? ????????? ?????? `DV < D` ??? `(DV + DF) < D`??? ????????? ??? ????????????.

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



**??????** `TrackUnDelegation`??? `DelegatedVesting` ??? `DelegatedFree` ????????? ??????????????? ???????????? ???????????? `amount`??? ???????????? `Coins` ????????? ???????????? ?????????.

**??????**: ????????? ???????????? ?????? ????????? ???????????? ????????? ???????????? ????????? ????????? ?????? `DV` ???????????? ????????????. ?????? ????????? ?????? ???????????? ?????? ???????????? ???????????????.

**??????**: ?????? ??????(?????? ??????) ????????? ?????? ????????? ?????? ????????? ????????? ???????????? ?????? ????????? ?????????(??????) ????????? ????????? ??? ?????????, ???????????? ?????? ????????? ?????? ?????? ???????????? ??????(tokens/shares)??? ?????? ????????? ??? ????????????. 



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

`VestingAccount` ????????? `x/auth`??? ????????????. ????????? ??????????????? ????????? ????????? ??????????????? ????????? ??????(???: `x/staking`??? ????????????)??? `SendCoins` ??? `SubtractCoins`??? ???????????? `x/bank` ????????? ?????? ????????? ????????? (e.g. `DelegateCoins`)??? ???????????? ?????????.

?????? ????????? ????????? ?????? ?????????????????? ?????? ?????? ????????? ????????? ??? ????????? ?????????. ????????? ?????? ????????? 'MsgSend' ???????????? ????????? ????????? ?????? ????????? ?????? ????????? ???????????? ????????? ???????????? ?????? ????????? ???????????? ?????????.

?????? ?????? ?????? ????????? ?????? ????????? ??????????????????.



## Genesis Initialization

????????? ????????? ???-????????? ????????? ?????? ??????????????? ?????? `GenesisAccount` ??????????????? `Vesting`, `StartTime` ??? `EndTime`??? ?????? ??? ????????? ???????????????. `BaseAccount` ?????? ?????? ???-????????? ????????? ???????????? `Vesting = false`??? ????????????. ???????????? ????????? ??????(???: `initFromGenesisState`)??? ????????? ????????? ???????????? ????????? ????????? ?????? ???????????? ???????????? ?????????.



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

10?????? ????????? ????????? ?????? ???????????? ????????? ????????? ???????????????.

```
OV = 10
DF = 0
DV = 0
BC = 10
V = 10
V' = 0
```



1. ?????? 1????????? ????????????.

   ```
   BC = 11
   ```

   

2. ????????? ?????? 2?????? ???????????????.

   ```
   V = 8 
   V' = 2
   ```

   

3. ????????? A?????? 4?????? ????????? ???????????????.

   ```
   DV = 4 
   BC = 7
   ```

   

4. 3?????? ????????? ????????????

   ```
   BC = 4
   ```

   

5. ????????? ?????? 2?????? ????????? ??? ???????????????.

   ```
   V = 6 
   V' = 4
   ```

   

6. 2????????? ????????????. ??? ???????????? ????????? ?????? ????????? ????????? ????????? ?????? ????????? ?????? ????????? ??? ?????? ?????? ??? ????????????. ????????? ????????? ????????? ??? ????????????.

   ```
   BC = 2
   ```
   
   



### Slashing

????????? ?????? ????????? ?????? ?????? ??????.

1. ????????? ?????? 5?????? ??????

   ```
   V = 5 
   V' = 5
   ```

   

2. ????????? A?????? 5?????? ????????? ???????????????.

   ```
   DV = 5 
   BC = 5
   ```

   

3. ????????? B?????? 5?????? ????????? ???????????????.

   ```
   DF = 5 
   BC = 0
   ```

   

4. ????????? A??? 50% ???????????? A??? ?????? ????????? ?????? 2.5????????? ????????? ????????????.

   

5. ????????? A????????? ?????? ??????(?????? 2.5???)

   ```
   DF = 5 - 2.5 = 2.5 
   BC = 0 + 2.5 = 2.5
   ```

   

6. ????????? B????????? ????????? ???????????????(5??????). ??? ???????????? ????????? ??? ?????? ????????? ????????? ??? ?????? ????????? ????????? ??? ????????? 2.5?????? ????????? ?????? ??? ????????????. ????????? ????????? ????????? ??? ????????????.

   ```
   DV = 5 - 2.5 = 2.5
   DF = 2.5 - 2.5 = 0 
   BC = 2.5 + 5 = 7.5
   ```

   

   ????????? ?????? 'DV'??? ????????? ???????????? ???????????????.



### Periodic Vesting

100?????? ????????? 1?????? ?????? ????????? ????????? ????????? ????????? ????????????, ??? ???????????? ????????? 1/4??? ??????????????????. ????????? ????????? ????????? ????????????.

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



1. ?????? 1????????? ????????????.

   ```
   BC = 101
   ```

   

2. ????????? ?????? 1 ??????, 25?????? ??????

   ```
   V = 75 
   V' = 25
   ```

   

3. ????????? ?????? 2 ??????, 5?????? ????????? ???????????? 5?????? ????????? ???????????????.

   ```
   DV = 5 
   BC = 91
   ```

   

4. ????????? ?????? 2 ??????, 25?????? ??????

   ```
   V = 50 
   V' = 50
   ```

   



## Glossary

- **OriginalVesting**: ????????? ????????? ????????? ????????? ????????? ???(?????????)?????????. ??? ????????? ??????????????? ???????????? ????????????.
- **StartTime**: ????????? ????????? ???????????? ???????????? BFT ???????????????.
- **EndTime**: ????????? ????????? ????????? ??????????????? BFT ???????????????.
- **DelegatedFree**: ?????? ????????? ????????? ????????? ????????? ???????????? ????????? ????????? ??????(?????????) ???????????????.
- **DelegatedVesting**: ?????? ?????? ????????? ???????????? ????????? ????????? ????????? ???(?????????)?????????.
- **ContinuousVestingAccount**: ????????? ?????? ??????????????? ????????? ??????????????? ????????? ?????? ??????.
- **DelayedVestingAccount**: ????????? ????????? ?????? ????????? ????????? ??????????????? ????????? ?????? ??????.
- **PeriodicVestingAccount**: ????????? ????????? ????????? ?????? ????????? ??????????????? ????????? ?????? ??????.
- **PermanentLockedAccount**: ????????? ???????????? ?????? ??????????????? ????????????. ??? ????????? ????????? ?????? ?????? ???????????? ?????? ??? ???????????? ????????? ?????? ????????? ??? ????????????.