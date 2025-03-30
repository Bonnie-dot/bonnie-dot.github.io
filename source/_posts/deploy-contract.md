---
title: 从零开始：发布你的智能合约
tags: [区块链]
categories: [区块链]
poster:
  topic: 标题上方的小字
  headline: 大标题
  caption: 标题下方的小字
  color: 标题颜色
abbrlink: b24a58d7
date: 2024-07-06 19:15:31
description:
cover: 
banner: 
---
当发布智能合约时，你不仅仅是在区块链上部署一段代码，而是在构建一个新的数字化世界的一部分。智能合约为去中心化应用（DApps）提供了基础，让人们可以进行安全、透明且无需信任的交易。但要让你的智能合约真正发挥作用，就需要将其正确地部署和发布到区块链上。

在本文中，我们将带你了解发布智能合约的全过程。从准备素材，到选择合适的区块链平台，再到实际的部署和验证过程，我们将一步步地为你详细解说。无论你是初次接触智能合约还是想要更深入地了解它们，我们都将提供实用的指导，让你能够顺利地发布你的智能合约，为区块链世界的发展做出贡献。让我们一起探索这个充满活力和机遇的领域吧！

## Foundry
1. 安装foundry
```
curl -L <https://foundry.paradigm.xyz> | bash
```
2. 使用foundry 创建一个新的项目
```
forge init hello_foundry
```
3.安装openzeppelin
```
npm install @openzeppelin/contracts
```
## 准备智能合约代码及部署脚本
1. 创建一个.env环境文件
```
vim .env
```
2. 加入环境变量
```
SEPOLIA_RPC_URL=PRC_URL
PRIVATE_KEY=0x000000
ETHERSCAN_API_KEY=000000
ADDRESS=0x
```
3.source .env,这样foundry 就可以自动加载环境变量了。每次更新.env,需要重新加载环境变量
4. 更新foundry.toml
```
[rpc_endpoints] sepolia = "${SEPOLIA_RPC_URL}" 
[etherscan] sepolia = { key = "${ETHERSCAN_API_KEY}" }
```
5. 写入合约Box
```
pragma solidity ^0.8.13;
import {Ownable} from "lib/openzeppelin-contracts/contracts/access/Ownable.sol";
import {console} from "lib/forge-std/src/Script.sol";

contract Box is Ownable {
    uint256 private value;

    constructor(address initialOwner) Ownable(initialOwner) {
        console.log("Box contract deployed by: ", initialOwner);
    }

    event ValueChanged(uint256 newValue);

    function store(uint256 newValue) public onlyOwner {
        value = newValue;
        emit ValueChanged(newValue);
    }

    function retrieve() public view returns (uint256) {
        return value;
    }
}

```
- 版本声明： `pragma solidity ^0.8.13;` 表明了这个合约的 Solidity 编译器版本。
- 导入语句： `import {Ownable} from "lib/openzeppelin-contracts/contracts/access/Ownable.sol"`; 导入了来自 `OpenZeppelin` 框架的 `Ownable` 合约，用于管理合约的所有权。
- 合约定义： `contract Box is Ownable { ... }` 定义了一个名为 `Box` 的合约，它继承了 `Ownable` 合约，意味着它具有所有权管理功能。
- 私有状态变量： `uint256 private value;` 声明了一个名为 `value` 的私有整数类型状态变量，用于存储合约中的数据。
- 构造函数： `constructor(address initialOwner) Ownable(initialOwner) { ... }` 是合约的构造函数，它在合约部署时执行初始化操作，其中 `initialOwner` 参数表示初始所有者地址。在构造函数中，通过 `console.log` 语句输出了合约的部署者地址。
- 事件定义： `event ValueChanged(uint256 newValue);` 定义了一个名为 `ValueChanged` 的事件，用于记录合约状态变量的变化。
- 存储函数： `function store(uint256 newValue) public onlyOwner { ... }` 定义了一个名为 `store` 的公共函数，用于存储新的值到合约中，并且只允许合约的所有者调用。
- 检索函数： `function retrieve() public view returns (uint256) { ... }` 定义了一个名为 `retrieve` 的公共视图函数，用于检索合约中存储的值，并返回给调用者。

6. 部署脚本
```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Script, console} from "lib/forge-std/src/Script.sol";
import {Box} from "../src/Box.sol";
import {Counter} from "../src/Counter.sol";

contract MyScript is Script {
    function setUp() public {}

    function run() external {
        console.log("Deploying Box contract...");
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        vm.startBroadcast(deployerPrivateKey);
        Box box = new Box(vm.envAddress("ADDRESS"));
        vm.stopBroadcast();
        console.log("Box contract deployed at address: ", address(box));
    }
}

```
- 导入语句： `import {Script, console} from "lib/forge-std/src/Script.sol";` 导入了来自 `Script.sol` 文件中的 `Script` 和 `console` 对象，以便在合约中使用。
- 合约定义： `contract MyScript is Script { ... }` 定义了一个名为 `MyScript` 的合约，它继承了 `Script` 合约。这个合约将用于部署其他合约。
- setUp 函数： `function setUp() public {} `定义了一个名为 `setUp` 的公共函数，但函数体为空，没有任何操作。
- run 函数： `function run() external { ... } `定义了一个名为 `run` 的外部函数，它将在合约被调用时执行。在函数体中，它首先- 使用 `console.log` 语句输出一条消息，然后从环境变量中获取私钥和地址信息，并使用这些信息部署了一个名为 `Box` 的合约实例。最后，它再次使用 `console.log` 语句输出了合约的部署地址。

## 部署合约
### 部署在Local Chain
1. set up local chain:
```
anvil --fork-url RPC
```
2. 部署
```
forge script script/Script.s.sol:MyScript --fork-url http://localhost:8545 --broadcast
```
### 部署Sepolia
```
forge script --chain sepolia script/Script.s.sol:MyScript --rpc-url $SEPOLIA_RPC_URL --broadcast --verify -vvvv

```
![contract detail](snapshot.png)

## 调用合约
本文使用viem进行调用合约方法store
```ts
const store = async (newValue: number): Promise<number> => {
  // Only the owner can store by walletClient instead of publicClient
  const txHash = await walletClient.writeContract({
    address: address,
    abi: abi,
    functionName: "store",
    args: [newValue],
  });
  const response = await publicClient.waitForTransactionReceipt({
    hash: txHash,
  });
  return parseInt(response.logs[0].data, 16);
};

```
以上是发布合约，调用合约的全部步骤，你如果想要看全部code,可以点击repo查看[详情](https://github.com/Bonnie-dot/contracts)。