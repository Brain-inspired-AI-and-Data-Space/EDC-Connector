# Eclipse Dataspace Connector (EDC) 源码结构与功能模块分析

源码地址： https://github.com/eclipse-edc/Connector

------------------------------------------------------------------------

# 1. 仓库总体结构

EDC 是一个大型 Gradle 多模块工程。核心目录包括：

-   core/
-   spi/
-   extensions/
-   data-protocols/
-   system-tests/
-   dist/bom/
-   docs/

整个项目通过 settings.gradle.kts 组织成大量子模块。

------------------------------------------------------------------------

# 2. 核心目录与功能分区

## 2.1 spi ------ 扩展接口层（系统边界）

SPI 是整个 EDC 的抽象能力层，定义：

-   领域接口
-   扩展点
-   模型对象
-   枚举与协议抽象

主要分区：

-   spi:common（通用能力，如 token、http、json-ld、policy-engine、store
    等）
-   spi:control-plane（合同、传输、目录相关接口）
-   spi:data-plane（数据流执行接口）
-   spi:data-plane-selector（数据面选择器接口）
-   spi:policy-monitor（策略监控接口）

理解方式： 如果要替换默认实现，必须先理解对应 SPI。

------------------------------------------------------------------------

## 2.2 core ------ 核心运行逻辑

core 目录提供最小可运行连接器能力。

### 2.2.1 core:common

-   runtime-core：运行时装配
-   connector-core：核心服务注册
-   token-core：token 处理
-   edr-store-core：EDR 管理
-   state-machine-lib：状态机引擎
-   policy-engine-lib：策略引擎实现

### 2.2.2 core:control-plane

负责：

-   合同协商状态机
-   传输流程编排
-   目录发布逻辑
-   模型转换

关键子模块：

-   control-plane-contract(-manager)
-   control-plane-transfer(-manager)
-   control-plane-catalog
-   control-plane-transform

### 2.2.3 core:data-plane

负责真实数据流执行：

-   data-plane-core
-   data-plane-util

### 2.2.4 data-plane-selector

-   data-plane-selector-core

用于多数据面实例选择与路由。

### 2.2.5 policy-monitor

-   policy-monitor-core

用于策略执行监控。

------------------------------------------------------------------------

## 2.3 data-protocols ------ 协议实现层

重点是 DSP（Dataspace Protocol）。

模块结构：

-   dsp-spi
-   dsp-http-spi
-   dsp-core
-   dsp-lib
-   dsp-dispatcher
-   dsp-2025（版本化实现）

功能：

-   HTTP 入口
-   协议校验
-   payload 转换
-   catalog / negotiation / transfer dispatcher

此外包含：

-   data-plane-signaling

用于控制面与数据面信令交互。

------------------------------------------------------------------------

## 2.4 extensions ------ 技术实现层

extensions 是对 SPI 的具体实现。

### 2.4.1 common 扩展

-   management-api
-   control-api
-   version-api
-   observability
-   OAuth2 / token-based auth
-   decentralized identity
-   verifiable credentials
-   hashicorp vault
-   SQL 存储实现
-   transaction 管理
-   AES 加密

### 2.4.2 control-plane 扩展

-   asset 管理 API
-   contract 协商 API
-   policy 定义 API
-   transfer process API
-   SQL store 实现
-   callback 机制

### 2.4.3 data-plane 扩展

-   HTTP 传输
-   Kafka 传输
-   OAuth2 数据面认证
-   self-registration
-   signaling API
-   SQL store

### 2.4.4 data-plane-selector 扩展

-   selector API
-   selector SQL store

### 2.4.5 policy-monitor 扩展

-   SQL 持久化

------------------------------------------------------------------------

## 2.5 system-tests ------ 系统级验证

包含：

-   e2e transfer tests
-   dataplane tests
-   DSP TCK
-   Management API tests
-   Version API tests

用于验证模块组合与行为正确性。

------------------------------------------------------------------------

## 2.6 dist:bom ------ 依赖管理

提供：

-   control-plane BOM
-   data-plane BOM
-   feature-sql BOM
-   dcp BOM

用于统一版本对齐与功能组合。

------------------------------------------------------------------------

# 3. 架构分层总结

  层级             作用
---------------- --------------------------
  SPI              定义扩展能力边界
  Core             提供领域状态机与核心编排
  Data Protocols   协议实现与适配
  Extensions       技术落地实现
  System Tests     行为验证
  BOM              依赖组合与版本对齐

------------------------------------------------------------------------

# 4. 推荐源码阅读路径

1.  先看 SPI（理解能力边界）
2.  阅读 core（理解状态机与编排逻辑）
3.  阅读 data-protocols（协议如何映射到核心能力）
4.  阅读 extensions（默认实现如何落地）
5.  使用 system-tests 验证理解

------------------------------------------------------------------------

# 5.EDC 的完整数据交换流程

EDC 的完整数据交换流程分为：

