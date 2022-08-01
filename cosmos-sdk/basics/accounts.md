<!--
order: 4
-->

# 계정(Accounts)

<!-- This document describes the in-built account and public key system of the Cosmos SDK. {synopsis} -->
이 문서에서는 코스모스 SDK의 내장된 계정 및 공개 키 시스템에 대해 설명합니다.

## 선행 학습 자료(Pre-requisite Readings)

* [코스모스 SDK 애플리케이션 분석](./app-anatomy.md) {prereq}

## 계정 정의(Account Definition)

<!-- In the Cosmos SDK, an _account_ designates a pair of _public key_ `PubKey` and _private key_ `PrivKey`. The `PubKey` can be derived to generate various `Addresses`, which are used to identify users (among other parties) in the application. `Addresses` are also associated with [`message`s](../building-modules/messages-and-queries.md#messages) to identify the sender of the `message`. The `PrivKey` is used to generate [digital signatures](#signatures) to prove that an `Address` associated with the `PrivKey` approved of a given `message`. -->
코스모스 SDK에서 _account_는 _public key_ `PubKey`와 _private key_ `PrivKey`의 쌍을 지정합니다. 
`PubKey`는 애플리케이션에서 사용자를 식별하는 데 사용되는 다양한 `주소(Addresses)`를 생성하기 위해 파생될 수 있습니다. 
`Addresses`는 또한 `메시지(message)`의 발신자를 식별하기 위해 [`메시지`](../building-modules/messages-and-queries.md#messages)와 연결됩니다. `PrivKey`는 `PrivKey`와 관련된 `Addresses`가 주어진 `Message`를 승인했음을 증명하기 위해 [디지털 서명](#signatures)을 생성하는 데 사용됩니다.

<!-- For HD key derivation the Cosmos SDK uses a standard called [BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki). The BIP32 allows users to create an HD wallet (as specified in [BIP44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki)) - a set of accounts derived from an initial secret seed. A seed is usually created from a 12- or 24-word mnemonic. A single seed can derive any number of `PrivKey`s using a one-way cryptographic function. Then, a `PubKey` can be derived from the `PrivKey`. Naturally, the mnemonic is the most sensitive information, as private keys can always be re-generated if the mnemonic is preserved. -->
HD 키 파생을 위해 코스모스 SDK는 [BIP32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki)라는 표준을 사용합니다.
BIP32를 통해 사용자는 HD 지갑([BIP44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki)에 지정됨) - 초기 비밀에서 파생된 계정 집합을 만들 수 있습니다. 
씨앗. 시드(seed)는 일반적으로 12 또는 24단어 니모닉으로 생성됩니다. 
단일 시드(seed)는 단방향 암호화 기능을 사용하여 원하는 수의 `PrivKey`를 파생할 수 있습니다. 
그러면 `PrivKey`에서 `PubKey`가 파생될 수 있습니다. 니모닉이 유지되면 개인 키가 항상 다시 생성될 수 있기 때문에, 니모닉은 가장 민감한 정보입니다. 


```text
     Account 0                         Account 1                         Account 2

+------------------+              +------------------+               +------------------+
|                  |              |                  |               |                  |
|    Address 0     |              |    Address 1     |               |    Address 2     |
|        ^         |              |        ^         |               |        ^         |
|        |         |              |        |         |               |        |         |
|        |         |              |        |         |               |        |         |
|        |         |              |        |         |               |        |         |
|        +         |              |        +         |               |        +         |
|  Public key 0    |              |  Public key 1    |               |  Public key 2    |
|        ^         |              |        ^         |               |        ^         |
|        |         |              |        |         |               |        |         |
|        |         |              |        |         |               |        |         |
|        |         |              |        |         |               |        |         |
|        +         |              |        +         |               |        +         |
|  Private key 0   |              |  Private key 1   |               |  Private key 2   |
|        ^         |              |        ^         |               |        ^         |
+------------------+              +------------------+               +------------------+
         |                                 |                                  |
         |                                 |                                  |
         |                                 |                                  |
         +--------------------------------------------------------------------+
                                           |
                                           |
                                 +---------+---------+
                                 |                   |
                                 |  Master PrivKey   |
                                 |                   |
                                 +-------------------+
                                           |
                                           |
                                 +---------+---------+
                                 |                   |
                                 |  Mnemonic (Seed)  |
                                 |                   |
                                 +-------------------+
```

<!-- In the Cosmos SDK, keys are stored and managed by using an object called a [`Keyring`](#keyring). -->
코스모스 SDK에서는 [`Keyring`](#keyring)이라는 객체를 사용하여 키를 저장하고 관리합니다.

## 키(keys), 계정(accounts), 주소(addresses) 및 서명(signatures) 

<!-- The principal way of authenticating a user is done using [digital signatures](https://en.wikipedia.org/wiki/Digital_signature). Users sign transactions using their own private key. Signature verification is done with the associated public key. For on-chain signature verification purposes, we store the public key in an `Account` object (alongside other data required for a proper transaction validation). -->
사용자 인증의 주요 방법은 [디지털 서명](https://en.wikipedia.org/wiki/Digital_signature)을 사용하여 수행됩니다. 
사용자는 자신의 개인 키를 사용하여 트랜잭션에 서명합니다. 
서명 확인은 연결된 공개 키로 수행됩니다. 온체인 서명 확인을 위해 공개 키를 `계정(account)` 개체에 저장합니다(적절한 트랜잭션 유효성 검사에 필요한 다른 데이터와 함께).

<!-- In the node, all data is stored using Protocol Buffers serialization. -->
노드에서 모든 데이터는 프로토콜 버퍼 직렬화를 사용하여 저장됩니다.

<!-- The Cosmos SDK supports the following digital key schemes for creating digital signatures: -->
코스모스 SDK는 디지털 서명 생성을 위해 다음과 같은 디지털 키 체계를 지원합니다.

<!-- * `secp256k1`, as implemented in the [Cosmos SDK's `crypto/keys/secp256k1` package](https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/crypto/keys/secp256k1/secp256k1.go).
* `secp256r1`, as implemented in the [Cosmos SDK's `crypto/keys/secp256r1` package](https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/crypto/keys/secp256r1/pubkey.go),
* `tm-ed25519`, as implemented in the [Cosmos SDK `crypto/keys/ed25519` package](https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/crypto/keys/ed25519/ed25519.go). This scheme is supported only for the consensus validation. -->

* `secp256k1`, [코스모스 SDK의 `crypto/keys/secp256k1` 패키지]에서 구현됨(https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/crypto/keys/secp256k1/ secp256k1.go).
* `secp256r1`, [코스모스 SDK의 `crypto/keys/secp256r1` 패키지]에서 구현됨(https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/crypto/keys/secp256r1/ pubkey.go),
* `tm-ed25519`, [코스모스 SDK `crypto/keys/ed25519` 패키지]에서 구현됨(https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/crypto/keys/에서 구현됨) ed25519/ed25519.go). 이 기술은 합의 검증에만 지원됩니다.

|              | Address length in bytes | Public key length in bytes | Used for transaction authentication | Used for consensus (tendermint) |
| :----------: | :---------------------: | :------------------------: | :---------------------------------: | :-----------------------------: |
| `secp256k1`  |           20            |             33             |                 yes                 |               no                |
| `secp256r1`  |           32            |             33             |                 yes                 |               no                |
| `tm-ed25519` |     -- not used --      |             32             |                 no                  |               yes               |

## 주소(Addresses)

<!-- `Addresses` and `PubKey`s are both public information that identifies actors in the application. `Account` is used to store authentication information. The basic account implementation is provided by a `BaseAccount` object. -->
`Addresses`와 `PubKey`는 모두 애플리케이션에서 행위자를 식별하는 공개 정보입니다. 
`Account`은 인증 정보를 저장하는 데 사용됩니다. 기본 계정 구현은 `BaseAccount` 개체에 의해 제공됩니다.

<!-- Each account is identified using `Address` which is a sequence of bytes derived from a public key. In the Cosmos SDK, we define 3 types of addresses that specify a context where an account is used: -->
각 계정은 공개 키에서 파생된 바이트 시퀀스인 `Addresses`를 사용하여 식별됩니다. 
코스모스 SDK에서는 계정이 사용되는 컨텍스트를 지정하는 3가지 유형의 주소를 정의합니다.

<!-- * `AccAddress` identifies users (the sender of a `message`).
* `ValAddress` identifies validator operators.
* `ConsAddress` identifies validator nodes that are participating in consensus. Validator nodes are derived using the **`ed25519`** curve. -->
* `AccAddress`는 사용자(`message`의 발신자)를 식별합니다.
* `ValAddress`는 유효성 검증인 운영자를 식별합니다.
* `ConsAddress`는 합의에 참여하는 검증인 노드를 식별합니다. 검증인 노드는 **`ed25519`** 곡선을 사용하여 파생됩니다.
* 
<!-- These types implement the `Address` interface: -->
이러한 타입은 `Address` 인터페이스를 구현합니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/types/address.go#L108-L125

Address construction algorithm is defined in [ADR-28](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-028-public-key-addresses.md).
Here is the standard way to obtain an account address from a `pub` public key:\
주소 생성 알고리즘은 [ADR-28](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-028-public-key-addresses.md)에 정의되어 있습니다.
다음은 `pub` 공개 키에서 계정 주소를 얻는 표준 방법입니다.

```go
sdk.AccAddress(pub.Address().Bytes())
```

<!-- Of note, the `Marshal()` and `Bytes()` method both return the same raw `[]byte` form of the address. `Marshal()` is required for Protobuf compatibility. -->
참고로 `Marshal()` 및 `Bytes()` 메서드는 모두 동일한 원시 `[]byte` 형식의 주소를 반환합니다. 
`Marshal()`은 Protobuf 호환성을 위해 필요합니다.

<!-- For user interaction, addresses are formatted using [Bech32](https://en.bitcoin.it/wiki/Bech32) and implemented by the `String` method. The Bech32 method is the only supported format to use when interacting with a blockchain. The Bech32 human-readable part (Bech32 prefix) is used to denote an address type. Example: -->
사용자 상호작용을 위해 주소는 [Bech32](https://en.bitcoin.it/wiki/Bech32)를 사용하여 형식화되고 `String` 방식으로 구현됩니다. 
Bech32 방법은 블록체인과 상호 작용할 때 사용할 수 있는 유일한 지원 형식입니다. 
사람이 읽을 수 있는 Bech32 부분(Bech32 접두사)은 주소 유형을 나타내는 데 사용됩니다. 예시:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/types/address.go#L272-L286

|                    | Address Bech32 Prefix |
| ------------------ | --------------------- |
| Accounts           | cosmos                |
| Validator Operator | cosmosvaloper         |
| Consensus Nodes    | cosmosvalcons         |

### 공개 키(Public Keys)

<!-- Public keys in Cosmos SDK are defined by `cryptotypes.PubKey` interface. Since public keys are saved in a store, `cryptotypes.PubKey` extends the `proto.Message` interface: -->
코스모스 SDK의 공개 키는 `cryptotypes.PubKey` 인터페이스에 의해 정의됩니다. 
공개 키는 저장소에 저장되기 때문에 `cryptotypes.PubKey`는 `proto.Message` 인터페이스를 확장합니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/crypto/types/types.go#L8-L17

<!-- A compressed format is used for `secp256k1` and `secp256r1` serialization. -->
압축 형식은 `secp256k1` 및 `secp256r1` 직렬화에 사용됩니다.

<!-- * The first byte is a `0x02` byte if the `y`-coordinate is the lexicographically largest of the two associated with the `x`-coordinate.
* Otherwise the first byte is a `0x03`. -->
* `y` 좌표가 `x` 좌표와 관련된 두 가지 사전순으로 가장 큰 경우, 첫 번째 바이트는 `0x02` 바이트입니다.
* 그렇지 않으면 첫 번째 바이트는 `0x03`입니다.

<!-- This prefix is followed by the `x`-coordinate. -->
이 접두사(prefix) 뒤에는 `x` 좌표가 옵니다.

<!-- Public Keys are not used to reference accounts (or users) and in general are not used when composing transaction messages (with few exceptions: `MsgCreateValidator`, `Validator` and `Multisig` messages).
For user interactions, `PubKey` is formatted using Protobufs JSON ([ProtoMarshalJSON](https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/codec/json.go#L14-L34) function). Example: -->
공개 키는 계정(또는 사용자)을 참조하는 데 사용되지 않으며, 일반적으로 트랜잭션 메시지를 작성할 때 사용되지 않습니다(몇 가지 예외: `MsgCreateValidator`, `Validator` 및 `Multisig` 메시지).
사용자 상호 작용을 위해 `PubKey`는 Protobufs JSON([ProtoMarshalJSON](https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/codec/json.go#L14-L34) 함수를 사용하여 형식이 지정됩니다. 예시:

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/crypto/keyring/output.go#L23-L39

## 키링(Keyring)

<!-- A `Keyring` is an object that stores and manages accounts. In the Cosmos SDK, a `Keyring` implementation follows the `Keyring` interface: -->
`키링(Keyring)`은 계정을 저장하고 관리하는 객체입니다.
코스모스 SDK에서 `Keyring` 구현은 `Keyring` 인터페이스를 따릅니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/crypto/keyring/keyring.go#L54-L101

<!-- The default implementation of `Keyring` comes from the third-party [`99designs/keyring`](https://github.com/99designs/keyring) library. -->
`Keyring`의 기본 구현은 서드파티인 [`99designs/keyring`](https://github.com/99designs/keyring) 라이브러리에서 제공됩니다.

<!-- A few notes on the `Keyring` methods: -->
`Keyring` 방법에 대한 몇 가지 참고 사항:

<!-- * `Sign(uid string, msg []byte) ([]byte, types.PubKey, error)` strictly deals with the signature of the `msg` bytes. You must prepare and encode the transaction into a canonical `[]byte` form. Because protobuf is not deterministic, it has been decided in [ADR-020](../architecture/adr-020-protobuf-transaction-encoding.md) that the canonical `payload` to sign is the `SignDoc` struct, deterministically encoded using [ADR-027](../architecture/adr-027-deterministic-protobuf-serialization.md). Note that signature verification is not implemented in the Cosmos SDK by default, it is deferred to the [`anteHandler`](../core/baseapp.md#antehandler). -->
* `Sign(uid string, msg []byte) ([]byte, types.PubKey, error)`는 `msg` 바이트의 서명을 엄격하게 다룹니다. 트랜잭션을 준비하고 정식 `[]byte` 형식으로 인코딩해야 합니다. protobuf는 결정적(deterministic)이지 않기 때문에 [ADR-020](../architecture/adr-020-protobuf-transaction-encoding.md)에서 서명할 표준 `페이로드`가 `SignDoc` 구조체로 결정되었는데, [ADR-027](../architecture/adr-027-deterministic-protobuf-serialization.md)을 사용하여 결정적으로 인코딩됩니다. 코스모스 SDK에서는 기본적으로 서명 확인이 구현되지 않으며 [`anteHandler`](../core/baseapp.md#antehandler)로 연기됩니다.
  +++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/proto/cosmos/tx/v1beta1/tx.proto#L48-L65

<!-- * `NewAccount(uid, mnemonic, bip39Passphrase, hdPath string, algo SignatureAlgo) (*Record, error)` creates a new account based on the [`bip44 path`](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki) and persists it on disk. The `PrivKey` is **never stored unencrypted**, instead it is [encrypted with a passphrase](https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/crypto/armor.go) before being persisted. In the context of this method, the key type and sequence number refer to the segment of the BIP44 derivation path (for example, `0`, `1`, `2`, ...) that is used to derive a private and a public key from the mnemonic. Using the same mnemonic and derivation path, the same `PrivKey`, `PubKey` and `Address` is generated. The following keys are supported by the keyring: -->
* `NewAccount(uid, mnemonic, bip39Passphrase, hdPath string, algo SignatureAlgo) (*Record, error)`는 [`bip44 path`](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki)를 기반으로 새 계정을 만들고, 디스크에 저장합니다. `PrivKey`는 **암호화되지 않은 상태로 저장되지 않습니다**, 대신 지속되기 전에 [암호구로 암호화](https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/crypto/armor.go)됩니다. 이 방법의 컨텍스트에서 키 유형 및 시퀀스 번호는 니모닉으로부터 공개키 및 개인키를 생성하기 위해 사용되는 BIP44 파생 경로의 세그먼트를 참조합니다. 동일한 니모닉 및 파생 경로를 사용하여 동일한 `PrivKey`, `PubKey` 및 `Address`가 생성됩니다. 키링은 다음 키를 지원합니다.

* `secp256k1`
* `ed25519`

<!-- * `ExportPrivKeyArmor(uid, encryptPassphrase string) (armor string, err error)` exports a private key in ASCII-armored encrypted format using the given passphrase. You can then either import the private key again into the keyring using the `ImportPrivKey(uid, armor, passphrase string)` function or decrypt it into a raw private key using the `UnarmorDecryptPrivKey(armorStr string, passphrase string)` function. -->
* `ExportPrivKeyArmor(uid, encryptPassphrase string) (armor string, err error)`는 주어진 암호를 사용하여 ASCII로 보호된 암호화 형식으로 개인 키를 내보냅니다. 그런 다음 `ImportPrivKey(uid, armor, passphrase string)` 기능을 사용하여 개인 키를 키링으로 다시 가져오거나 `UnarmorDecryptPrivKey(armorStr string, passphrase string)` 기능을 사용하여 원시 개인 키로 해독할 수 있습니다.
* 
<!-- ## Next {hide} -->
## 다음

<!-- Learn about [gas and fees](./gas-fees.md) {hide} -->
[가스 및 요금](./gas-fees.md)에 대해 알아보기.