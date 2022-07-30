# gRPC, REST, and Tendermint Endpoints



이 문서는 노드가 노출하는 모든 엔드포인트(gRPC, REST 및 기타 엔드포인트)에 대한 개요를 제공합니다.



## An Overview of All Endpoints

각 노드는 사용자가 노드와 상호 작용할 수 있도록 다음 엔드포인트를 노출하며 각 엔드포인트는 서로 다른 포트에서 제공됩니다. 각 엔드포인트를 구성하는 방법에 대한 세부 정보는 엔드포인트의 자체 섹션에서 제공됩니다.

- gRPC 서버(기본 포트: `9090`)
- REST 서버(기본 포트: `1317`)
- Tendermint RPC 엔드포인트(기본 포트: `26657`)

노드는 Tendermint P2P 엔드포인트 또는 [Prometheus 엔드포인트](https://docs.tendermint.com/master/nodes/metrics.html#metrics)와 같은 일부 다른 엔드포인트를 노출합니다. Cosmos SDK와 직접적인 관련은 없습니다. 이러한 엔드포인트에 대한 자세한 내용은 [Tendermint 문서](https://docs.tendermint.com/master/tendermint-core/using-tendermint.html#configuration)를 참조하세요.



## gRPC Server

Cosmos SDK의 기본 [인코딩](https://docs.cosmos.network/v0.46/core/encoding) 라이브러리는 Protobuf입니다. 이는 Cosmos SDK에 연결할 수 있는 광범위한 Protobuf 기반 도구를 제공합니다. 그러한 도구 중 하나는 [gRPC](https://grpc.io/)이며, 여러 언어로 적절한 클라이언트 지원을 제공하는 최신 오픈소스 고성능 RPC 프레임워크입니다.

각 모듈은 상태 쿼리를 정의하는 [Protobuf `Query` 서비스](https://docs.cosmos.network/v0.46/building-modules/messages-and-queries.html#queries)를 노출합니다. 트랜잭션을 브로드캐스트하는 데 사용되는 `Query` 서비스와 트랜잭션 서비스는 애플리케이션 내부의 다음 기능을 통해 gRPC 서버에 연결됩니다.

```go
		// RegisterGRPCServer registers gRPC services directly with the gRPC
		// server.
		RegisterGRPCServer(grpc.Server)
```



참고: gRPC를 통해 [Protobuf `Msg` 서비스](https://docs.cosmos.network/v0.46/building-modules/messages-and-queries.html#messages) 엔드포인트를 노출할 수 없습니다. 트랜잭션은 gRPC를 사용하여 브로드캐스트되기 전에 CLI를 사용하거나 프로그래밍 방식으로 생성 및 사이닝되어야 합니다. 자세한 내용은 [Generating, Signing, and Broadcasting Transactions](https://docs.cosmos.network/v0.46/run-node/txs.html)를 참조하세요.

`grpc.Server`는 모든 gRPC 쿼리 요청과 브로드캐스트 트랜잭션 요청을 생성하고 제공하는 구체적인 gRPC 서버입니다. 이 서버는 `~/.simapp/config/app.toml`에서 구성할 수 있습니다.

- `grpc.enable = true|false` 필드는 gRPC 서버를 활성화해야 하는지 여부를 정의합니다. 기본값은 'true'입니다.
- `grpc.address = {string}` 필드는 서버가 바인딩해야 하는 주소(실제로 포트, 호스트는 `0.0.0.0`에 유지되어야 하기 때문에)를 정의합니다. 기본값은 '0.0.0.0:9090'입니다.

`~/.simapp`은 노드의 구성과 데이터베이스가 저장되는 디렉토리입니다. 기본적으로 `~/.{app_name}`으로 설정됩니다.

gRPC 서버가 시작되면 gRPC 클라이언트를 사용하여 요청을 보낼 수 있습니다. 몇 가지 예는 [Interact with the Node](https://docs.cosmos.network/v0.46/run-node/interact-node.html#using-grpc) 자습서에 나와 있습니다.

Cosmos SDK와 함께 제공되는 사용 가능한 모든 gRPC 엔드포인트에 대한 개요는 [Protobuf 문서](https://buf.build/cosmos/cosmos-sdk)입니다.



## REST Server

Cosmos SDK supports REST routes via gRPC-gateway.

All routes are configured under the following fields in `~/.simapp/config/app.toml`:

- `api.enable = true|false` field defines if the REST server should be enabled. Defaults to `false`.
- `api.address = {string}` field defines the address (really, the port, since the host should be kept at `0.0.0.0`) the server should bind to. Defaults to `tcp://0.0.0.0:1317`.
- some additional API configuration options are defined in `~/.simapp/config/app.toml`, along with comments, please refer to that file directly.



Cosmos SDK는 gRPC-gateway를 통해 REST 경로를 지원합니다.

모든 경로는 `~/.simapp/config/app.toml`의 다음 필드에서 구성됩니다.

- `api.enable = true|false` 필드는 REST 서버를 활성화해야 하는지 여부를 정의합니다. 기본값은 '거짓'입니다.
- `api.address = {string}` 필드는 서버가 바인딩해야 하는 주소(실제로는 포트, 호스트는 `0.0.0.0`에 유지되어야 하므로)를 정의합니다. 기본값은 `tcp://0.0.0.0:1317`입니다.
- 일부 추가 API 구성 옵션은 `~/.simapp/config/app.toml`에 정의되어 있으며 주석과 함께 해당 파일을 직접 참조하십시오.



### gRPC-gateway REST Routes

여러 가지 이유로 gRPC를 사용할 수 없는 경우(예: 웹 애플리케이션을 만들고 있는데, 브라우저가 gRPC가 기반으로 하는 HTTP2를 지원하지 않는 경우), Cosmos SDK는 gRPC-gateway를 통해 REST 경로를 제공합니다.

[gRPC-gateway](https://grpc-ecosystem.github.io/grpc-gateway/)는 gRPC 엔드포인트를 REST 엔드포인트로 노출하는 도구입니다. Protobuf `Query` 서비스에 정의된 각 gRPC 엔드포인트에 대해 Cosmos SDK는 이에 상응하는 REST를 제공합니다. 예를 들어 밸런스 쿼리는 `/cosmos.bank.v1beta1.QueryAllBalances` gRPC 엔드포인트 또는 gRPC-gateway `"/cosmos/bank/v1beta1/balances/{address}"` REST 엔드포인트를 통해 수행할 수 있습니다. 둘 다 같은 결과를 반환합니다. Protobuf `Query` 서비스에 정의된 각 RPC 메소드에 대해 해당 REST 엔드포인트가 옵션으로 정의됩니다.

```go

  // AllBalances queries the balance of all coins for a single account.
  rpc AllBalances(QueryAllBalancesRequest) returns (QueryAllBalancesResponse) {
    option (google.api.http).get = "/cosmos/bank/v1beta1/balances/{address}";
```


애플리케이션 개발자는 gRPC 게이트웨이 REST 경로를 REST 서버에 연결해야 하며, 이는 ModuleManager에서 `RegisterGRPCGatewayRoutes` 함수를 호출하여 수행됩니다.



### Swagger

API 서버의 '/swagger' 경로 아래에 [Swagger](https://swagger.io/)(또는 OpenAPIv2) 스펙 파일이 노출됩니다. Swagger는 각 엔드포인트에 대한 설명, 입력 인수, 반환 유형 등을 포함하여 서버가 제공하는 API 엔드포인트를 설명하는 공개 스펙입니다.

`/swagger` 엔드포인트 활성화는 기본적으로 true로 설정되는 `api.swagger` 필드를 통해 `~/.simapp/config/app.toml` 내부에서 구성할 수 있습니다.

애플리케이션 개발자의 경우 사용자 정의 모듈을 기반으로 고유한 Swagger 정의를 생성할 수 있습니다. 코스모스 SDK의 [Swagger 생성 스크립트](https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/scripts/protoc-swagger-gen.sh)가 좋은 출발점입니다.



## Tendermint RPC

Cosmos SDK와 별도로 Tendermint는 RPC 서버도 노출합니다. 이 RPC 서버는 `~/.simapp/config/config.toml`의 `rpc` 테이블에서 매개변수를 조정하여 구성할 수 있으며 기본 수신 주소는 `tcp://0.0.0.0:26657`입니다. 모든 Tendermint RPC 엔드포인트의 OpenAPI 사양은 [여기](https://docs.tendermint.com/master/rpc/)에서 확인할 수 있습니다.

일부 Tendermint RPC 끝점은 Cosmos SDK와 직접 관련이 있습니다.

- `/abci_query`: 이 엔드포인트는 애플리케이션에 상태 정보를 쿼리합니다. `path` 매개변수로 다음 문자열을 보낼 수 있습니다.
  - `/cosmos.bank.v1beta1.QueryAllBalances`와 같은 모든 Protobuf 정규화된 서비스 방법. 그런 다음 `data` 필드는 Protobuf를 사용하여 바이트로 인코딩된 메서드의 요청 매개변수를 포함해야 합니다.
  - `/app/simulate`: 트랜잭션을 시뮬레이션하고 사용된 가스와 같은 일부 정보를 반환합니다.
  - `/app/version`: 애플리케이션의 버전을 반환합니다.
  - `/store/{path}`: 저장소(store)를 직접 쿼리합니다.
  - `/p2p/filter/addr/{port}`: 주소 포트 별로 필터링된 노드의 P2P 피어 목록을 반환합니다.
  - `/p2p/filter/id/{id}`: ID 별로 필터링된 노드의 P2P 피어 목록을 반환합니다.
- `/broadcast_tx_{sync,async,commit}`: 이 3개의 엔드포인트는 트랜잭션을 다른 피어에게 브로드캐스트합니다. CLI, gRPC 및 REST는 브로드캐스트 트랜잭션 방법을 제공하지만 모두 내부적으로 이 3개의 Tendermint RPC를 사용합니다.



## Comparison Table

| Name           | Advantages                                                   | Disadvantages                                                |
| :------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| gRPC           | - 다양한 언어로 생성된 코드 스텁을 사용 가능<br/>- 스트리밍 및 양방향 통신(HTTP2) 지원<br/>- 작은 와이어 바이너리 크기, 더 빠른 전송 | - HTTP2 기반, 브라우저에서 사용할 수 없음<br/>- 학습 곡선(주로 Protobuf로 인해) |
| REST           | - 유비쿼터스<br/>- 모든 언어의 클라이언트 라이브러리, 더 빠른 구현 | - 단항 요청-응답 통신만 지원(HTTP1.1)<br/>- 더 큰 유선(over-the-wire) 메시지 크기(JSON) |
| Tendermint RPC | - 사용하기 쉬움                                              | - 더 큰 유선(over-the-wire) 메시지 크기(JSON)                |