1. 目录发现（Catalog）
2. 合同协商（Contract Negotiation）
3. 传输执行（Transfer Process）

完整链路全貌

下面是从 Consumer 角度的真实完整流程。

------

## 第一阶段：目录发现

Consumer 调用：

```
/catalog/request
```

流程：

1. DSP API 接收请求
2. CatalogService 查询资产
3. Policy 附带 Offer
4. 返回 CatalogResponse

这一步还没有任何合同。

------

## 第二阶段：合同协商

Consumer 发起：

```
/contractnegotiations
```

流程：

1. 创建 ContractNegotiation 实体
2. 进入协商状态机
3. 双方交换 Offer / CounterOffer
4. Provider PolicyEngine 评估
5. 达成 Agreement
6. 生成 ContractAgreement

这个阶段的状态机是：

```
INITIAL → REQUESTED → OFFERED → AGREED → FINALIZED
```

------

## 第三阶段：数据传输

有了 Agreement 之后：

1. Consumer 创建 TransferRequest
2. TransferProcess 状态机启动
3. Provision
4. DataPlane 执行
5. 回调
6. 完成

------

所以完整链路是：

```
Catalog
   ↓
ContractNegotiation
   ↓
Agreement
   ↓
TransferProcess
   ↓
DataPlane Execution
```

------

# SPI总体设计思路

研究 EDC 源码时，SPI 就是“核心逻辑（core）允许你替换/插入的边界”。官方也明确：`spi` 是 connector 的主要扩展点，包含需要实现的接口、必要的模型与枚举，决定了用户能自定义到什么程度。

EDC 的 SPI 基本遵循同一套模式：

1. **服务接口（Service / Manager / Resolver）**：定义“做什么”。
2. **存储接口（Store）**：定义“怎么持久化 + 怎么按状态/条件取数”。控制平面尤其重。
3. **模型与上下文（Model / Context）**：跨模块传递的 DTO/上下文对象。
4. **可插拔策略/规则（Policy / CEL / Validator）**：定义“允许/不允许”的计算入口。
5. **协议与安全（Protocol / Auth / Token / VC）**：定义“怎么跟外部对接、怎么信任”。

你解读源码时，正确顺序是：**SPI 定义边界 → core 使用这些接口做状态机/编排 → extensions 给出默认实现**（SQL/Vault/OAuth2/HTTP…）。

把 SPI 按“你到底想换什么”归类：

