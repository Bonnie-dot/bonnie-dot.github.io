---
title: Merkle Tree：区块链中的数据完整性与高效验证之道
tags:
  - 区块链
categories:
  - 区块链
poster:
  topic: 标题上方的小字
  headline: 大标题
  caption: 标题下方的小字
  color: 标题颜色
abbrlink: 2fd92054
date: 2025-03-27 07:57:22
description:
cover:
banner:
---
### 什么是Merkle Tree？

Merkle Tree（默克尔树）是一种树形数据结构，其设计初衷是高效且安全地验证数据的完整性和一致性。它在区块链、分布式系统以及文件系统中广泛应用，例如比特币和以太坊，IPFS。Merkle Tree的核心思想是通过递归的哈希计算，将大数据量归约为一个小的哈希值，即“Merkle Root”。


![image.png](merkle-tree.png)

### Merkle Tree的原理

1.  **叶子节点（Leaf Nodes）：**

    -   叶子节点存储原始数据的哈希值（通常是通过加密哈希函数如SHA-256计算得出）,eg:H(A),H(B)。

1.  **非叶子节点（Non-Leaf Nodes）：**

    -   非叶子节点的值是其子节点的哈希值组合计算得出的,eg: H0,H1。
    -   常见的计算方式是： Hash(Left Child+Right Child)
        -   当数据块数量不足时，可重复使用最后一个节点的哈希值以填充。
        -   在稀疏Merkle树（Sparse Merkle Tree）中，非叶子节点还可以存储空数据的默认哈希值，从而适应更大的数据范围。

1.  **根节点（Root Node）：**

    -   最终的Merkle Root是树的根节点，表示整棵树的数据摘要,eg:Root Hash。

### 基础操作
下面会使用js代码和`crypto`库来展示基础操作，这里有一个前置内容，我们先定义一个class和其属性：
```js
const crypto = require("crypto");
class MerkleTree {
  constructor(data) {
    this.levels = [data];
    this.buildTree(data);
    this.merkleRoot = this.levels[this.levels.length - 1][0];
  }
  hashData(data) {
    return crypto.createHash("sha256").update(data).digest("hex");
  }
 }
```
1.  **Build Tree（构建Merkle树）：**

    -   输入数据块，将其依次计算哈希值作为叶子节点。
    -   逐层向上计算非叶子节点的哈希值，直到得到根节点。
```js
buildTree(blocks) {
    if (blocks.length === 1) return blocks;
    const nextLevel = [];
    for (let i = 0; i < blocks.length; i += 2) {
      const left = blocks[i];
      const right = blocks[i + 1] || left;
      const combinedHash = this.hashData(
        left < right ? left + right : right + left
      );
      nextLevel.push(combinedHash);
    }
    this.levels.push(nextLevel);
    return this.buildTree(nextLevel);
 }
  ```

2.  **Get Path（获取验证路径）：**

    -   对于某个叶子节点，验证路径是从该节点到根节点的所有兄弟节点的哈希值。
    -   验证路径用于高效地证明某个数据块是否属于这棵树。
```js
  getPath(targetBlock) {
    if (!this.levels[0].includes(targetBlock)) {
      throw new Error("Target block not found in data blocks");
    }

    let path = [];
    let currentLevel = this.levels[0];
    let levelIndex = 0;
    while (currentLevel.length > 1) {
      const index = currentLevel.indexOf(targetBlock);
      const siblingIndex = index % 2 === 0 ? index + 1 : index - 1;
      console.log("siblingIndex:", siblingIndex);
      // 添加兄弟节点的哈希值
      if (siblingIndex < currentLevel.length) {
        path.push(currentLevel[siblingIndex]);
      } else {
        path.push(currentLevel[index]); // 奇数节点时重复自身
      }
      currentLevel = this.levels[++levelIndex];
      targetBlock = this.hashData(
        targetBlock < path[path.length - 1]
          ? targetBlock + path[path.length - 1]
          : path[path.length - 1] + targetBlock
      );
    }

    return path;
  }
```
比如以A为例，标注为红色的为其路径

![image.png](get-path.png)
3.  **Verify Path（验证路径）：**

