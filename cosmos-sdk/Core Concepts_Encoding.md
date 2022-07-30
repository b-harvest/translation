# Encoding



**개요**

Cosmos SDK의 인코딩은 주로 `go-amino` 코덱으로 처리되었지만 이제 Cosmos SDK는 상태 및 클라이언트 측 인코딩 모두에 `gogoprotobuf`를 사용하는 방향으로 이동하고 있습니다.



## Encoding

Cosmos SDK는 객체 인코딩 사양인 [Amino](https://github.com/tendermint/go-amino/)와 [Protocol Buffers](https://developers.google.com/protocol-buffers) 두 가지 바이너리 와이어 인코딩 프로토콜을 사용합니다. Protocol Buffers는 Proto3의 하위 집합으로 인터페이스 지원을 위한 확장 기능을 가집니다. Amino가 대부분 호환되는 Proto3는 Amino와 대부분 호환(Proto2와는 호환되지 않음)됩니다. 이에 대한 자세한 내용은 [Proto3 사양](https://developers.google.com/protocol-buffers/docs/proto3)을 참조하세요.

리플렉션 기반인 Amino는 의미 있는 교차 언어/클라이언트 지원이 없고, 심각한 성능 결점이 있기 때문에 Protocol Buffer, 특히 [gogoprotobuf](https://github.com/gogo/protobuf/)가 Amino 대신 사용됩니다. Amino를 Protocol Buffer로 대체하는 프로세스는 여전히 진행 중입니다.

Cosmos SDK 내 타입(types)에 대한 바이너리 와이어 인코딩은 클라이언트 인코딩과 스토어 인코딩 등 두 가지 주요 범주로 나눌 수 있습니다. 클라이언트 인코딩은 주로 트랜잭션 처리 및 사이닝을 중심으로 진행되는 반면 스토어 인코딩은 상태 머신 이행에 사용되는 타입과 궁극적으로 머클 트리에 저장되는 내용을 중심으로 진행됩니다.

스토어 인코딩의 경우, protobuf 정의는 모든 타입에 대해 존재할 수 있으며 일반적으로 Amino 기반 "중개" 타입 갖습니다. 특히 protobuf 기반 타입 정의는 직렬화와 지속성에 사용되는 반면 Amino 기반 타입은 전후(back-n-forth)로 변환할 수 있는 상태 머신의 비즈니스 로직에 사용됩니다. Amino 기반 유형은 향후 단계적으로 중단될 수 있으므로 개발자는 가능하면 protobuf 메시지 정의를 사용하도록 주의해야 합니다.

`codec` 패키지에는 `BinaryCodec`, `JSONCodec` 등 두 가지 핵심 인터페이스가 있습니다. BinaryCodec은 일반 `interface{}` 타입 대신 JSONCodec을 구현하는 타입에서 작동한다는 점을 제외하고 현재 Amino 인터페이스를 캡슐화합니다.

또한 `Codec`에는 두 가지 구현이 있습니다. 첫 번째는 바이너리 및 JSON 직렬화가 Amino를 통해 처리되는 `AminoCodec`입니다. 두 번째는 바이너리 및 JSON 직렬화가 Protobuf를 통해 처리되는 `ProtoCodec`입니다.

즉, 모듈은 Amino 또는 Protobuf 인코딩을 사용할 수 있지만 타입은 `ProtoMarshaler`를 구현해야 합니다. 모듈이 자신의 타입에 대해 이 인터페이스를 구현하지 않으려면 Amino 코덱을 직접 사용할 수 있습니다.



### Amino

모든 모듈은 타입과 인터페이스를 직렬화하는데 Amino를 사용합니다. 이 코덱에는 일반적으로 해당 모듈의 도메인(예: 메시지)에만 등록된 타입과 인터페이스가 있지만, `x/gov`와 같은 예외도 있습니다. 각 모듈은 사용자가 코덱을 제공하고 모든 타입을 등록할 수 있도록 하는 `RegisterLegacyAminoCodec` 기능을 노출합니다. 애플리케이션은   필요하다면 각 모듈에서 이 메서드를 호출합니다.

모듈에 대한 protobuf 기반 타입이 없는 경우 (아래 코드 참고), 원시 와이어 바이트를 구체적인 타입이나 인터페이스르 인코딩/디코딩하는데 Amino를 사용합니다.

```go
bz := keeper.cdc.MustMarshal(typeOrInterface)
keeper.cdc.MustUnmarshal(bz, &typeOrInterface)
```

위 기능에는 앞 부분 길이가 다른 변형들이 있고  이는 일반적으로 데이터를 스트리밍하거나 함께 그룹화해야 할 때 사용됩니다(예: `ResponseDeliverTx.Data`).



#### Authz authorizations

Since the `MsgExec` message type can contain different messages instances, it is important that developers add the following code inside the `init` method of their module's `codec.go` file:

`MsgExec` 메시지 타입에는 다른 메시지 인스턴스가 포함될 수 있으므로 개발자가 모듈의 `codec.go` 파일의 `init` 메소드 안에 다음 코드를 추가하는 것이 중요합니다.

```go
import authzcodec "github.com/cosmos/cosmos-sdk/x/authz/codec"

init() {
    // Register all Amino interfaces and concrete types on the authz Amino codec so that this can later be
    // used to properly serialize MsgGrant and MsgExec instances
    RegisterLegacyAminoCodec(authzcodec.Amino)
}

```



이렇게 하면 `x/authz` 모듈이 Amino를 사용하여 `MsgExec` 인스턴스를 올바르게 직렬화 및 역직렬화할 수 있습니다. 이는 Ledger를 사용하여 이런 종류의 메시지에 사이닝할 때 필요합니다.



### Gogoproto

모듈은 자신의 타입에 Protobuf 인코딩을 사용하는 것이 좋습니다. Cosmos SDK에서는 공식 [Google protobuf implementation](https://github.com/protocolbuffers/protobuf) 에 비해 속도 및 DX 개선을 제공하는 Protobuf 스펙의  [Gogoproto](https://github.com/gogo/protobuf) 특정 구현을 사용합니다. 



### Guidelines for protobuf message definitions

In addition to [following official Protocol Buffer guidelines](https://developers.google.com/protocol-buffers/docs/proto3#simple), we recommend using these annotations in .proto files when dealing with interfaces:

[공식 프로토콜 버퍼 지침 가이드라인](https://developers.google.com/protocol-buffers/docs/proto3#simple) 외에도 인터페이스를 처리할 때 .proto 파일에서 다음 주석을 사용하는 것이 좋습니다.

- 'cosmos_proto.accepts_interface'를 사용하여 인터페이스를 허용하는 필드를 기록합니다.
- `protoName`과 동일한 정규화된 이름을 `InterfaceRegistry.RegisterInterface`에 전달합니다.
- `cosmos_proto.implements_interface`로 인터페이스 구현에 주석을 답니다.



### Transaction Encoding

Another important use of Protobuf is the encoding and decoding of [transactions](https://docs.cosmos.network/v0.46/core/transactions.html). Transactions are defined by the application or the Cosmos SDK but are then passed to the underlying consensus engine to be relayed to other peers. Since the underlying consensus engine is agnostic to the application, the consensus engine accepts only transactions in the form of raw bytes.

Protobuf의 또 다른 중요한 용도는 [transactions](https://docs.cosmos.network/v0.46/core/transactions.html)의 인코딩 및 디코딩입니다. 트랜잭션은 애플리케이션 또는 Cosmos SDK에 의해 정의되지만 기본 합의 엔진으로 전달되어 다른 피어에게 전달됩니다. 기본 컨센서스 엔진은 애플리케이션-애그노스틱하기 때문에 컨센서스 엔진은 원시 바이트 형태의 트랜잭션만 허용합니다.

- `TxEncoder` 객체가 인코딩을 수행합니다.
- `TxDecoder` 객체가 디코딩을 수행합니다.

```
// TxDecoder unmarshals transaction bytes
type TxDecoder func(txBytes []byte) (Tx, error)

// TxEncoder marshals transaction to bytes
type TxEncoder func(tx Tx) ([]byte, error)
```



이 두 객체의 표준 구현은 [`auth` 모듈](https://docs.cosmos.network/v0.46/x/auth/spec/)에서 찾을 수 있습니다.

```go
package tx

import (
	"fmt"

	"google.golang.org/protobuf/encoding/protowire"

	"github.com/cosmos/cosmos-sdk/codec"
	"github.com/cosmos/cosmos-sdk/codec/unknownproto"
	sdk "github.com/cosmos/cosmos-sdk/types"
	sdkerrors "github.com/cosmos/cosmos-sdk/types/errors"
	"github.com/cosmos/cosmos-sdk/types/tx"
)

// DefaultTxDecoder returns a default protobuf TxDecoder using the provided Marshaler.
func DefaultTxDecoder(cdc codec.ProtoCodecMarshaler) sdk.TxDecoder {
	return func(txBytes []byte) (sdk.Tx, error) {
		// Make sure txBytes follow ADR-027.
		err := rejectNonADR027TxRaw(txBytes)
		if err != nil {
			return nil, sdkerrors.Wrap(sdkerrors.ErrTxDecode, err.Error())
		}

		var raw tx.TxRaw

		// reject all unknown proto fields in the root TxRaw
		err = unknownproto.RejectUnknownFieldsStrict(txBytes, &raw, cdc.InterfaceRegistry())
		if err != nil {
			return nil, sdkerrors.Wrap(sdkerrors.ErrTxDecode, err.Error())
		}

		err = cdc.Unmarshal(txBytes, &raw)
		if err != nil {
			return nil, err
		}

		var body tx.TxBody

		// allow non-critical unknown fields in TxBody
		txBodyHasUnknownNonCriticals, err := unknownproto.RejectUnknownFields(raw.BodyBytes, &body, true, cdc.InterfaceRegistry())
		if err != nil {
			return nil, sdkerrors.Wrap(sdkerrors.ErrTxDecode, err.Error())
		}

		err = cdc.Unmarshal(raw.BodyBytes, &body)
		if err != nil {
			return nil, sdkerrors.Wrap(sdkerrors.ErrTxDecode, err.Error())
		}

		var authInfo tx.AuthInfo

		// reject all unknown proto fields in AuthInfo
		err = unknownproto.RejectUnknownFieldsStrict(raw.AuthInfoBytes, &authInfo, cdc.InterfaceRegistry())
		if err != nil {
			return nil, sdkerrors.Wrap(sdkerrors.ErrTxDecode, err.Error())
		}

		err = cdc.Unmarshal(raw.AuthInfoBytes, &authInfo)
		if err != nil {
			return nil, sdkerrors.Wrap(sdkerrors.ErrTxDecode, err.Error())
		}

		theTx := &tx.Tx{
			Body:       &body,
			AuthInfo:   &authInfo,
			Signatures: raw.Signatures,
		}

		return &wrapper{
			tx:                           theTx,
			bodyBz:                       raw.BodyBytes,
			authInfoBz:                   raw.AuthInfoBytes,
			txBodyHasUnknownNonCriticals: txBodyHasUnknownNonCriticals,
		}, nil
	}
}

// DefaultJSONTxDecoder returns a default protobuf JSON TxDecoder using the provided Marshaler.
func DefaultJSONTxDecoder(cdc codec.ProtoCodecMarshaler) sdk.TxDecoder {
	return func(txBytes []byte) (sdk.Tx, error) {
		var theTx tx.Tx
		err := cdc.UnmarshalJSON(txBytes, &theTx)
		if err != nil {
			return nil, sdkerrors.Wrap(sdkerrors.ErrTxDecode, err.Error())
		}

		return &wrapper{
			tx: &theTx,
		}, nil
	}
}

// rejectNonADR027TxRaw rejects txBytes that do not follow ADR-027. This is NOT
// a generic ADR-027 checker, it only applies decoding TxRaw. Specifically, it
// only checks that:
// - field numbers are in ascending order (1, 2, and potentially multiple 3s),
// - and varints are as short as possible.
// All other ADR-027 edge cases (e.g. default values) are not applicable with
// TxRaw.
func rejectNonADR027TxRaw(txBytes []byte) error {
	// Make sure all fields are ordered in ascending order with this variable.
	prevTagNum := protowire.Number(0)

	for len(txBytes) > 0 {
		tagNum, wireType, m := protowire.ConsumeTag(txBytes)
		if m < 0 {
			return fmt.Errorf("invalid length; %w", protowire.ParseError(m))
		}
		// TxRaw only has bytes fields.
		if wireType != protowire.BytesType {
			return fmt.Errorf("expected %d wire type, got %d", protowire.BytesType, wireType)
		}
		// Make sure fields are ordered in ascending order.
		if tagNum < prevTagNum {
			return fmt.Errorf("txRaw must follow ADR-027, got tagNum %d after tagNum %d", tagNum, prevTagNum)
		}
		prevTagNum = tagNum

		// All 3 fields of TxRaw have wireType == 2, so their next component
		// is a varint, so we can safely call ConsumeVarint here.
		// Byte structure: <varint of bytes length><bytes sequence>
		// Inner  fields are verified in `DefaultTxDecoder`
		lengthPrefix, m := protowire.ConsumeVarint(txBytes[m:])
		if m < 0 {
			return fmt.Errorf("invalid length; %w", protowire.ParseError(m))
		}
		// We make sure that this varint is as short as possible.
		n := varintMinLength(lengthPrefix)
		if n != m {
			return fmt.Errorf("length prefix varint for tagNum %d is not as short as possible, read %d, only need %d", tagNum, m, n)
		}

		// Skip over the bytes that store fieldNumber and wireType bytes.
		_, _, m = protowire.ConsumeField(txBytes)
		if m < 0 {
			return fmt.Errorf("invalid length; %w", protowire.ParseError(m))
		}
		txBytes = txBytes[m:]
	}

	return nil
}

// varintMinLength returns the minimum number of bytes necessary to encode an
// uint using varint encoding.
func varintMinLength(n uint64) int {
	switch {
	// Note: 1<<N == 2**N.
	case n < 1<<(7):
		return 1
	case n < 1<<(7*2):
		return 2
	case n < 1<<(7*3):
		return 3
	case n < 1<<(7*4):
		return 4
	case n < 1<<(7*5):
		return 5
	case n < 1<<(7*6):
		return 6
	case n < 1<<(7*7):
		return 7
	case n < 1<<(7*8):
		return 8
	case n < 1<<(7*9):
		return 9
	default:
		return 10
	}
}
```



```
package tx

import (
	"fmt"

	"github.com/gogo/protobuf/proto"

	"github.com/cosmos/cosmos-sdk/codec"
	sdk "github.com/cosmos/cosmos-sdk/types"
	txtypes "github.com/cosmos/cosmos-sdk/types/tx"
)

// DefaultTxEncoder returns a default protobuf TxEncoder using the provided Marshaler
func DefaultTxEncoder() sdk.TxEncoder {
	return func(tx sdk.Tx) ([]byte, error) {
		txWrapper, ok := tx.(*wrapper)
		if !ok {
			return nil, fmt.Errorf("expected %T, got %T", &wrapper{}, tx)
		}

		raw := &txtypes.TxRaw{
			BodyBytes:     txWrapper.getBodyBytes(),
			AuthInfoBytes: txWrapper.getAuthInfoBytes(),
			Signatures:    txWrapper.tx.Signatures,
		}

		return proto.Marshal(raw)
	}
}

// DefaultJSONTxEncoder returns a default protobuf JSON TxEncoder using the provided Marshaler.
func DefaultJSONTxEncoder(cdc codec.ProtoCodecMarshaler) sdk.TxEncoder {
	return func(tx sdk.Tx) ([]byte, error) {
		txWrapper, ok := tx.(*wrapper)
		if ok {
			return cdc.MarshalJSON(txWrapper.tx)
		}

		protoTx, ok := tx.(*txtypes.Tx)
		if ok {
			return cdc.MarshalJSON(protoTx)
		}

		return nil, fmt.Errorf("expected %T, got %T", &wrapper{}, tx)
	}
}
```



트랜잭션이 인코딩되는 방법에 대한 자세한 내용은 [ADR-020](https://docs.cosmos.network/v0.46/architecture/adr-020-protobuf-transaction-encoding.html)을 참조하세요.



### Interface Encoding and Usage of `Any`

The Protobuf DSL is strongly typed, which can make inserting variable-typed fields difficult. Imagine we want to create a `Profile` protobuf message that serves as a wrapper over [an account](https://docs.cosmos.network/v0.46/basics/accounts.html):

Protobuf DSL은 강-타입(strong type) 언어라서 변수 타입 필드를 삽입하기 어려울 수 있습니다. [accounts](https://docs.cosmos.network/v0.46/basics/accounts.html)에 대한 래퍼 역할을 하는 `Profile` protobuf 메시지를 생성한다고 상상해 보세요.

```go
message Profile {
  // account is the account associated to a profile.
  cosmos.auth.v1beta1.BaseAccount account = 1;
  // bio is a short description of the account.
  string bio = 4;
}
```



이 `Profile` 예에서는 `account`를 `BaseAccount`로 하드코딩했습니다. 그러나 `BaseVestingAccount` 또는 `ContinuousVestingAccount`와 같은 [베스팅과 관련된 사용자 계정](https://docs.cosmos.network/v0.46/x/auth/spec/05_vesting.html)의 다른 유형이 몇 가지 더 있습니다. 이 계정들은 서로 다르지만 모두 `AccountI` 인터페이스를 구현합니다. `AccountI` 인터페이스를 허용하는 `account` 필드가 있는 계정 타입들을 모두 허용하는 `Profile`을 어떻게 만드시겠습니까?

```
// AccountI is an interface used to store coins at a given address within state.
// It presumes a notion of sequence numbers for replay protection,
// a notion of account numbers for replay protection for previously pruned accounts,
// and a pubkey for authentication purposes.
//
// Many complex conditions can be used in the concrete struct which implements AccountI.
type AccountI interface {
	proto.Message

	GetAddress() sdk.AccAddress
	SetAddress(sdk.AccAddress) error // errors if already set.

	GetPubKey() cryptotypes.PubKey // can return nil.
	SetPubKey(cryptotypes.PubKey) error

	GetAccountNumber() uint64
	SetAccountNumber(uint64) error

	GetSequence() uint64
	SetSequence(uint64) error

	// Ensure that account implements stringer
	String() string
}
```



[ADR-019](https://docs.cosmos.network/v0.46/architecture/adr-019-protobuf-state-encoding.html)에서는 protobuf에서 인터페이스를 인코딩하기 위해 `Any`를 사용합니다. `Any`는 직렬화된 임의의 메시지를 바이트로 포함하며 해당 메시지 타입에 대해 전역적으로 고유한 식별자 역할을 하고 해당 메시지 타입으로 해석되는 URL을 포함합니다. 이 전략을 사용하면 protobuf 메시지 내부에 임의의 Go 타입을 패킹할 수 있습니다. 그러면 새로운 `Profile`이 다음과 같이 표시됩니다.

```
message Profile {
  // account is the account associated to a profile.
  google.protobuf.Any account = 1 [
    (cosmos_proto.accepts_interface) = "AccountI"; // Asserts that this field only accepts Go types implementing `AccountI`. It is purely informational for now.
  ];
  // bio is a short description of the account.
  string bio = 4;
}

```



프로필 내부에 계정을 추가하려면 먼저 `codectypes.NewAnyWithValue`를 사용하여 `Any` 내부에 계정을 패킹해야 합니다.

```
var myAccount AccountI
myAccount = ... // Can be a BaseAccount, a ContinuousVestingAccount or any struct implementing `AccountI`

// Pack the account into an Any
accAny, err := codectypes.NewAnyWithValue(myAccount)
if err != nil {
  return nil, err
}

// Create a new Profile with the any.
profile := Profile {
  Account: accAny,
  Bio: "some bio",
}

// We can then marshal the profile as usual.
bz, err := cdc.Marshal(profile)
jsonBz, err := cdc.MarshalJSON(profile)

```



요약하자면, 인터페이스를 인코딩하려면 1) 인터페이스를 `Any`로 압축하고 2) `Any`를 마샬링해야 합니다. 편의를 위해 Cosmos SDK는 이 두 단계를 묶을 수 있는 `MarshalInterface` 메소드를 제공합니다. [x/auth 모듈의 실제 예](https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/x/auth/keeper/keeper.go# L230-L233)를 참고하세요.



 `Any` 내부에서 구체적인 Go 유형을 검색하는 "unpacking"이라고 하는 리버스 오퍼레이션은 `Any`의 `GetCachedValue()`로 수행됩니다.

```
profileBz := ... // The proto-encoded bytes of a Profile, e.g. retrieved through gRPC.
var myProfile Profile
// Unmarshal the bytes into the myProfile struct.
err := cdc.Unmarshal(profilebz, &myProfile)

// Let's see the types of the Account field.
fmt.Printf("%T\n", myProfile.Account)                  // Prints "Any"
fmt.Printf("%T\n", myProfile.Account.GetCachedValue()) // Prints "BaseAccount", "ContinuousVestingAccount" or whatever was initially packed in the Any.

// Get the address of the accountt.
accAddr := myProfile.Account.GetCachedValue().(AccountI).GetAddress()

```



It is important to note that for `GetCachedValue()` to work, `Profile` (and any other structs embedding `Profile`) must implement the `UnpackInterfaces` method:

`GetCachedValue()`가 작동하려면 `Profile`(과 `Profile`을 포함하는 다른 모든 구조체)이 `UnpackInterfaces` 메서드를 구현해야 합니다.

```
func (p *Profile) UnpackInterfaces(unpacker codectypes.AnyUnpacker) error {
  if p.Account != nil {
    var account AccountI
    return unpacker.UnpackAny(p.Account, &account)
  }

  return nil
}

```



`UnpackInterfaces`는 이 메서드를 구현하는 모든 구조체에서 재귀적으로 호출되어 모든 'Any'가 `GetCachedValue()`를 올바르게 채울 수 있도록 합니다.

인터페이스 인코딩, 특히 `UnpackInterfaces`에 대한 자세한 내용과 `InterfaceRegistry`를 사용하여 `Any`의 `type_url`을 확인하는 방법은 [ADR-019](https://docs.cosmos.network /v0.46/architecture/adr-019-protobuf-state-encoding.html)을 확인하세요.



#### `Any` Encoding in the Cosmos SDK

The above `Profile` example is a fictive example used for educational purposes. In the Cosmos SDK, we use `Any` encoding in several places (non-exhaustive list):

- the `cryptotypes.PubKey` interface for encoding different types of public keys,
- the `sdk.Msg` interface for encoding different `Msg`s in a transaction,
- the `AccountI` interface for encodinig different types of accounts (similar to the above example) in the x/auth query responses,
- the `Evidencei` interface for encoding different types of evidences in the x/evidence module,
- the `AuthorizationI` interface for encoding different types of x/authz authorizations,
- the [`Validator`](https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/x/staking/types/staking.pb.go#L306-L339)struct that contains information about a validator.

A real-life example of encoding the pubkey as `Any` inside the Validator struct in x/staking is shown in the following example:

위의 `Profile` 예제는 교육용으로 사용된 가상의 예제입니다. Cosmos SDK에서는 여러 위치에서 `Any` 인코딩을 사용합니다(전체 목록은 아님).

- 다양한 타입의 공개키를 인코딩하기 위한 `cryptotypes.PubKey` 인터페이스
- 트랜잭션에서 다른 `Msg`를 인코딩하기 위한 `sdk.Msg` 인터페이스
- x/auth 쿼리 응답에서 다양한 타입의 계정(위의 예와 유사)을 인코딩하기 위한 `AccountI` 인터페이스
- x/evidence 모듈에서 다양한 타입의 증거를 인코딩하기 위한 `Evidencei` 인터페이스
- 다양한 타입의 x/authz 인증을 인코딩하기 위한 `AuthorizationI` 인터페이스
- 밸리데이터에 대한 정보가 포함된 [`Validator`](https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/x/staking/types/staking.pb.go#L306-L339) 구조체

x/staking의 Validator 구조체 내에서 pubkey를 `Any`로 인코딩하는 실제 예가 다음에 나와 있습니다.



```go
// NewValidator constructs a new Validator
//nolint:interfacer
func NewValidator(operator sdk.ValAddress, pubKey cryptotypes.PubKey, description Description) (Validator, error) {
	pkAny, err := codectypes.NewAnyWithValue(pubKey)
	if err != nil {
		return Validator{}, err
	}

	return Validator{
		OperatorAddress:   operator.String(),
		ConsensusPubkey:   pkAny,
		Jailed:            false,
		Status:            Unbonded,
		Tokens:            sdk.ZeroInt(),
		DelegatorShares:   sdk.ZeroDec(),
		Description:       description,
		UnbondingHeight:   int64(0),
		UnbondingTime:     time.Unix(0, 0).UTC(),
		Commission:        NewCommission(sdk.ZeroDec(), sdk.ZeroDec(), sdk.ZeroDec()),
		MinSelfDelegation: sdk.OneInt(),
	}, nil
}
```



## FAQ

### How to create modules using protobuf encoding

#### Defining module types

인코딩하기 위해 Protobuf 타입을 정의할 수 있습니다.

- state
- [`Msg`s](https://docs.cosmos.network/v0.46/building-modules/messages-and-queries.html#messages)
- [Query services](https://docs.cosmos.network/v0.46/building-modules/query-services.html)
- [genesis](https://docs.cosmos.network/v0.46/building-modules/genesis.html)



#### Naming and conventions

개발자는 [프로토콜 버퍼 스타일 가이드](https://developers.google.com/protocol-buffers/docs/style)와 [Buf](https://buf.build/docs/style- 가이드), [ADR 023](https://docs.cosmos.network/v0.46/architecture/adr-023-protobuf-naming.html)에서 자세한 내용을 참조하세요.



### How to update modules to protobuf encoding

However, if a module type composes an interface, it must wrap it in the `sdk.Any` (from `/types` package) type. To do that, a module-level .proto file must use [`google.protobuf.Any`](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/any.proto)for respective message type interface types.

For example, in the `x/evidence` module defines an `Evidence` interface, which is used by the `MsgSubmitEvidence`. The structure definition must use `sdk.Any` to wrap the evidence file. In the proto file we define it as follows:

모듈에 인터페이스(예: `Account` 또는 `Content`)가 포함되어 있지 않으면 구체적인 Amino 코덱을 통해 인코딩 및 저장되는 기존 유형을 Protobuf (추가 지침은 1. 참조)로 마이그레이션하고 추가 사용자 정의 없이 `ProtoCodec`을 통해 구현되는 ` Marshaler` 를 코덱으로 받습니다.

그러나 모듈 타입이 인터페이스를 구성하는 경우 `sdk.Any`(`/types` 패키지에서) 타입으로 포장해야 합니다. 그렇게 하려면 모듈 수준 .proto 파일에서 각각의 메시지 타입-인터페이스 타입들에 대한 [`google.protobuf.Any`](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/any.proto)를 사용해야 합니다. 

예를 들어 `x/evidence` 모듈에서 `MsgSubmitEvidence`가 사용하는 `Evidence` 인터페이스를 정의합니다. 구조 정의는 증거 파일을 래핑하기 위해 `sdk.Any`를 사용해야 합니다. proto 파일에서는 다음과 같이 정의합니다.

```go
// proto/cosmos/evidence/v1beta1/tx.proto

message MsgSubmitEvidence {
  string              submitter = 1;
  google.protobuf.Any evidence  = 2 [(cosmos_proto.accepts_interface) = "Evidence"];
}

```



Cosmos SDK의 `codec.Codec` 인터페이스는 `Any`로 상태를 쉽게 인코딩할 수 있도록 `MarshalInterface` 및 `UnmarshalInterface` 지원 메소드를 제공합니다.

모듈은 인터페이스 등록을 위한 메커니즘을 제공하는 `InterfaceRegistry`를 사용하여 인터페이스를 등록해야 합니다. `RegisterInterface(protoName string, iface interface{}, impls ...proto.Message)` 및 구현: `RegisterImplementations(iface interface{}, impls .. .proto.Message)`는 Amino의 타입 등록과 유사하게 Any에서 안전하게 언패킹할 수 있습니다.

```go
// InterfaceRegistry provides a mechanism for registering interfaces and
// implementations that can be safely unpacked from Any
type InterfaceRegistry interface {
	AnyUnpacker
	jsonpb.AnyResolver

	// RegisterInterface associates protoName as the public name for the
	// interface passed in as iface. This is to be used primarily to create
	// a public facing registry of interface implementations for clients.
	// protoName should be a well-chosen public facing name that remains stable.
	// RegisterInterface takes an optional list of impls to be registered
	// as implementations of iface.
	//
	// Ex:
	//   registry.RegisterInterface("cosmos.base.v1beta1.Msg", (*sdk.Msg)(nil))
	RegisterInterface(protoName string, iface interface{}, impls ...proto.Message)

	// RegisterImplementations registers impls as concrete implementations of
	// the interface iface.
	//
	// Ex:
	//  registry.RegisterImplementations((*sdk.Msg)(nil), &MsgSend{}, &MsgMultiSend{})
	RegisterImplementations(iface interface{}, impls ...proto.Message)

	// ListAllInterfaces list the type URLs of all registered interfaces.
	ListAllInterfaces() []string

	// ListImplementations lists the valid type URLs for the given interface name that can be used
	// for the provided interface type URL.
	ListImplementations(ifaceTypeURL string) []string
}
```



또한 `UnpackInterfaces` 단계는 인터페이스가 필요하기 전에 인터페이스들을 언패킹하기 위해 역직렬화에 도입되어야 합니다. 직접 또는 멤버 중 하나를 통해 protobuf `Any`를 포함하는 Protobuf 타입은 `UnpackInterfacesMessage` 인터페이스를 구현해야 합니다.

```go
type UnpackInterfacesMessage interface {
  UnpackInterfaces(InterfaceUnpacker) error
}

```

