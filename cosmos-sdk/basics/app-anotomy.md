# Anatomy of a Cosmos SDK Application



이 문서는 Cosmos SDK 애플리케이션의 핵심 부분을 설명합니다. 문서 전체에서 `app`이라는 플레이스-홀더 애플리케이션이 사용됩니다.



## Node Client

Daemon 또는 [Full-Node Client](https://docs.cosmos.network/v0.46/core/node.html)는 Cosmos SDK 기반 블록체인의 핵심 프로세스입니다. 네트워크의 참가자는 이 프로세스를 실행하여 상태 머신을 초기화하고, 다른 전체 노드와 연결하고, 새 블록이 들어올 때 상태 머신을 업데이트합니다.

```
                ^  +-------------------------------+  ^
                |  |                               |  |
                |  |  State-machine = Application  |  |
                |  |                               |  |   Built with Cosmos SDK
                |  |            ^      +           |  |
                |  +----------- | ABCI | ----------+  v
                |  |            +      v           |  ^
                |  |                               |  |
Blockchain Node |  |           Consensus           |  |
                |  |                               |  |
                |  +-------------------------------+  |   Tendermint Core
                |  |                               |  |
                |  |           Networking          |  |
                |  |                               |  |
                v  +-------------------------------+  v

```



블록체인 전체 노드는 일반적으로 "데몬"의 경우 접미사가 `-d`인 바이너리로 표시됩니다(예: `app`의 경우 `appd`, `gaia`의 경우 `gaiad`). 이 바이너리는 `./cmd/appd/`에 있는 간단한 [`main.go`](https://docs.cosmos.network/v0.46/core/node.html#main-function) 함수를 실행하여 빌드됩니다. 이 작업은 일반적으로 [Makefile](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#dependencies-and-makefile)을 통해 발생합니다.
기본 바이너리가 빌드되면 [`start` 명령](https://docs.cosmos.network/v0.46/core/node.html#start-command)을 실행하여 노드를 시작할 수 있습니다. 이 명령 함수는 주로 다음 세 가지 작업을 수행합니다.

1.  [`app.go`](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#core-application-file)에 정의된 상태 머신의 인스턴스를 만듭니다.
2. `~/.app/data` 폴더에 저장된 db에서 추출한 최신 상태로 상태 머신을 초기화합니다. 이 시점에서 상태 머신은 `appBlockHeight` 높이에 있습니다.
3. 새 Tendermint 인스턴스를 만들고 시작합니다. 무엇보다도 노드는 피어와 핸드셰이크를 수행합니다. 최신 `blockHeight`를 가져오고 로컬 `appBlockHeight`보다 큰 경우 이 높이와 동기화할 블록을 재생합니다. `appBlockHeight`가 0이면 노드가 제네시스에서 시작되고 Tendermint는 ABCI를 통해 `InitChain` 메시지를 앱으로 전송하여  [`InitChainer`](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#initchainer)를 트리거합니다.



## Core Application File

일반적으로 상태 머신의 코어는  `app.go`라는 파일에 정의되어 있습니다. 주로 애플리케이션의 타입 정의와 애플리케이션을 만들고 초기화하는 함수를 포함합니다.



### Type Definition of the Application

`app.go`에 정의된 첫 번째 것은 애플리케이션의 타입입니다. 일반적으로 다음 부분으로 구성됩니다.

- **[`baseapp`](https://docs.cosmos.network/v0.46/core/baseapp.html)에 대한 참조** : `app.go`에 정의된 사용자 지정 애플리케이션은`baseapp`의 확장입니다.  트랜잭션이 Tendermint에 의해 애플리케이션으로 릴레이되면, 앱은 `baseapp`의 메소드를 사용하여 트랜잭션을 적절한 모듈로 라우팅합니다.  baseapp은 모든 [ABCI 메소드](https://docs.tendermint.com/master/spec/abci/abci.html)  및 [라우팅 로직](https://docs.cosmos.network/v0.46/core/baseapp.html#routing)을 포함하여 애플리케이션에 대한 대부분의 핵심 로직을 구현합니다.

- **스토어 키 목록** : 전체 상태를 담고 있는 [스토어(Store, 저장소)](https://docs.cosmos.network/v0.46/core/store.html)는 Cosmos SDK에서 [`멀티스토어`](https://docs.cosmos. 코스모스 SDK의 network/v0.46/core/store.html#multistore) (즉, a store of store)로 구현됩니다. 각 모듈은 상태의 일부를 유지하기 위해 다중 스토어에 있는 하나 이상의 스토어를 사용합니다. 이러한 스토어는 `app` 타입에 선언된 특정 키로 액세스할 수 있습니다. 이 키는 `Keeper`와 함께 Cosmos SDK의 [객체 기능 모델](https://docs.cosmos.network/v0.46/core/ocap.html)의 핵심입니다.

- **모듈의 `키퍼` 목록** : 각 모듈은 이 모듈의 스토어에 대한 읽기 및 쓰기를 처리하는  [`keeper`](https://docs.cosmos.network/v0.46/building-modules/keeper.html)라는 추상화를 정의합니다.  한 모듈의 `keeper` 메소드는 승인된 경우 다른 모듈에서 호출될 수 있습니다. 이것이 키퍼들이 애플리케이션의 타입으로 선언되고 다른 모듈에 대한 인터페이스로 내보내져 그 모듈에서 키퍼에 대해 승인된 기능에만 액세스할 수 있도록 하는 이유입니다.

- **[`appCodec`](https://docs.cosmos.network/v0.46/core/encoding.html)에 대한 참조** : 애플리케이션의 `appCodec`은 데이터 구조를 직렬화 및 역직렬화하여 스토어에 저장합니다. 스토어는 `[]bytes`만 저장할 수 있기 때문에 데이터를 직렬화하여 []bytes 형태로 변환해야 합니다. 기본 코덱은 [프로토콜 버퍼](https://docs.cosmos.network/v0.46/core/encoding.html)입니다.

- **[`legacyAmino`](https://docs.cosmos.network/v0.46/core/encoding.html) 코덱에 대한 참조** : Cosmos SDK의 일부는 아직 위의 `appCodec`을 사용하도록 마이그레이션되지 않아서 Amino를 사용하도록 하드코딩되어 있습니다. 다른 부분은 이전 버전과의 호환성을 위해 Amino를 명시적으로 사용합니다. 이러한 이유로 애플리케이션은 여전히 레거시 아미노 코덱에 대한 참조를 보유하고 있습니다. Amino 코덱은 향후 릴리스에서 SDK에서 제거될 예정입니다.

- **[모듈 관리자](https://docs.cosmos.network/v0.46/building-modules/module-manager.html#manager) 및 [기본 모듈 관리자](https: //docs.cosmos.network/v0.46/building-modules/module-manager.html#basicmanager)에 대한 참조** : 모듈 관리자는 애플리케이션의 모듈 목록을 포함하는 개체입니다. [`Msg` 서비스](https://docs.cosmos.network/v0.46/core/baseapp.html#msg-services) 및 [gRPC `Query` service](https://docs.cosmos.network/v0.46/core/baseapp.html#grpc-query-services) 등록,  [`InitChainer`](https:// docs.cosmos.network/v0.46/basics/app-anatomy.html#initchainer), [`BeginBlocker` 및 `EndBlocker`](https://docs.cosmos.network/v0.46/basics/app-anatomy .html#beginblocker-and-blocker) 같은 다양한 기능들을 위한 모듈 사이의 실행 순서를 세팅하는 등 이러한 모듈과 관련된 작업을 용이하게 합니다. 

  

데모 및 테스트 목적으로 사용되는 Cosmos SDK의 자체 앱인 `simapp`의 애플리케이션 타입 정의 예제를 참조하세요.

```go
// SimApp은 ABCI 애플리케이션을 확장하지만 대부분의 매개변수를 내보냅니다.
// 테스트에 개체 기능이 필요하지 않으므로 helper 기능을 만들 때 편의를 위해 내보냅니다.
type SimApp struct {
	*baseapp.BaseApp
	legacyAmino       *codec.LegacyAmino
	appCodec          codec.Codec
	interfaceRegistry types.InterfaceRegistry

	invCheckPeriod uint

	// keys to access the substores
	keys    map[string]*storetypes.KVStoreKey
	tkeys   map[string]*storetypes.TransientStoreKey
	memKeys map[string]*storetypes.MemoryStoreKey

	// keepers
	AccountKeeper    authkeeper.AccountKeeper
	BankKeeper       bankkeeper.Keeper
	CapabilityKeeper *capabilitykeeper.Keeper
	StakingKeeper    stakingkeeper.Keeper
	SlashingKeeper   slashingkeeper.Keeper
	MintKeeper       mintkeeper.Keeper
	DistrKeeper      distrkeeper.Keeper
	GovKeeper        govkeeper.Keeper
	CrisisKeeper     crisiskeeper.Keeper
	UpgradeKeeper    upgradekeeper.Keeper
	ParamsKeeper     paramskeeper.Keeper
	AuthzKeeper      authzkeeper.Keeper
	EvidenceKeeper   evidencekeeper.Keeper
	FeeGrantKeeper   feegrantkeeper.Keeper
	GroupKeeper      groupkeeper.Keeper
	NFTKeeper        nftkeeper.Keeper

	// the module manager
	mm *module.Manager

	// simulation manager
	sm *module.SimulationManager

	// module configurator
	configurator module.Configurator
}
```



### Constructor Function

이 함수는 위 섹션에 정의된 새 애플리케이션 타입을 구성합니다. 애플리케이션 데몬의 [`start` 명령](https://docs.cosmos.network/v0.46/core/node.html#start-command)에서 사용하려면 `AppCreator` 서명을 충족해야 합니다.

```	go
  // AppCreator는 다양한 구성을 사용하여 응용 프로그램을 느리게 초기화할 수 있는 기능입니다.
	AppCreator func(log.Logger, dbm.DB, io.Writer, AppOptions) Application
```



이 기능이 수행하는 주요 작업은 다음과 같습니다.

- 새로운 [`codec`](https://docs.cosmos.network/v0.46/core/encoding.html)를 인스턴스화하고 [기본 관리자](https://docs.cosmos.network/v0.46/building-modules/module-manager.html#basicmanager)를 사용하여 각 애플리케이션 모듈의 `codec`을 초기화합니다.
- `baseapp` 인스턴스, 코덱 및 모든 적절한 저장소 키에 대한 참조로 새 애플리케이션을 인스턴스화합니다.
- 애플리케이션의 각 모듈에 있는 NewKeeper 기능을 사용해 애플리케이션 타입에 정의된 모든 [키퍼들](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#keeper)을 인스턴스화합니다. 키퍼들은 정확한 순서 대로 인스턴스화되어야 하는데, 한 모듈의 `NewKeeper`는 다른 모듈의 `keeper`에 대한 레퍼런스를 필요로 하기 때문입니다.
- 애플리케이션의 각 모듈의 [`AppModule`](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#application-module-interface)을 사용하여 애플리케이션의 [모듈 관리자](https://docs.cosmos.network/v0.46/building-modules/module-manager.html#manager)를 인스턴스화합니다.
- 모듈 관리자를 사용하여 애플리케이션의 [`Msg` 서비스](https://docs.cosmos.network/v0.46/core/baseapp.html#msg-services), [gRPC `Query` 서비스](https: //docs.cosmos.network/v0.46/core/baseapp.html#grpc-query-services), [기존 `Msg` 경로](https://docs.cosmos.network/v0.46/core/baseapp .html#routing) 및 [기존 쿼리 경로](https://docs.cosmos.network/v0.46/core/baseapp.html#query-routing)를 인스턴스화합니다. 트랜잭션이 ABCI를 통해 Tendermint에서 애플리케이션으로 릴레이되면 여기에 정의된 경로를 사용해 해당 모듈의 [`Msg` 서비스](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#msg-services)로 라우팅됩니다. 마찬가지로 애플리케이션에서 gRPC 쿼리 요청을 받으면 여기에 정의된 gRPC 경로를 사용해 해당 모듈의 [`gRPC 쿼리 서비스`](https://docs.cosmos.network/v0.46/basics/app-anatomy.html# grpc-query-services)로 라우팅됩니다. Cosmos SDK는 각각 레거시 `Msg` 경로와 레거시 쿼리 경로를 사용하여 라우팅되는 레거시 `Msg` 및 레거시 Tendermint 쿼리를 계속 지원합니다.
- 모듈 관리자를 사용하여 [애플리케이션 모듈의 불변(invariant)](https://docs.cosmos.network/v0.46/building-modules/invariants.html)을 등록합니다. 불변(Invariant)는 각 블록의 끝에서 평가되는 변수(예: 토큰의 총 공급량)입니다. 불변을 확인하는 프로세스는 [`InvariantsRegistry`](https://docs.cosmos.network/v0.46/building-modules/invariants.html#invariant-registry)라는 특수 모듈을 통해 수행됩니다. 불변 값은 모듈에 정의된 예측 값과 같아야 합니다. 값이 예측된 값과 다른 경우 불변 레지스트리에 정의된 특수 논리가 트리거됩니다(일반적으로 체인이 중지됨). 이것은 중요한 버그가 눈에 띄지 않게 하고 수정하기 어려운 오래 지속되는 효과를 생성하는지 확인하는 데 유용합니다.
- 모듈 관리자를 사용하여 [애플리케이션의 각 모듈](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#application-module-interface)의 `InitGenesis`, `BeginBlocker`, `EndBlocker` 함수 사이의 실행 순서를 설정합니다. 모든 모듈이 이러한 기능을 구현하는 것은 아닙니다.
- 나머지 애플리케이션 매개변수를 설정합니다.
  - [`InitChainer`](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#initchainer): 애플리케이션이 처음 시작될 때 애플리케이션을 초기화하는 데 사용됨
  - [`BeginBlocker`, `EndBlocker`](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#beginblocker-and-endlbocker): 모든 블록의 시작과 끝에서 호출
  - [`anteHandler`](https://docs.cosmos.network/v0.46/core/baseapp.html#antehandler): 수수료 및 서명 확인을 처리하는 데 사용
- 스토어들을 마운트합니다.
- 애플리케이션을 반환합니다.



이 기능은 앱의 인스턴스만 생성하는 반면, 실제 상태는 노드가 다시 시작되면 `~/.app/data` 폴더에서 전달되거나 노드가 처음으로 시작된 경우에는 제네시스 파일에서 생성됩니다. 

`simapp`의 애플리케이션 생성자의 예를 참조하세요.

```go
// NewSimApp은 초기화된 SimApp에 대한 참조를 반환합니다.
func NewSimApp(
	logger log.Logger, db dbm.DB, traceStore io.Writer, loadLatest bool, skipUpgradeHeights map[int64]bool,
	homePath string, invCheckPeriod uint, encodingConfig simappparams.EncodingConfig,
	appOpts servertypes.AppOptions, baseAppOptions ...func(*baseapp.BaseApp),
) *SimApp {
	appCodec := encodingConfig.Codec
	legacyAmino := encodingConfig.Amino
	interfaceRegistry := encodingConfig.InterfaceRegistry

	bApp := baseapp.NewBaseApp(appName, logger, db, encodingConfig.TxConfig.TxDecoder(), baseAppOptions...)
	bApp.SetCommitMultiStoreTracer(traceStore)
	bApp.SetVersion(version.Version)
	bApp.SetInterfaceRegistry(interfaceRegistry)

	keys := sdk.NewKVStoreKeys(
		authtypes.StoreKey, banktypes.StoreKey, stakingtypes.StoreKey,
		minttypes.StoreKey, distrtypes.StoreKey, slashingtypes.StoreKey,
		govtypes.StoreKey, paramstypes.StoreKey, upgradetypes.StoreKey, feegrant.StoreKey,
		evidencetypes.StoreKey, capabilitytypes.StoreKey,
		authzkeeper.StoreKey, nftkeeper.StoreKey, group.StoreKey,
	)
	tkeys := sdk.NewTransientStoreKeys(paramstypes.TStoreKey)
  // 참고: testingkey는 테스트 목적으로 마운트되었습니다. 실제 애플리케이션에는 이 키가 포함되지 않아야 합니다.
	memKeys := sdk.NewMemoryStoreKeys(capabilitytypes.MemStoreKey, "testingkey")

  // AppOptions를 사용하여 상태 수신 기능 구성
  // 이 경우 반환된 streamingServices 및 waitGroup으로 아무 작업도 하지 않습니다.
	if _, _, err := streaming.LoadStreamingServices(bApp, appOpts, appCodec, keys); err != nil {
		tmos.Exit(err.Error())
	}

	app := &SimApp{
		BaseApp:           bApp,
		legacyAmino:       legacyAmino,
		appCodec:          appCodec,
		interfaceRegistry: interfaceRegistry,
		invCheckPeriod:    invCheckPeriod,
		keys:              keys,
		tkeys:             tkeys,
		memKeys:           memKeys,
	}

	app.ParamsKeeper = initParamsKeeper(appCodec, legacyAmino, keys[paramstypes.StoreKey], tkeys[paramstypes.TStoreKey])

	// set the BaseApp's parameter store
  bApp.SetParamStore(app.ParamsKeeper.Subspace(baseapp.Paramspace).WithKeyTable(paramstypes.ConsensusParamsKeyTable()))

	app.CapabilityKeeper = capabilitykeeper.NewKeeper(appCodec, keys[capabilitytypes.StoreKey], memKeys[capabilitytypes.MemStoreKey])
	// 정적으로 생성된 ScopedKeeper를 적용하려는 애플리케이션은 `ScopeToModule`을 사용하여 `NewApp`에서 범위 지정 모듈을 생성한 후 `Seal`을 호출해야 합니다.
	app.CapabilityKeeper.Seal()

	// add keepers
	app.AccountKeeper = authkeeper.NewAccountKeeper(
		appCodec, keys[authtypes.StoreKey], app.GetSubspace(authtypes.ModuleName), authtypes.ProtoBaseAccount, maccPerms, sdk.Bech32MainPrefix,
	)
	app.BankKeeper = bankkeeper.NewBaseKeeper(
		appCodec, keys[banktypes.StoreKey], app.AccountKeeper, app.GetSubspace(banktypes.ModuleName), app.ModuleAccountAddrs(),
	)
	stakingKeeper := stakingkeeper.NewKeeper(
		appCodec, keys[stakingtypes.StoreKey], app.AccountKeeper, app.BankKeeper, app.GetSubspace(stakingtypes.ModuleName),
	)
	app.MintKeeper = mintkeeper.NewKeeper(
		appCodec, keys[minttypes.StoreKey], app.GetSubspace(minttypes.ModuleName), &stakingKeeper,
		app.AccountKeeper, app.BankKeeper, authtypes.FeeCollectorName,
	)
	app.DistrKeeper = distrkeeper.NewKeeper(
		appCodec, keys[distrtypes.StoreKey], app.GetSubspace(distrtypes.ModuleName), app.AccountKeeper, app.BankKeeper,
		&stakingKeeper, authtypes.FeeCollectorName,
	)
	app.SlashingKeeper = slashingkeeper.NewKeeper(
		appCodec, keys[slashingtypes.StoreKey], &stakingKeeper, app.GetSubspace(slashingtypes.ModuleName),
	)
	app.CrisisKeeper = crisiskeeper.NewKeeper(
		app.GetSubspace(crisistypes.ModuleName), invCheckPeriod, app.BankKeeper, authtypes.FeeCollectorName,
	)

	app.FeeGrantKeeper = feegrantkeeper.NewKeeper(appCodec, keys[feegrant.StoreKey], app.AccountKeeper)

	// register the staking hooks
	// NOTE: stakingKeeper above is passed by reference, so that it will contain these hooks
	app.StakingKeeper = *stakingKeeper.SetHooks(
		stakingtypes.NewMultiStakingHooks(app.DistrKeeper.Hooks(), app.SlashingKeeper.Hooks()),
	)

	app.AuthzKeeper = authzkeeper.NewKeeper(keys[authzkeeper.StoreKey], appCodec, app.MsgServiceRouter(), app.AccountKeeper)

	groupConfig := group.DefaultConfig()
	/*
		Example of setting group params:
		groupConfig.MaxMetadataLen = 1000
	*/
	app.GroupKeeper = groupkeeper.NewKeeper(keys[group.StoreKey], appCodec, app.MsgServiceRouter(), app.AccountKeeper, groupConfig)

	// register the proposal types
	govRouter := govv1beta1.NewRouter()
	govRouter.AddRoute(govtypes.RouterKey, govv1beta1.ProposalHandler).
		AddRoute(paramproposal.RouterKey, params.NewParamChangeProposalHandler(app.ParamsKeeper)).
		AddRoute(distrtypes.RouterKey, distr.NewCommunityPoolSpendProposalHandler(app.DistrKeeper)).
		AddRoute(upgradetypes.RouterKey, upgrade.NewSoftwareUpgradeProposalHandler(app.UpgradeKeeper))
	govConfig := govtypes.DefaultConfig()
	/*
		Example of setting gov params:
		govConfig.MaxMetadataLen = 10000
	*/
	govKeeper := govkeeper.NewKeeper(
		appCodec, keys[govtypes.StoreKey], app.GetSubspace(govtypes.ModuleName), app.AccountKeeper, app.BankKeeper,
		&stakingKeeper, govRouter, app.MsgServiceRouter(), govConfig,
	)

	app.GovKeeper = *govKeeper.SetHooks(
		govtypes.NewMultiGovHooks(
		// register the governance hooks
		),
	)
	// set the governance module account as the authority for conducting upgrades
	app.UpgradeKeeper = upgradekeeper.NewKeeper(skipUpgradeHeights, keys[upgradetypes.StoreKey], appCodec, homePath, app.BaseApp, authtypes.NewModuleAddress(govtypes.ModuleName).String())

	// RegisterUpgradeHandlers is used for registering any on-chain upgrades
	app.RegisterUpgradeHandlers()

	app.NFTKeeper = nftkeeper.NewKeeper(keys[nftkeeper.StoreKey], appCodec, app.AccountKeeper, app.BankKeeper)

	// create evidence keeper with router
	evidenceKeeper := evidencekeeper.NewKeeper(
		appCodec, keys[evidencetypes.StoreKey], &app.StakingKeeper, app.SlashingKeeper,
	)
	// If evidence needs to be handled for the app, set routes in router here and seal
	app.EvidenceKeeper = *evidenceKeeper

	/****  Module Options ****/

	// NOTE: 모듈 생성자 내에서 `appOpts`를 구문 분석하는 것을 고려할 수 있습니다. 
  // 현재로서는 모듈이 기대하는 인수에 대해 더 엄격한 것을 선호합니다.
	skipGenesisInvariants := cast.ToBool(appOpts.Get(crisis.FlagSkipGenesisInvariants))

	// NOTE: 나중에 수정되는 모듈 관리자에서 인스턴스화된 모든 모듈은 여기에서 참조로 전달되어야 합니다.
	app.mm = module.NewManager(
		genutil.NewAppModule(
			app.AccountKeeper, app.StakingKeeper, app.BaseApp.DeliverTx,
			encodingConfig.TxConfig,
		),
		auth.NewAppModule(appCodec, app.AccountKeeper, authsims.RandomGenesisAccounts),
		vesting.NewAppModule(app.AccountKeeper, app.BankKeeper),
		bank.NewAppModule(appCodec, app.BankKeeper, app.AccountKeeper),
		capability.NewAppModule(appCodec, *app.CapabilityKeeper),
		crisis.NewAppModule(&app.CrisisKeeper, skipGenesisInvariants),
		feegrantmodule.NewAppModule(appCodec, app.AccountKeeper, app.BankKeeper, app.FeeGrantKeeper, app.interfaceRegistry),
		gov.NewAppModule(appCodec, app.GovKeeper, app.AccountKeeper, app.BankKeeper),
		mint.NewAppModule(appCodec, app.MintKeeper, app.AccountKeeper, nil),
		slashing.NewAppModule(appCodec, app.SlashingKeeper, app.AccountKeeper, app.BankKeeper, app.StakingKeeper),
		distr.NewAppModule(appCodec, app.DistrKeeper, app.AccountKeeper, app.BankKeeper, app.StakingKeeper),
		staking.NewAppModule(appCodec, app.StakingKeeper, app.AccountKeeper, app.BankKeeper),
		upgrade.NewAppModule(app.UpgradeKeeper),
		evidence.NewAppModule(app.EvidenceKeeper),
		params.NewAppModule(app.ParamsKeeper),
		authzmodule.NewAppModule(appCodec, app.AuthzKeeper, app.AccountKeeper, app.BankKeeper, app.interfaceRegistry),
		groupmodule.NewAppModule(appCodec, app.GroupKeeper, app.AccountKeeper, app.BankKeeper, app.interfaceRegistry),
		nftmodule.NewAppModule(appCodec, app.NFTKeeper, app.AccountKeeper, app.BankKeeper, app.interfaceRegistry),
	)

  // 시작하는 동안 블록 슬래싱은 distr.BeginBlocker 이후에 발생하므로 
  // CanWithdrawInvariant를 불변으로 유지하기 위해 유효성 검사기 수수료 풀에 아무것도 남지 않습니다.
	// NOTE: HistoricalEntries 매개변수 > 0인 경우 스테이킹 모듈이 필요합니다.
	// NOTE: capability 모듈의 beginblocker는 기능(예: IBC)을 사용하는 모든 모듈보다 먼저 와야 합니다.
	app.mm.SetOrderBeginBlockers(
		upgradetypes.ModuleName, capabilitytypes.ModuleName, minttypes.ModuleName, distrtypes.ModuleName, slashingtypes.ModuleName,
		evidencetypes.ModuleName, stakingtypes.ModuleName,
		authtypes.ModuleName, banktypes.ModuleName, govtypes.ModuleName, crisistypes.ModuleName, genutiltypes.ModuleName,
		authz.ModuleName, feegrant.ModuleName, nft.ModuleName, group.ModuleName,
		paramstypes.ModuleName, vestingtypes.ModuleName,
	)
	app.mm.SetOrderEndBlockers(
		crisistypes.ModuleName, govtypes.ModuleName, stakingtypes.ModuleName,
		capabilitytypes.ModuleName, authtypes.ModuleName, banktypes.ModuleName, distrtypes.ModuleName,
		slashingtypes.ModuleName, minttypes.ModuleName,
		genutiltypes.ModuleName, evidencetypes.ModuleName, authz.ModuleName,
		feegrant.ModuleName, nft.ModuleName, group.ModuleName,
		paramstypes.ModuleName, upgradetypes.ModuleName, vestingtypes.ModuleName,
	)

	// NOTE: genutils 모듈은 스테이킹 후에 발생해야 풀이 제네시스 계정의 토큰으로 올바르게 초기화됩니다.
	// NOTE: genutils 모듈은 인증에서 매개변수에 액세스할 수 있도록 인증 후에도 발생해야 합니다.
  // NOTE: Capability 모듈은 가장 먼저 발생해야 합니다. 
  // InitChain에서 capabilities를 생성하거나 요구하려는 다른 모듈이 안전하게 수행할 수 있도록 
  // 모든 capabilities를 초기화하기 위해서 입니다.
	app.mm.SetOrderInitGenesis(
		capabilitytypes.ModuleName, authtypes.ModuleName, banktypes.ModuleName, distrtypes.ModuleName, stakingtypes.ModuleName,
		slashingtypes.ModuleName, govtypes.ModuleName, minttypes.ModuleName, crisistypes.ModuleName,
		genutiltypes.ModuleName, evidencetypes.ModuleName, authz.ModuleName,
		feegrant.ModuleName, nft.ModuleName, group.ModuleName,
		paramstypes.ModuleName, upgradetypes.ModuleName, vestingtypes.ModuleName,
	)

	// 여기에서 사용자 지정 마이그레이션 순서를 설정하려면 주석 처리를 제거하세요.
	// app.mm.SetOrderMigrations(custom order)

	app.mm.RegisterInvariants(&app.CrisisKeeper)
	app.mm.RegisterRoutes(app.Router(), app.QueryRouter(), encodingConfig.Amino)
	app.configurator = module.NewConfigurator(app.appCodec, app.MsgServiceRouter(), app.GRPCQueryRouter())
	app.mm.RegisterServices(app.configurator)

	// add test gRPC service for testing gRPC queries in isolation
	testdata_pulsar.RegisterQueryServer(app.GRPCQueryRouter(), testdata_pulsar.QueryImpl{})

	// 시뮬레이션 관리자를 생성하고 결정적 시뮬레이션을 위한 모듈의 순서를 정의합니다.
	//
  // 참고: 이것은 퍼지 테스트 트랜잭션에 시뮬레이터를 사용하지 않는 앱에서는 불필요합니다.
	app.sm = module.NewSimulationManager(
		auth.NewAppModule(appCodec, app.AccountKeeper, authsims.RandomGenesisAccounts),
		bank.NewAppModule(appCodec, app.BankKeeper, app.AccountKeeper),
		capability.NewAppModule(appCodec, *app.CapabilityKeeper),
		feegrantmodule.NewAppModule(appCodec, app.AccountKeeper, app.BankKeeper, app.FeeGrantKeeper, app.interfaceRegistry),
		gov.NewAppModule(appCodec, app.GovKeeper, app.AccountKeeper, app.BankKeeper),
		mint.NewAppModule(appCodec, app.MintKeeper, app.AccountKeeper, nil),
		staking.NewAppModule(appCodec, app.StakingKeeper, app.AccountKeeper, app.BankKeeper),
		distr.NewAppModule(appCodec, app.DistrKeeper, app.AccountKeeper, app.BankKeeper, app.StakingKeeper),
		slashing.NewAppModule(appCodec, app.SlashingKeeper, app.AccountKeeper, app.BankKeeper, app.StakingKeeper),
		params.NewAppModule(app.ParamsKeeper),
		evidence.NewAppModule(app.EvidenceKeeper),
		authzmodule.NewAppModule(appCodec, app.AuthzKeeper, app.AccountKeeper, app.BankKeeper, app.interfaceRegistry),
		groupmodule.NewAppModule(appCodec, app.GroupKeeper, app.AccountKeeper, app.BankKeeper, app.interfaceRegistry),
		nftmodule.NewAppModule(appCodec, app.NFTKeeper, app.AccountKeeper, app.BankKeeper, app.interfaceRegistry),
	)

	app.sm.RegisterStoreDecoders()

	// initialize stores
	app.MountKVStores(keys)
	app.MountTransientStores(tkeys)
	app.MountMemoryStores(memKeys)

	// initialize BaseApp
	app.SetInitChainer(app.InitChainer)
	app.SetBeginBlocker(app.BeginBlocker)
	app.SetEndBlocker(app.EndBlocker)
	app.setAnteHandler(encodingConfig.TxConfig, cast.ToStringSlice(appOpts.Get(server.FlagIndexEvents)))
  // v0.46에서 SDK는 _postHandlers_를 도입합니다. 
  // PostHandler는 antehandler와 비슷하지만 `runMsgs` 실행 _후에_ 실행됩니다. 
  // 또한 체인으로 정의되며 antehandler와 동일한 서명을 갖습니다.
  //
	// baseapp에서 postHandler는 `runMsgs`와 동일한 저장소 분기에서 실행됩니다. 
  // 즉, `runMsgs`와 `postHandler` 상태가 둘 다 성공하면 커밋되고, 둘 중 하나가 실패하면 둘 다 되돌려집니다.
  // 
  // SDK는 단 하나의 데코레이터인 트랜잭션 팁 데코레이터로 구성된 기본 postHandlers 체인을 노출합니다. 
  // 그러나 일부 체인은 기본적으로 필요하지 않으므로 팁이 필요하지 않은 경우 다음 줄을 주석 처리하세요.
	// To read more about tips:
	// https://docs.cosmos.network/main/core/tips.html
	//
	// anteHandler 또는 postHandler 체인을 변경하는 것은 상태 머신을 중단시킬 수 있기 때문에
  // 조정된 업그레이드가 필요할 수 있습니다.
	app.setPostHandler()

	if loadLatest {
		if err := app.LoadLatestVersion(); err != nil {
			tmos.Exit(err.Error())
		}
	}

	return app
}
```



### InitChainer

`InitChainer`는 제네시스 파일(즉, 제네시스 계정의 토큰 잔액)에서 애플리케이션의 상태를 초기화하는 기능입니다. 애플리케이션이 텐더민트 엔진으로부터 `InitChain` 메시지를 수신할 때 호출되며, 이는 노드가 `appBlockHeight == 0`(즉, 제네시스)에서 시작될 때 발생합니다. 애플리케이션은 [`SetInitChainer`](https://pkg.go.dev/github.com/cosmos/cosmos-sdk/baseapp#BaseApp.SetInitChainer) 메소드를 통해 자체 [constructor](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#constructor-function)  안에 `InitChainer`를 설정합니다.

In general, the `InitChainer` is mostly composed of the [`InitGenesis`](https://docs.cosmos.network/v0.46/building-modules/genesis.html#initgenesis) function of each of the application's modules. This is done by calling the `InitGenesis` function of the module manager, which in turn will call the `InitGenesis` function of each of the modules it contains. Note that the order in which the modules' `InitGenesis` functions must be called has to be set in the module manager using the [module manager's](https://docs.cosmos.network/v0.46/building-modules/module-manager.html) `SetOrderInitGenesis` method. This is done in the [application's constructor](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#application-constructor), and the `SetOrderInitGenesis` has to be called before the `SetInitChainer`.

일반적으로 `InitChainer`는 대부분 애플리케이션의 각 모듈에 [`InitGenesis`](https://docs.cosmos.network/v0.46/building-modules/genesis.html#initgenesis) 기능으로 구성됩니다. 이것은 모듈 관리자의 `InitGenesis` 함수를 호출하여 수행되며, 차례로 포함된 각 모듈에 포함된 `InitGenesis` 함수를 호출합니다. 모듈의 `InitGenesis` 함수가 호출되어야 하는 순서는 [module manager](https://docs.cosmos.network/v0.46/building-modules/module-manager.html) 의 `SetOrderInitGenesis` 메소드를 사용하여 모듈 관리자에서 설정해야 합니다 . 이는 [애플리케이션의 생성자](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#application-constructor)에서 수행되며, `SetOrderInitGenesis`는 `SetInitChainer ` 전에 호출되어야 합니다.

`simapp`에서 `InitChainer`의 예를 참조하십시오.

```go
// 체인 초기화 시 InitChainer 애플리케이션 업데이트
func (app *SimApp) InitChainer(ctx sdk.Context, req abci.RequestInitChain) abci.ResponseInitChain {
	var genesisState GenesisState
	if err := json.Unmarshal(req.AppStateBytes, &genesisState); err != nil {
		panic(err)
	}
	app.UpgradeKeeper.SetModuleVersionMap(ctx, app.mm.GetVersionMap())
	return app.mm.InitGenesis(ctx, app.appCodec, genesisState)
}
```



### BeginBlocker and EndBlocker

Cosmos SDK는 개발자에게 애플리케이션의 일부로 코드의 자동 실행을 구현할 수 있는 가능성을 제공합니다. 이는 `BeginBlocker`와 `EndBlocker`라는 두 가지 기능을 통해 구현됩니다. 애플리케이션이 각 블록의 시작과 끝에서 각각 발생하는 `BeginBlock` 및 `EndBlock` 메시지를 Tendermint 엔진으로부터 수신할 때 호출됩니다. 애플리케이션은 [`SetBeginBlocker`](https://pkg.go.dev/github.com/cosmos/cosmos-sdk/baseapp#BaseApp.SetBeginBlocker)와 [`SetEndBlocker`](https://pkg.go.dev/github.com/cosmos/cosmos-sdk/baseapp#BaseApp.SetEndBlocker)를 통해 자체  [constructor](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#constructor-function) 안에 `BeginBlock` 및 `EndBlock`를 설정해야 합니다.

일반적으로 `BeginBlocker` 및 `EndBlocker` 기능은 대부분 애플리케이션 각 모듈의  [`BeginBlock` 및 `EndBlock`](https://docs.cosmos.network/v0.46/building-modules/beginblock-endblock.html ) 기능들로 구성됩니다. 이는 모듈 관리자의 'BeginBlock' 및 'EndBlock' 함수를 호출하여 수행되며, 이는 차례로 포함된 각 모듈의 'BeginBlock' 및 'EndBlock' 함수를 호출합니다. 모듈의 'BeginBlock' 및 'EndBlock' 함수가 호출되어야 하는 순서는 각각 'SetOrderBeginBlockers' 및 'SetOrderEndBlockers' 메소드를 사용하여 모듈 관리자에서 설정해야 합니다. 이는 [애플리케이션 생성자](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#application-constructor)의 [모듈 관리자](https://docs.cosmos.network/v0.46/building-modules/module-manager.html)를 통해 수행됩니다.  `SetOrderBeginBlockers` 및 `SetOrderEndBlockers` 메소드는 `SetBeginBlocker` 및 `SetEndBlocker` 함수보다 먼저 호출되어야 합니다.

As a sidenote, it is important to remember that application-specific blockchains are deterministic. Developers must be careful not to introduce non-determinism in `BeginBlocker` or `EndBlocker`, and must also be careful not to make them too computationally expensive, as [gas](https://docs.cosmos.network/v0.46/basics/gas-fees.html) does not constrain the cost of `BeginBlocker` and `EndBlocker` execution.

참고로 애플리케이션 특화 블록체인은 결정적이라는 점을 기억하는 것이 중요합니다. 개발자는 `BeginBlocker` 또는 `EndBlocker`에 비결정론을 도입하지 않도록 주의해야 하며, [gas](https://docs.cosmos.network/v0.46/basics/gas-fees.html)가 `BeginBlocker` 및 `EndBlocker`의 실행 비용을 제한하지 않으므로 계산 비용이 너무 많이 들지 않도록 주의해야 합니다.

`simapp`의 `BeginBlocker` 및 `EndBlocker` 함수의 예를 참조하십시오.

```go
// BeginBlocker application updates every begin block
func (app *SimApp) BeginBlocker(ctx sdk.Context, req abci.RequestBeginBlock) abci.ResponseBeginBlock {
	return app.mm.BeginBlock(ctx, req)
}

// EndBlocker application updates every end block
func (app *SimApp) EndBlocker(ctx sdk.Context, req abci.RequestEndBlock) abci.ResponseEndBlock {
	return app.mm.EndBlock(ctx, req)
}
```



### Register Codec

The `EncodingConfig` structure is the last important part of the `app.go` file. The goal of this structure is to define the codecs that will be used throughout the app.

`EncodingConfig` 구조는 `app.go` 파일의 마지막 중요한 부분입니다. 이 구조의 목표는 앱 전체에서 사용할 코덱을 정의하는 것입니다.

```go
// EncodingConfig는 지정된 앱에 사용할 구체적인 인코딩 유형을 지정합니다.
// 이것은 protobuf와 amino 구현 간의 호환성을 위해 제공됩니다.
type EncodingConfig struct {
	InterfaceRegistry types.InterfaceRegistry
	Codec             codec.Codec
	TxConfig          client.TxConfig
	Amino             *codec.LegacyAmino
}
```



다음은 4개의 필드 각각이 의미하는 바에 대한 설명입니다.

- `InterfaceRegistry`: `InterfaceRegistry`는 [`google.protobuf.Any`](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/any.proto)를 사용하여 인코딩 및 디코딩된("압축 해제"라고도 함) 인터페이스를 처리하기 위해 Protobuf 코덱에서 사용됩니다. `Any`는 `type_url`(인터페이스를 구현하는 구체적인 타입의 이름)과 `value`(인코딩된 바이트)를 포함하는 구조체로 생각할 수 있습니다. `InterfaceRegistry`는 `Any`에서 안전하게 압축을 풀 수 있는 인터페이스 및 구현을 등록하기 위한 메커니즘을 제공합니다. 애플리케이션의 각 모듈은 모듈의 고유한 인터페이스와 구현을 등록하는 데 사용할 수 있는 `RegisterInterfaces` 메소드를 구현합니다.

  - [ADR-19](https://docs.cosmos.network/v0.46/architecture/adr-019-protobuf-state-encoding.html#usage-of-any-to-encode-interfaces)에서 Any 에 대해 자세히 알아볼 수 있습니다.

  - 더 자세히 알아보면, Cosmos SDK는 [`gogoprotobuf`](https://github.com/gogo/protobuf)라는 Protobuf 사양의 구현을 사용합니다. 기본적으로 ['Any'의 gogo protobuf 구현](https://pkg.go.dev/github.com/gogo/protobuf/types)은 [글로벌 유형 등록](https:// github.com/gogo/protobuf/blob/master/proto/properties.go#L540)을 사용하여 `Any`에 패킹된 값을 구체적인 Go 타입으로 디코딩합니다. 이것은 종속성 트리의 모든 악성 모듈이 전역 protobuf 레지스트리에 타입을 등록하고 `type_url` 필드에서 해당 유형을 참조하는 트랜잭션에 의해 로드 및 역직렬화될 수 있는 취약점을 만듭니다. 자세한 내용은 [ADR-019](https://docs.cosmos.network/v0.46/architecture/adr-019-protobuf-state-encoding.html)를 참조하세요.

- `Codec`: Cosmos SDK 전체에서 사용되는 기본 코덱입니다. 상태를 인코딩 및 디코딩하는 데 사용되는 `BinaryCodec`과 사용자에게 데이터를 출력하는 데 사용되는 'JSONCodec'(예: [CLI](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#cli))으로 구성됩니다.. 기본적으로 SDK는 `Protobuf`를 `Codec`으로 사용합니다.
- `TxConfig`: `TxConfig`는 클라이언트가 애플리케이션에서 정의한 구체적인 트랜잭션 타입을 생성하는 데 사용할 수 있는 인터페이스를 정의합니다. 현재 SDK는 두 가지 트랜잭션 유형을 처리합니다:  `SIGN_MODE_DIRECT`(Protobuf 바이너리를 무선 인코딩으로 사용) 및 `SIGN_MODE_LEGACY_AMINO_JSON`(Amino에 따라 다름). 트랜잭션에 대한 자세한 내용은 [여기](https://docs.cosmos.network/v0.46/core/transactions.html)를 참조하세요.
- `Amino`: Cosmos SDK의 일부 레거시 부분은 이전 버전과의 호환성을 위해 여전히 Amino를 사용합니다. 각 모듈은 `RegisterLegacyAmino` 메소드를 노출하여 Amino 내에서 모듈의 특정 유형을 등록합니다. 이 `Amino` 코덱은 앱 개발자가 더 이상 사용하지 않아야 하며 향후 릴리스에서 제거될 예정입니다.



Cosmos SDK는 앱 생성자(`NewApp`)에 대한 `EncodingConfig`를 생성하는 데 사용되는 `MakeTestEncodingConfig` 함수를 노출합니다. Protobuf를 기본 `Codec`으로 사용합니다.

참고: 이 기능은 더 이상 사용되지 않으며 앱을 생성하거나 테스트에서만 사용해야 합니다.

`simapp`의 `MakeTestEncodingConfig` 예를 참조하세요.

```go
// MakeTestEncodingConfig는 테스트를 위한 EncodingConfig를 생성합니다. 
// 이 함수는 테스트 또는 새 앱 인스턴스(NewApp*())를 생성할 때만 사용해야 합니다.
// 앱 사용자는 새 코덱을 만들면 안 됩니다. 대신 app.AppCodec를 사용하세요.
// [더 이상 사용되지 않음]
func MakeTestEncodingConfig() simappparams.EncodingConfig {
	encodingConfig := simappparams.MakeTestEncodingConfig()
	std.RegisterLegacyAminoCodec(encodingConfig.Amino)
	std.RegisterInterfaces(encodingConfig.InterfaceRegistry)
	ModuleBasics.RegisterLegacyAminoCodec(encodingConfig.Amino)
	ModuleBasics.RegisterInterfaces(encodingConfig.InterfaceRegistry)
	return encodingConfig
}
```



## Modules

[모듈](https://docs.cosmos.network/v0.46/building-modules/intro.html)은 코스모스 SDK 애플리케이션의 심장이자 영혼입니다. 그것들은 상태 머신 내에 중첩된 상태 머신으로 간주될 수 있습니다. 트랜잭션이 ABCI를 통해 기본 Tendermint 엔진에서 애플리케이션으로 릴레이되면 [`baseapp`](https://docs.cosmos.network/v0.46/core/baseapp.html)은 그것을 적절한 처리할 수 있는 모듈로 라우팅합니다. 이 패러다임을 통해 개발자는 필요한 대부분의 모듈이 이미 존재하기 때문에 복잡한 상태 머신을 쉽게 구축할 수 있습니다. **개발자의 경우 Cosmos SDK 애플리케이션 구축과 관련된 대부분의 작업은 아직 존재하지 않는 애플리케이션에 필요한 사용자 지정 모듈을 구축하고 이미 존재하는 모듈과 통합하여 하나의 일관된 애플리케이션으로 구성됩니다**. 애플리케이션 디렉토리에서 표준 관행은 모듈을 `x/` 폴더에 저장하는 것입니다(이미 빌드된 모듈이 포함된 Cosmos SDK의 `x/` 폴더와 혼동하지 마십시오).



### Application Module Interface

모듈은 Cosmos SDK에 정의된 인터페이스들,  [`AppModuleBasic`](https://docs.cosmos.network/v0.46/building-modules/module-manager.html#appmodulebasic) 및 [`AppModule`](https://docs.cosmos.network/v0.46/building-modules/module-manager.html#appmodule), 을 구현해야 합니다. 전자는 `Codec`과 같은 모듈의 기본적인 비의존적 요소를 구현하는 반면 후자는 모듈 메소드의 대부분을 처리합니다(다른 모듈의 `Keeper`에 대한 참조가 필요한 메소드 포함). `AppModule` 및 `AppModuleBasic` 타입은 관례상 `module.go`라는 파일에 정의되어 있습니다.

`AppModule`은 모듈을 일관된 애플리케이션으로 구성하는 데 도움이 되는 유용한 메소드 모음을 모듈에 표시합니다. 이러한 메소드는 애플리케이션의 모듈 컬렉션을 관리하는 [`module manager`](https://docs.cosmos.network/v0.46/building-modules/module-manager.html#manager)에서 호출됩니다.



### `Msg` Services

각 모듈은 두 개의 [Protobuf 서비스](https://developers.google.com/protocol-buffers/docs/proto#services)를 정의합니다. 하나는 메시지를 처리하는 `Msg` 서비스이고 다른 하나는 쿼리를 처리하는 gRPC `Query` 서비스입니다. 모듈을 상태 머신으로 간주하면 `Msg` 서비스는 상태 전환 RPC 메소드 집합입니다. 각 Protobuf `Msg` 서비스 메소드는 `sdk.Msg` 인터페이스를 구현해야 하는 Protobuf 요청 유형과 1:1 관계입니다. `sdk.Msg`는 [transactions](https://docs.cosmos.network/v0.46/core/transactions.html)에 번들로 제공되며 각 트랜잭션에는 하나 이상의 메시지가 포함됩니다.

When a valid block of transactions is received by the full-node, Tendermint relays each one to the application via [`DeliverTx` (opens new window)](https://docs.tendermint.com/master/spec/abci/apps.html#delivertx). Then, the application handles the transaction:

풀 노드에서 유효한 트랜잭션 블록을 수신하면 Tendermint는 [`DeliverTx`](https://docs.tendermint.com/master/spec/abci/apps.html#delivertx)를 통해 각 트랜잭션을 애플리케이션에 전달합니다. 그런 다음 애플리케이션은 트랜잭션을 처리합니다.

1. 트랜잭션을 수신하면 애플리케이션은 먼저 `[]byte`에서 트랜잭션을 언마샬링합니다.
2. 그런 다음 트랜잭션에 포함된 'Msg'를 추출하기 전에 [수수료 지불 및 서명](https://docs.cosmos.network/v0.46/basics/gas-fees.html#antehandler)과 같은 트랜잭션에 대한 몇 가지 사항을 확인합니다.
3. `sdk.Msg`는 Protobuf [`Any`s](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#register-codec)를 사용하여 인코딩됩니다. 각 `Any`의 `type_url`을 분석하여 baseapp의 `msgServiceRouter`는 `sdk.Msg`를 해당 모듈의 `Msg` 서비스로 라우팅합니다.
4. 메시지가 성공적으로 처리되면 상태가 업데이트됩니다.



자세한 내용은 트랜잭션[lifecycle](https://docs.cosmos.network/v0.46/basics/tx-lifecycle.html)을 참조하세요.

모듈 개발자는 자신의 모듈을 빌드할 때 사용자 지정 `Msg` 서비스를 만듭니다. 일반적인 관행은 `tx.proto` 파일에 `Msg` Protobuf 서비스를 정의하는 것입니다. 예를 들어 `x/bank` 모듈은 토큰을 전송하는 두 가지 방법으로 서비스를 정의합니다.

```go
// Msg는 뱅크 Msg 서비스를 정의합니다.
service Msg {
  // Send는 한 계정에서 다른 계정으로 코인을 보내는 방법을 정의합니다.
  rpc Send(MsgSend) returns (MsgSendResponse);

  // MultiSend는 일부 계정에서 다른 계정으로 코인을 보내는 방법을 정의합니다.
  rpc MultiSend(MsgMultiSend) returns (MsgMultiSendResponse);
}
```



서비스 메소드는 모듈 상태를 업데이트하기 위해 `keeper`를 사용합니다.

또한 각 모듈은 [`AppModule` 인터페이스](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#application-module-interface)의 일부로 `RegisterServices` 메소드를 구현해야 합니다. 이 메소드는 생성된 Protobuf 코드에서 제공하는 `RegisterMsgServer` 함수를 호출해야 합니다.



### gRPC `Query` Services

gRPC `Query` 서비스를 사용하면 [gRPC](https://grpc.io/)를 사용하여 상태를 쿼리할 수 있습니다. 기본적으로 활성화되어 있으며 [`app.toml`](https://docs.cosmos.network/v0.46/run-node/run-node.html#configuring-the-node-using-apptoml) 내의 `grpc.enable` 및 `grpc.address` 필드에서 구성할 수 있습니다.

gRPC `Query` 서비스는 모듈의 Protobuf 정의 파일, 특히 `query.proto` 내부에 정의됩니다. `query.proto` 정의 파일은 단일 `Query` [Protobuf 서비스](https://developers.google.com/protocol-buffers/docs/proto#services)를 노출합니다. 각 gRPC 쿼리 엔드포인트는 `Query` 서비스 내에서 `rpc` 키워드로 시작하는 서비스 메소드에 해당합니다.

Protobuf는 모든 서비스 메소드를 포함하는 각 모듈에 대한 `QueryServer` 인터페이스를 생성합니다. 그런 다음 모듈의 [`keeper`](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#keeper)는 각 서비스 메소드의 구체적인 구현을 제공하여 이 `QueryServer` 인터페이스를 구현해야 합니다. 이 구체적인 구현은 해당 gRPC 쿼리 엔드포인트의 핸들러입니다.

마지막으로 각 모듈은 [`AppModule` 인터페이스](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#application-module-interface)의 일부로 `RegisterServices` 메소드도 구현해야 합니다. 이 메소드는 생성된 Protobuf 코드에서 제공하는 `RegisterQueryServer` 함수를 호출해야 합니다.



### Keeper

[`Keepers`](https://docs.cosmos.network/v0.46/building-modules/keeper.html)는 모듈 스토어의 게이트키퍼입니다. 모듈 저장소에서 읽거나 쓰려면 'Keeper'의 메소드 중 하나를 반드시 거쳐야 합니다. 이는 Cosmos SDK의 [object-capabilities](https://docs.cosmos.network/v0.46/core/ocap.html) 모델에 의해 보장됩니다. 저장소에 대한 키를 보유하고 있는 객체만 액세스할 수 있으며 모듈의 `키퍼`만이 모듈 저장소에 대한 키를 보유해야 합니다.

`Keepers`는 일반적으로 `keeper.go`라는 파일에 정의되어 있습니다. 여기에는 `keeper`의 타입 정의와 메서드가 포함되어 있습니다.

'keeper' 타입 정의는 일반적으로 다음으로 구성됩니다.

- 다중 저장소(multistore)의 모듈 저장소에 대한 **키**.
- **다른 모듈의 `keepers`**에 대한 참조. '키퍼'가 다른 모듈의 저장소에 액세스해야 하는 경우에만 필요합니다(읽거나 쓰기 위해).
- 애플리케이션의 **코덱**에 대한 참조. '키퍼'는 구조체를 저장하기 전에 마샬링하거나 조회할 때 구조체를 언마샬링하는 데 필요합니다. 저장소는 `[]bytes`만 값으로 받아들이기 때문입니다.

타입 정의와 함께 `keeper.go` 파일의 다음으로 중요한 구성 요소는 `keeper`의 생성자 함수인 `NewKeeper`입니다. 이 함수는 위에서 정의한 타입의 새로운 `키퍼`를 `코덱`으로 인스턴스화하고 `키`를 저장하고 잠재적으로 다른 모듈의 `키퍼`를 매개변수로 참조합니다. `NewKeeper` 함수는 [어플리케이션의 생성자](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#constructor-function)에서 호출됩니다. 파일의 나머지 부분은 주로 getter와 setter인 `keeper`의 메소드를 정의합니다.



### Command-Line, gRPC Services and REST Interfaces

각 모듈은 [애플리케이션의 인터페이스](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#application)를 통해 최종 사용자에게 노출될 명령줄 명령, gRPC 서비스 및 REST 경로를 정의합니다. 이를 통해 최종 사용자는 모듈에 정의된 유형의 메시지를 생성하거나 모듈에서 관리하는 상태의 하위 집합을 쿼리할 수 있습니다.



#### CLI

일반적으로 [모듈 관련 명령어](https://docs.cosmos.network/v0.46/building-modules/module-interfaces.html#cli)는 모듈의 폴더 내의  `client/cli`라는 폴더에 정의되어 있습니다. CLI는 명령을 `client/cli/tx.go` 및 `client/cli/query.go`에 각각 정의된 트랜잭션과 쿼리의 두 가지 범주로 나눕니다. 두 명령 모두 [Cobra Library](https://github.com/spf13/cobra)를 기반으로 합니다.

- 트랜잭션 명령을 사용하면 사용자가 새 트랜잭션을 생성하여 블록에 포함하고 결국 상태를 업데이트할 수 있습니다. 모듈에 정의된 각 [메시지 유형](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#message-types)에 대해 하나의 명령을 생성해야 합니다. 이 명령은 최종 사용자가 제공한 매개변수를 사용하여 메시지의 생성자를 호출하고 이를 트랜잭션으로 래핑합니다. Cosmos SDK는 서명(signing) 및 기타 트랜잭션 메타데이터 추가를 처리합니다.
- Queries let users query the subset of the state defined by the module. Query commands forward queries to the [application's query router](https://docs.cosmos.network/v0.46/core/baseapp.html#query-routing), which routes them to the appropriate [querier](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#querier) the `queryRoute` parameter supplied.
- 쿼리를 통해 사용자는 모듈에서 정의한 상태의 하위 집합을 쿼리할 수 있습니다. 쿼리 명령은 쿼리를 [애플리케이션의 쿼리 라우터](https://docs.cosmos.network/v0.46/core/baseapp.html#query-routing)로 전달하고,  적절한 [querier](https:/ /docs.cosmos.network/v0.46/basics/app-anatomy.html#querier) 에게 `queryRoute` 매개변수를 라우팅합니다.



#### gRPC

[gRPC](https://grpc.io/)is a modern open-source high performance RPC framework that has support in multiple languages. It is the recommended way for external clients (such as wallets, browsers and other backend services) to interact with a node.

Each module can expose gRPC endpoints, called [service methods](https://grpc.io/docs/what-is-grpc/core-concepts/#service-definition)and are defined in the [module's Protobuf `query.proto` file](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#grpc-query-services). A service method is defined by its name, input arguments and output response. The module then needs to:

[gRPC](https://grpc.io/)는 여러 언어를 지원하는 최신 오픈 소스 고성능 RPC 프레임워크입니다. 외부 클라이언트(예: 지갑, 브라우저 및 기타 백엔드 서비스)가 노드와 상호 작용하는 데 권장되는 방법입니다.

각 모듈은 [서비스 메소드](https://grpc.io/docs/what-is-grpc/core-concepts/#service-definition)이라고 하는 gRPC 엔드포인트을 노출할 수 있으며, 서비스 메소드는 [모듈의 Protobuf `query.proto ` 파일](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#grpc-query-services)에 정의됩니다. 서비스 메소드는 이름, 입력 인수 및 출력 응답으로 정의됩니다. 그런 다음 모듈은 다음을 수행해야 합니다.

- `AppModuleBasic`에 `RegisterGRPCGatewayRoutes` 메소드를 정의하여 클라이언트 gRPC 요청을 모듈 내부의 올바른 핸들러에 연결합니다.
- 각 서비스 메소드에 대해 해당 핸들러를 정의합니다. 핸들러는 gRPC 요청을 처리하는 데 필요한 핵심 로직을 구현하며 `keeper/grpc_query.go` 파일에 있습니다.



#### gRPC-gateway REST Endpoints

일부 외부 클라이언트는 gRPC 사용을 원하지 않을 수 있습니다. 이 경우 Cosmos SDK는 각 gRPC 서비스를 해당 REST 엔드포인트로 노출하는 gRPC 게이트웨이 서비스를 제공합니다. 자세한 내용은 [grpc-gateway](https://grpc-ecosystem.github.io/grpc-gateway/) 문서를 참조하십시오.

REST 엔드포인트는 Protobuf 주석을 사용하여 gRPC 서비스와 함께 Protobuf 파일에 정의됩니다. REST 쿼리를 노출하려는 모듈은 `rpc` 메소드에 `google.api.http` 주석을 추가해야 합니다. 기본적으로 SDK에 정의된 모든 REST 엔드포인트에는 `/cosmos/` 접두사로 시작하는 URL이 있습니다.

The Cosmos SDK also provides a development endpoint to generate [Swagger (opens new window)](https://swagger.io/)definition files for these REST endpoints. This endpoint can be enabled inside the [`app.toml`](https://docs.cosmos.network/v0.46/run-node/run-node.html#configuring-the-node-using-apptoml) config file, under the `api.swagger` key.

또한 Cosmos SDK는 이러한 REST 엔드포인트에 대한 [Swagger](https://swagger.io/) 정의 파일을 생성하기 위한 개발 엔드포인트를 제공합니다. 이 끝점은 [`app.toml`](https://docs.cosmos.network/v0.46/run-node/run-node.html#configuring-the-node-using-apptoml) 구성 내에서 활성화할 수 있습니다. 파일에서 `api.swagger` 키 아래에 있습니다.



## Application Interface

[인터페이스](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#command-line-grpc-services-and-rest-interfaces)를 통해 최종 사용자가 풀노드 클라이언트와 상호 작용할 수 있습니다. 이는 풀노드에서 데이터를 쿼리하거나 풀노드에 의해 릴레이되어 결국 블록에 포함될 새 트랜잭션을 생성 및 전송하는 것을 의미합니다.

The main interface is the [Command-Line Interface](https://docs.cosmos.network/v0.46/core/cli.html). The CLI of a Cosmos SDK application is built by aggregating [CLI commands](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#cli) defined in each of the modules used by the application. The CLI of an application is the same as the daemon (e.g. `appd`), and defined in a file called `appd/main.go`. The file contains:

기본 인터페이스는 [명령줄 인터페이스](https://docs.cosmos.network/v0.46/core/cli.html)입니다. Cosmos SDK 애플리케이션의 CLI는 애플리케이션에서 사용하는 각 모듈에 정의된 [CLI 명령](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#cli)을 집계하여 구축됩니다. 애플리케이션의 CLI는 데몬(예: `appd`)과 동일하며 `appd/main.go`라는 파일에 정의되어 있습니다. 파일에는 다음이 포함됩니다.

- **`main()` 함수**, `appd` 인터페이스 클라이언트를 빌드하기 위해 실행됩니다. 이 함수는 각 명령을 준비하고 빌드하기 전에 `rootCmd`에 추가합니다. `appd`의 루트에 `status`, `keys` 및 `config`, 쿼리 명령, tx 명령 및 `rest-server`와 같은 일반 명령을 추가합니다.
- **쿼리 명령어**는 `queryCmd` 함수를 호출하여 추가됩니다. 이 함수는 애플리케이션의 각 모듈에 정의된 쿼리 명령(`main()` 함수에서 `sdk.ModuleClients`의 배열로 전달됨)과 블록 또는 밸리데이터 쿼리 같은 다른 하위 수준 쿼리 명령을 포함하는 Cobra 명령을 반환합니다. Query 명령어는 CLI의 `appd query [query]` 명령어를 사용하여 호출합니다.
- **Transaction commands** are added by calling the `txCmd` function. Similar to `queryCmd`, the function returns a Cobra command that contains the tx commands defined in each of the application's modules, as well as lower level tx commands like transaction signing or broadcasting. Tx commands are called by using the command `appd tx [tx]` of the CLI.
- **트랜잭션 명령어**는 `txCmd` 함수를 호출하여 추가됩니다. `queryCmd`와 유사하게 이 함수는 각 애플리케이션 모듈에 정의된 tx 명령과 트랜잭션 서명 또는 브로드캐스팅과 같은 하위 수준 tx 명령이 포함된 Cobra 명령을 반환합니다. Tx 명령은 CLI의 `appd tx [tx]` 명령을 사용하여 호출됩니다.



[Cosmos Hub](https://github.com/cosmos/gaia)에서 애플리케이션의 기본 명령줄 파일의 예를 참조하세요.

```go
func NewRootCmd() (*cobra.Command, params.EncodingConfig) {
	encodingConfig := gaia.MakeEncodingConfig()
	initClientCtx := client.Context{}.
		WithCodec(encodingConfig.Marshaler).
		WithInterfaceRegistry(encodingConfig.InterfaceRegistry).
		WithTxConfig(encodingConfig.TxConfig).
		WithLegacyAmino(encodingConfig.Amino).
		WithInput(os.Stdin).
		WithAccountRetriever(types.AccountRetriever{}).
		WithHomeDir(gaia.DefaultNodeHome).
		WithViper("")

	rootCmd := &cobra.Command{
		Use:   "gaiad",
		Short: "Stargate Cosmos Hub App",
		PersistentPreRunE: func(cmd *cobra.Command, _ []string) error {
			initClientCtx, err := client.ReadPersistentCommandFlags(initClientCtx, cmd.Flags())
			if err != nil {
				return err
			}

			initClientCtx, err = config.ReadFromClientConfig(initClientCtx)
			if err != nil {
				return err
			}

			if err = client.SetCmdClientContextHandler(initClientCtx, cmd); err != nil {
				return err
			}

			customTemplate, customGaiaConfig := initAppConfig()
			return server.InterceptConfigsPreRunHandler(cmd, customTemplate, customGaiaConfig)
		},
	}

	initRootCmd(rootCmd, encodingConfig)

	return rootCmd, encodingConfig
}
```



## Dependencies and Makefile

`go-grpc v1.34.0`에 도입된 패치로 인해 gRPC가 `gogoproto` 라이브러리와 호환되지 않아 일부 [gRPC 쿼리](https://github.com/cosmos/cosmos-sdk/issues/ 8426)가 패닉에 빠집니다. 따라서 Cosmos SDK를 사용하려면 `go.mod`에 `go-grpc <=v1.33.2`가 설치되어 있어야 합니다.

gRPC가 제대로 작동하는지 확인하려면 애플리케이션의 `go.mod`에 다음 행을 추가하는 것이 **매우 권장됩니다**:

```go
replace google.golang.org/grpc => google.golang.org/grpc v1.33.2

```

자세한 내용은 [issue #8392](https://github.com/cosmos/cosmos-sdk/issues/8392)를 참조하세요.



개발자가 종속성 관리자와 프로젝트 구축 방법을 자유롭게 선택할 수 있으므로 이 섹션은 선택 사항입니다. 즉, 현재 가장 많이 사용되는 버전 관리 프레임워크는 [`go.mod` ](https://github.com/golang/go/wiki/Modules)입니다. 이는 애플리케이션 전체에서 사용되는 각 라이브러리를 올바른 버전으로 가져오도록 합니다.

아래는 [Cosmos Hub](https://github.com/cosmos/gaia)에서 `go.mod`의 예시로 제공합니다.

```go
module github.com/cosmos/gaia/v7

go 1.17

require (
	github.com/cosmos/cosmos-sdk v0.45.1
	github.com/cosmos/go-bip39 v1.0.0
	github.com/cosmos/ibc-go/v3 v3.0.0
	github.com/gorilla/mux v1.8.0
	github.com/gravity-devs/liquidity v1.5.0
	github.com/ory/dockertest/v3 v3.8.1
	github.com/rakyll/statik v0.1.7
	github.com/spf13/cast v1.4.1
	github.com/spf13/cobra v1.3.0
	github.com/spf13/viper v1.10.1
	github.com/strangelove-ventures/packet-forward-middleware/v2 v2.1.1
	github.com/stretchr/testify v1.7.0
	github.com/tendermint/tendermint v0.34.14
	github.com/tendermint/tm-db v0.6.4
)
```



어플리케이션을 구축하기 위해서는 일반적으로 [Makefile](https://en.wikipedia.org/wiki/Makefile)을 사용합니다. Makefile은 기본적으로 애플리케이션에 대한 두 진입점([`appd`](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#node-client) and [`appd`](https://docs.cosmos.network/v0.46/basics/app-anatomy.html#application-interface))을 빌드하기 전에 `go.mod`가 실행되도록 합니다. 

[Cosmos Hub Makefile](https://github.com/cosmos/gaia/blob/Theta-main/Makefile)의 예시입니다.