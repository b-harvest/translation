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

## [#](https://docs.cosmos.network/v0.46/basics/tx-lifecycle.html#inclusion-in-a-block)Inclusion in a Block

Consensus, the process through which validator nodes come to agreement on which transactions to accept, happens in **rounds**. Each round begins with a proposer creating a block of the most recent transactions and ends with **validators**, special full-nodes with voting power responsible for consensus, agreeing to accept the block or go with a `nil` block instead. Validator nodes execute the consensus algorithm, such as [Tendermint BFT (opens new window)](https://docs.tendermint.com/master/spec/consensus/consensus.html#terms), confirming the transactions using ABCI requests to the application, in order to come to this agreement.

The first step of consensus is the **block proposal**. One proposer amongst the validators is chosen by the consensus algorithm to create and propose a block - in order for a `Tx` to be included, it must be in this proposer's mempool.

## [#](https://docs.cosmos.network/v0.46/basics/tx-lifecycle.html#state-changes)State Changes

The next step of consensus is to execute the transactions to fully validate them. All full-nodes that receive a block proposal from the correct proposer execute the transactions by calling the ABCI functions [`BeginBlock`](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#beginblocker-and-endblocker), `DeliverTx` for each transaction, and [`EndBlock`](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#beginblocker-and-endblocker). While each full-node runs everything locally, this process yields a single, unambiguous result, since the messages' state transitions are deterministic and transactions are explicitly ordered in the block proposal.

Copy		----------------------- 	|Receive Block Proposal| 	----------------------- 	          | 		  v 	----------------------- 	| BeginBlock	      | 	----------------------- 	          | 		  v 	----------------------- 	| DeliverTx(tx0)      | 	| DeliverTx(tx1)      | 	| DeliverTx(tx2)      | 	| DeliverTx(tx3)      | 	|	.	      | 	|	.	      | 	|	.	      | 	----------------------- 	          | 		  v 	----------------------- 	| EndBlock	      | 	----------------------- 	          | 		  v 	----------------------- 	| Consensus	      | 	----------------------- 	          | 		  v 	----------------------- 	| Commit	      | 	-----------------------

```
```



### [#](https://docs.cosmos.network/v0.46/basics/tx-lifecycle.html#delivertx)DeliverTx

The `DeliverTx` ABCI function defined in [`BaseApp`](https://docs.cosmos.network/v0.46/core/baseapp.html) does the bulk of the state transitions: it is run for each transaction in the block in sequential order as committed to during consensus. Under the hood, `DeliverTx` is almost identical to `CheckTx` but calls the [`runTx`](https://docs.cosmos.network/v0.46/core/baseapp.html#runtx) function in deliver mode instead of check mode. Instead of using their `checkState`, full-nodes use `deliverState`:

- **Decoding:** Since `DeliverTx` is an ABCI call, `Tx` is received in the encoded `[]byte` form. Nodes first unmarshal the transaction, using the [`TxConfig`](https://docs.cosmos.network/v0.46/basics/app-anatomy#register-codec) defined in the app, then call `runTx` in `runTxModeDeliver`, which is very similar to `CheckTx` but also executes and writes state changes.
- **Checks and AnteHandler:** Full-nodes call `validateBasicMsgs` and `AnteHandler` again. This second check happens because they may not have seen the same transactions during the addition to Mempool stage and a malicious proposer may have included invalid ones. One difference here is that the `AnteHandler` will not compare `gas-prices` to the node's `min-gas-prices` since that value is local to each node - differing values across nodes would yield nondeterministic results.
- **`MsgServiceRouter`:** While `CheckTx` would have exited, `DeliverTx` continues to run [`runMsgs`](https://docs.cosmos.network/v0.46/core/baseapp.html#runtx-antehandler-runmsgs-posthandler) to fully execute each `Msg` within the transaction. Since the transaction may have messages from different modules, `BaseApp` needs to know which module to find the appropriate handler. This is achieved using `BaseApp`'s `MsgServiceRouter` so that it can be processed by the module's Protobuf [`Msg` service](https://docs.cosmos.network/v0.46/building-modules/msg-services.html). For `LegacyMsg` routing, the `Route` function is called via the [module manager](https://docs.cosmos.network/v0.46/building-modules/module-manager.html) to retrieve the route name and find the legacy [`Handler`](https://docs.cosmos.network/v0.46/building-modules/msg-services.html#handler-type) within the module.
- **`Msg` service:** Protobuf `Msg` service is responsible for executing each message in the `Tx` and causes state transitions to persist in `deliverTxState`.
- **PostHandlers:** [`PostHandler`](https://docs.cosmos.network/v0.46/core/baseapp.html#posthandler)s run after the execution of the message. If they fail, the state change of `runMsgs`, as well of `PostHandlers` are both reverted.
- **Gas:** While a `Tx` is being delivered, a `GasMeter` is used to keep track of how much gas is being used; if execution completes, `GasUsed` is set and returned in the `abci.ResponseDeliverTx`. If execution halts because `BlockGasMeter` or `GasMeter` has run out or something else goes wrong, a deferred function at the end appropriately errors or panics.

If there are any failed state changes resulting from a `Tx` being invalid or `GasMeter` running out, the transaction processing terminates and any state changes are reverted. Invalid transactions in a block proposal cause validator nodes to reject the block and vote for a `nil` block instead.

### [#](https://docs.cosmos.network/v0.46/basics/tx-lifecycle.html#commit)Commit

The final step is for nodes to commit the block and state changes. Validator nodes perform the previous step of executing state transitions in order to validate the transactions, then sign the block to confirm it. Full nodes that are not validators do not participate in consensus - i.e. they cannot vote - but listen for votes to understand whether or not they should commit the state changes.

When they receive enough validator votes (2/3+ *precommits* weighted by voting power), full nodes commit to a new block to be added to the blockchain and finalize the state transitions in the application layer. A new state root is generated to serve as a merkle proof for the state transitions. Applications use the [`Commit`](https://docs.cosmos.network/v0.46/core/baseapp.html#commit) ABCI method inherited from [Baseapp](https://docs.cosmos.network/v0.46/core/baseapp.html); it syncs all the state transitions by writing the `deliverState` into the application's internal state. As soon as the state changes are committed, `checkState` start afresh from the most recently committed state and `deliverState` resets to `nil` in order to be consistent and reflect the changes.

Note that not all blocks have the same number of transactions and it is possible for consensus to result in a `nil` block or one with none at all. In a public blockchain network, it is also possible for validators to be **byzantine**, or malicious, which may prevent a `Tx` from being committed in the blockchain. Possible malicious behaviors include the proposer deciding to censor a `Tx` by excluding it from the block or a validator voting against the block.

At this point, the transaction lifecycle of a `Tx` is over: nodes have verified its validity, delivered it by executing its state changes, and committed those changes. The `Tx` itself, in `[]byte` form, is stored in a block and appended to the blockchain.