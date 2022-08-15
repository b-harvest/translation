<!--
order: 4
-->

# 노드 클라이언트 (데몬)

Cosmos SDK 응용 프로그램의 메인 endpoint는 데몬 클라이언트(풀노드 클라이언트라고도 함)입니다. 풀노드는 제네시스 파일에서 시작하여 state-machine를 실행합니다. 트랜잭션을 수신 및 릴레이하고 블록 프로포절 및 서명을 하기 위해 동일한 클라이언트를 실행하는 피어에 연결합니다. 풀노드는 코스모스 SDK로 정의된 애플리케이션과 ABCI를 통해 애플리케이션과 연결된 합의 엔진으로 구성된다. {개요}

## 미리 읽어 보아야할 것들

* [SDK 어플리케이션의 해부학적 구조](../basics/app-anatomy.md) {prereq}

## `main` 함수

모든 Cosmos SDK 애플리케이션의 풀노드 클라이언트는 `main` 함수를 실행하여 빌드됩니다. 클라이언트는 일반적으로 애플리케이션 이름에 접미사 `-d`를 추가하여 이름을 지정하고(예: `app`이라는 애플리케이션의 경우 `appd`), `main` 함수는 `./appd/cmd/main.go` 파일에 정의됩니다. 이 함수를 실행하면 command들을 포함하는 `appd` 실행파일이 생성됩니다. `app`이라는 앱의 경우 메인 명령은 풀노드를 시작하는 [`appd start`](#start-command)입니다.


일반적으로, 개발자는 다음과 같은 구조로 `main.go` 함수를 구현합니다.

* 첫번째로, [`encodingCodec`](./encoding.md)는 어플리케이션을 위해서 인스턴스화 되어있습니다.
* 그리고나서, `config`가 해석되고, 설정변수들이 셋팅됩니다. 이것은 주로 [addresses](../basics/accounts.md#addresses)를 위한 Bech32접두사를 셋팅하는 것을 포함합니다.
  +++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/types/config.go#L14-L29
* [cobra](https://github.com/spf13/cobra)를 사용해 풀노드 클라이언트의 root 커멘드가 생성됩니다. 그 이후에, 모든 커스텀 커멘드는 `rootCmd`의 방법인 `AddCommand()`를 이용하여 추가됩니다.
* `server.AddCommands()`를 이용해 `rootCmd`에 기본 서버 커멘드를 추가합니다. 이러한 명령어는 Cosmos SDK 레벨에서 정의된 표준으로 위에서 추가한 커멘드와 구분됩니다. 이러한 커멘드는 모든 Cosmos SDK 기반 애플리케이션에서 공유해야 합니다. 여기에는 가장 중요한 명령인 [`start` 명령](#start-command)가 포함됩니다.
* `executor`를 준비하고 실행합니다.
   +++ https://github.com/tendermint/tendermint/blob/v0.35.4/libs/cli/setup.go#L74-L78

데모 목적을 위해 Cosmos SDK기반의 어플리케이션인 `simapp`의 `main`함수 예시를 확인합니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/simapp/simd/main.go

## `start` 커멘드

`start` 커멘드는 Cosmos SDK의 `/server` 폴더에 정의되어 있습니다. [`main` 함수](#main-function)에서 풀노드 클라이언트의 루트 명령에 추가되어 있고, end-user가 노드를 시작하기 위해 호출합니다.

```bash
# 예를들어 어플리케이션 이름이 "app" 일 때, 다음 커멘드는 풀노드를 시작합니다.
appd start

# Cosmos SDK내의 simapp을 사용하면, 다음 커멘드는 simapp을 실행합니다.
simd start
```

다시 말해, 풀노드는 네트워킹 계층, 합의 계층 및 어플리케이션 프로그램 계층의 세 가지 개념 계층으로 구성됩니다. 처음 두 개는 일반적으로 합의 엔진(기본적으로 텐더민트 코어)이라는 엔티티에 함께 번들로 제공되는 반면, 세 번째는 Cosmos SDK의 도움으로 정의된 state-machine입니다. 현재 Cosmos SDK는 기본 합의 엔진으로 텐더민트를 사용합니다. 즉, start 커멘드는 텐더민트 노드를 부팅하기 위해 구현되었습니다.

`start` 커멘드의 흐름은 매우 쉽습니다. 먼저, `db` (기본적으로 [`leveldb`](https://github.com/syndtr/goleveldb)입니다)를 열기 위해, `context`에서 `config`를 검색합니다.

이 `db`는 가장 최근의 어플리케이션 state를 저장하고 있습니다.(만약 어플리케이션이 처음실행되었다면, 비어있습니다.)

`db`와 함께, `start` 커멘드는 `appCreator` 함수를 활용환 어플리케이션의 새로운 인스턴스를 생성합니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/server/start.go#L209-L209

`appCreator`는  `AppCreator` 서명을 만족하는 함수입니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/server/types/app.go#L57-L59

실제로, 어플리케이션 [생성자](../basics/app-anatomy.md#constructor-function)는 `appCreator`로 전달됩니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/simapp/simd/cmd/root.go#L246-L295

그리고 나서, `app`의 인스턴스는 새로운 텐더민트 노드에 의해서 인스턴스화 됩니다.

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/server/start.go#L291-L294

The Tendermint node can be created with `app` because the latter satisfies the [`abci.Application` interface](https://github.com/tendermint/tendermint/blob/v0.35.4/abci/types/application.go#L7-L32) (given that `app` extends [`baseapp`](./baseapp.md)).
텐더민트 노드는 `app`함께 생성가능 합니다. 왜냐하면, 후자는 [`abci.Application` 인터페이스](https://github.com/tendermint/tendermint/blob/v0.35.4/abci/types/application.go#L7-L32)하기 떄문입니다. (이러한 사항은 `app` 에서 이어서 확인 [`baseapp`](./baseapp.md))

'node.New' 방법의 일부로 텐더민트는 애플리케이션의 height(즉, genesis 이 후의 블록 수)가 텐더민트 노드의 높이와 동일한지 확인합니다.

이 두 높이의 차이는 항상 음수이거나 null이어야 합니다. 특히 음수인 경우 `node.New`는 애플리케이션의 높이가 텐더민트 노드의 높이에 도달할 때까지 블록을 재생합니다.

마지막으로 애플리케이션의 높이가 '0'이면 텐더민트 노드는 애플리케이션에서 [`InitChain`](./baseapp.md#initchain)을 호출하여 제네시스 파일에서 상태를 초기화합니다.

텐더민트 노드가 인스턴스화되고 애플리케이션과 동기화되면 노드를 시작할 수 있습니다

+++ https://github.com/cosmos/cosmos-sdk/blob/v0.46.0-rc1/server/start.go#L296-L298

시작 시 노드는 RPC 및 P2P 서버를 부트스트랩하고 피어에 연결을 시작합니다. 피어와 핸드셰이크하는 동안 노드가 자신이 앞서고 있음을 인식하면 따라잡기 위해 모든 블록을 순차적으로 쿼리합니다. 그런 다음 새로운 진행을 위해, 새로운 블록 프로퍼절과 벨리데이터의 블록 사인을 기다립니다.

## 댜른 커멘드

노드를 구체적으로 실행하고 상호작용하는 방법을 알아보려면 [노드, API 및 CLI 실행](../run-node/README.md) 가이드를 참조하세요.

## 다음 {hide}

[저장](./store.md)에 대하여 배우세요. {hide}
