# Client



# Auth



## CLI

사용자는 CLI를 사용하여 `auth` 모듈을 쿼리하고 상호 작용할 수 있습니다.



### Query

`query` 명령을 사용하면 사용자가 `auth` 상태를 쿼리할 수 있습니다.

```bash
simd query auth --help	
```



#### account

`account` 명령을 사용하면 사용자가 주소로 계정을 쿼리할 수 있습니다.

```bash
simd query auth account [address] [flags]
```



Example:

```bash
simd query auth account cosmos1...
```



Example Output:

```yaml
'@type': /cosmos.auth.v1beta1.BaseAccount
account_number: "0"
address: cosmos1zwg6tpl8aw4rawv8sgag9086lpw5hv33u5ctr2
pub_key:
  '@type': /cosmos.crypto.secp256k1.PubKey
  key: ApDrE38zZdd7wLmFS9YmqO684y5DG6fjZ4rVeihF/AQD
sequence: "1"

```



#### accounts

`accounts` 명령을 사용하면 사용 가능한 모든 계정을 쿼리할 수 있습니다.

```bash
simd query auth accounts [flags]	
```



Example:

```bash
simd query auth accounts	
```



Example Output:

```yaml
accounts:
- '@type': /cosmos.auth.v1beta1.BaseAccount
  account_number: "0"
  address: cosmos1zwg6tpl8aw4rawv8sgag9086lpw5hv33u5ctr2
  pub_key:
    '@type': /cosmos.crypto.secp256k1.PubKey
    key: ApDrE38zZdd7wLmFS9YmqO684y5DG6fjZ4rVeihF/AQD
  sequence: "1"
- '@type': /cosmos.auth.v1beta1.ModuleAccount
  base_account:
    account_number: "8"
    address: cosmos1yl6hdjhmkf37639730gffanpzndzdpmhwlkfhr
    pub_key: null
    sequence: "0"
  name: transfer
  permissions:
  - minter
  - burner
- '@type': /cosmos.auth.v1beta1.ModuleAccount
  base_account:
    account_number: "4"
    address: cosmos1fl48vsnmsdzcv85q5d2q4z5ajdha8yu34mf0eh
    pub_key: null
    sequence: "0"
  name: bonded_tokens_pool
  permissions:
  - burner
  - staking
- '@type': /cosmos.auth.v1beta1.ModuleAccount
  base_account:
    account_number: "5"
    address: cosmos1tygms3xhhs3yv487phx3dw4a95jn7t7lpm470r
    pub_key: null
    sequence: "0"
  name: not_bonded_tokens_pool
  permissions:
  - burner
  - staking
- '@type': /cosmos.auth.v1beta1.ModuleAccount
  base_account:
    account_number: "6"
    address: cosmos10d07y265gmmuvt4z0w9aw880jnsr700j6zn9kn
    pub_key: null
    sequence: "0"
  name: gov
  permissions:
  - burner
- '@type': /cosmos.auth.v1beta1.ModuleAccount
  base_account:
    account_number: "3"
    address: cosmos1jv65s3grqf6v6jl3dp4t6c9t9rk99cd88lyufl
    pub_key: null
    sequence: "0"
  name: distribution
  permissions: []
- '@type': /cosmos.auth.v1beta1.BaseAccount
  account_number: "1"
  address: cosmos147k3r7v2tvwqhcmaxcfql7j8rmkrlsemxshd3j
  pub_key: null
  sequence: "0"
- '@type': /cosmos.auth.v1beta1.ModuleAccount
  base_account:
    account_number: "7"
    address: cosmos1m3h30wlvsf8llruxtpukdvsy0km2kum8g38c8q
    pub_key: null
    sequence: "0"
  name: mint
  permissions:
  - minter
- '@type': /cosmos.auth.v1beta1.ModuleAccount
  base_account:
    account_number: "2"
    address: cosmos17xpfvakm2amg962yls6f84z3kell8c5lserqta
    pub_key: null
    sequence: "0"
  name: fee_collector
  permissions: []
pagination:
  next_key: null
  total: "0"

```



