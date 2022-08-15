# 권장 폴더 구조(Recommended Folder Structure)

이 문서에서는 Cosmos SDK 모듈의 권장 구조를 간략히 설명합니다. 이러한 아이디어는 제안으로 적용하기 위한 것입니다. 응용 프로그램 개발자는 모듈 구조 및 개발 설계를 개선하고 기여하도록 권장됩니다.

## 구조(Structure)

일반적인 Cosmos SDK 모듈은 다음과 같이 구성할 수 있습니다.

```shell
proto
└── {project_name}
    └── {module_name}
        └── {proto_version}
            ├── {module_name}.proto
            ├── event.proto
            ├── genesis.proto
            ├── query.proto
            └── tx.proto
```

- `{module_name}.proto`: 모듈의 공통 메시지 유형 정의
- `event.proto`: 이벤트와 관련된 모듈의 메시지 유형 정의
- `genesis.proto`: 생성 상태와 관련된 모듈의 메시지 유형 정의
- `query.proto`: 모듈의 쿼리 서비스 및 관련 메시지 유형 정의
- `tx.proto`: 모듈의 메시지 서비스 및 관련 메시지 유형 정의

```shell
x/{module_name}
├── client
│   ├── cli
│   │   ├── query.go
│   │   └── tx.go
│   └── testutil
│       ├── cli_test.go
│       └── suite.go
├── exported
│   └── exported.go
├── keeper
│   ├── genesis.go
│   ├── grpc_query.go
│   ├── hooks.go
│   ├── invariants.go
│   ├── keeper.go
│   ├── keys.go
│   ├── msg_server.go
│   └── querier.go
├── module
│   └── module.go
├── simulation
│   ├── decoder.go
│   ├── genesis.go
│   ├── operations.go
│   └── params.go
├── spec
│   ├── 01_concepts.md
│   ├── 02_state.md
│   ├── 03_messages.md
│   └── 04_events.md
├── {module_name}.pb.go
├── abci.go
├── codec.go
├── errors.go
├── events.go
├── events.pb.go
├── expected_keepers.go
├── genesis.go
├── genesis.pb.go
├── keys.go
├── msgs.go
├── params.go
├── query.pb.go
└── tx.pb.go
```

- `client/`: 모듈의 CLI 클라이언트 기능 구현 및 모듈의 통합 테스트 제품군
- `exported/`: 모듈의 내보낸 유형 - 일반적으로 인터페이스 유형입니다. 모듈이 다른 모듈의 키퍼에 의존하는 경우 키퍼를 구현하는 모듈에 대한 직접적인 종속성을 피하기 위해 `expected_keepers.go` 파일(아래 참조)을 통해 인터페이스 계약으로 키퍼를 수신해야 합니다. 그러나 이러한 인터페이스 계약은 키퍼를 구현하는 모듈에 특정한 반환 유형 및/또는 작동하는 메서드를 정의할 수 있으며 여기에서 `exported/`가 작동합니다. `exported/`에 정의된 인터페이스 유형은 표준 유형을 사용하므로 모듈이 `expected_keepers.go` 파일을 통해 인터페이스 계약으로 키퍼를 수신할 수 있습니다. 이 패턴을 사용하면 코드를 DRY 상태로 유지하고 가져오기 주기 혼돈을 완화할 수 있습니다.
- `keeper/`: 모듈의 `Keeper` 및 `MsgServer` 구현
- `module/`: 모듈의 `AppModule` 및 `AppModuleBasic` 구현
- `simulation/`: 모듈의 [simulation](./simulator.html) 패키지는 블록체인 시뮬레이터 애플리케이션(`simapp`)에서 사용하는 기능을 정의합니다.
- `spec/`: 모듈의 사양 문서는 중요한 개념, 상태 저장 구조, 메시지 및 이벤트 유형 정의를 간략하게 설명합니다.
- 루트 디렉토리에는 프로토콜 버퍼에 의해 생성된 유형 정의를 포함하여 메시지, 이벤트 및 기원 상태에 대한 유형 정의가 포함됩니다.
  - `abci.go`: 모듈의 `BeginBlocker` 및 `EndBlocker` 구현(이 파일은 `BeginBlocker` 및/또는 `EndBlocker`를 정의해야 하는 경우에만 필요합니다.)
  - `codec.go`: 인터페이스 유형에 대한 모듈의 레지스트리 메소드
  - `errors.go`: 모듈의 센티넬 오류
  - `events.go`: 모듈의 이벤트 유형 및 생성자
  - `expected_keepers.go`: 모듈의 [예상 키퍼](./keeper.html#type-definition) 인터페이스
  - `genesis.go`: 모듈의 제네시스 상태 메서드 및 도우미 기능
  - `keys.go`: 모듈의 저장 키 및 관련 도우미 기능
  - `msgs.go`: 모듈의 메시지 유형 정의 및 관련 메서드
  - `params.go`: 모듈의 매개변수 유형 정의 및 관련 메소드
  - `*.pb.go`: 프로토콜 버퍼에 의해 생성된 모듈의 유형 정의(위의 각 `*.proto` 파일에 정의됨)
