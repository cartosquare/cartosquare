---
title: SOLANA 编程模型
published: true
---

在 Solana 编程模型中，`app` 通过 `transactions` 携带 `instructions` 来与 `Solana` 集群中的 `programs` 交互。 `programs` 事先由开发者上传到集群中。

一个 instruction，有可能是从一个账户转账到另一个账户，或者是创建一个合约来规定如何转账。在 Solana 集群中，instructions 是顺序执行的，且具有原子性，即只要有一个 instruction 执行失败，整个 transaction 就会回滚。

# Transaction 格式

一个 transaction 由`signatures` 数组以及 `message` 组成。

## signatures 数组
  
signatures 数组以 `compact-array` 的格式存储，这个格式先序列化数组长度，然后紧跟着序列化每一个数组元素。数组长度使用一个特殊的，称为 `compact-u16` 的多比特编码方式来编码。

compact-u16 的工作方式：首先把待编码数值的低7位写入第一个 byte 的低7位；如果数值大于0x7f，把第一个 byte 的高位置为1，然后把下7位写入第二个 byte；如果数值大于0x3fff，就把第二个 byte 的高位置为1，然后把剩下的2位写入第三个 byte。

signatures 数组中的每一个数字签名是以 `ed25519` 二进制格式存储，该格式消耗64 bytes。

## message 结构

一个 message 包含一个 `header`，紧跟着一个 compact-array 格式的 account addresses 数组，再接着一个最近的 `blockhash`，最后是一个 compact-array 格式的 instructions 数组。

* Message Header 格式
  
Header包含3个 unsigned 8-bit 数值，分别是：

1. transaction 需要的签名数
1. 需要签名的的只读账户地址数
1. 不需要签名的只读账户地址数

* Account Addresses 数组格式

需要签名的账户地址出现在 Account Addresses 数组的前面，在这些地址中，可读写权限的账号又排在只读账号的前面。
不需要签名的账户地址排在需要签名的账户地址后面，在这些地址中，也是可读写账号排在只读账号的前面。

Account Address 是一个 32-bytes 的任意数据，在数字签名中，这个地址通常是 `ed25519 keypair` 中的 `public key`。

* Blockhash 格式

一个 blockhash 包含一个 32-byte 的 `SHA-256` 哈希，用来表明客户端上次与账本交互的时间。如果 blockhash 太老的话，验证节点会拒绝交易。

blockhash 可以用来防止 replay 攻击。blockhash 实质上给了 transaction 生命周期。

* Instruction 格式

一个 instruction 包含了一个 program id，紧跟着一个 compact-array 格式的 account address 索引数组，然后是一个 compact-array格式的 opaque 8-bit 数据。

program id 用来识别链上的程序， unsigned 8-bit 大小。program 的 `account owner` 指定使用哪个 `loader` 来加载 program，以及 `runtime` 如何执行程序。