-   验证路径通过逐层哈希计算并与Merkle Root对比，确保数据的完整性。
-   如果计算结果与Merkle Root一致，则证明数据块未被篡改。

```js
verifyPath(targetBlock, path) {
    let currentHash = targetBlock;
    for (const siblingHash of path) {
      currentHash = this.hashData(
        currentHash < siblingHash
          ? currentHash + siblingHash
          : siblingHash + currentHash
      );
      console.log("currentHash:", currentHash);
    }
    return currentHash === this.merkleRoot;
  }
```

<p align="center">完整代码可以查看<a href="https://github.com/Bonnie-dot/data-structure-and-algorithm/blob/master/src/07_tree/merkleTree/mekleTree.ts">这里</a></p>

### Merkle Tree的核心功能

1.  **数据完整性验证：**

    -   通过比较数据的验证路径和Merkle Root，可以快速验证某数据块是否属于树的一部分。
    -   即使是海量数据，也只需提供一条较短的验证路径，节省存储和计算成本。

1.  **快速数据同步：**

    -   在分布式系统中，可以通过Merkle Root快速对比不同节点的数据是否一致。
    -   不一致的部分通过验证路径定位后进行同步，无需同步整个数据集。

1.  **隐私保护：**

    -   零知识证明的应用场景中，Merkle Tree可以验证数据的存在性而无需暴露具体内容。
 例如，只需提供验证路径即可证明某笔交易属于某个区块。

1.  **版本管理：**

    -   Merkle Tree可用于跟踪数据版本的变化，例如Git中的文件变更管理，每次变更都会生成新的Merkle Root。

1.  **动态更新：**

    -   通过扩展结构（如动态Merkle Tree或稀疏Merkle Tree），可以高效地插入、删除或修改数据块。




### Merkle Tree在比特币中的应用

在比特币中，Merkle Tree主要用于区块中交易数据的组织和验证：

1.  每个交易数据的哈希值作为叶子节点。
1.  根节点（Merkle Root）存储在区块头中。
1.  当需要验证某个交易是否在区块内时，仅需提供验证路径，而不需要下载整个区块。

这种方式显著降低了存储和计算成本，同时保证了验证的高效性和安全性。

![image.png](merkle-tree-bitcoin.png)

比特币的客户端设计主要依赖于 **简单支付验证（SPV, Simplified Payment Verification）** ，它能够证明某笔交易存在于区块链中，但在状态验证方面存在几个关键缺陷：

#### **1. 无法验证当前账户状态**

比特币客户端只能证明某笔交易是否包含在区块中（即**包含性证明**），但无法得知账户的当前状态。例如：

-   你无法直接查询某个地址当前有多少比特币。
-   你只能依赖外部节点提供的信息，并假设至少一个节点会通知你所有相关交易。
-   如果某个交易的最终效果依赖于之前的多个交易（如多次 UTXO 消耗），你需要回溯整个交易链条来计算余额。

#### **2. 依赖第三方节点，增加信任假设**
  
比特币客户端通常需要连接多个全节点，并假设至少一个诚实节点会返回正确的信息。但如果所有你连接的节点都试图欺骗你，或者拒绝提供某些交易信息，你可能无法及时得知你的账户发生了变化。

#### **3. 无法高效验证复杂合约**

比特币的 UTXO（未花费交易输出）模型适用于简单的支付交易，但难以扩展到智能合约：

-   UTXO 仅包含“**某个地址是否可以花费某笔交易**”的信息，而不像以太坊账户模型那样记录账户状态。
-   复杂合约需要追踪状态变更，而比特币客户端无法高效完成这项工作。

#### **4. 交易影响的验证成本高**

-   在比特币中，一笔交易的影响可能取决于一系列过去的交易，因此客户端若想验证交易最终是否有效，往往需要回溯整个历史。
-   例如，要验证某个地址的余额，必须检查所有相关的 UTXO，而 UTXO 本身是分散在整个区块链中的，没有集中存储。
### Merkle Tree在以太坊中的应用

以太坊使用了多种Merkle树（例如Patricia Merkle Tree）来组织账户状态和交易数据：

![image.png](./merkle-tree-etherum.png)
#### 以太坊的状态（State）、交易（Transaction）和收据（Receipt）

