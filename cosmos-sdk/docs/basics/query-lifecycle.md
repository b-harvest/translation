<!--
order: 3
-->

# 쿼리 라이프사이클(Query Lifecycle)

<!-- This document describes the lifecycle of a query in a Cosmos SDK application, from the user interface to application stores and back. {synopsis} -->
이 문서는 SDK 애플리케이션에서 사용자 인터페이스부터 애플리케이션 스토어까지의 쿼리(query)의 라이프사이클(동작하는 전반적인 방법)에 대해 설명한다.

## 선행 학습 자료(Pre-requisite Readings)

* [트랜잭션 라이프사이클](./tx-lifecycle.md) 

## 쿼리 생성(Query Creation)

<!-- A [**query**](../building-modules/messages-and-queries.md#queries) is a request for information made by end-users of applications through an interface and processed by a full-node. Users can query information about the network, the application itself, and application state directly from the application's stores or modules. Note that queries are different from [transactions](../core/transactions.md) (view the lifecycle [here](./tx-lifecycle.md)), particularly in that they do not require consensus to be processed (as they do not trigger state-transitions); they can be fully handled by one full-node. -->
[**쿼리**](../building-modules/messages-and-queries.md#queries)는 애플리케이션의 사용자가 인터페이스를 통해 만들고, 전체 노드에서 처리하는 정보에 대한 요청입니다. 
사용자는 네트워크, 애플리케이션 자체, 그리고 애플리케이션 상태에 대한 정보를 애플리케이션의 저장소 또는 모듈로부터 직접 쿼리할 수 있습니다.
쿼리는 [트랜잭션](../core/transactions.md)과 다르며([여기](./tx-lifecycle.md)에서 트랜잭션 라이프사이클 보기) 특히 처리에 합의가 필요하지 않습니다(상태 전환을 트리거하지 않기 때문입니다).
쿼리는 하나의 전체 노드에서 완전히 처리될 수 있습니다.

<!-- For the purpose of explaining the query lifecycle, let's say `MyQuery` is requesting a list of delegations made by a certain delegator address in the application called `simapp`. As to be expected, the [`staking`](../../x/staking/spec/README.md) module handles this query. But first, there are a few ways `MyQuery` can be created by users. -->
쿼리 라이프사이클을 설명하기 위해, `MyQuery`가 `simapp`이라는 애플리케이션에서 특정 위임자 주소가 만든 위임 목록을 요청한다고 가정해 보겠습니다.
예상대로 [`스테이킹`](../../x/staking/spec/README.md) 모듈이 이 쿼리를 처리합니다.
그러나 먼저 사용자가 `MyQuery`를 만들 수 있는 몇 가지 방법이 있습니다.

### 커맨드라인 인터페이스(CLI)

<!-- The main interface for an application is the command-line interface. Users connect to a full-node and run the CLI directly from their machines - the CLI interacts directly with the full-node. To create `MyQuery` from their terminal, users type the following command: -->
애플리케이션의 기본 인터페이스는 커맨드라인 인터페이스(CLI, command-line interface)입니다. 
사용자는 전체 노드에 연결하고 컴퓨터에서 직접 CLI를 실행합니다. 
CLI는 전체 노드와 직접 상호 작용합니다. 터미널에서 `MyQuery`를 생성하려면 사용자가 다음 명령을 입력합니다.

```bash
simd query staking delegations <delegatorAddress>
```

<!-- This query command was defined by the [`staking`](../../x/staking/spec/README.md) module developer and added to the list of subcommands by the application developer when creating the CLI. -->
이 쿼리 명령은 [`staking`](../../x/staking/spec/README.md) 모듈 개발자에 의해 정의되며, CLI를 만들 때 애플리케이션 개발자가 하위 명령 목록에 추가합니다.

<!-- Note that the general format is as follows: -->
일반적인 형식은 다음과 같습니다.

```bash
simd query [moduleName] [command] <arguments> --flag <flagArg>
```

<!-- To provide values such as `--node` (the full-node the CLI connects to), the user can use the [`app.toml`](../run-node/run-node.md#configuring-the-node-using-apptoml) config file to set them or provide them as flags. -->
`--node`(CLI가 연결하는 전체 노드)와 같은 값을 제공하기 위해, 사용자는 [`app.toml`](../run-node/run-node.md#configuring-the -node-using-apptoml) 구성 파일을 사용하여 설정하거나 플래그로 제공합니다.

<!-- The CLI understands a specific set of commands, defined in a hierarchical structure by the application developer: from the [root command](../core/cli.md#root-command) (`simd`), the type of command (`Myquery`), the module that contains the command (`staking`), and command itself (`delegations`). Thus, the CLI knows exactly which module handles this command and directly passes the call there. -->
CLI는 애플리케이션 개발자가 계층 구조로 정의한 특정 명령 세트를 이해합니다. [루트 명령](../core/cli.md#root-command)(`simd`)에서 커맨드 타입( `Myquery`), 커맨드(`staking`) 및 명령 자체(`delegations`)가 포함된 모듈입니다. 
따라서 CLI는 이 명령을 처리하는 모듈을 정확히 알고 거기에서 호출을 직접 전달합니다.

<!-- Another interface through which users can make queries is [gRPC](https://grpc.io) requests to a [gRPC server](../core/grpc_rest.md#grpc-server). The endpoints are defined as [Protocol Buffers](https://developers.google.com/protocol-buffers) service methods inside `.proto` files, written in Protobuf's own language-agnostic interface definition language (IDL). The Protobuf ecosystem developed tools for code-generation from `*.proto` files into various languages. These tools allow to build gRPC clients easily. -->
사용자가 쿼리할 수 있는 또 다른 인터페이스는 [gRPC 서버](../core/grpc_rest.md#grpc-server)에 대한 [gRPC](https://grpc.io) 요청입니다. 
엔드포인트는 Protobuf의 고유한 언어에 구애받지 않는 IDL(인터페이스 정의 언어)로 작성된 `.proto` 파일 내 [프로토콜 버퍼](https://developers.google.com/protocol-buffers) 서비스 메서드로 정의됩니다. 
Protobuf 생태계는 `*.proto` 파일에서 다양한 언어로 코드 생성을 위한 도구를 개발했습니다. 
이러한 도구를 사용하면 gRPC 클라이언트를 쉽게 구축할 수 있습니다.

<!-- One such tool is [grpcurl](https://github.com/fullstorydev/grpcurl), and a gRPC request for `MyQuery` using this client looks like: -->
이러한 도구 중 하나는 [grpcurl](https://github.com/fullstorydev/grpcurl)이며 이 클라이언트를 사용하는 `MyQuery`에 대한 gRPC 요청은 다음과 같습니다.

```bash
grpcurl \
    -plaintext                                           # We want results in plain test
    -import-path ./proto \                               # Import these .proto files
    -proto ./proto/cosmos/staking/v1beta1/query.proto \  # Look into this .proto file for the Query protobuf service
    -d '{"address":"$MY_DELEGATOR"}' \                   # Query arguments
    localhost:9090 \                                     # gRPC server endpoint
    cosmos.staking.v1beta1.Query/Delegations             # Fully-qualified service method name
```

### REST

<!-- Another interface through which users can make queries is through HTTP Requests to a [REST server](../core/grpc_rest.md#rest-server). The REST server is fully auto-generated from Protobuf services, using [gRPC-gateway](https://github.com/grpc-ecosystem/grpc-gateway). -->
사용자가 쿼리할 수 있는 또 다른 인터페이스는 [REST 서버](../core/grpc_rest.md#rest-server)에 대한 HTTP 요청을 통한 것입니다. 
REST 서버는 [gRPC-gateway](https://github.com/grpc-ecosystem/grpc-gateway)를 사용하여 Protobuf 서비스에서 완전히 자동 생성됩니다.

<!-- An example HTTP request for `MyQuery` looks like: -->
`MyQuery`에 대한 예시 HTTP 요청은 다음과 같습니다.

```bash
GET http://localhost:1317/cosmos/staking/v1beta1/delegators/{delegatorAddr}/delegations
```

## CLI가 쿼리를 핸들링하는 방법 (How Queries are Handled by the CLI)

<!-- The examples above show how an external user can interact with a node by querying its state. To understand more in details the exact lifecycle of a query, let's dig into how the CLI prepares the query, and how the node handles it. The interactions from the users' perspective are a bit different, but the underlying functions are almost identical because they are implementations of the same command defined by the module developer. This step of processing happens within the CLI, gRPC or REST server and heavily involves a `client.Context`. -->
위의 예는 외부 사용자가 상태를 쿼리하여 노드와 상호 작용할 수 있는 방법을 보여줍니다. 
쿼리의 정확한 라이프사이클을 자세히 이해하기 위해 CLI가 쿼리를 준비하는 방법과 노드가 쿼리를 처리하는 방법을 살펴보겠습니다. 
사용자 관점에서의 상호 작용은 약간 다르지만 기본 기능은 모듈 개발자가 정의한 동일한 명령의 구현이기 때문에 거의 동일합니다. 
이 처리 단계는 CLI, gRPC 또는 REST 서버 내에서 발생하며 `client.Context`를 많이 포함합니다.

### Context

<!-- The first thing that is created in the execution of a CLI command is a `client.Context`. A `client.Context` is an object that stores all the data needed to process a request on the user side. In particular, a `client.Context` stores the following: -->
CLI 명령을 실행할 때 가장 먼저 생성되는 것은 `client.Context`입니다. 
`client.Context`는 사용자 측에서 요청을 처리하는 데 필요한 모든 데이터를 저장하는 객체입니다. 특히 `client.Context`는 다음을 저장합니다.

<!-- * **Codec**: The [encoder/decoder](../core/encoding.md) used by the application, used to marshal the parameters and query before making the Tendermint RPC request and unmarshal the returned response into a JSON object. The default codec used by the CLI is Protobuf.
* **Account Decoder**: The account decoder from the [`auth`](../../x/auth/spec/README.md) module, which translates `[]byte`s into accounts.
* **RPC Client**: The Tendermint RPC Client, or node, to which the request will be relayed to.
* **Keyring**: A [Key Manager](../basics/accounts.md#keyring) used to sign transactions and handle other operations with keys.
* **Output Writer**: A [Writer](https://pkg.go.dev/io/#Writer) used to output the response.
* **Configurations**: The flags configured by the user for this command, including `--height`, specifying the height of the blockchain to query and `--indent`, which indicates to add an indent to the JSON response.

The `client.Context` also contains various functions such as `Query()` which retrieves the RPC Client and makes an ABCI call to relay a query to a full-node. -->

* **코덱**: 애플리케이션에서 사용하는 [인코더/디코더](../core/encoding.md), 텐더민트 RPC 요청을 하기 전에 매개변수와 쿼리를 마샬링하고 반환된 응답을 JSON 객체로 언마샬링하는데 사용됩니다. CLI에서 사용하는 기본 코덱은 Protobuf입니다.
* **계정 디코더**: `[]byte`를 계정으로 변환하는 [`auth`](../../x/auth/spec/README.md) 모듈의 계정 디코더.
* **RPC 클라이언트**: 요청이 중계될 텐더민트 RPC 클라이언트 또는 노드입니다.
* **Keyring**: 트랜잭션에 서명하고 키로 다른 작업을 처리하는 데 사용되는 [Key Manager](../basics/accounts.md#keyring)입니다.
* **Output Writer**: 응답을 출력하는 데 사용되는 [Writer](https://pkg.go.dev/io/#Writer)입니다.
* **구성**: 쿼리할 블록체인의 높이를 지정하는 `--height` 및 JSON 응답에 들여쓰기를 추가함을 나타내는 `--indent`를 포함하여 이 명령에 대해 사용자가 구성한 플래그입니다.

`client.Context`에는 RPC 클라이언트를 검색하고 전체 노드에 쿼리를 전달하기 위해 ABCI 호출을 만드는 `Query()`와 같은 다양한 기능도 포함되어 있습니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/client/context.go#L25-L63

<!-- The `client.Context`'s primary role is to store data used during interactions with the end-user and provide methods to interact with this data - it is used before and after the query is processed by the full-node. Specifically, in handling `MyQuery`, the `client.Context` is utilized to encode the query parameters, retrieve the full-node, and write the output. Prior to being relayed to a full-node, the query needs to be encoded into a `[]byte` form, as full-nodes are application-agnostic and do not understand specific types. The full-node (RPC Client) itself is retrieved using the `client.Context`, which knows which node the user CLI is connected to. The query is relayed to this full-node to be processed. Finally, the `client.Context` contains a `Writer` to write output when the response is returned. These steps are further described in later sections. -->
`client.Context`의 주요 역할은 최종 사용자와의 상호 작용 중에 사용되는 데이터를 저장하고 이 데이터와 상호 작용하는 방법을 제공하는 것입니다. 
이 데이터는 전체 노드에서 쿼리를 처리하기 전후에 사용됩니다. 특히 `MyQuery`를 처리할 때 `client.Context`는 쿼리 매개변수를 인코딩하고, 전체 노드를 검색하고, 출력을 작성하는 데 사용됩니다. 
풀 노드로 중계되기 전에 쿼리를 `[]byte` 형식으로 인코딩해야 합니다. 풀 노드는 애플리케이션에 구애받지 않고 특정 유형을 이해하지 못하기 때문입니다. 전체 노드(RPC 클라이언트) 자체는 사용자 CLI가 연결된 노드를 알고 있는 `client.Context`를 사용하여 검색됩니다. 
쿼리는 처리될 이 전체 노드로 전달됩니다. 마지막으로 `client.Context`에는 응답이 반환될 때 출력을 작성하는 `Writer`가 포함되어 있습니다. 이러한 단계는 이후 섹션에서 자세히 설명합니다.

### 인수 및 경로 생성 (Arguments and Route Creation)

<!-- At this point in the lifecycle, the user has created a CLI command with all of the data they wish to include in their query. A `client.Context` exists to assist in the rest of the `MyQuery`'s journey. Now, the next step is to parse the command or request, extract the arguments, and encode everything. These steps all happen on the user side within the interface they are interacting with. -->
라이프사이클의 이 시점에서 사용자는 쿼리에 포함하려는 모든 데이터로 CLI 명령을 생성했습니다. 
`client.Context`는 `MyQuery`의 나머지 부분을 지원하기 위해 존재합니다. 
이제 다음 단계는 명령 또는 요청을 구문 분석하고 인수를 추출하고 모든 것을 인코딩하는 것입니다. 이러한 단계는 모두 상호 작용하는 인터페이스 내에서 사용자 측에서 발생합니다.

#### 인코딩 (Encoding)

<!-- In our case (querying an address's delegations), `MyQuery` contains an [address](./accounts.md#addresses) `delegatorAddress` as its only argument. However, the request can only contain `[]byte`s, as it will be relayed to a consensus engine (e.g. Tendermint Core) of a full-node that has no inherent knowledge of the application types. Thus, the `codec` of `client.Context` is used to marshal the address.

Here is what the code looks like for the CLI command: -->

여기(주소의 위임 쿼리)에서는 `MyQuery`에는 [address](./accounts.md#addresses) `delegatorAddress`가 유일한 인수로 포함됩니다. 
그러나 요청은 애플리케이션 유형에 대한 고유한 지식이 없는 전체 노드의 합의 엔진(예: 텐더민트 코어)으로 중계되기 때문에 `[]byte`만 포함할 수 있습니다. 
따라서 `client.Context`의 `codec`는 주소를 마샬링하는 데 사용됩니다.

CLI 명령에 대한 코드는 다음과 같습니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/x/staking/client/cli/query.go#L323-L326

#### gRPC 쿼리 클라이언트 생성 (gRPC Query Client Creation)

<!-- The Cosmos SDK leverages code generated from Protobuf services to make queries. The `staking` module's `MyQuery` service generates a `queryClient`, which the CLI will use to make queries. Here is the relevant code: -->
코스모스 SDK는 Protobuf 서비스에서 생성된 코드를 활용하여 쿼리를 만듭니다. `staking` 모듈의 `MyQuery` 서비스는 CLI가 쿼리를 만드는 데 사용할 `queryClient`를 생성합니다. 관련 코드는 다음과 같습니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/x/staking/client/cli/query.go#L317-L341

<!-- Under the hood, the `client.Context` has a `Query()` function used to retrieve the pre-configured node and relay a query to it; the function takes the query fully-qualified service method name as path (in our case: `/cosmos.staking.v1beta1.Query/Delegations`), and arguments as parameters. It first retrieves the RPC Client (called the [**node**](../core/node.md)) configured by the user to relay this query to, and creates the `ABCIQueryOptions` (parameters formatted for the ABCI call). The node is then used to make the ABCI call, `ABCIQueryWithOptions()`.

Here is what the code looks like: -->

내부적으로 `client.Context`에는 사전 구성된 노드를 검색하고 쿼리를 전달하는 데 사용되는 `Query()` 함수가 있습니다. 
이 함수는 쿼리 정규화된 서비스 메서드 이름을 경로(이 경우 `/cosmos.staking.v1beta1.Query/Delegations`)로, 인수를 매개변수로 사용합니다. 먼저 이 쿼리를 전달하도록 사용자가 구성한 RPC 클라이언트([**노드**](../core/node.md)라고 함)를 검색하고 `ABCIQueryOptions`(ABCI 호출에 대해 형식이 지정된 매개변수)를 만듭니다). 그런 다음 노드는 ABCI 호출 `ABCIQueryWithOptions()`를 만드는 데 사용됩니다.

코드는 다음과 같습니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/client/query.go#L80-L114

## RPC

<!-- With a call to `ABCIQueryWithOptions()`, `MyQuery` is received by a [full-node](../core/encoding.md) which will then process the request. Note that, while the RPC is made to the consensus engine (e.g. Tendermint Core) of a full-node, queries are not part of consensus and will not be broadcasted to the rest of the network, as they do not require anything the network needs to agree upon.

Read more about ABCI Clients and Tendermint RPC in the [Tendermint documentation](https://docs.tendermint.com/master/rpc/). -->

`ABCIQueryWithOptions()`를 호출하면 `MyQuery`가 [전체 노드](../core/encoding.md)에 의해 수신되어 요청을 처리합니다. 
RPC는 전체 노드의 합의 엔진(예: 텐더민트 코어)에 대해 이루어지지만 쿼리는 합의의 일부가 아니며 네트워크 합의가 필요한 어떤 것도 필요로 하지 않기 때문에 나머지 네트워크에 브로드캐스트되지 않습니다.

ABCI 클라이언트 및 텐더민트 RPC에 대한 자세한 내용은 [텐더민트 문서](https://docs.tendermint.com/master/rpc/)를 참조하십시오.

## 애플리케이션 쿼리 처리 (Application Query Handling)

<!-- When a query is received by the full-node after it has been relayed from the underlying consensus engine, it is now being handled within an environment that understands application-specific types and has a copy of the state. [`baseapp`](../core/baseapp.md) implements the ABCI [`Query()`](../core/baseapp.md#query) function and handles gRPC queries. The query route is parsed, and it matches the fully-qualified service method name of an existing service method (most likely in one of the modules), then `baseapp` will relay the request to the relevant module. -->
기본 합의 엔진에서 중계된 후 전체 노드에서 쿼리를 수신하면 이제 애플리케이션별 유형을 이해하고 상태 사본이 있는 환경 내에서 처리됩니다. 
[`baseapp`](../core/baseapp.md)는 ABCI [`Query()`](../core/baseapp.md#query) 함수를 구현하고 gRPC 쿼리를 처리합니다. 
쿼리 경로가 구문 분석되고 기존 서비스 메서드의 정규화된 서비스 메서드 이름(대부분 모듈 중 하나에 있음)과 일치하면 `baseapp`이 요청을 관련 모듈로 중계합니다.

<!-- Apart from gRPC routes, `baseapp` also handles four different types of queries: `app`, `store`, `p2p`, and `custom`. The first three types (`app`, `store`, `p2p`) are purely application-level and thus directly handled by `baseapp` or the stores, but the `custom` query type requires `baseapp` to route the query to a module's [legacy queriers](../building-modules/query-services.md#legacy-queriers). To learn more about these queries, please refer to [this guide](../core/grpc_rest.md#tendermint-rpc). -->
gRPC 경로 외에도 `baseapp`은 `app`, `store`, `p2p` 및 `custom`과 같은 4가지 다른 유형의 쿼리도 처리합니다. 처음 세 가지 유형(`app`, `store`, `p2p`)은 순전히 애플리케이션 수준이므로 `baseapp` 또는 상점에서 직접 처리하지만 `custom` 쿼리 유형은 쿼리를 다음으로 라우팅하기 위해 `baseapp`이 필요합니다. 
모듈의 [레거시 쿼리](../building-modules/query-services.md#legacy-queriers). 
이러한 쿼리에 대한 자세한 내용은 [본 가이드](../core/grpc_rest.md#tendermint-rpc)를 참조하세요.

<!-- Since `MyQuery` has a Protobuf fully-qualified service method name from the `staking` module (recall `/cosmos.staking.v1beta1.Query/Delegations`), `baseapp` first parses the path, then uses its own internal `GRPCQueryRouter` to retrieve the corresponding gRPC handler, and routes the query to the module. The gRPC handler is responsible for recognizing this query, retrieving the appropriate values from the application's stores, and returning a response. Read more about query services [here](../building-modules/query-services.md).

Once a result is received from the querier, `baseapp` begins the process of returning a response to the user. -->

`MyQuery`에는 `staking` 모듈의 Protobuf 정규화된 서비스 메서드 이름이 있으므로(`/cosmos.staking.v1beta1.Query/Delegations`를 생각해보세요) `baseapp`은 먼저 경로를 구문 분석한 다음 자체 내부 `GRPCQueryRouter ` 해당 gRPC 핸들러를 검색하고 쿼리를 모듈로 라우팅합니다. 
gRPC 핸들러는 이 쿼리를 인식하고 애플리케이션 저장소에서 적절한 값을 검색하고 응답을 반환하는 역할을 합니다. 
쿼리 서비스에 대한 자세한 내용은 [여기](../building-modules/query-services.md)를 참조하세요.

쿼리자로부터 결과가 수신되면 `baseapp`은 사용자에게 응답을 반환하는 프로세스를 시작합니다.

## 응답 (Response)

<!-- Since `Query()` is an ABCI function, `baseapp` returns the response as an [`abci.ResponseQuery`](https://docs.tendermint.com/master/spec/abci/abci.html#query-2) type. The `client.Context` `Query()` routine receives the response and. -->
`Query()`는 ABCI 함수이므로 `baseapp`은 응답을 [`abci.ResponseQuery`](https://docs.tendermint.com/master/spec/abci/abci.html#query-2) 타입으로 반환합니다.  유형. `client.Context` `Query()` 루틴은 응답을 수신합니. 

### CLI 응답 (CLI Response)

<!-- The application [`codec`](../core/encoding.md) is used to unmarshal the response to a JSON and the `client.Context` prints the output to the command line, applying any configurations such as the output type (text, JSON or YAML).

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/client/context.go#L315-L343

And that's a wrap! The result of the query is outputted to the console by the CLI. -->

[`codec`](../core/encoding.md) 애플리케이션은 JSON에 대한 응답을 언마샬하는데 사용되며 `client.Context`는 결과를 커맨드라인에 결과 타입(텍스트, JSON 또는 YAML)과 같은 설정을 적용하여 출력합니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/client/context.go#L315-L343

여기까지입니다. 쿼리 결과는 CLI에 의해 콘솔에 출력됩니다.

## 다음 {hide}

어카운트에 대해 더 읽어보세요 [어카운트](./accounts.md). {hide}