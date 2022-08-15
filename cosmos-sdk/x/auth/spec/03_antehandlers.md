# AnteHandlers



`x/auth` 모듈에는 현재 자체 트랜잭션 핸들러가 없지만 트랜잭션에 대한 기본 유효성 검사를 수행하는 데 사용되는 특수한  `AnteHandler`를 노출하여 멤풀에서 제외될 수 있습니다. 'AnteHandler'는 [ADR 010](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-010-modular-antehandler.md)에 따라 현재 컨텍스트 내에서 트랜잭션을 확인하는 데코레이터 세트로 볼 수 있습니다. 

Tendermint 제안자는 현재 제안된 블록 트랜잭션에 `CheckTx`에 실패한 블록 트랜잭션을 포함할 수 있는 기능이 있으므로 `AnteHandler`는 `CheckTx`와 `DeliverTx` 모두에서 호출됩니다.



## Decorators



auth 모듈은 다음 순서로 단일 `AnteHandler`로 재귀적으로 연결된 `AnteDecorator`를 제공합니다.

- `SetUpContextDecorator`: `Context`의 `GasMeter`를 설정하고 다음 `AnteHandler`를 defer 절로 래핑하여 `AnteHandler` 체인의 다운스트림 `OutOfGas` 패닉으로부터 복구하여 제공된 가스와 사용된 가스에 대한 정보와 함께 오류를 반환합니다. 
- `RejectExtensionOptionsDecorator`: protobuf 트랜잭션에 선택적으로 포함될 수 있는 모든 확장 옵션을 거부합니다.
- `MempoolFeeDecorator`: `CheckTx` 동안 `tx` 수수료가 로컬 mempool `minFee` 매개변수보다 높은지 확인합니다.
- `ValidateBasicDecorator`: `tx.ValidateBasic`을 호출하고 nil이 아닌 오류를 반환합니다.
- `TxTimeoutHeightDecorator`: `tx` 높이 제한 시간(height timeout)을 확인합니다.
- `ValidateMemoDecorator`: 애플리케이션 매개변수로 `tx` 메모의 유효성을 검사하고 nil이 아닌 오류를 반환합니다.
- `ConsumeGasTxSizeDecorator`: 애플리케이션 매개변수를 기반으로 `tx` 크기에 비례하여 가스를 소비합니다.
- `DeductFeeDecorator`: `tx`의 첫 번째 서명자로부터 `FeeAmount`를 공제합니다. `x/feegrant` 모듈이 활성화되고 수수료 부여자가 설정되면 수수료 부여자 계정에서 수수료를 공제합니다.
- `SetPubKeyDecorator`: 상태 머신과 현재 컨텍스트에 해당 pubkey가 저장되어 있지 않은 `tx` 서명자의 pubkey를 설정합니다.
- `ValidateSigCountDecorator`: 앱 매개변수를 기반으로 `tx`의 서명 수를 검증합니다.
- `SigGasConsumeDecorator`: 각 서명에 대해 매개변수-정의된 가스 양을 소비합니다. 이를 위해서는 `SetPubKeyDecorator`의 일부로 모든 서명자의 컨텍스트에서 pubkey를 설정해야 합니다.
- `SigVerificationDecorator`: 모든 서명이 유효한지 확인합니다. 이를 위해서는 `SetPubKeyDecorator`의 일부로 모든 서명자의 컨텍스트에서 pubkey를 설정해야 합니다.
- `IncrementSequenceDecorator`: 리플레이 어택을 방지하기 위해 각 서명자에 대한 계정 순서를 증가시킵니다.