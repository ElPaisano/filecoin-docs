---
title: "Differences with Ethereum"
description: ""
lead: ""
draft: false
images: []
type: docs
menu:
  smart-contracts:
    parent: "smart-contracts-filecoin-evm-runtime"
    identifier: "differences-with-ethereum-e2c404b7736d394fbdb9f96395bedbb3"
weight: 220
toc: true
---

While Filecoin EVM runtime aims to be compatible with the Ethereum ecosystem, it has some marked differences.

## Gas costs

Filecoin charges Filecoin gas only. This includes the Filecoin EVM runtime. Instead of the Filecoin EVM runtime charging gas according to the EVM spec for each EVM opcode executed, the Filecoin virtual machine (FVM) charges Filecoin gas for executing the EVM interpreter itself. The [How gas works]({{< relref "how-gas-works" >}} page goes into this in more detail). Importantly, this means that Filecoin EVM runtime gas costs and EVM gas costs will be very different:

1. EVM and Filecoin gas are different units of measurement and are not 1:1. Purely based on chain throughput (gas/second), the ratio of Ethereum gas to Filecoin gas is about 1:444. Expect Filecoin gas numbers to look _much_ larger than those in Ethereum.
1. Because Filecoin charges Filecoin gas for executing the Filecoin EVM runtime interpreter:
    1. Some instructions may be more expensive and/or cheaper in Filecoin EVM runtime than they are in the EVM.
    1. EVM instruction costs can depend on the exact Filecoin EVM runtime code-paths taken, and caching.

{{< alert >}}
⚠️ Filecoin gas costs are not set in stone and should never be hard-coded. Future network upgrades will break any smart contracts that depend on gas costs not changing.
{{< /alert >}}

## Gas stipend

Solidity calls to `address.transfer` and `address.send` grant a fixed gas stipend of 2300 Ethereum gas to the called contract. The Filecoin EVM runtime automatically detects such calls, and sets the gas limit to 10 million Filecoin gas. This is a relatively more generous limit than Ethereum's, but it's future-proof. You should expect the callee to be able to carry out more work than in Ethereum.

## Self destruct

Filecoin EVM runtime emulates EVM self-destruct behavior but isn't able to entirely duplicate it:

1. There is no gas refund for self-destruct.
2. On self-destruct, the contract is marked as self-destructed, but is not actually deleted from the Filecoin state-tree. Instead, it simply behaves as if it does not exist. It acts like an empty contract.
3. Unlike in the EVM, in Filecoin EVM runtime, self-destruct can _fail_ causing the executing contract to revert. Specifically, this can happen if the specified beneficiary address is an embedded [ID address]({{< relref "/smart-contracts/filecoin-evm-runtime/address-types" >}}) and no actor exists with the specified ID.
4. If funds are sent to a self-destructed contract after it self-destructs but before the end of the transaction, those funds remain with the self-destructed contract. In Ethereum, these funds would vanish after the transaction finishes executing.

## CALLCODE

The `CALLCODE` opcode has not been implemented. Use the newer `DELEGATECALL` opcode.

## Bare-value sends

In Ethereum, `SELFDESTRUCT` is the only way to send funds to a smart contract without giving the target smart contract a chance to execute code.

In Filecoin, any actor can use `method 0`, also called a bare-value send, to transfer funds to any other actor without invoking the target actor's code. You can think of this behaviour as having the suggested [`PAY` opcode](https://eips.ethereum.org/EIPS/eip-5920) already implemented in Filecoin.

## Precompiles

The Filecoin EVM runtime, unlike Ethereum, does not usually enforce gas limits when calling precompiles. This means that it isn't possible to prevent a precompile from consuming all remaining gas. The `call actor` and `call actor id` precompiles are the exception. However, they apply the passed gas limit to the actor call, not the entire precompile operation (i.e., the full precompile execution end-to-end can use more gas than specified, it's only the final `send` to the target actor that will be limited).