#### 1. State Trie（状态树）
**作用**：存储所有账户（EOA 和合约账户）的状态，包括余额、Nonce、合约存储等。  
**存储位置**：区块头中的 `State Root`。  

#### **包含的信息**
每个账户由 **地址（Address） → 账户状态（Account State）** 组成，账户状态包含：

| 字段             | 作用 |
|----------------|---------------------------------------------|
| **Nonce**     | 交易计数，防止重放攻击 |
| **Balance**   | 账户余额（以 Wei 计算） |
| **Storage Root** | 该账户存储状态的默克尔树根，仅合约账户有 |
| **Code Hash** | 账户的合约代码哈希，仅合约账户有 |

---

#### 2. Transaction Trie（交易树）
**作用**：存储该区块的所有交易数据。  
**存储位置**：区块头中的 `Transactions Root`。  

#### **包含的信息**
每个交易由 **交易索引（Index） → 交易数据（Transaction Data）** 组成。  

| 字段           | 作用 |
|--------------|---------------------------------------------|
| **Nonce**    | 交易发送方的交易计数 |
| **Gas Price** | 交易支付的 Gas 费用 |
| **Gas Limit** | 交易最大 Gas 消耗限制 |
| **To**       | 交易接收方地址 |
| **Value**    | 交易发送的 ETH 数量 |
| **Data**     | 交易附带的数据（如合约调用参数） |
| **V, R, S**  | 交易签名 |

---

#### 3. Receipts Trie（收据树）
**作用**：存储该区块中所有交易的执行结果，包括日志（Logs）和 Gas 消耗等信息。  
**存储位置**：区块头中的 `Receipts Root`。  

#### **包含的信息**
每个交易执行完成后，会生成一条收据，由 **交易索引（Index） → 交易收据（Receipt Data）** 组成。  

| 字段             | 作用 |
|----------------|---------------------------------------------|
| **Status**    | 交易执行结果（1 = 成功, 0 = 失败） |
| **Cumulative Gas Used** | 当前区块内所有交易的总 Gas 消耗 |
| **Bloom**     | 用于快速索引日志的 Bloom 过滤器 |
| **Logs**      | 交易触发的事件日志 |

---

#### **总结**
| Trie 类型 | 作用 | 存储位置 | 主要内容 |
|----------|-----|---------|---------|
| **State Trie（状态树）** | 记录账户的状态 | 区块头的 `State Root` | 账户余额、Nonce、存储根、合约代码哈希 |
| **Transaction Trie（交易树）** | 记录该区块的所有交易 | 区块头的 `Transactions Root` | 交易数据（From、To、Value、Gas、Data 等） |
| **Receipts Trie（收据树）** | 记录交易的执行结果 | 区块头的 `Receipts Root` | 交易状态、Gas 消耗、日志（Logs） |

这三棵 Trie 的根哈希被存储在区块头中，使得客户端可以通过 **Merkle Proof（默克尔证明）** 验证账户状态、交易数据和交易执行结果，而无需下载整个区块链。

为了比特币的问题，以太坊引入了 **Merkle Patricia Trie** 结构，使得客户端不仅可以验证交易是否存在，还可以高效验证账户余额、智能合约状态等数据：

-   **状态证明（State Proofs）** ：以太坊的区块不仅包含交易数据，还包含整个区块链状态的默克尔根，使得客户端可以高效验证账户状态，而不需要下载整个链。
-   **智能合约支持**：以太坊的账户模型记录了合约状态，并通过 Trie 结构组织，使得状态变化可以通过默克尔证明高效查询。

这些改进使得以太坊客户端能够在**无需信任全节点的情况下，直接验证账户余额、智能合约状态等信息**，从而比比特币的 SPV 机制更加强大。

### 总结

Merkle Tree是高效且安全的数据结构，其核心功能包括数据完整性验证、快速数据同步、隐私保护、版本管理等。在区块链技术中，Merkle Tree的应用极大地提升了数据验证的效率和可靠性。比特币通过Merkle树简化了交易验证，以太坊则扩展了其功能，用于存储和验证账户状态和交易数据。

### 引用
- [Merkling in Ethereum](https://blog.ethereum.org/2015/11/15/merkling-in-ethereum)