对于on-chain BPF programs来说，program的 owner` 是 `BPF Loader`，并且 acoount data 是 `BPF bytecode` 格式，一旦 program 成功部署，就永久地标记为可执行的。

被 instruction 引用的 accounts 代表了链上的状态，是 program 的输入和输出。

每一个 instruction 表示一个单独的程序，每个 transaction 中的部分 accounts 需要传入到这个程序中，同时一个byte array的数据也会被传入到程序中。链上程序会解析数据，对账户做指定的操作。

通常 program 会提供帮助函数来创建它们支持的 instructions。比如 system program 提供下面的 Rust 帮助函数来创建一个`SystemInstruction::CreateAccount` 指令：

```rust
pub fn create_account(
	from_pubkey: &PubKey,
  to_pubkey: &Pubkey,
  lamports: u64,
  space: u64,
  owner: &Pubkey,
) -> Instruction {
  let account_metas = vec![
    AccountMeta::new(*from_pubkey, true),
    AccountMeta::new(*to_pubkey, true),
  ];

  Instruction::new_with_bincode(
    system_program::id(),
    &systemInstruction::CreateAccount {
      lamports,
      space,
      owner: *owner,
    },
    account_metas,
  );
}
```
# Accounts

## Account 面面观

*  State

账户用来存储交易之间的状态，账户存在的前提是里面需要有一定的 `token` ，在Solana中叫做 `lamports` 。账户被存在验证节点的内存中，需要定期交租（ `rent` ），否则会被删除（也可以一次性交租换取永久豁免交租）。

账户由一个 256-bit 的 public key 表示。

* Signers

账户可以作为一个 signer 给一个 transaction 签名，以授权 transaction 对账户的改动。

* Read-only

账户可以被指定为可读的，这样不同的 transactions 可以同时执行。

* Executable

账户可以是可执行的，这表明这个账户代表的是一个 program，账户的 public key 就是 program id。

## 创建账户的流程

1. 客户端生成一个 `keypair`，包含 public key 和 private key
2. 把账户地址（public key）以及账户需占用的存储空间（目前最大是10M）传入 `SystemProgram::CreateAccount` 指令

账户地址可以是任意的256位数据，高级用户可以创建派生的账户（通过 `SystemProgram::CreateAccountWithSeed`, `Pubkey::CreateProgramAddress`）。

## 账户的所属权

账户有一个owner的属性，刚创建的账户的 owner 是 `system account aptly`。如果一个 program 是账户的 owner 的话，就可以对账户进行转账操作，否则只能读取账户的信息。

## Rent

在 Solana 集群的每个 `epoch`，账户都会根据存储空间大小被收取租金。

### 租金计算

Solana 集群的每个 epoch 是2天的时间，目前的租金费率计算如下：

```
3.56 SQL/年/Mb = 0.01 SQL /天/Mb = 19.055441478439427 lamport/epoch/byte
```

一个 account 的大小最少是128 bytes，因此，初始资金为10000 lamports 的最小空间账户在创建完后还剩：

```
Rent: 2439 = 19.055441478439427 (rent rate) * 128 bytes (minimum account size) * 1 (epoch)
Account Balance: 7,561 = 10,000 (transfered lamports) - 2,439 (this account's rent fee for an epoch)
```

注意到，账户刚创建就被收租了，收的租金是下一个 epoch 的。

### 租金豁免

一次性付足2年的租金可以永久豁免租金。可以使用 `getMinimumBalanceForRentExemption` 接口来获取最小的永久豁免租金是多少。

比如，对于一个包含 150000 bytes 数据的账户，一次性豁免租金为：

```
105,290,880 = 19.055441478439427 (fee rate) * (128 + 15_000)(account size including metadata) * ((365.25/2) * 2)(epochs in 2 years)
```

# Runtime

## program 的能力

program 可以根据自己的规则修改在它之下的 accounts。如果一个 account 的 owner 不是这个 program，但是提供了对这个 transaction 的私钥签名，那这个 program 就得到授权。

## program 修改 account 的规则

- 只有 account 的 owner 可以改变 owner
  - 并且仅当 account 是可写入的
  - 并且仅当 account 不是可执行的
  - 并且仅当账户的数据是刚初始化的，或是空的
- program 不能让没有传给它的账户资金减少
- 只读和可执行账户不能被修改
- 只有 system program 可以改变 account data 的大小，并且仅当 system program 拥有这个 account
- 只有账号的 owner 可以改变 account data
  - 并且仅当 account 是可写入的
  - 并且仅当 account 不是可执行的
- 可执行的设定是不可逆的，并且只有是 account owner 可以设定
- 没有人可以修改 account 的租金费率

## Compute Budget

为了防止 program 滥用计算资源，每个程序被赋予计算预算，超出预算程序会被暂停并报错。

下面的操作会消耗计算成本：

- 执行 BPF 指令
- 调用系统函数
  - 日志
  - 创建 program 地址
  - 跨 program 调用
  - ...