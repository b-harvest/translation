# Transaction Lifecycle

이 문서는 생성부터 커밋된 상태 변경까지의 트랜잭션 수명 주기를 설명합니다. 트랜잭션 정의는 [다른 문서](https://docs.cosmos.network/v0.46/core/transactions.html)에 설명되어 있습니다. 트랜잭션은 `Tx`로 칭합니다. 



## Creation

### Transaction Creation

주 애플리케이션 인터페이스들 중 하나는 명령줄 인터페이스(CLI, Command-line interface)입니다. 트랜잭션 `Tx`는 사용자가 ,`[command]`의 트랜잭션의 타입, `[args]`의 인수, `[flags]`의 가스 가격과 같은 구성인, [command-line](https://docs.cosmos.network/v0.46/core/cli.html)에서 다음 형식의 명령을 입력하여 생성할 수 있습니다. 

```go
[appname] tx [command] [args] [flags]
```



이 명령은 트랜잭션을 자동으로 **생성**하고 계정의 비밀 키를 사용하여 **서명**한 다음 지정된 피어 노드로 **브로드캐스트**합니다.

트랜잭션 생성을 위한 몇 가지 필수 및 선택적 flags가 있습니다.`--from` 플래그는 트랜잭션을 시작하는 [계정](https://docs.cosmos.network/v0.46/basics/accounts.html)을  지정합니다. 예를 들어 트랜잭션이 코인을 보내는 경우, 지정된 `from` 주소에서 자금이 인출됩니다.



#### Gas and Fees

또한 사용자가 [fees](https://docs.cosmos.network/v0.46/basics/gas-fees.html)로 지불할 의사가 있는 금액을 나타내는 데 사용할 수 있는 여러 [flags](https://docs.cosmos.network/v0.46/core/cli.html)가 있습니다.

- `--gas`는 `Tx`가 소비하는 연산 자원을 나타내는 [gas](https://docs.cosmos.network/v0.46/basics/gas-fees.html)의 양을 나타냅니다. Gas는 트랜잭션에 따라 달라지며 실행 전까지 정확하게 계산되지 않지만 `--gas`의 값으로 `auto`를 제공하여 추정할 수 있습니다.
- `--gas-adjustment`(선택 사항)는 과소 측정을 피하기 위해 `가스`를 확장하는 데 사용할 수 있습니다. 예를 들어 사용자는 가스 조정을 1.5로 지정하여 예상 가스의 1.5배를 사용할 수 있습니다.
- `--gas-prices`는 사용자가 가스 단위당 지불하고자 하는 금액을 지정합니다. 이는 토큰의 하나 또는 여러개의 화폐일 수 있습니다. 예를 들어 `--gas-prices=0.025uatom, 0.025upho`는 사용자가 가스 단위당 0.025uatom 및 0.025upho를 지불할 의향이 있음을 의미합니다.
- `--fees`는 사용자가 총 지불하고자 하는 수수료를 지정합니다.
- `--timeout-height`는 tx가 지나간 특정 높이에  커밋되는 것을 방지하기 위해 블록 타임아웃 높이를 지정합니다.

지불할 수수료의 최종 값은 가스에 가스 가격을 곱한 값과 같습니다. 즉, `fees = ceil(gas * gasPrices)`입니다. 따라서 수수료는 가스 가격을 사용하여 계산할 수 있고 그 반대도 가능하므로 사용자는 둘 중 하나만 지정합니다.

나중에 벨리데이터들은 주어진 또는 계산된 `gas-prices`을 로컬 `min-gas-prices`과 비교하여 블록에 트랜잭션을 포함할지 여부를 결정합니다. `gas-prices`가 충분히 높지 않으면 `Tx`가 거부되므로 사용자는 더 많은 비용을 지불하도록 유도됩니다.



#### CLI Example



애플리케이션 `app` 사용자는 CLI에 다음 명령을 입력하여 `senderAddress`에서 `recipientAddress`로 1000uatom을 보내는 트랜잭션을 생성할 수 있습니다. 그것은 그들이 지불할 의향이 있는 가스의 양을 지정합니다: 단위 가스당 0.025uatom의 가스 가격으로 1.5배 확대된 자동 추정.

```go
appd tx send <recipientAddress> 1000uatom --from <senderAddress> --gas auto --gas-adjustment 1.5 --gas-prices 0.025uatom
```



#### Other Transaction Creation Methods

명령줄은 애플리케이션과 상호 작용하는 쉬운 방법이지만 `Tx`는 [gRPC 또는 REST 인터페이스](https://docs.cosmos.network/v0.46/core/grpc_rest.html) 또는 애플리케이션 개발자가 정의한 다른 진입점을 사용하여 생성할 수도 있습니다. 사용자의 관점에서 상호 작용은 사용 중인 웹 인터페이스 또는 지갑에 따라 다릅니다(예: [Lunie.io](https://lunie.io/#/)를 사용 및 Ledger Nano S 서명으로 `Tx` 생성). 

## Addition to Mempool

`Tx`를 수신하는 각 풀 노드(Tendermint로 구동된)는 [ABCI 메시지](https://docs.tendermint.com/master/spec/abci/abci.html#messages) 즉,` CheckTx`를 유효성을 확인하기 위해 애플리케이션 계층에 전달하여 `abci.ResponseCheckTx`를 수신합니다.`Tx`가 검사를 통과하면 한 블록에 포함될 때까지 노드들의 [**Mempool**](https://docs.tendermint.com/master/tendermint-core/mempool/) (각 노드에 고유한 트랜잭션의 인메모리 풀)에 보관됩니다  - 정직한 노드는 유효하지 않은 것으로 판명되면 'Tx'를 버립니다. 합의에 앞서 노드는 들어오는 트랜잭션을 지속적으로 확인하고 이를 동료에게 퍼뜨립니다.



### Types of Checks

전체 노드는 계산 낭비를 피하기 위해 가능한 한 조기에 잘못된 트랜잭션을 식별하고 거부하는 것을 목표로 `CheckTx` 동안 `Tx`에 대해 상태 비저장 및 상태 저장 검사를 수행합니다.

***상태 비저장*** 검사는 노드가 상태에 액세스할 필요가 없습니다. 라이트 클라이언트 또는 오프라인 노드가 이를 수행할 수 있으므로 계산 비용이 적게 듭니다. 상태 비저장 검사에는 주소가 비어 있지 않은지 확인하고, 음수가 아닌 숫자를 적용하고, 정의에 지정된 기타 논리가 포함됩니다.

***상태 저장*** 검사는 커밋된 상태를 기반으로 트랜잭션 및 메시지의 유효성을 검사합니다. 예를 들어 관련 값이 존재하고 거래할 수 있는지, 주소에 충분한 자금이 있는지, 보낸 사람이 거래할 수 있는 권한이 있거나 올바른 소유권이 있는지 확인하는 것이 포함됩니다. 주어진 순간에 전체 노드에는 일반적으로 다양한 목적을 위한 애플리케이션 내부 상태의 [여러 버전](https://docs.cosmos.network/v0.46/core/baseapp.html#state-updates)이 있습니다. 예를 들어 노드는 트랜잭션을 확인하는 동안 상태 변경을 실행하지만 쿼리에 응답하려면 마지막 커밋된 상태의 복사본이 여전히 필요합니다. 커밋되지 않은 변경이 있는 상태를 사용하여 응답해서는 안 됩니다.

`Tx`를 확인하기 위해 전체 노드는 *stateless* 및 *stateful* 검사를 모두 포함하는 `CheckTx`를 호출합니다. 추가 검증은 [`DeliverTx`](https://docs.cosmos.network/v0.46/basics/tx-lifecycle.html#delivertx) 단계에서 나중에 발생합니다. `CheckTx`는 `Tx` 디코딩을 시작으로 여러 단계를 거칩니다.



### Decoding

응용 프로그램이 기본 합의 엔진(예: Tendermint)에서 `Tx`를 수신하면 여전히 [인코딩된](https://docs.cosmos.network/v0.46/core/encoding.html) `[ ]byte` 형식이며 처리하려면 비정렬화해야 합니다. 그런 다음 [`runTx`](https://docs.cosmos.network/v0.46/core/baseapp.html#runtx-antehandler-runmsgs-posthandler) 함수가 호출되어 `runTxModeCheck` 모드에서 실행됩니다. 즉, 함수는 모든 검사를 실행하지만 메시지를 실행하고 상태 변경을 쓰기 전에 종료합니다.

### ValidateBasic

메시지([`sdk.Msg`](https://docs.cosmos.network/v0.46/core/transactions.html#messages))는 트랜잭션(`Tx`)에서 추출됩니다. 모듈 개발자가 구현한 `sdk.Msg` 인터페이스의 `ValidateBasic` 메소드는 각 트랜잭션에 대해 실행됩니다. 분명히 잘못된 메시지를 버리기 위해 `BaseApp` 유형은 [`CheckTx`](https://docs.cosmos.network/v0.46/core/baseapp)및 [`DeliverTx`](https://docs.cosmos.network/v0.46/core/baseapp.html#delivertx) 트랜잭션에서 메시지 처리 초기에 `ValidateBasic` 메소드를 호출합니다. `ValidateBasic`은 **stateless** 검사(상태에 대한 액세스가 필요하지 않은 검사)만 포함할 수 있습니다.

#### Guideline

Gas는 `ValidateBasic`이 실행될 때 청구되지 않으므로 미들웨어 작업을 활성화하기 위해 필요한 모든 상태 비저장 검사(예: 미들웨어에 의한 서명의 유효성을 검사하기 위해 필요한 서명자 계정 구문 분석)와 CheckTx 단계에서 성능에 영향을 미치지 않는 상태 비저장 온전성 검사만 수행하는  것이 좋습니다.  모듈 메시지 서버에서 [메시지 처리](https://docs.cosmos.network/v0.46/building-modules/msg-services#Validation) 시 다른 검증 작업을 수행해야 합니다.

예를 들어 메시지가 한 주소에서 다른 주소로 코인을 보내는 것이라면 `ValidateBasic`은 비어 있지 않은 주소와 음수가 아닌 코인 금액을 확인하지만 주소의 계정 잔액과 같은 상태에 대한 지식이 필요하지 않습니다.

[메시지 서비스 유효성 검사](https://docs.cosmos.network/v0.46/building-modules/msg-services.html#Validation)도 참조하세요.



### AnteHandler

ValidateBasic 검사 후 `AnteHandler`가 실행됩니다. 기술적으로는 선택 사항이지만 실제로는 서명 확인, 가스 계산, 수수료 공제 및 기타 블록체인 거래와 관련된 핵심 작업을 수행하기 위해 매우 자주 실행됩니다.

캐시된 컨택스트의 복사본은 트랜잭션 유형에 대해 지정된 제한된 검사를 수행하는 `AnteHandler` 에 제공됩니다. 복사본을 사용하면 `AnteHandler`가 마지막 커밋 상태를 수정하지 않고 `Tx`에 대한 상태 저장 검사를 수행하고 실행이 실패하면 원래 상태로 되돌릴 수 있습니다.

예를 들어, [`auth` ](https://github.com/cosmos/cosmos-sdk/tree/main/x/auth/spec)module `AnteHandler`는 시퀀스 번호를 확인하고 증가시킵니다. 서명 및 계좌 번호, 거래의 첫 번째 서명자로부터 수수료 공제 - 모든 상태 변경은 `checkState`를 사용하여 이루어집니다.

### Gas

`Tx` 실행 중에 사용된 가스의 양을 추적하는 `GasMeter`를 유지하는 [`Context`](https://docs.cosmos.network/v0.46/core/context.html) 는 초기화됩니다. 'Tx'에 대해 사용자가 제공한 가스 양을 `GasWanted`라고 합니다. 실행 중에 소비되는 가스의 양인 `GasConsumed`가 `GasWanted`를 초과하면 실행이 중지되고 상태의 캐시된 복사본에 대한 변경 사항이 커밋되지 않습니다. 그렇지 않으면 `CheckTx`는 `GasUsed`를 `GasConsumed`와 동일하게 설정하고 결과에 반환합니다. 가스 및 수수료 값을 계산한 후 검증자 노드는 사용자가 지정한 `gas-prices`이 현지에서 정의된 `min-gas-prices`보다 큰지 확인합니다.





### Discard or Addition to Mempool

`CheckTx` 중 어느 시점에서든 `Tx`가 실패하면 해당 항목이 삭제되고 트랜잭션 수명 주기가 여기서 종료됩니다. 그렇지 않고 `CheckTx`를 성공적으로 통과하면 기본 프로토콜은 이를 피어 노드에 릴레이하고 Mempool에 추가하여 `Tx`가 다음 블록에 포함될 후보가 되도록 합니다.

**mempool**은 모든 전체 노드에서 볼 수 있는 트랜잭션을 추적하는 목적으로 사용됩니다. 전체 노드는 재 공격을 방지하기 위한 첫 번째 방어선으로 자신이 본 마지막 `mempool.cache_size` 트랜잭션의 **mempool 캐시**를 유지합니다.

현재 존재하는 예방 조치에는 수수료와 동일하지만 유효한 트랜잭션으로부터 재생된 트랜잭션을 구별하기 위한 `시퀀스`(nonce) 카운터가 포함됩니다. 공격자가 많은 `Tx` 사본으로 노드를 스팸하려고 하면 mempool 캐시를 유지하는 전체 노드가 모든 노드에서 `CheckTx`를 실행하는 대신 동일한 사본을 거부합니다. 복사본에 `시퀀스` 번호가 증가하더라도 공격자는 수수료를 지불해야 하므로 의욕을 잃게 됩니다.

Validator 노드는 전체 노드와 마찬가지로 재생 공격을 방지하기 위해 mempool을 유지하지만 블록 포함을 준비하기 위해 확인되지 않은 트랜잭션의 풀로도 사용합니다. `Tx`가 이 단계에서 모든 검사를 통과하더라도 `CheckTx`가 트랜잭션을 완전히 검증하지 않기 때문에(즉, 실제로 메시지를 실행하지 않기 때문에) 나중에 여전히 유효하지 않은 것으로 판명될 수 있습니다.



## Inclusion in a Block

검증자 노드가 수락할 트랜잭션에 대해 합의하는 프로세스인 합의(Consensus)는 **라운드** 내에서 발생합니다. 각 라운드는 제안자가 가장 최근 거래의 블록을 생성하는 것으로 시작하여 **검증인**, 즉 합의를 책임지는 투표권을 가진 특별한 풀 노드로 블록을 수락하거나 대신 'nil' 블록으로 처리하는 데 동의합니다. 검증자 노드는 합의에 이르기 위해 [Tendermint BFT](https://docs.tendermint.com/master/spec/consensus/consensus.html#terms)와 같은 합의 알고리즘을 실행하여 애플리케이션에 ABCI 요청을 사용하여 트랜잭션을 확인합니다. 

합의의 첫 번째 단계는 **블록 제안**입니다. 검증자 중 하나의 제안자는 블록을 생성하고 제안하기 위해 합의 알고리즘에 의해 선택됩니다. 'Tx'가 포함되려면 이 제안자의 멤풀에 있어야 합니다.



## State Changes

합의의 다음 단계는 트랜잭션을 실행하여 완전히 검증하는 것입니다. 올바른 제안자로부터 블록 제안을 받은 모든 풀 노드는 ABCI 함수 [`BeginBlock`](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#beginblocker-and-endblocker), 각 트랜잭션에 대한 `DeliverTx` 및 [`EndBlock`](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#beginblocker-and-endblocker)를 호출하여 트랜잭션을 실행합니다. 각 풀 노드가 모든 것을 로컬에서 실행하는 동안 메시지의 상태 전환이 결정적이며 트랜잭션이 블록 제안에서 명시적으로 정렬되기 때문에 이 프로세스는 단일하고 명확한 결과를 산출합니다.

```go
		-----------------------
		|Receive Block Proposal|
		-----------------------
		          |
			  v
		-----------------------
		| BeginBlock	      |
		-----------------------
		          |
			  v
		-----------------------
		| DeliverTx(tx0)      |
		| DeliverTx(tx1)      |
		| DeliverTx(tx2)      |
		| DeliverTx(tx3)      |
		|	.	      |
		|	.	      |
		|	.	      |
		-----------------------
		          |
			  v
		-----------------------
		| EndBlock	      |
		-----------------------
		          |
			  v
		-----------------------
		| Consensus	      |
		-----------------------
		          |
			  v
		-----------------------
		| Commit	      |
		-----------------------

```





### DeliverTx

[`BaseApp`](https://docs.cosmos.network/v0.46/core/baseapp.html)에 정의된 `DeliverTx` ABCI 함수는 상태 전환의 대부분을 수행합니다. 블록의 각 트랜잭션에 대해 합의 중에 커밋된 대로 순차적으로 실행됩니다. 내부적으로 `DeliverTx`는 `CheckTx`와 거의 동일하지만 전달 모드에서 대신 [`runTx`](https://docs.cosmos.network/v0.46/core/baseapp.html#runtx) 함수를 호출합니다. 풀 노드는 `checkState`를 사용하는 대신 `deliverState`를 사용합니다.

- **Decoding:** `DeliverTx`는 ABCI 호출이므로 `Tx`는 인코딩된 `[]byte` 형식으로 수신됩니다. 노드는 먼저 앱에 정의된 [`TxConfig`](https://docs.cosmos.network/v0.46/basics/app-anatomy#register-codec)를 사용하여 트랜잭션을 언마샬링한 다음 `runTxModeDeliver` 모드에서 `runTx`를 호출합니다.  `runTx`는 `CheckTx`와 매우 유사하지만 상태 변경을 실행하고 기록하기도 합니다.
- **Checks and AnteHandler:** 풀 노드는 `validateBasicMsgs` 및 `AnteHandler`를 다시 호출합니다. 이 두 번째 검사는 Mempool에 추가하는 동안 동일한 트랜잭션을 보지 못했고 악의적인 제안자가 잘못된 트랜잭션을 포함했을 수 있기 때문에 발생합니다. 여기서 한 가지 차이점은 `AnteHandler`가 `gas-prices`를 노드의 `min-gas-prices`와 비교하지 않는다는 것입니다. 그 값은 각 노드에 국한되어 있기 때문입니다. 노드 간에 다른 값은 비결정적 결과를 산출합니다.
- **`MsgServiceRouter`:** `CheckTx`가 종료되었지만 `DeliverTx`는 [`runMsgs`](https://docs.cosmos.network/v0.46/core/baseapp.html#runtx-antehandler-runmsgs-posthandler)를 계속 실행하여  트랜잭션 내의 각 `Msg`를 완전히 실행합니다. 트랜잭션에는 다른 모듈의 메시지가 있을 수 있으므로 `BaseApp`은 적절한 핸들러를 찾기 위해 어떤 모듈을 알아야 합니다. 이는 모듈의 Protobuf [`Msg` 서비스](https://docs.cosmos.network/v0.46/building-modules/msg-services.html)에서 처리할 수 있도록 `BaseApp`의 `MsgServiceRouter`를 사용하여 수행됩니다. . `LegacyMsg` 라우팅의 경우 `Route` 기능이 [모듈 관리자](https://docs.cosmos.network/v0.46/building-modules/module-manager.html)를 통해 호출되어 경로 이름을 검색하고 모듈 내에서 레거시 [`Handler`](https://docs.cosmos.network/v0.46/building-modules/msg-services.html#handler-type)를 찾습니다.
- **`Msg` service:** Protobuf `Msg` 서비스는 `Tx`의 각 메시지 실행을 담당하고 상태 전환이 `deliverTxState`에서 지속되도록 합니다.
- **PostHandlers:** [`PostHandler`](https://docs.cosmos.network/v0.46/core/baseapp.html#posthandler)는 메시지 실행 후 실행됩니다. 실패하면 `runMsgs`와 `PostHandlers`의 상태 변경이 모두 되돌려집니다.
- **Gas:** While a `Tx` is being delivered, a `GasMeter` is used to keep track of how much gas is being used; if execution completes, `GasUsed` is set and returned in the `abci.ResponseDeliverTx`. If execution halts because `BlockGasMeter` or `GasMeter` has run out or something else goes wrong, a deferred function at the end appropriately errors or panics.
- 'Tx'가 배달되는 동안 'GasMeter'는 사용 중인 가스의 양을 추적하는 데 사용됩니다. 실행이 완료되면 `GasUsed`가 설정되고 `abci.ResponseDeliverTx`에 반환됩니다. `BlockGasMeter` 또는 `GasMeter`가 부족하거나 다른 문제가 발생하여 실행이 중단되면 마지막에 지연된 함수가 적절하게 오류 또는 패닉을 발생시킵니다.

'Tx'가 유효하지 않거나 'GasMeter'가 실행되지 않아 실패한 상태 변경이 있는 경우 트랜잭션 처리가 종료되고 모든 상태 변경이 되돌려집니다. 블록 제안의 유효하지 않은 트랜잭션으로 인해 밸리데이터 노드가 블록을 거부하고 대신 'nil' 블록에 투표합니다.



### Commit

마지막 단계는 노드가 블록 및 상태 변경을 커밋하는 것입니다. 밸리데이터 노드는 트랜잭션을 검증하기 위해 상태 전환을 실행하는 이전 단계를 수행한 다음 블록에 서명하여 확인합니다. 밸리데이터가 아닌 풀 노드는 합의에 참여하지 않습니다. 즉, 투표할 수 없습니다. 그러나 상태 변경을 커밋해야 하는지 여부를 이해하기 위해 투표를 수신합니다.

충분한 검증인 투표를 받으면(2/3+ *precommits* - 보팅 파워로 가중치 부여), 풀 노드는 블록체인에 추가할 새 블록을 커밋하고 애플리케이션 계층에서 상태 전환을 완료합니다. 상태 전환에 대한 머클 증거로 사용하기 위해 새 상태 루트가 생성됩니다. 애플리케이션은 [Baseapp](https://docs.cosmos.network/v0.46/core/baseapp.html)에서 상속된 [`Commit`](https://docs.cosmos.network/v0.46/core/baseapp.html#commit) ABCI 메서드를 사용합니다. ; 애플리케이션의 내부 상태에 `deliverState`를 작성하여 모든 상태 전환을 동기화합니다. 상태 변경이 커밋되자마자 `checkState`는 가장 최근에 커밋된 상태에서 새로 시작하고 `deliverState`는 일관성을 유지하고 변경 사항을 반영하기 위해 `nil`로 재설정됩니다.

모든 블록에 동일한 수의 트랜잭션이 있는 것은 아니며 합의에 따라 블록이 'nil'이거나 전혀 없는 블록이 될 수 있습니다. 퍼블릭 블록체인 네트워크에서는 검증인이 **비잔틴**이거나 악의적일 수 있으며, 이로 인해 블록체인에서 `Tx`가 커밋되지 않을 수 있습니다. 가능한 악의적 행동에는 제안자가 `Tx`를 블록에서 제외하여 검열하기로 결정하거나 블록에 대해 반대 투표하는 검증인이 포함됩니다.

이 시점에서 `Tx`의 트랜잭션 수명 주기는 끝났습니다. 노드는 유효성을 확인하고 상태 변경을 실행하여 전달하고 해당 변경 사항을 커밋합니다. `[]byte` 형식의 `Tx` 자체는 블록에 저장되고 블록체인에 추가됩니다.