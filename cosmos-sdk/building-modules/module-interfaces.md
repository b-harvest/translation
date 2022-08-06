# 모듈 인터페이스(Module Interfaces)

이 문서에서는 모듈에 대한 CLI 및 REST 인터페이스를 구축하는 방법을 자세히 설명합니다. 다양한 Cosmos SDK 모듈의 예제가 포함되어 있습니다.

## 읽어야 하는 선행 문서(Prerequisite Readings)

- [빌딩 모듈 소개](./intro.md)

## CLI

응용 프로그램의 주요 인터페이스 중 하나는 [명령줄 인터페이스](../core/cli.md)입니다. 이 진입점은 최종 사용자가 트랜잭션과 [**쿼리**](./messages- and-queries.md#queries)로 래핑된 [**messages**](./messages-and-queries.md#messages)를 생성할 수 있도록하는 애플리케이션 모듈의 명령을 추가합니다. CLI 파일은 일반적으로 모듈의 `./client/cli` 폴더에 있습니다.

### 트랜잭션 명령(Transaction Commands)

상태 변경을 트리거하는 메시지를 생성하려면 최종 사용자가 메시지를 래핑하고 전달하는 [트랜잭션](../core/transactions.md)을 생성해야 합니다. 트랜잭션 명령은 하나 이상의 메시지를 포함하는 트랜잭션을 생성합니다.

트랜잭션 명령에는 일반적으로 모듈의 `./client/cli` 폴더 내에 있는 자체 `tx.go` 파일이 있습니다. 명령은 getter 함수에 지정되며 함수 이름에는 명령 이름이 포함되어야 합니다.

다음은 `x/bank` 모듈의 예입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/x/bank/client/cli/tx.go#L35-L71

예제에서 `NewSendTxCmd()`는 `MsgSend`를 래핑하고 전달하는 트랜잭션에 대한 트랜잭션 명령을 생성하고 반환합니다. `MsgSend`는 한 계정에서 다른 계정으로 토큰을 보내는 데 사용되는 메시지입니다.

일반적으로 getter 함수는 다음을 수행합니다.

- **명령 구성:** 명령을 만드는 방법에 대한 자세한 내용은 [Cobra Documentation](https://pkg.go.dev/github.com/spf13/cobra)를 참조하세요.
  - **Use:** 명령을 호출하는 데 필요한 사용자 입력 형식을 지정합니다. 위의 예에서 `send`는 트랜잭션 명령의 이름이고 `[from_key_or_address]`, `[to_address]`, `[amount]`는 인수입니다.
  - **Args:** 사용자가 제공하는 인수의 수입니다. 이 경우 `[from_key_or_address]`, `[to_address]`, `[amount]`의 세 가지가 있습니다.
  - **Short and Long:** 명령에 대한 설명입니다. `짧은` 설명이 필요합니다. `긴` 설명은 사용자가 `--help` 플래그를 추가할 때 표시되는 추가 정보를 제공하는 데 사용할 수 있습니다.
  - **RunE:** 오류를 반환할 수 있는 함수를 정의합니다. 이것은 명령이 실행될 때 호출되는 함수입니다. 이 함수는 모든 로직을 캡슐화하여 새 트랜잭션을 생성합니다.
    - 함수는 일반적으로 `client.GetClientTxContext(cmd)`로 수행할 수 있는 `clientCtx`를 가져오는 것으로 시작합니다. `clientCtx`에는 사용자에 대한 정보를 포함하여 트랜잭션 처리와 관련된 정보가 포함되어 있습니다. 이 예에서 `clientCtx`는 `clientCtx.GetFromAddress()`를 호출하여 발신자의 주소를 검색하는 데 사용됩니다.
    - 해당되는 경우 명령의 인수가 구문 분석됩니다. 이 예에서 `[to_address]`와 `[amount]` 인수는 모두 구문 분석됩니다.
    - 구문 분석된 인수와 `clientCtx`의 정보를 사용하여 [message](./messages-and-queries.md)가 생성됩니다. 메시지 유형의 생성자 함수는 직접 호출됩니다. 이 경우 `types.NewMsgSend(fromAddr, toAddr, amount)`입니다. 메시지를 브로드캐스트하기 전에 [`msg.ValidateBasic()`](../basics/tx-lifecycle.md#ValidateBasic) 및 기타 유효성 검사 메서드를 호출하는 것이 좋습니다.
    - 사용자가 원하는 것에 따라 트랜잭션은 오프라인으로 생성되거나 `tx.GenerateOrBroadcastTxCLI(clientCtx, flags, msg)`를 사용하여 사전 구성된 노드에서 서명되고 브로드캐스트됩니다.
- **트랜잭션 플래그 추가:** 모든 트랜잭션 명령은 트랜잭션 [flags](#flags) 세트를 추가해야 합니다. 트랜잭션 플래그는 사용자로부터 추가 정보를 수집하는 데 사용됩니다.(예: 사용자가 지불할 수수료 금액) 트랜잭션 플래그는 `AddTxFlagsToCmd(cmd)`를 사용하여 구성된 명령에 추가됩니다.
- **명령 반환:** 마지막으로 트랜잭션 명령이 반환됩니다.

각 모듈은 모듈의 모든 트랜잭션 명령을 집계하는 `NewTxCmd()`를 구현해야 합니다. 다음은 `x/bank` 모듈의 예입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/x/bank/client/cli/tx.go#L17-L33

각 모듈은 단순히 `NewTxCmd()`를 반환하는 `AppModuleBasic`에 대한 `GetTxCmd()` 메서드도 구현해야 합니다. 이를 통해 루트 명령은 각 모듈에 대한 모든 트랜잭션 명령을 쉽게 집계할 수 있습니다. 다음은 예입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/x/bank/module.go#L70-L73

### 쿼리 명령(Query Commands)

[쿼리](./messages-and-queries.md#queries)를 통해 사용자는 애플리케이션 또는 네트워크 상태에 대한 정보를 수집할 수 있습니다. 그것들은 애플리케이션에 의해 라우팅되고 정의된 모듈에 의해 처리됩니다. 쿼리 명령에는 일반적으로 모듈의 `./client/cli` 폴더에 자체 `query.go` 파일이 있습니다. 트랜잭션 명령과 마찬가지로 getter 함수에 지정됩니다. 다음은 `x/auth` 모듈의 쿼리 명령 예입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/x/auth/client/cli/query.go#L83-L125

예제에서 `GetAccountCmd()`는 제공된 계정 주소를 기반으로 계정의 상태를 반환하는 쿼리 명령을 생성하고 반환합니다.

일반적으로 getter 함수는 다음을 수행합니다.

- **명령 구성:** 명령을 만드는 방법에 대한 자세한 내용은 [Cobra 설명서](https://pkg.go.dev/github.com/spf13/cobra)를 참조하세요.
  - **Use:** 명령을 호출하는 데 필요한 사용자 입력 형식을 지정합니다. 위의 예에서 `account`는 쿼리 명령의 이름이고 `[address]`는 인수입니다.
  - **Args:** 사용자가 제공하는 인수의 수입니다. 이 경우 딱 하나로 `[address]`가 있습니다.
  - **Short and Long:** 명령에 대한 설명입니다. `짧은` 설명이 필요합니다. `긴` 설명은 사용자가 `--help` 플래그를 추가할 때 표시되는 추가 정보를 제공하는 데 사용할 수 있습니다.
  - **RunE:** 오류를 반환할 수 있는 함수를 정의합니다. 이것은 명령이 실행될 때 호출되는 함수입니다. 이 함수는 모든 로직을 캡슐화하여 새 쿼리를 생성합니다.
    - 함수는 일반적으로 `client.GetClientQueryContext(cmd)`로 수행할 수 있는 `clientCtx`를 가져오는 것으로 시작합니다. `clientCtx`에는 쿼리 처리와 관련된 정보가 포함되어 있습니다.
    - 해당되는 경우 명령의 인수가 구문 분석됩니다. 이 예에서는 `[address]` 인수가 구문 분석됩니다.
    - 새로운 `queryClient`는 `NewQueryClient(clientCtx)`를 사용하여 초기화됩니다. 그런 다음 `queryClient`를 사용하여 적절한 [query](./messages-and-queries.md#grpc-queries)를 호출합니다.
    - `clientCtx.PrintProto` 메소드는 `proto.Message` 개체의 형식을 지정하여 결과를 사용자에게 다시 인쇄할 수 있도록 하는 데 사용됩니다.
- **쿼리 플래그 추가:** 모든 쿼리 명령은 쿼리 [flags](#flags) 세트를 추가해야 합니다. 쿼리 플래그는 `AddQueryFlagsToCmd(cmd)`를 사용하여 구성된 명령에 추가됩니다.
- **명령 반환:** 마지막으로 쿼리 명령이 반환됩니다.

각 모듈은 모듈의 모든 쿼리 명령을 집계하는 `GetQueryCmd()`를 구현해야 합니다. 다음은 `x/auth` 모듈의 예입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/x/auth/client/cli/query.go#L25-L42

또한 각 모듈은 `GetQueryCmd()` 함수를 반환하는 `AppModuleBasic`에 대한 `GetQueryCmd()` 메서드를 구현해야 합니다. 이를 통해 루트 명령은 각 모듈에 대한 모든 쿼리 명령을 쉽게 집계할 수 있습니다. 다음은 예입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/x/auth/client/cli/query.go#L32-L50

### 플래그(Flags)

[Flags](../core/cli.md#flags)를 사용하면 사용자가 명령을 사용자 정의할 수 있습니다. `--fees` 및 `--gas-prices`는 사용자가 거래에 대한 [fees](../basics/gas-fees.md) 및 가스 가격을 설정할 수 있는 플래그의 예입니다.

모듈에 특정한 플래그는 일반적으로 모듈의 `./client/cli` 폴더에 있는 `flags.go` 파일에 생성됩니다. 플래그를 생성할 때 개발자는 값 유형, 플래그 이름, 기본값 및 플래그에 대한 설명을 설정합니다. 또한 개발자는 플래그를 *필수*로 표시하여 사용자가 플래그 값을 포함하지 않으면 오류가 발생하도록 할 수 있습니다.

다음은 명령에 `--from` 플래그를 추가하는 예입니다.

```go
cmd.Flags().String(FlagFrom, "", "서명할 개인 키의 이름 또는 주소")
```

이 예에서 플래그의 값은 `String`이고 플래그의 이름은 `from`(`FlagFrom` 상수의 값)이며 플래그의 기본값은 `""`이며 사용자가 명령에 `--help`를 추가할 때 표시될 설명입니다.

다음은 `--from` 플래그를 *required*로 표시하는 예입니다.

```go
cmd.MarkFlagRequired(FlagFrom)
```

플래그 생성에 대한 자세한 내용은 [Cobra 문서](https://github.com/spf13/cobra)를 참조하세요.

[트랜잭션 명령](#transaction-commands)에서 언급했듯이 모든 트랜잭션 명령이 추가해야 하는 플래그 집합이 있습니다. 이는 Cosmos SDK의 `./client/flags` 패키지에 정의된 `AddTxFlagsToCmd` 메소드로 수행됩니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/client/flags/flags.go#L103-L131

`AddTxFlagsToCmd(cmd *cobra.Command)`에는 트랜잭션 명령에 필요한 모든 기본 플래그가 포함되어 있으므로 모듈 개발자는 자체 플래그를 추가하지 않도록 선택할 수 있습니다(대신 인수를 지정하는 것이 더 적절할 수 있음).

마찬가지로 모듈 쿼리 명령에 공통 플래그를 추가하는 `AddQueryFlagsToCmd(cmd *cobra.Command)`가 있습니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/client/flags/flags.go#L92-L101

## gRPC

[gRPC](https://grpc.io/)는 RPC(원격 프로시저 호출) 프레임워크입니다. RPC는 지갑 및 거래소와 같은 외부 클라이언트가 블록체인과 상호 작용하는 데 선호되는 방법입니다.

ABCI 쿼리 경로를 제공하는 것 외에도 Cosmos SDK는 gRPC 쿼리 요청을 ABCI 쿼리 요청으로 라우팅하는 gRPC 프록시 서버를 제공합니다.

그렇게 하기 위해 모듈은 클라이언트 gRPC 요청을 모듈 내부의 올바른 핸들러에 연결하기 위해 `AppModuleBasic`에 `RegisterGRPCGatewayRoutes(clientCtx client.Context, mux *runtime.ServeMux)`를 구현해야 합니다.

다음은 `x/auth` 모듈의 예입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/x/auth/module.go#L61-L66

## gRPC-gateway REST

애플리케이션은 HTTP 요청을 사용하는 웹 서비스를 지원해야 합니다.(예: [Keplr](https://keplr.app)와 같은 웹 지갑) [grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway)는 REST 호출을 gRPC 호출로 변환하므로 gRPC를 사용하지 않는 클라이언트에 유용할 수 있습니다.

REST 쿼리를 노출하려는 모듈은 `x/auth` 모듈의 아래 예와 같이 `google.api.http` 주석을 `rpc` 메소드에 추가해야 합니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/proto/cosmos/auth/v1beta1/query.proto#L13-L59

gRPC 게이트웨이는 애플리케이션 및 텐더민트와 함께 프로세스 내에서 시작됩니다. [`app.toml`](../run-node/run-node.md#configuring-the-node-using-apptoml)에서 gRPC 구성을 `enable`로 설정하여 활성화하거나 비활성화할 수 있습니다.

Cosmos SDK는 [Swagger](https://swagger.io/) 문서(`protoc-gen-swagger`) 생성 명령을 제공합니다. [`app.toml`](../run-node/run-node.md#configuring-the-node-using-apptoml)에서 `swagger`를 설정하면 swagger 문서가 자동으로 등록되어야 하는지 여부가 정의됩니다.