- **我要换存储实现（SQL/NoSQL/Memory）**：看各域 `*Store` 接口（多在 control-plane SPI + common store 抽象），配合 `QuerySpec`。 [GitHub+1](https://github.com/eclipse-edc/Connector/blob/main/spi/common/core-spi/src/main/java/org/eclipse/edc/spi/query/QuerySpec.java?utm_source=chatgpt.com)
- **我要换策略决策方式**：看 `policy-engine-spi` + `policy-model` + request context + `cel-spi`。 [GitHub+1](https://github.com/eclipse-edc/Connector/discussions/1836?utm_source=chatgpt.com)
- **我要换身份与信任栈**：看 auth/token/oauth2/jwt + did/vc/decentralized-claims。 [GitHub](https://raw.githubusercontent.com/eclipse-edc/Connector/main/settings.gradle.kts)
- **我要换协议入口（DSP/自定义协议）**：看 `protocol-spi` + json-ld/transform/validator。 [GitHub+1](https://raw.githubusercontent.com/eclipse-edc/Connector/main/settings.gradle.kts)
- **我要扩展数据传输类型**：看 `data-plane-spi` + data-address spis + data-plane-http-spi。

------

# 控制平面内部的传输状态机逻辑（第三阶段）

有了 Agreement 之后：

1. Consumer 创建 TransferRequest
2. TransferProcess 状态机启动
3. Provision
4. DataPlane 执行
5. 回调
6. 完成

搞清楚 **TransferProcess 从创建 → 协商完成 → 数据面执行 → 完成/失败** 的完整控制链路。

按“抽象能力面 → 状态机驱动 → 存储并发控制 → 数据面协作”四层拆。

------

## 一、Transfer 的抽象能力面（SPI 层）

先看 `spi:control-plane:transfer-spi`。

这是状态机的“接口边界”。

### 1️⃣ 核心模型：TransferProcess

抽象表达：

- id
- type（pull/push）
- state
- contractId
- dataRequest
- callbackAddresses
- errorDetail
- provisionedResources
- createdAt / updatedAt

核心意义：

> TransferProcess 是一个持久化的状态机实体。

它不是一次 HTTP 请求。
 它是一个长期存在的“流程对象”。

------

### 2️⃣ 核心接口：TransferProcessStore

典型能力面：

- save()
- findById()
- query(QuerySpec)
- nextForState(int state, int batchSize)
- update()
- delete()

⚠ 关键点在：

```
nextForState(...)
```

这是状态机调度器拉任务的入口。

它必须满足：

- 按状态取一批
- 加锁（防止多节点重复处理）
- 原子更新状态

否则分布式部署会乱。

------

### 3️⃣ TransferProcessManager（SPI 抽象）

定义能力：

- initiateTransfer()
- terminate()
- complete()
- error()
- start()

Manager 是状态机驱动者。

------

## 二、Core 层：状态机驱动逻辑

进入：

```
core:control-plane:control-plane-transfer(-manager)
```

这里是整个控制平面的“发动机”。

------

### 1️⃣ 典型架构

TransferProcessManager 内部通常包含：

- StateMachine
- ProvisionManager
- DataPlaneSignalingClient
- TransferProcessStore
- Clock
- Executor / Scheduler

核心思想：

> 这是一个定时轮询驱动的状态推进器。

不是事件驱动。
 是“定时扫库 + 推进状态”。

------

### 2️⃣ 状态机推进逻辑（概念模型）

伪代码结构：

```
while (running) {
  processes = store.nextForState(INITIAL, batch)
  for each process:
      transitionTo(PROVISIONING)
}
```

然后针对每个状态都有处理逻辑：

| 状态         | 处理逻辑              |
| ------------ | --------------------- |
| INITIAL      | 校验 + 创建资源       |
| PROVISIONING | 调用 ProvisionManager |
| PROVISIONED  | 发起数据面请求        |
| IN_PROGRESS  | 等待数据面回调        |
| COMPLETED    | 终态                  |
| FAILED       | 终态                  |

核心：

> 状态推进是幂等的。

如果服务重启：

- 未完成的流程会继续被拉起
- 状态从数据库恢复

------

## 三、Provision 阶段（资源准备）

ProvisionManager 的作用：

- 为数据访问生成资源
- 生成 EDR（Endpoint Data Reference）
- 创建 token
- 生成临时访问地址

例如：

- S3 bucket 临时凭证
- HTTP proxy endpoint

Provision 不是数据传输。

它只是“准备数据访问条件”。

------

## 四、数据面协作机制

核心接口：

- DataPlaneSignalingClient

控制平面不会直接传输数据。

它只：

1. 构造 DataFlowRequest
2. 调用数据面
3. 等待回调

------

### 数据面执行流程

控制平面发：

```
START data flow
```

数据面执行：

- 拉数据
- 推数据
- 完成

数据面回调：

```
TRANSFER_COMPLETED
TRANSFER_FAILED
```

控制平面更新状态。

------

## 五、并发与分布式一致性（关键设计）

这是必须重点研究的部分。

### 1️⃣ nextForState 的核心作用

它通常需要：

- SELECT ... FOR UPDATE
- 或 optimistic locking
- 或 lease 机制

否则：

两个节点会同时推进同一个 TransferProcess。

------

### 2️⃣ 乐观锁字段

TransferProcess 通常包含：

- state
- stateCount
- updatedAt

更新时：

```
WHERE id = ? AND state = oldState
```

更新失败说明已被其他节点处理。

------

## 六、状态机是“数据库驱动型”

EDC 的设计不是：

> 事件驱动 + 消息队列

而是：

> 状态持久化 + 轮询驱动

优点：

- 简单
- 易恢复
- 不依赖 MQ

缺点：

- 高并发下数据库压力大
- 延迟依赖轮询频率

------

## 七、完整链路总结（控制平面视角）

1. 用户调用 Management API 创建 transfer
2. TransferProcess 被持久化，状态 = INITIAL
3. StateMachine 轮询发现
4. 进入 PROVISIONING
5. 资源准备完成
6. 调用 DataPlaneSignalingClient
7. 状态 = IN_PROGRESS
8. 数据面回调
9. 状态 = COMPLETED 或 FAILED

------

## 八、应该读的具体代码顺序

按顺序：

1. TransferProcess（模型）
2. TransferProcessStore（接口）
3. TransferProcessManager（核心驱动）
4. ProvisionManager
5. DataPlaneSignalingClient
6. SQL Store 实现（看 nextForState 实现）

------

# **合同协商**（**Contract Negotiation**）的状态机主线（第二阶段）

Consumer 发起：

```
/contractnegotiations
```

流程：

1. 创建 ContractNegotiation 实体
2. 进入协商状态机
3. 双方交换 Offer / CounterOffer
4. Provider PolicyEngine 评估
5. 达成 Agreement
6. 生成 ContractAgreement

从 `contract-spi` 的 **Store** / **Service 接口** 开始，逐步追踪到 **核心状态机如何执行**，并深入分析 **SQL 层如何控制并发** 和 **状态迁移**。

------

## **一、合同协商的 SPI 接口**

### 1.1 `contract-spi`（合同协商的接口定义）

合同协商的核心在于 `ContractNegotiation` 和 `ContractAgreement` 两个概念。整个协商过程需要依赖一系列的接口来完成，包括：

- **ContractNegotiation**（合同协商）：用于表示合同的协商过程，是一个可变状态的实体。
- **ContractAgreement**（合同协议）：合同协商完成后生成的协议对象，表示最终达成的协议。

**`contract-spi`** 包含以下重要接口：

1. **ContractNegotiationStore**：用于持久化合同协商的状态。
2. **ContractNegotiationManager**：管理合同协商状态的逻辑，处理合同的不同阶段（创建、修改、批准、拒绝等）。

------

## **二、合同协商的生命周期与状态机**

合同协商有一个明确的生命周期，这个生命周期通过 `ContractNegotiation` 和 `ContractAgreement` 来管理。下面是合同协商的状态机转换。

### 2.1 `ContractNegotiation` 状态机

状态机从以下几个阶段流转：

1. **INITIAL**：合同协商刚开始。
2. **REQUESTED**：协商已被请求，等待对方响应。
3. **OFFERED**：对方已经做出了回应，双方已交换要素。
4. **AGREED**：合同已达成一致，准备签署。
5. **FINALIZED**：合同已最终签署。

------

## **三、合同协商在核心中的实现**

在 `core:control-plane:control-plane-contract` 中，合同协商的管理逻辑被封装在 **`ContractNegotiationManager`** 类中。它通过状态机驱动合同协商的整个过程。

### 3.1 `ContractNegotiationManager`：合同协商管理器

`ContractNegotiationManager` 类的作用是处理合同协商的整个生命周期，包括：

- 接收请求，创建 `ContractNegotiation` 对象
- 更新协商状态
- 生成最终的 `ContractAgreement`
- 调用存储接口持久化协商过程

### `ContractNegotiationManager` 的关键方法

- **`initiateNegotiation()`**：启动合同协商，初始化 `ContractNegotiation` 对象，状态设置为 `REQUESTED`。
- **`offerAgreement()`**：在协商中，提供一份合同，状态变为 `OFFERED`。
- **`finalizeAgreement()`**：达成协议，状态变为 `AGREED`，并最终生成 `ContractAgreement`。
- **`rejectAgreement()`**：拒绝合同，状态变为 `REJECTED`。
- **`completeNegotiation()`**：协商完成，状态变为 `FINALIZED`。

------

## **四、SQL 层并发控制与锁机制**

### 4.1 `ContractNegotiationStore`（合同协商存储）

`ContractNegotiationStore` 负责持久化合同协商的过程，提供基本的增、删、改、查操作。主要方法：

- **save()**：保存合同协商。
- **findById()**：通过 ID 获取合同协商信息。
- **update()**：更新合同协商状态。
- **nextForState()**：根据当前状态查询下一个待处理的合同协商。

特别需要关注 **`nextForState()`** 方法，它的实现决定了如何在并发环境中拉取合同协商对象并更新状态。

```
public List<ContractNegotiation> nextForState(int state, int batchSize) {
    String query = "SELECT * FROM contract_negotiations WHERE state = ? FOR UPDATE LIMIT ?";
    return jdbcTemplate.query(query, new Object[] { state, batchSize }, new ContractNegotiationRowMapper());
}
```

这段 SQL 使用了 **`FOR UPDATE`**，这意味着：

- **行级锁**：数据库会锁定查询出来的合同协商记录，防止其他节点重复处理。
- **`batchSize`**：控制每次查询的数量，避免一次性查询过多记录导致性能瓶颈。

### 4.2 锁的作用

使用 **行级锁** 保证了即使在分布式环境中，多个实例在同一时刻处理合同协商时，不会出现竞争条件（race condition）。这对于保证状态机的正确性至关重要。

- 当一个节点（例如 A）查询到某个合同协商并更新状态时，其他节点（例如 B）将无法获取相同的记录进行修改，直到 A 释放锁。
- **乐观锁**：如果使用了乐观锁（例如，通过 `stateCount` 字段），那么在更新时，如果 `stateCount` 被修改，则更新操作会失败，这时可以尝试重新执行协商。

### 4.3 并发控制机制（nextForState）

`nextForState()` 方法会查询当前状态的合同协商对象，并加锁。每次查询时，都会执行一个 **SELECT FOR UPDATE** 操作，确保只有一个节点能够获取到某个状态的协商对象进行处理。

```
SELECT * FROM contract_negotiations WHERE state = ? FOR UPDATE LIMIT ?
```

这个查询确保：

- **仅一个节点**能获得该合同协商进行修改（防止重复执行）。
- **批量处理**：通过 `batchSize` 参数控制每次拉取的数据量，避免大规模查询带来的性能问题。

------

## **五、合同协商完整流程总结**

1. **初始化协商**：Consumer 发起合同协商请求，创建 `ContractNegotiation`，状态为 `REQUESTED`。
2. **提供合同**：Provider 提供合同条款，状态变为 `OFFERED`。
3. **协议达成**：双方达成一致，状态变为 `AGREED`。
4. **完成协商**：生成 `ContractAgreement`，状态变为 `FINALIZED`，合同协商过程结束。

------

# 目录发现（Catalog Discovery）主线（第三阶段）

按照 **`catalog-spi`** 的接口 / **核心服务管理器** / **SQL 存储与并发控制** 这条主线，详细剖析整个目录发现的流程。

Consumer 调用：

```
/catalog/request
```

流程：

1. DSP API 接收请求
2. CatalogService 查询资产
3. Policy 附带 Offer
4. 返回 CatalogResponse

这一步还没有任何合同。

目录发现的关键目标是：**Consumer 查找并查询 Provider 发布的资源，获取数据资产信息**。这一过程涉及 **目录查询**、**资源映射**、**协议约定** 等步骤。

------

## 一、目录发现的 SPI 接口

### 1.1 `catalog-spi`：目录相关接口定义

**`catalog-spi`** 是 EDC 中目录管理的 SPI 层，负责定义目录查询、发布、存储等操作的接口。`catalog-spi` 包含以下关键接口：

1. **Catalog**：目录，描述了一个资产集合（assets）。
2. **CatalogStore**：目录存储接口，用于保存和查询目录。
3. **CatalogService**：核心服务，提供对外的目录查询服务。

这三者是实现目录发现的核心：

- **`Catalog`**：表示一个包含数据资产的集合。
- **`CatalogStore`**：负责将 `Catalog` 持久化，支持查询与更新操作。
- **`CatalogService`**：对外提供目录查询 API。

具体接口定义：

```
public interface CatalogService {
    List<Catalog> queryCatalog(String filter, QuerySpec querySpec);
    Catalog getCatalogById(String catalogId);
    void publishCatalog(Catalog catalog);
}

public interface CatalogStore {
    Catalog save(Catalog catalog);
    Catalog findById(String catalogId);
    List<Catalog> query(QuerySpec querySpec);
}
```

------

## 二、目录查询核心逻辑：`CatalogService`

### 2.1 `CatalogService` 核心功能

`CatalogService` 是与外部系统交互的主要服务，它提供了 **查询目录** 和 **发布目录** 的能力。该服务通过调用 **`CatalogStore`** 来实现持久化操作，并通过定义的查询方法来访问和筛选数据。

**关键方法**：

- **`queryCatalog()`**：查询目录，支持根据条件（如过滤器、分页、排序）进行检索。
- **`getCatalogById()`**：通过 ID 获取特定目录信息。
- **`publishCatalog()`**：发布新目录，将目录资源保存到存储中。

```
public class CatalogServiceImpl implements CatalogService {

    private final CatalogStore catalogStore;

    public CatalogServiceImpl(CatalogStore catalogStore) {
        this.catalogStore = catalogStore;
    }

    @Override
    public List<Catalog> queryCatalog(String filter, QuerySpec querySpec) {
        return catalogStore.query(querySpec);  // 调用 catalogStore 执行查询
    }

    @Override
    public Catalog getCatalogById(String catalogId) {
        return catalogStore.findById(catalogId);  // 调用 catalogStore 查找指定目录
    }

    @Override
    public void publishCatalog(Catalog catalog) {
        catalogStore.save(catalog);  // 调用 catalogStore 保存目录
    }
}
```

### 2.2 目录查询的实现原理

在 `CatalogServiceImpl` 类中，查询方法会依赖 **`CatalogStore`** 来持久化和查询目录数据。目录存储的操作通常涉及 **数据库查询**，并结合 **`QuerySpec`** 来支持过滤、分页、排序等复杂查询。

------

## 三、目录存储：`CatalogStore` 和 SQL 实现

### 3.1 `CatalogStore`：目录存储的接口

`CatalogStore` 负责将 `Catalog` 对象持久化到数据库中。它通常包括：

- **save()**：保存目录对象。
- **findById()**：根据目录 ID 查找目录。
- **query()**：根据查询条件（如 `QuerySpec`）查询多个目录。

```
public interface CatalogStore {
    Catalog save(Catalog catalog);
    Catalog findById(String catalogId);
    List<Catalog> query(QuerySpec querySpec);
}
```

在 `CatalogServiceImpl` 中，调用 `catalogStore` 进行数据的增、查、改操作。

### 3.2 SQL 层的并发控制：`nextForState` 和 `FOR UPDATE`

在目录存储的实现中，查询操作会涉及到 **`SELECT`** 语句，并且可能使用 **行级锁** 来避免并发问题。类似于之前提到的 **`nextForState()`** 在传输过程中的作用，目录存储在并发环境下同样需要确保数据一致性。

对于 **`query()`** 方法，可能会使用如下的 SQL 语句来获取符合条件的目录条目：

```
SELECT * FROM catalog WHERE filter = ? FOR UPDATE LIMIT ?;
```

- **`FOR UPDATE`**：在查询过程中为检索到的记录加锁，避免多个并发节点同时修改同一目录对象。
- **`LIMIT`**：控制查询的数量，避免一次性拉取过多数据，减轻数据库负担。

### 3.3 查询流程与锁机制

- 当 **`CatalogService.queryCatalog()`** 方法被调用时，首先会构造出查询条件，并传递给 **`CatalogStore.query()`** 方法。
- 查询方法会通过 SQL 查询，从数据库中检索符合条件的目录数据，并且会在查询时加上 **行级锁**（`FOR UPDATE`），防止其他节点在处理过程中同时修改这些目录条目。
- 如果数据库中没有合适的目录条目，查询会返回空列表。如果找到了符合条件的条目，`CatalogService` 将返回这些目录条目。

------

## 四、目录发现的完整流程总结

1. **Consumer 查询目录**：Consumer 调用 `CatalogService.queryCatalog()`，查询符合条件的目录资源。
2. **目录存储查询**：`CatalogService` 将查询请求传递给 `CatalogStore`，并使用 SQL 查询符合条件的目录。
3. **加锁处理**：在查询过程中，通过 **`FOR UPDATE`** 来保证在并发环境下只有一个节点可以修改目录信息。
4. **返回目录信息**：查询完成后，`CatalogService` 将返回符合条件的目录。

------

## 五、SQL 并发控制与锁的作用

在 **目录发现阶段**，并发控制主要通过 SQL 行级锁来保证：

1. **行级锁（`FOR UPDATE`）**：在查询过程中加锁，确保在分布式环境中，不同节点不会同时修改同一个目录。
2. **`LIMIT`**：通过限制查询的条目数，避免大量数据查询对系统造成负担。
3. **乐观锁/版本控制**：在某些情况下，可能会采用乐观锁来确保没有其他节点修改同一目录数据。

------

# 合同协商状态机和数据传输状态机完整集成

如果你想真正理解 EDC 架构，你必须：

先搞清楚：

- ContractNegotiation 状态机
- TransferProcess 状态机

这两个是 EDC 控制平面的“双核心”。

接下来我们将 **合同协商（Contract Negotiation）** 和 **数据传输（Transfer Process）** 的状态机流程集成起来，形成一个完整的流程。我们将深入分析：

1. **合同协商与数据传输的关联**：合同协商（Contract Negotiation）如何触发数据传输（Transfer Process）。
2. **状态机的集成**：从合同协商到数据传输的状态流转，如何通过状态机保持整个过程的连贯性。
3. **核心管理器如何协调**：`ContractNegotiationManager` 和 `TransferProcessManager` 如何互相协作，确保合同达成后数据传输可以无缝启动。

------

### **一、合同协商与数据传输的关联**

合同协商和数据传输是两个独立的过程，但它们之间有着明确的关联。在 EDC 的工作流程中，合同协商是数据传输的前置步骤。合同协商的目的是确保数据的交换符合预定的政策和协议，确保双方都同意数据的使用方式、访问权限等条件。

#### 1.1 合同协商的状态机

合同协商有一套明确的状态机，通常如下：

| 状态          | 描述                               |
| ------------- | ---------------------------------- |
| **INITIAL**   | 协商刚开始，尚未发起。             |
| **REQUESTED** | 请求已发送，等待对方回应。         |
| **OFFERED**   | 对方已回应，提供了合同要素。       |
| **AGREED**    | 双方达成一致，准备签署合同。       |
| **FINALIZED** | 合同最终签署，达成协议，合同完成。 |

在合同协商的过程中，`ContractNegotiationManager` 会管理这些状态的切换，从 `REQUESTED` 到 `OFFERED`，然后进入 `AGREED`，最终进入 `FINALIZED`。

#### 1.2 数据传输的触发

一旦合同进入 `FINALIZED` 状态，意味着双方已经达成了协议，接下来便可以启动数据传输。

- **Contract Agreement** 是合同协商完成后生成的协议实体。这个协议包含了数据的交换条件、传输的要求等信息。
- 当 `ContractNegotiation` 进入 `FINALIZED` 状态时，`ContractNegotiationManager` 会通知 **TransferProcessManager** 启动数据传输流程。

------

### **二、完整的状态机集成流程**

合同协商完成后，数据传输流程会依赖合同的状态。在 EDC 中，合同协商的成功达成（`FINALIZED`）会触发数据传输的开始，整个流程如下：

#### 2.1 合同协商完成后触发数据传输

1. **合同协商成功**：当 `ContractNegotiation` 进入 `FINALIZED` 状态时，`ContractNegotiationManager` 会生成 `ContractAgreement`，这个协议对象包含了传输数据的必要信息。
2. **触发数据传输**：`ContractNegotiationManager` 会调用 **`TransferProcessManager`** 来启动数据传输过程。这通常是通过调用 `TransferProcessManager.initiateTransfer()` 来完成的。

#### 2.2 数据传输的状态机

数据传输的状态机和合同协商的状态机类似，它管理了数据传输的整个生命周期。数据传输的状态通常如下：

| 状态             | 描述                               |
| ---------------- | ---------------------------------- |
| **INITIAL**      | 数据传输刚开始，尚未准备。         |
| **PROVISIONING** | 正在准备数据访问权限或资源。       |
| **PROVISIONED**  | 数据资源准备好，可以开始数据传输。 |
| **IN_PROGRESS**  | 数据正在传输。                     |
| **COMPLETED**    | 数据传输完成。                     |
| **FAILED**       | 数据传输失败。                     |

#### 2.3 状态机集成流转

1. **合同协商完成后触发数据传输**：一旦合同协商达成并进入 `FINALIZED`，`ContractNegotiationManager` 会通知 `TransferProcessManager` 启动数据传输。
2. **TransferProcessManager** 会创建一个新的 `TransferProcess`，并将其状态设置为 `INITIAL`。
3. **Provisioning 阶段**：`TransferProcessManager` 会调用 **ProvisionManager** 来准备数据的访问权限（例如，生成临时的 S3 凭证、生成 token 等），状态进入 `PROVISIONING`。
4. **数据传输阶段**：准备好后，`TransferProcessManager` 会调用 **DataPlaneSignalingClient** 来触发数据传输的执行，状态进入 `IN_PROGRESS`，数据开始传输。
5. **数据传输完成或失败**：当数据传输完成或发生失败时，回调机制会触发状态转换，最终数据传输进入 `COMPLETED` 或 `FAILED`。

------

### **三、管理器如何协调合同协商与数据传输**

#### 3.1 `ContractNegotiationManager`

`ContractNegotiationManager` 主要负责合同协商的各个阶段的状态推进。它的职责是确保合同从 `REQUESTED` 到 `FINALIZED` 的顺利过渡，一旦合同完成，它会生成 `ContractAgreement` 并通知 **`TransferProcessManager`** 启动数据传输。

关键方法：

- **`initiateNegotiation()`**：启动合同协商。
- **`offerAgreement()`**：提供合同要素。
- **`finalizeAgreement()`**：合同完成，生成 `ContractAgreement`。
- **`completeNegotiation()`**：协商完成，合同最终达成。

#### 3.2 `TransferProcessManager`

`TransferProcessManager` 负责数据传输的整个状态机，它管理数据的传输过程，包括准备阶段、传输阶段和完成阶段。

关键方法：

- **`initiateTransfer()`**：初始化数据传输，创建 `TransferProcess` 并进入 `INITIAL` 状态。
- **`provision()`**：为数据传输准备资源，进入 `PROVISIONING` 状态。
- **`transferData()`**：开始数据传输，进入 `IN_PROGRESS` 状态。
- **`completeTransfer()`**：数据传输完成，进入 `COMPLETED` 状态。

------

### **四、完整的流程总结**

1. **合同协商阶段**：
   - Consumer 和 Provider 通过 `ContractNegotiationManager` 协商合同。
   - 当合同达成一致（进入 `FINALIZED` 状态），生成 `ContractAgreement`。
2. **数据传输阶段**：
   - `ContractNegotiationManager` 通知 `TransferProcessManager` 启动数据传输。
   - `TransferProcessManager` 创建 `TransferProcess`，进入 `INITIAL` 状态。
   - 进入 **Provisioning** 阶段准备数据资源，进入 `PROVISIONING` 状态。
   - 数据传输开始，进入 `IN_PROGRESS` 状态。
   - 数据传输完成或失败，最终进入 `COMPLETED` 或 `FAILED` 状态。

------

# 数据面与控制平面的通信机制

尤其是在数据传输阶段，如何通过 **回调机制** 实现 **事件通知** 和 **数据同步**。这一部分的实现对理解 **异步执行** 和 **回调处理** 至关重要。

### **一、数据面与控制平面通信概述**

数据面和控制平面的通信机制通常包括以下几个部分：

1. **控制平面（Control Plane）**：负责合同协商、数据传输的状态管理、策略应用等。
2. **数据面（Data Plane）**：实际执行数据传输的组件。它负责从数据源获取数据并将其传输到目标，执行数据传输操作。

在 **数据传输阶段**，控制平面与数据面通过事件通知和回调机制进行交互。数据面执行数据传输时，必须定期通知控制平面当前的进度和状态，控制平面根据这些状态做出相应的操作。

------

### **二、控制平面如何调用数据面执行数据操作**

在前面的分析中，我们讲到 **`TransferProcessManager`** 是如何启动数据传输的。当 **合同协商** 完成并生成 `ContractAgreement` 后，控制平面通过 **`TransferProcessManager`** 来初始化数据传输。

#### 2.1 控制平面调用数据面：`DataPlaneSignalingClient`

控制平面通过 **`DataPlaneSignalingClient`** 来与数据面进行通信，主要有以下几个功能：

- 向数据面发送数据传输请求。
- 传递传输状态。
- 等待数据面完成传输后返回状态。

**`DataPlaneSignalingClient`** 是控制平面与数据面之间的通信桥梁，它通过以下方式与数据面进行交互：

1. 启动数据传输（发起请求）。
2. 接收数据传输的状态更新（例如，数据传输开始、进行中、完成或失败）。
3. 接收回调事件（例如，数据传输成功或失败的回调）。

```
public class DataPlaneSignalingClient {

    private final DataPlaneApi dataPlaneApi;

    public DataPlaneSignalingClient(DataPlaneApi dataPlaneApi) {
        this.dataPlaneApi = dataPlaneApi;
    }

    public void startTransfer(DataTransferRequest request) {
        // 发起数据传输请求
        dataPlaneApi.startTransfer(request);
    }

    public void transferCompleted(String transferId) {
        // 接收数据传输完成回调
        dataPlaneApi.transferCompleted(transferId);
    }

    public void transferFailed(String transferId) {
        // 接收数据传输失败回调
        dataPlaneApi.transferFailed(transferId);
    }
}
```

### **三、数据面如何通知控制平面（回调机制）**

数据面通过回调机制将数据传输的状态（如开始、进行中、完成、失败等）传递给控制平面。这个机制是 **异步的**，意味着数据面和控制平面不会在同一线程中同步执行，回调会在数据面执行完相关任务后返回结果。

#### 3.1 异步执行与回调

- **控制平面发起传输请求**：控制平面通过 `DataPlaneSignalingClient` 向数据面发起数据传输请求。这是一个同步的调用，控制平面会等待数据面处理完请求。
- **数据面处理**：数据面开始数据传输后，它会异步执行数据传输任务。这可能包括读取数据、处理数据、上传到目标位置等。
- **回调通知控制平面**：在数据传输完成或发生错误时，数据面通过回调接口通知控制平面传输状态。
  - **`transferCompleted()`**：数据传输成功。
  - **`transferFailed()`**：数据传输失败。

回调机制的关键在于数据面和控制平面之间的松耦合。控制平面只关心事件的结果，而不需要处理数据传输的具体细节。

#### 3.2 回调机制的实现

`DataPlaneSignalingClient` 通过监听 **事件总线** 或 **回调队列** 来接收数据面发出的事件。这些事件通常是 **异步消息**，可以通过 **消息队列** 或 **WebSocket** 等方式进行推送。

例如，控制平面可能会使用 **事件总线（Event Bus）** 来处理数据传输的回调事件：

```
public class DataTransferCallbackHandler {

    private final EventBus eventBus;

    public DataTransferCallbackHandler(EventBus eventBus) {
        this.eventBus = eventBus;
    }

    public void register() {
        eventBus.register(this);
    }

    @Subscribe
    public void onTransferCompleted(TransferCompletedEvent event) {
        // 处理数据传输成功回调
        System.out.println("Transfer completed: " + event.getTransferId());
    }

    @Subscribe
    public void onTransferFailed(TransferFailedEvent event) {
        // 处理数据传输失败回调
        System.out.println("Transfer failed: " + event.getTransferId());
    }
}
```

`EventBus` 在这里充当了 **回调通知中心**，它负责分发来自数据面的回调事件。当数据面发出 `TransferCompletedEvent` 或 `TransferFailedEvent` 时，事件会被发送到监听该事件的控制平面组件（如 `DataTransferCallbackHandler`）。

### **四、回调事件的流转**

回调事件的流转通常会有如下步骤：

1. **数据面执行传输**：数据面根据 `TransferProcess` 对象中的信息执行传输任务。这个过程是异步的，因此数据面会在完成任务后通过回调机制通知控制平面。
2. **数据面通知控制平面**：数据面通过调用 `DataPlaneSignalingClient` 的 `transferCompleted()` 或 `transferFailed()` 方法，通知控制平面当前传输的状态。
3. **控制平面接收回调**：控制平面通过 **事件总线** 或 **回调队列** 接收到数据面发出的回调事件。控制平面根据回调事件更新状态，例如从 `IN_PROGRESS` 转换为 `COMPLETED` 或 `FAILED`。
4. **状态更新**：控制平面根据回调的状态更新 **`TransferProcess`** 的状态，完成数据传输的生命周期管理。

------

### **五、回调机制与异步执行的关键优势**

1. **松耦合**：数据面和控制平面不直接依赖于彼此的执行，控制平面只关注事件结果，而不需要关心数据传输的具体实现。
2. **异步执行**：数据传输是异步的，数据面可以在后台执行数据传输任务，而控制平面则通过回调机制接收通知，避免了同步执行的阻塞。
3. **可扩展性**：回调机制使得数据面和控制平面之间的通信方式更加灵活，能够处理高并发的请求，并适应不同的传输协议和策略。
