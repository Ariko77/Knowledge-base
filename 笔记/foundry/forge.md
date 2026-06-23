# forge

## **forge**：Solidity 构建与测试工具

### 常用命令

| 命令 | 说明 |
| --- | --- |
| <font style="background-color:#f3bb2f;">build</font> | 编译项目合约 \[别名: compile] |
| <font style="background-color:#f3bb2f;">clean</font> | 清除构建缓存和产物 |
| <font style="background-color:#f3bb2f;">create</font> | 部署合约 |
| <font style="background-color:#f3bb2f;">doc</font> | 生成合约文档 |
| `eip712` | 生成 EIP-712 编码结构 |
| <font style="background-color:#f3bb2f;">flatten</font> | 打平成一个合约文件 |
| <font style="background-color:#f3bb2f;">fmt</font> | 格式化 Solidity 代码 |
| `geiger` | 检测危险 cheatcode |
| `init` | 初始化新项目 |
| <font style="background-color:#f3bb2f;">install</font> | 安装依赖 |
| <font style="background-color:#f3bb2f;">remove</font> | 移除依赖 |
| ==remappings | 查看路径映射 |
| <font style="background-color:#f3bb2f;">script</font> | 运行合约脚本 |
| `selectors` | 函数选择器工具 |
| <font style="background-color:#f3bb2f;">snapshot</font> | 保存 gas 使用快照 |
| `soldeer` | 依赖管理工具 |
| <font style="background-color:#f3bb2f;">test</font> | 运行测试 |
| `tree` | 显示依赖关系树 |
| `update` | 更新依赖 |
| <font style="background-color:#f3bb2f;">verify-contract</font> | 在 Etherscan 验证合约 |
| `verify-bytecode` | 验证字节码与源码一致性 |
| `verify-check` | 检查验证状态 |

### 通用选项

| 选项 | 说明 |
| --- | --- |
| `-h, --help` | 显示帮助信息 |
| `-V, --version` | 显示版本 |
| `-j, --threads` | 设置线程数量 |
| `--json` | JSON 输出 |
| `-q, --quiet` | 安静模式 |
| `-v, --verbosity` | 日志等级（可叠加） |

更多内容：[forge 官方文档](https://book.getfoundry.sh/reference/forge/forge.html)

### 常见命令+实例

#### `forge build` — 编译项目合约

编译整个 Foundry 项目，生成字节码和 ABI，构建缓存等。

```solidity
forge build
```

**中文**：编译项目中的 Solidity 合约，生成构建产物（在 `out/` 和 `cache/` 文件夹中）。

***

#### `forge clean` — 清除构建缓存

```solidity
forge clean
```

**中文**：删除 `out/` 和 `cache/`，清理构建产物，便于重新构建。

***

#### `forge test` — 运行测试

运行 `test/` 目录中的测试合约。

```solidity
forge test
```

**运行指定测试文件或函数：**

```solidity
forge test --match-path test/Fallback.t.sol
forge test --match-test testFallbackAttack
```

**中文**：运行测试合约中所有以 `test` 开头的函数。

***

#### `forge script` — 运行合约脚本（可选广播）

```solidity
forge script script/Deploy.s.sol:Deploy \
--rpc-url <RPC_URL> --private-key <私钥> --broadcast
```

**中文**：运行指定脚本（如部署、调用函数），并用私钥签名广播交易。

***

#### `forge create` — 快速部署合约

```solidity
forge create src/MyContract.sol:MyContract \
--private-key <私钥> --rpc-url <RPC_URL>
```

**中文**：直接部署一个合约实例，不需写脚本。

***

#### `forge install` — 安装依赖包

```solidity
forge install rari-capital/solmate
```

**中文**：从 GitHub 安装依赖，保存在 `lib/` 目录中。

***

#### `forge remove` — 删除依赖包

```solidity
forge remove rari-capital/solmate
```

**中文**：移除已安装的依赖。

***

#### `forge flatten` — 扁平化合约源码

```solidity
forge flatten src/MyContract.sol > Flat.sol
```

**中文**：将多文件合约整合成一个文件，便于 Etherscan 验证。

***

#### `forge doc` — 生成项目文档

```solidity
forge doc
```

**中文**：根据合约注释生成 Markdown 文档。

***

#### `forge fmt` — 格式化 Solidity 代码

```solidity
forge fmt
forge fmt --check  # 只检查格式是否正确
```

**中文**：自动统一代码风格。

***

#### `forge inspect` — 检查合约结构

```solidity
forge inspect MyContract abi
forge inspect MyContract storage
```

**中文**：查看合约的 ABI 或存储结构等。

***

#### `forge snapshot` — Gas 使用快照

```solidity
forge snapshot
```

**中文**：记录每个测试函数的 gas 使用量，输出到 `gas-snapshot` 文件。

***

#### `forge remappings` — 查看路径映射

```solidity
forge remappings
```

**中文**：查看 `remappings.txt` 中配置的库路径映射。

***

#### `forge verify-contract` — Etherscan 合约验证

```solidity
forge verify-contract <合约地址> MyContract <etherscan-api-key> --chain-id 1
```

**中文**：将合约源码提交到 Etherscan 进行验证。


> 更新: 2025-07-11 10:40:08  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/ep7ofy85emu1raf0>