# State

## Accounts

Accounts contain authentication information for a uniquely identified external user of an SDK blockchain, including public key, address, and account number / sequence number for replay protection. For efficiency, since account balances must also be fetched to pay fees, account structs also store the balance of a user as `sdk.Coins`.

Accounts are exposed externally as an interface, and stored internally as either a base account or vesting account. Module clients wishing to add more account types may do so.

계정에는 리플레이 프로텍션을 위한 공개 키, 주소 및 계정 번호/순서 번호를 포함하여 SDK 블록체인의 고유하게 식별된 외부 사용자에 대한 인증 정보가 포함됩니다. 효율성을 위해 수수료를 지불하기 위해 계정 잔액도 가져와야 하므로 계정 구조체는 사용자의 잔액도 `sdk.Coins`로 저장합니다.

계정은 외부적으로 인터페이스로 노출되고 내부적으로는 베이스 어카운트 또는 베스팅 어카운트로 저장됩니다. 더 많은 계정 유형을 추가하려는 모듈 클라이언트는 그렇게 할 수 있습니다.

- `0x01 | Address -> ProtocolBuffer(account)`



### Account Interface

계정 인터페이스는 표준 계정 정보를 읽고 쓰는 메소드를 제공합니다. 이 모든 메서드는 스토어에 계정을 쓰기 위해 인터페이스에 확인되는 계정 구조에서 작동하므로, 어카운트 키퍼를 사용해야 합니다.

```go
// AccountI는 상태 내의 주어진 주소에 코인을 저장하는 데 사용되는 인터페이스입니다. 
// 리플레이 프로텍션를 위한 시퀀스 번호 개념, 
// 이전에 정리된 계정에 대한 리플레이 프로텍션을 위한 계정 번호 개념, 
// 인증 목적을 위한 pubkey를 가정합니다.
// 
// AccountI를 구현하는 구체적인 구조체에는 많은 복잡한 조건이 사용될 수 있습니다.
type AccountI interface {
	proto.Message

	GetAddress() sdk.AccAddress
	SetAddress(sdk.AccAddress) error // errors if already set.

	GetPubKey() crypto.PubKey // can return nil.
	SetPubKey(crypto.PubKey) error

	GetAccountNumber() uint64
	SetAccountNumber(uint64) error

	GetSequence() uint64
	SetSequence(uint64) error

	// Ensure that account implements stringer
	String() string
}

```



#### Base Account

베이스 어카운트(기본 계정)은 모든 필수 필드를 구조체에 직접 저장하는 가장 단순하고 일반적인 계정 타입입니다.

```go
// BaseAccount는 기본 계정 타입을 정의합니다. 기본 계정 기능에 필요한 모든 필드가 포함되어 있습니다. 
// 모든 사용자 정의 계정 타입은 추가 기능(예: vesting)을 위해 이 타입을 확장해야 합니다.
message BaseAccount {
  string address = 1;
  google.protobuf.Any pub_key = 2;
  uint64 account_number = 3;
  uint64 sequence       = 4;
}

```





### Vesting Account

[베스팅](https://docs.cosmos.network/v0.46/modules/auth/05_vesting.html)을 참조하세요.