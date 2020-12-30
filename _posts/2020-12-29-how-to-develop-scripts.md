---
layout: post
title: 如何在 Nervos CKB 上开发智能合约
---

## 概述

Nervos CKB 是一条基于 PoW 的 layer1 公链，其 Cell 模型是比特币 UTXO 模型的泛化，因此它的智能合约开发有别于基于以太坊账户模型的智能合约开发。以太坊的合约是链上计算，合约调用者需要给出合约方法的输入，链上会完成计算并得到输出，而 CKB 的合约是链上验证，合约调用者需要同时给出输入和输出，链上完成输入到输出的验证。

举一个简单的类比，如果你想实现在合约中实现 `y = sqrt(x)` 函数，对于以太坊你需要给出 `x` 的值，合约会计算 `y` 的值；而对于 CKB 来说，你需要同时给出 `x` 和 `y` 的值，合约负责验证 `x` 与 `y` 是否满足 `y = sqrt(x)`。

从这个例子中可以看到，以太坊合约开发只需要关注输入和需要调用的合约函数即可，链上会完成计算和状态的更新，而 CKB 则需要在链外提前计算输入和输出，合约只需要按照相同的计算规则来验证输入和输出是否满足要求，换言之，CKB 需要同时实现链外的 Generator 和链上的 Validator，这两者的验证规则是一致的。

对于熟悉以太坊智能合约的开发者来说，CKB 的智能合约相当于是一种全新的开发模型，所有的状态改变都需要链外的 Generator 提前设定好，链上要做的只是验证状态改变是否符合规则。相比于以太坊的只需要在链上实现合约规则，CKB 需要在链外和链上同时实现两套相同的规则，这在一定程度上增加了合约开发的复杂度，不过好处是合约运行的复杂度可以大大降低，因为验证通常要比计算更简单。

还是上文提到的例子，如果你想在合约中实现 `y = sqrt(x)` 函数，以太坊需要在合约中实现根据输入 `x` 做开平方运算得到 `y`，而 CKB 的合约其实只需要判断 `x` 和 `y` 是否满足 `x = y^2`，显然平方的计算复杂度要远小于开平方的复杂度。换言之，CKB 的合约算法可以不需要跟链外 Generator 完全保持一致，只要两者的计算是等价的即可。

## Cell 和 Transaction 的数据结构