#### params

`params` 명령을 사용하면 사용자가 현재 인증 매개변수를 쿼리할 수 있습니다.

```bash
simd query auth params [flags]	
```



Example:

```bash
simd query auth params
```



Example Output:

```bash
max_memo_characters: "256"
sig_verify_cost_ed25519: "590"
sig_verify_cost_secp256k1: "1000"
tx_sig_limit: "7"
tx_size_cost_per_byte: "10"
	
```



## gRPC

사용자는 gRPC 엔드포인트를 사용하여 `auth` 모듈을 쿼리할 수 있습니다.



### Account

`account` 엔드포인트를 사용하면 사용자가 주소로 계정을 쿼리할 수 있습니다.

```bash
cosmos.auth.v1beta1.Query/Account	
```



Example:

```bash
grpcurl -plaintext \
    -d '{"address":"cosmos1.."}' \
    localhost:9090 \
    cosmos.auth.v1beta1.Query/Account

```



Example Output:

```json
{
  "account":{
    "@type":"/cosmos.auth.v1beta1.BaseAccount",
    "address":"cosmos1zwg6tpl8aw4rawv8sgag9086lpw5hv33u5ctr2",
    "pubKey":{
      "@type":"/cosmos.crypto.secp256k1.PubKey",
      "key":"ApDrE38zZdd7wLmFS9YmqO684y5DG6fjZ4rVeihF/AQD"
    },
    "sequence":"1"
  }
}

```





### Accounts

`accounts` 엔드포인트를 통해 사용자는 사용 가능한 모든 계정을 쿼리할 수 있습니다.

```
cosmos.auth.v1beta1.Query/Accounts
```



Example:

```bash
grpcurl -plaintext \
    localhost:9090 \
    cosmos.auth.v1beta1.Query/Accounts	
```



Example Output:

```json
{
   "accounts":[
      {
         "@type":"/cosmos.auth.v1beta1.BaseAccount",
         "address":"cosmos1zwg6tpl8aw4rawv8sgag9086lpw5hv33u5ctr2",
         "pubKey":{
            "@type":"/cosmos.crypto.secp256k1.PubKey",
            "key":"ApDrE38zZdd7wLmFS9YmqO684y5DG6fjZ4rVeihF/AQD"
         },
         "sequence":"1"
      },
      {
         "@type":"/cosmos.auth.v1beta1.ModuleAccount",
         "baseAccount":{
            "address":"cosmos1yl6hdjhmkf37639730gffanpzndzdpmhwlkfhr",
            "accountNumber":"8"
         },
         "name":"transfer",
         "permissions":[
            "minter",
            "burner"
         ]
      },
      {
         "@type":"/cosmos.auth.v1beta1.ModuleAccount",
         "baseAccount":{
            "address":"cosmos1fl48vsnmsdzcv85q5d2q4z5ajdha8yu34mf0eh",
            "accountNumber":"4"
         },
         "name":"bonded_tokens_pool",
         "permissions":[
            "burner",
            "staking"
         ]
      },
      {
         "@type":"/cosmos.auth.v1beta1.ModuleAccount",
         "baseAccount":{
            "address":"cosmos1tygms3xhhs3yv487phx3dw4a95jn7t7lpm470r",
            "accountNumber":"5"
         },
         "name":"not_bonded_tokens_pool",
         "permissions":[
            "burner",
            "staking"
         ]
      },
      {
         "@type":"/cosmos.auth.v1beta1.ModuleAccount",
         "baseAccount":{
            "address":"cosmos10d07y265gmmuvt4z0w9aw880jnsr700j6zn9kn",
            "accountNumber":"6"
         },
         "name":"gov",
         "permissions":[
            "burner"
         ]
      },
      {
         "@type":"/cosmos.auth.v1beta1.ModuleAccount",
         "baseAccount":{
            "address":"cosmos1jv65s3grqf6v6jl3dp4t6c9t9rk99cd88lyufl",
            "accountNumber":"3"
         },
         "name":"distribution"
      },
      {
         "@type":"/cosmos.auth.v1beta1.BaseAccount",
         "accountNumber":"1",
         "address":"cosmos147k3r7v2tvwqhcmaxcfql7j8rmkrlsemxshd3j"
      },
      {
         "@type":"/cosmos.auth.v1beta1.ModuleAccount",
         "baseAccount":{
            "address":"cosmos1m3h30wlvsf8llruxtpukdvsy0km2kum8g38c8q",
            "accountNumber":"7"
         },
         "name":"mint",
         "permissions":[
            "minter"
         ]
      },
      {
         "@type":"/cosmos.auth.v1beta1.ModuleAccount",
         "baseAccount":{
            "address":"cosmos17xpfvakm2amg962yls6f84z3kell8c5lserqta",
            "accountNumber":"2"
         },
         "name":"fee_collector"
      }
   ],
   "pagination":{
      "total":"9"
   }
}

```





