# Keepers

auth 모듈은 계정을 읽고 쓰는 데 사용할 수 있는 하나의 키퍼인 계정 키퍼만 노출합니다.



## Account Keeper

현재 모든 계정의 모든 필드를 읽고 쓸 수 있고 저장된 모든 계정에 대해 반복할 수 있는 완전히 허가된 계정 키퍼(fully-permissioned account keeper)가 하나만 노출되어 있습니다.



```
// AccountKeeperI는 x/auth의 키퍼가 구현하는 인터페이스 계약입니다.
type AccountKeeperI interface {
	// 다음 계정 번호와 지정된 주소로 새 계정을 반환합니다. 새 계정을 스토어에 저장하지 않습니다.
	NewAccountWithAddress(sdk.Context, sdk.AccAddress) types.AccountI

	// 다음 계좌 번호로 새 계좌를 반환합니다. 새 계정을 스토어에 저장하지 않습니다.
	NewAccount(sdk.Context, types.AccountI) types.AccountI

	// 스토어에 계정이 있는지 확인합니다.
	HasAccount(sdk.Context, sdk.AccAddress) bool

	// 저장소에서 계정을 검색합니다.
	GetAccount(sdk.Context, sdk.AccAddress) types.AccountI

	// 저장소에 계정을 설정합니다.
	SetAccount(sdk.Context, types.AccountI)

	// 저장소에서 계정을 제거합니다.
	RemoveAccount(sdk.Context, types.AccountI)

	// 제공된 함수를 호출하여 모든 계정을 반복합니다. true를 반환하면 반복을 중지합니다.
	IterateAccounts(sdk.Context, func(types.AccountI) bool)

	// 지정된 주소에서 계정의 공개 키 가져오기
	GetPubKey(sdk.Context, sdk.AccAddress) (crypto.PubKey, error)

	// 지정된 주소에서 계정의 시퀀스를 가져옵니다.
	GetSequence(sdk.Context, sdk.AccAddress) (uint64, error)

	// 다음 계정 번호를 가져오고 내부 카운터를 증가시킵니다.
	GetNextAccountNumber(sdk.Context) uint64
}

```