因为 CKB 的合约本质上是通过交易来改变 Cell 的状态，因此我们强烈建议先熟悉 Cell 和 Transaction 的数据结构，否则会影响后续合约的理解，详情可以参考[Transaction Structure](https://github.com/nervosnetwork/rfcs/blob/master/rfcs/0022-transaction-structure/0022-transaction-structure.md)。

```
// Cell
{
  capacity: uint64,
  lock: Script,
  type: Script,
}

// Script
{
  code_hash: H256,
  args: Bytes,
  hash_type: String    // type or data
}
```

`inputs`、`outputs` 和 `outputs_data` 代表了 Cell 在一笔交易前后的状态变化，Cell 包含了 `lock script`(必需)和 `type script`(非必需)，CKB VM 会执行 `inputs` 中的所有的 `lock scripts`，以及 `inputs` 和 `outputs` 中的所有 `type scripts`，`lock script`和 `type script` 包含了对 Cell 状态约束的合约规则。

关于 Script 中 code_hash、args 以及 hash_type 可以参考[Code Locating](https://github.com/nervosnetwork/rfcs/blob/master/rfcs/0022-transaction-structure/0022-transaction-structure.md#code-locating)，请务必先阅读，否则会影响后续合约的理解。

## VM Syscall

由于我们需要在合约中判断 Cell 在一笔交易中前后的状态变化是否符合一定的规则，那么首先我们就需要在合约中可以获取到 Cell 和 Transaction 中的数据，CKB VM 提供了 syscall 帮助我们在合约中访问 Cell 和 Transaction 中的数据:

```
- ckb_load_tx_hash
- ckb_load_transaction
- ckb_load_script_hash
- ckb_load_script
- ckb_load_cell
- ckb_load_cell_by_field
- ckb_load_cell_data
- ckb_load_cell_data_as_code
- ckb_load_input
- ckb_load_input_by_field
- ckb_load_header
- ckb_load_header_by_field
- ckb_load_witness
```

可以看到 [VM Syscall](https://github.com/nervosnetwork/rfcs/blob/master/rfcs/0009-vm-syscalls/0009-vm-syscalls.md) 提供了大量获取 Cell 和 Transaction 数据的方法，这些方法可以在 C 语言代码中直接调用，具体的参数和调用细节可以参考[VM Syscall](https://github.com/nervosnetwork/rfcs/blob/master/rfcs/0009-vm-syscalls/0009-vm-syscalls.md)。

```c
#include "ckb_syscalls.h"

// We are limiting the script size loaded to be 32KB at most. This should be
// more than enough. We are also using blake2b with 256-bit hash here, which is
// the same as CKB.
#define BLAKE2B_BLOCK_SIZE 32
#define SCRIPT_SIZE 32768

#define ERROR_SYSCALL -3
#define ERROR_SCRIPT_TOO_LONG -21

int main() {
  // First, let's load current running script, so we can extract owner lock script hash from script args.

  unsigned char script[SCRIPT_SIZE];
  uint64_t len = SCRIPT_SIZE;

  int ret = ckb_load_script(script, &len, 0);
  if (ret != CKB_SUCCESS) {
    return ERROR_SYSCALL;
  }
  if (len > SCRIPT_SIZE) {
    return ERROR_SCRIPT_TOO_LONG;
  }

  return CKB_SUCCESS;
}
```

上面的合约例子展示了如何读取当前 script 数据，以及判断 script 数据是否符合长度要求，CKB 的系统合约都是 C 语言实现的，详情可以参考 [ckb-system-scripts](https://github.com/nervosnetwork/ckb-system-scripts/tree/master/c) 以及 [ckb-miscellaneous-scripts](https://github.com/nervosnetwork/ckb-miscellaneous-scripts/tree/master/c)。

## Capsule

为了降低合约开发、调试、测试和部署的门槛，Nervos CKB 推出了基于 Rust 语言的智能合约开发框架 [Capsule](https://github.com/nervosnetwork/capsule)，旨在提供开箱即用的解决方案，以帮助开发者快速而轻松地完成常见的开发任务。

```
USAGE:
capsule [SUBCOMMAND]

FLAGS:
    -h, --help       Prints help information
    -V, --version    Prints version information

SUBCOMMANDS:
    check           Check environment and dependencies
    new             Create a new project
    new-contract    Create a new contract
    build           Build contracts
    run             Run command in contract build image
    test            Run tests
    deploy          Deploy contracts, edit deployment.toml to custodian deployment recipe.
    debugger        CKB debugger
    help            Prints this message or the help of the given subcommand(s)
```

通过 Capsule 命令行可以完成智能合约的创建、编译、测试、调试和部署，关于 Capsule 的详细使用说明可以参考 [Write a SUDT script by Capsule](https://docs.nervos.org/docs/labs/sudtbycapsule)。

为了能让 Rust 开发者在 Capsule 框架中调用 VM Syscall 方法，Nervos CKB 提供了 [ckb-std](https://github.com/jjyr/ckb-std) 以及相关[使用文档](https://nervosnetwork.github.io/ckb-std/riscv64imac-unknown-none-elf/doc/ckb_std/index.html)，开发者可以在合约中引入 `ckb-std`，从而使用 [high_level](https://nervosnetwork.github.io/ckb-std/riscv64imac-unknown-none-elf/doc/ckb_std/high_level/index.html) 模块下的方法完成对 CKB Cell 和 Transaction 数据的调用。

```rust
// Module ckb-std::high_level
find_cell_by_data_hash
load_cell
load_cell_capacity
load_cell_data
load_cell_data_hash
load_cell_lock
load_cell_lock_hash
load_cell_occupied_capacity
load_cell_type
load_cell_type_hash
load_header
load_header_epoch_length
load_header_epoch_number
load_header_epoch_start_block_number
load_input
load_input_out_point
load_input_since
load_script
load_script_hash
load_transaction
load_tx_hash
load_witness_args
```

以下是一些常见的使用 `high_level` 方法的例子：

```rust
// Call current script and check script args length
let script = load_script()?;
let args: Bytes = script.args().unpack();
if args.len() != 20 {
    return Err(Error::InvalidArgument);
}

// Call the input of index 0
let cell_input = load_cell(0, Source::Input)?

// Call the output of index 0
let cell_output = load_cell(0, Source::Output)?

// Filter inputs whose lock script hash is equal to the given lock hash and calculate the sum of inputs' capacity
let cell_inputs = QueryIter::new(load_cell, Source::Input)
            .position(|cell| &hash::blake2b(cell.lock().as_slice()) == lock_hash)
let inputs_sum_capacity = cell_inputs.into_iter().fold(0, |sum, c| sum + c.capacity().unpack())

// Check if there is an output with lock script hash equal to the given lock hash
let has_output = QueryIter::new(load_cell, Source::Output)
            .any(|cell| &hash::blake2b(cell.lock().as_slice()) == lock_hash)

// Check whether the witness args' lock is none of witness whose index in witnesses is 0
match load_witness_args(0, Source::Input) {
  Ok(witness_args) => {
    if witness_args.lock().to_opt().is_none() {
      Err(Error::WitnessSignatureWrong)
    } else {
      Ok(())
    }
  },
  Err(_) => Err(Error::WitnessSignatureWrong)
}

```

如果需要在合约中验证签名，[ckb-dynamic-loading-secp256k1](https://github.com/jjyr/ckb-dynamic-loading-secp256k1) 给出了如何通过 rust 代码调用系统 Secp256k1 的 C 代码，[ckb-dynamic-loading-rsa](https://github.com/XuJiandong/ckb-dynamic-loading-rsa) 给出了如何通过 rust 代码调用 RSA 签名算法的 C 代码。

更多关于 Capsule 开发智能合约的例子可以参考以下项目：

- https://github.com/jjyr/my-sudt
- https://github.com/duanyytop/ckb-cheque-script
- https://github.com/duanyytop/ckb-passport-lock

### 调试

在合约开发过程中遇到不符合预期的错误是很常见的，比较常见的调试方式是在合约中打印日志，`ckb-std` 提供了 `debug!` ，其用法类似于 rust 语言中的 `print!` ，而在合约的 tests 中，可以直接使用 `print!` 和 `println!` 来打印。

### 测试

对于 CKB 智能合约来说，Capsule 可以帮助开发者实现合约的本地测试，而无需部署到 Nervos CKB 开发链或者测试链，可以极大得降低合约调试难度，提升合约测试效率。关于如何在 Capsule 中实现合约的测试用例，可以参考 [Write a SUDT script by Capsule # Test](https://docs.nervos.org/docs/labs/sudtbycapsule#testing)

### 部署

对于 CKB 智能合约来说，除了常规的二进制代码直接部署，用二进制代码的 hash 作为 `code hash` 的方式，还有 [Type ID](https://xuejie.space/2020_02_03_introduction_to_ckb_script_programming_type_id/) 部署方式，`code hash` 取自 `type script hash`。

TYPE ID 以及 dep_group 等部署方式可以在 `deployment.toml` 文件中配置，最终的部署可以参考 [Write a SUDT script by Capsule # Deployment](https://docs.nervos.org/docs/labs/sudtbycapsule#deployment)。

## 常见错误

合约在开发过程中，难免会遇到各种各样的错误，如何快速定位问题并修复就显得很重要了，如果你的合约中用到了 CKB 的系统合约，例如 `secp256k1_blake160_sighash_all`、`secp256k1_blake160_multisig_all` 或者 Nervos DAO，那么你可以参考系统合约错误码以及相应的错误解释来快速定位问题，错误码详情参考 [Error Codes](https://github.com/nervosnetwork/ckb-system-scripts/wiki/Error-codes)。

比较常见的错误有：

1： 数组越界，检查是否访问了超过数组长度的索引
2： 缺少某项数据，例如某个 Cell 需要有 type script，但是在拼装交易的时候漏掉了
-1： 参数长度错误，有可能是 script args 或者 signature 长度不对
-2： 编码异常，检查 Cell 和 Transaction 的数据是否符合 molecule 要求，比如多了或者少了 0x，hex string 长度为奇数等等
-101 ~ -103：Secp256k1 验签失败，检查合约和 Transaction Witnesses 以及 Script 参数是否正确

当然还有很多系统合约错误，上面只是列举了比较常见的错误类型，详情可以参考 [Error Codes](https://github.com/nervosnetwork/ckb-system-scripts/wiki/Error-codes)。

除了系统合约的错误码，对于特定的业务合约也会有自己的错误码，这个时候就需要去看定义在业务合约中的错误码，定位可能出错的地方，例如 [ckb-cheque-script error code](https://github.com/duanyytop/ckb-cheque-script/blob/main/contracts/ckb-cheque-script/src/error.rs)。

对于合约可能出错的地方都应该抛出相应的错误码，这也不仅有利于合约本身的调试，也可以帮助链外 Generator 在拼装交易的时候更容易定位问题。