### Params

`params` 엔드포인트를 사용하면 사용자가 현재 인증 매개변수를 쿼리할 수 있습니다.

```
cosmos.auth.v1beta1.Query/Params
```



Example:

```bash
grpcurl -plaintext \
    localhost:9090 \
    cosmos.auth.v1beta1.Query/Params

```



Example Output:

```json
{
  "params": {
    "maxMemoCharacters": "256",
    "txSigLimit": "7",
    "txSizeCostPerByte": "10",
    "sigVerifyCostEd25519": "590",
    "sigVerifyCostSecp256k1": "1000"
  }
}

```



## REST

사용자는 REST 엔드포인트를 사용하여 `auth` 모듈을 쿼리할 수 있습니다.



### Account

`account` 엔드포인트를 사용하면 사용자가 주소로 계정을 쿼리할 수 있습니다.

```
/cosmos/auth/v1beta1/account?address={address}
```





### Accounts

'accounts' 엔드포인트를 통해 사용자는 사용 가능한 모든 계정을 쿼리할 수 있습니다.

```
/cosmos/auth/v1beta1/accounts
```





### Params

`params` 엔드포인트를 사용하면 사용자가 현재 인증 매개변수를 쿼리할 수 있습니다.

```
/cosmos/auth/v1beta1/params
```





# Vesting

## CLI

사용자는 CLI를 사용하여 `vesting` 모듈을 쿼리하고 상호 작용할 수 있습니다.



### Transactions

`tx` 명령을 사용하면 사용자가 `vesting` 모듈과 상호 작용할 수 있습니다.

```
simd tx vesting --help
```



#### create-periodic-vesting-account

`create-periodic-vesting-account` 명령은 일련의 코인과 기간 길이(초)가 설정된 토큰 할당으로 자금을 조달하는 새로운 베스팅 계정을 생성합니다.  기간은 이전 기간이 끝날 때만 기간이 시작된다는 점에서 순차적입니다. 첫 번째 기간의 기간은 계정 생성 시 시작됩니다.

```bash
simd tx vesting create-periodic-vesting-account [to_address] [periods_json_file] [flags]
```



Example:

```bash
simd tx vesting create-periodic-vesting-account cosmos1.. periods.json
```



#### create-vesting-account

`create-vesting-account` 명령은 토큰 할당으로 자금을 조달하는 새로운 베스팅 계정을 생성합니다. 계정은 '--delayed' 플래그에 의해 결정되는 지연 또는 지속적인 베스팅 계정일 수 있습니다. 생성된 모든 베스팅 계정의 시작 시간은 커밋된 블록의 시간으로 설정됩니다. end_time은 UNIX epoch 타임스탬프로 제공되어야 합니다.

```
simd tx vesting create-vesting-account [to_address] [amount] [end_time] [flags]
```



Example:

```
simd tx vesting create-vesting-account cosmos1.. 100stake 2592000
```

