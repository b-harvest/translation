# 명령줄 인터페이스

이 문서는 [**application**](../basics/app-anatomy.md)에 대해 CLI(명령줄 인터페이스)가 high-level에서 작동하는 방식을 설명합니다. Cosmos SDK [**module**](../building-modules/intro.md)용 CLI 구현을 위한 별도 문서는 [여기](../building-modules/module-interfaces.md#)에서 찾을 수 있습니다.

## 명령줄 인터페이스

### 예제 명령

CLI를 생성하는 정해진 방법은 없으나 Cosmos SDK 모듈은 일반적으로 [Cobra Library](https://github.com/spf13/cobra)를 사용합니다. Cobra를 통해 CLI를 구축하려면 명령, 인수 및 플래그를 정의해야 합니다. [**명령**](#commands)은 트랜잭션 생성을 위한 `tx` 및 애플리케이션 쿼리를 위한 `query`와 같이 사용자가 수행하고자 하는 작업을 이해합니다. 각 명령에는 특정 트랜잭션 유형의 이름을 지정하는 데 필요한 중첩된 하위 명령이 있을 수도 있습니다. 사용자는 또한 코인을 보낼 계좌 번호와 같은 **인수**와 가스 가격 또는 브로드캐스트할 노드와 같은 명령의 다양한 측면을 수정하기 위해 [**Flags**](#flags)를 제공합니다.

다음은 사용자가 simapp CLI `simd`와 상호 작용하기 위해 입력할 수 있는 명령으로 일부 토큰을 보내는 명령의 예입니다.

```bash
simd tx bank send $MY_VALIDATOR_ADDRESS $RECIPIENT 1000stake --gas auto --gas-prices <gasPrices>
```

처음부터 4개의 문자열(simd, tx, bank, send)은 각각 다음을 뜻합니다.

- `simd`: 전체 애플리케이션 에 대한 루트 명령입니다.
- `tx`: 사용자가 트랜잭션을 생성할 수 있는 모든 명령을 포함하는 하위 명령입니다.
- `bank`: 명령를 라우팅할 모듈을 나타내는 하위 명령입니다.(이 경우 [`x/bank`](../../x/bank/spec/README.md) 모듈)
- `send`: 트랜잭션 `send`의 유형(type)입니다.

다음 두 문자열($MY_VALIDATOR_ADDRESS, $RECIPIENT)은 토큰을 보내려는 유저의 주소인 `from_address`, 받는 유저의 주소인 `to_address`, 보내려는 토큰의 양 `amount`으로 모두 인수(argument)입니다. 마지막으로, 명령의 마지막 몇 문자열은 사용자가 수수료로 얼마를 지불할 의향이 있는지 나타내는 선택적 플래그입니다.(거래를 실행하는 데 사용된 가스의 양과 유저가 제공한 가스 가격을 사용하여 계산됨)

CLI는 [node](../core/node.md)와 상호 작용하여 이 명령을 처리합니다. 인터페이스 자체는 `main.go` 파일에 정의되어 있습니다.

### CLI 빌드

`main.go` 파일에는 루트 명령을 생성하는 `main()` 함수가 있어야 하며, 여기에는 모든 애플리케이션 명령이 하위 명령으로 추가됩니다. 루트 명령은 다음을 추가로 처리합니다.

- **구성 설정**: 구성 파일(예: Cosmos SDK 구성 파일)을 읽어 구성을 설정합니다.
- **플래그 추가**: 예를 들어 `--chain-id` 가 있습니다.
- **`codec` 을 인스턴스화**: 애플리케이션의 `MakeCodec()` 함수(`simapp`에서 `MakeTestEncodingConfig`라고 함)를 호출하여 `codec` 을 인스턴스화 합니다. [`codec`](../core/encoding.md)은 애플리케이션의 데이터 구조를 인코딩 및 디코딩하는 데 사용됩니다. 저장소는 `[]byte`만 유지할 수 있으므로 개발자는 데이터 구조에 대한 직렬화 형식을 정의해야 합니다. 또는 기본값인 Protobuf를 사용합니다.
- **하위 명령 추가**: [트랜잭션 명령](#transaction-commands) 및 [쿼리 명령](#query-commands)을 포함하여 가능한 모든 사용자 상호 작용에 대한 하위 명령을 추가합니다.

`main()` 함수는 마지막으로 실행기(executor)를 만들고 루트 명령을 [실행](https://pkg.go.dev/github.com/spf13/cobra#Command.Execute)합니다. `simapp` 애플리케이션의 `main()` 함수의 예를 참조하세요.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/simapp/simd/main.go#L12-L24

문서의 나머지 부분에서는 각 단계에 대해 구현해야 하는 사항을 자세히 설명하고 `simapp` CLI 파일의 코드 일부를 포함합니다.

## CLI에 명령 추가

모든 애플리케이션 CLI는 먼저 루트 명령을 생성한 다음 `rootCmd.AddCommand()`를 사용하여 하위 명령(종종 추가 중첩 하위 명령 포함)을 집계하여 기능을 추가합니다. 애플리케이션 고유 기능의 대부분은 각각 `TxCmd` 및 `QueryCmd`라고 하는 트랜잭션 및 쿼리 명령에 있습니다.

### 루트 명령

루트 명령(`rootCmd`라고 함)은 사용자가 상호 작용하려는 응용 프로그램을 나타내기 위해 명령줄에서 가장 처음에 입력하는 것입니다. 명령을 호출하는 데 사용되는 문자열("Use" 필드)은 일반적으로 `-d`접미사가 붙은 응용 프로그램의 이름입니다(예:`simd`또는`gaiad`). 루트 명령에는 일반적으로 응용 프로그램의 기본 기능을 지원하는 다음 명령이 포함됩니다.

- **Status 명령**: Cosmos SDK rpc 클라이언트 도구의 status 명령은 연결된 [`Node`](../core/node.md)의 상태에 대한 정보를 인쇄합니다. 노드의 상태에는 `NodeInfo`, `SyncInfo` 및 `ValidatorInfo`가 포함됩니다.
- **Key 명령**: Key [commands](https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/client/keys)에 대한 하위 명령 모음이 포함된 Cosmos SDK 클라이언트 도구 새 키 추가 및 키링에 저장, 키링에 저장된 모든 공개 키 나열, 키 삭제를 포함하여 Cosmos SDK 암호화 도구의 키 기능을 사용합니다. 예를 들어 사용자는 `simd keys add <name>`을 입력하여 새 키를 추가하고 암호화된 사본을 키링에 저장할 수 있습니다. 플래그 `--recover`를 사용하여 시드 구문에서 개인 키를 복구하거나 플래그 `--multisig`를 사용하여 여러 키를 그룹화하여 다중 서명 키를 만듭니다. `add` 키 명령에 대한 자세한 내용은 [여기](https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/client/keys/add.go) 코드를 참조하세요.
- **Server 명령**: Cosmos SDK 서버 패키지에 존재하는 명령입니다. 이러한 명령은 ABCI Tendermint 애플리케이션을 시작하는 데 필요한 메커니즘을 제공하고 애플리케이션을 완전히 부트스트랩하는 데 필요한 CLI 프레임워크([cobra](github.com/spf13/cobra) 기반)를 제공합니다. 이 패키지는 `StartCmd` 및 `ExportCmd`라는 두 가지 핵심 기능을 노출하여 애플리케이션을 시작하고 상태를 내보내는 명령을 각각 생성합니다. 자세한 내용은 [여기](https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/server)를 클릭하세요.
- **[트랜잭션](#transaction-commands) 명령**
- **[Query](#query-commands) 명령**

다음은 `simapp` 애플리케이션의 `rootCmd` 함수의 예입니다. 루트 명령을 인스턴스화하고 모든 실행 전에 실행할 [_persistent_ flag](#flags) 및 `PreRun` 기능을 추가하고 필요한 모든 하위 명령을 추가합니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/simapp/simd/cmd/root.go#L39-L85

`rootCmd`에는 `initAppConfig()`라는 함수가 있어 애플리케이션의 사용자 지정 구성을 설정하는 데 유용합니다.
기본적으로 앱은 `initAppConfig()`를 통해 덮어쓸 수 있는 Cosmos SDK의 Tendermint 앱 구성 템플릿을 사용합니다.
다음은 기본 `app.toml` 템플릿을 재정의하는 예제 코드입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/simapp/simd/cmd/root.go#L99-L153

`initAppConfig()`는 기본 Cosmos SDK의 [서버 구성](https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/server/config/config.go#L235)을 재정의할 수도 있습니다. 한 가지 예는 `min-gas-prices` 구성으로, 검증인이 거래 처리를 위해 수락할 최소 가스 가격을 정의합니다. 기본적으로 Cosmos SDK는 이 매개변수를 `""`(빈 문자열)로 설정하여 모든 유효성 검사자가 자체 `app.toml`을 조정하고 비어 있지 않은 값을 설정하도록 합니다. 그렇지 않으면 노드가 시작 시 중지됩니다. 이것은 검증자를 위한 최고의 UX가 아닐 수 있으므로 체인 개발자는 이 `initAppConfig()` 함수 내에서 검증인에 대한 기본 `app.toml` 값을 설정할 수 있습니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/simapp/simd/cmd/root.go#L119-L134

루트 수준의 `status` 및 `keys` 하위 명령은 대부분의 응용 프로그램에서 일반적이며 응용 프로그램 상태와 상호 작용하지 않습니다. 사용자가 실제로 _할 수 있는_ 대부분의 애플리케이션 기능은 `tx` 및 `query` 명령으로 활성화됩니다.

### 트랜잭션 명령

[Transactions](./transactions.md)는 상태 변경을 트리거하는 [`Msg`](../building-modules/messages-and-queries.md#messages)를 래핑하는 객체입니다. CLI 인터페이스를 사용하여 트랜잭션 생성을 활성화하기 위해 일반적으로 `txCommand` 기능이 `rootCmd`에 추가됩니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/simapp/simd/cmd/root.go#L175-L181

이 `txCommand` 기능은 애플리케이션에 대해 최종 사용자가 사용할 수 있는 모든 트랜잭션을 추가합니다. 여기에는 일반적으로 다음이 포함됩니다.

- **Sign 명령**: 트랜잭션에서 메시지에 서명 하는[`auth`](../../x/auth/spec/README.md) 모듈의 sign 명령입니다. 다중서명을 활성화하려면 `auth` 모듈의 `MultiSign` 명령을 추가하십시오. 모든 트랜잭션이 유효하려면 일종의 서명이 필요하기 때문에 서명 명령은 모든 애플리케이션에 필요합니다.
- **Broadcast 명령**: Cosmos SDK 클라이언트 도구에서 broadcast 명령을 사용하여 트랜잭션을 브로드캐스트합니다.
- **모든 [모듈 트랜잭션 명령](../building-modules/module-interfaces.md#transaction-commands)**: 응용 프로그램이 의존하는 모든 모듈 트랜잭션 명령은 [기본 모듈 관리자](../building- module/module-manager.md#basic-manager) `AddTxCommands()` 기능을 사용하여 검색됩니다.

다음은 `simapp` 애플리케이션에서 이러한 하위 명령을 집계하는 `txCommand`의 예입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/simapp/simd/cmd/root.go#L215-L240

### 쿼리 명령

[**Queries**](../building-modules/messages-and-queries.md#queries)는 사용자가 애플리케이션 상태에 대한 정보를 검색할 수 있도록 하는 객체입니다. CLI 인터페이스를 사용하여 쿼리 생성을 활성화하기 위해 일반적으로 `queryCommand` 기능이 `rootCmd`에 추가됩니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/simapp/simd/cmd/root.go#L175-L181

이 `queryCommand` 기능은 최종 사용자가 애플리케이션에 사용할 수 있는 모든 쿼리를 추가합니다. 여기에는 일반적으로 다음이 포함됩니다.

- **QueryTx** 및/또는 기타 트랜잭션 쿼리 명령] 사용자가 해시, 태그 목록 또는 블록 높이를 입력하여 트랜잭션을 검색할 수 있도록 하는 `auth` 모듈. 이러한 쿼리를 통해 사용자는 트랜잭션이 블록에 포함되었는지 확인할 수 있습니다.
- `auth` 모듈의 **계정 명령**은 주소가 지정된 계정의 상태(예: 계정 잔액)를 표시합니다.
- Cosmos SDK rpc 클라이언트 도구의 **Validator 명령**은 주어진 높이의 유효성 검사기 세트를 표시합니다.
- 주어진 높이에 대한 블록 데이터를 표시하는 Cosmos SDK rpc 클라이언트 도구의 **블록 명령**.
- **모든 [모듈 쿼리 명령](../building-modules/module-interfaces.md#query-commands)** [기본 모듈 관리자](../building- module/module-manager.md#basic-manager) `AddQueryCommands()` 함수.

다음은 `simapp` 애플리케이션에서 하위 명령을 집계하는 `queryCommand`의 예입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/simapp/simd/cmd/root.go#L191-L213

## 플래그

플래그는 명령을 수정하는 데 사용됩니다. 개발자는 CLI를 사용하여 `flags.go` 파일에 이를 포함할 수 있습니다. 사용자는 명령에 명시적으로 포함하거나 [`app.toml`](../run-node/run-node.md#configuring-the-node-using-apptoml) 내부에서 사전 구성할 수 있습니다. 일반적으로 사전 구성된 플래그에는 연결할 `--node`와 사용자가 상호 작용하려는 블록체인의 `--chain-id`가 포함됩니다.

명령에 추가된 _persistent_ 플래그(_local_ 플래그와 반대)는 모든 하위 명령을 초월합니다. 하위 명령은 이러한 플래그에 대해 구성된 값을 상속합니다. 또한 모든 플래그는 명령에 추가될 때 기본값을 갖습니다. 일부는 옵션을 해제하지만 다른 일부는 사용자가 유효한 명령을 생성하기 위해 재정의해야 하는 빈 값입니다. 플래그는 명시적으로 *required*로 표시되어 사용자가 값을 제공하지 않으면 자동으로 오류가 발생하지만 예기치 않은 누락 플래그를 다르게 처리하는 것도 허용됩니다.

플래그는 명령에 직접 추가되며(일반적으로 모듈 명령이 정의된 [모듈의 CLI 파일](../building-modules/module-interfaces.md#flags)) `rootCmd` 영구 플래그를 제외하고는 플래그가 없어야 합니다. 응용 프로그램 수준에서 추가됩니다. 루트 명령에 애플리케이션이 속한 블록체인의 고유 식별자인 `--chain-id`에 대한 _persistent_ 플래그를 추가하는 것이 일반적입니다. 이 플래그를 추가하는 것은 `main()` 함수에서 할 수 있습니다. 이 플래그를 추가하는 것은 이 애플리케이션 CLI의 명령 간에 체인 ID가 변경되어서는 안 되므로 의미가 있습니다. 다음은 `simapp` 애플리케이션의 예입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/simapp/simd/cmd/root.go#L210-L210

## 환경 변수

각 플래그는 해당하는 명명된 환경 변수에 바인딩됩니다. 그런 다음 환경 변수의 이름은 대문자 `basename` 뒤에 플래그의 플래그 이름이 오는 두 부분으로 구성됩니다. `-`는 `_`로 대체되어야 합니다. 예를 들어 기본 이름이 `GAIA`인 애플리케이션의 플래그 `--home`은 `GAIA_HOME`에 바인딩됩니다. 일상적인 작업에 대해 입력되는 플래그의 양을 줄일 수 있습니다. 예를 들어 다음 대신:

```sh
gaia --home=./ --node=<node address> --chain-id="testchain-1" --keyring-backend=test tx ... --from=<key name>
```

이것이 더 편리할 것입니다:

```sh
# .env, .envrc 등의 환경 변수를 정의합니다.
GAIA_HOME=<path to home>
GAIA_NODE=<node address>
GAIA_CHAIN_ID="testchain-1"
GAIA_KEYRING_BACKEND="test"

# 나중에 그냥 사용
gaia tx ... --from=<key name>
```

## 구성

애플리케이션의 루트 명령은 명령을 실행하기 위해 `PersistentPreRun()` cobra 명령 속성을 사용하는 것이 중요하므로 모든 하위 명령이 서버 및 클라이언트 컨텍스트에 액세스할 수 있습니다. 이러한 컨텍스트는 초기에 기본값으로 설정되며 각각의 `PersistentPreRun()` 함수에서 명령 범위로 수정될 수 있습니다. `client.Context`는 일반적으로 모든 명령이 필요한 경우 상속하고 재정의하는 데 유용한 "기본값" 값으로 미리 채워져 있습니다.

다음은 `simapp`의 `PersistentPreRun()` 함수의 예입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/simapp/simd/cmd/root.go#L56-L79

`SetCmdClientContextHandler` 호출은 `client.Context`를 만들고 루트 명령의 `Context`에 설정하는 `ReadPersistentCommandFlags`를 통해 영구 플래그를 읽습니다.

`InterceptConfigsPreRunHandler` 호출은 바이퍼 리터럴, 기본 `server.Context` 및 로거를 생성하고 이를 루트 명령의 `Context`에 설정합니다. `server.Context`는 제공된 홈 경로를 기반으로 Tendermint 구성을 읽거나 생성하는 내부 `interceptConfigs` 호출을 통해 수정되고 디스크에 저장됩니다. 또한 `interceptConfigs`는 애플리케이션 구성 `app.toml`을 읽고 로드하여 `server.Context` 바이퍼 리터럴에 바인딩합니다. 이는 애플리케이션이 CLI 플래그뿐만 아니라 이 파일에서 제공하는 애플리케이션 구성 값에도 액세스할 수 있도록 하는 데 매우 중요합니다.

## 다음

[events](./events.md)에 대해 알아보기
