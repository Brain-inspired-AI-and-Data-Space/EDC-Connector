# Eclipse EDC（Eclipse Dataspace Connector）本地部署与跑通指南

> 目标：在 **Windows** 环境下，使用 **WSL2 + Docker Desktop**，拉起 Eclipse EDC 的 **Provider/Consumer** 两个连接器，并跑通 **Asset/Policy/ContractDefinition -> Contract Negotiation -> TransferProcess** 的端到端流程。  
> 方案：直接复用 Eclipse 官方 **Samples** 中可运行的 `advanced-01-open-telemetry` 示例（包含 provider、consumer、Jaeger、Prometheus）。

---

## 1. 环境准备

### 1.1 必装
- Windows 10/11
- **Docker Desktop**（启用 WSL2 backend）
- **WSL2**（建议 Ubuntu 22.04+）
- Git

### 1.2 建议装（在 WSL2 内）
```bash
sudo apt-get update
sudo apt-get install -y curl jq
```

---

## 2. 解决 WSL 的 Docker 凭据助手报错

典型错误：
- `docker: error getting credentials ... docker-credential-desktop.exe: exec format error`

原因：
- WSL 的 `~/.docker/config.json` 里配置了 `"credsStore": "desktop"`，导致在 Linux 中尝试执行 Windows 的 `.exe`。

修复（WSL 内执行）：
```bash
mkdir -p ~/.docker
cp -a ~/.docker/config.json ~/.docker/config.json.bak 2>/dev/null || true
cat > ~/.docker/config.json <<'EOF'
{}
EOF
```

验证：
```bash
docker version
docker pull hello-world
```

---

## 3. 获取示例代码（WSL2 内执行）

> **建议把仓库放在 WSL 的 Linux 文件系统（`~/`）**，避免 `/mnt/c`、`/mnt/e` 的性能/权限问题。

```bash
cd ~
git clone https://github.com/eclipse-edc/Samples.git
cd Samples
```

---

## 4. 构建示例运行时（只用 Docker，不依赖本机 JAVA）

> 你之前遇到的 `JAVA_HOME is not set`，直接用本步骤绕过。

在 `Samples` 根目录执行：
```bash
docker run --rm -v "$PWD":/workspace -w /workspace gradle:8-jdk17   ./gradlew :advanced:advanced-01-open-telemetry:open-telemetry-runtime:build
```

若 `gradlew` 没有执行权限：
```bash
chmod +x gradlew
```

---

## 5. 启动 Provider/Consumer（docker compose）

```bash
docker compose -f advanced/advanced-01-open-telemetry/docker-compose.yaml up -d --build
docker compose -f advanced/advanced-01-open-telemetry/docker-compose.yaml ps
```

![image-20260304150321443](C:\Users\Think\AppData\Roaming\Typora\typora-user-images\image-20260304150321443.png)

![image-20260304150350671](C:\Users\Think\AppData\Roaming\Typora\typora-user-images\image-20260304150350671.png)

### 5.1 端口信息（后续写接口文档必用）

| 角色 | Management API | Protocol API（DSP） | Control API |
|---|---|---|---|
| Provider | `http://localhost:19193/management` | `http://localhost:19194/protocol` | `http://localhost:19192/control` |
| Consumer | `http://localhost:29193/management` | `http://localhost:29194/protocol` | `http://localhost:29192/control` |

Management API 鉴权 Header：
- `X-Api-Key: password`

查看日志：
```bash
docker logs -n 120 provider
docker logs -n 120 consumer
```

---

## 6. 常见问题：拉基础镜像超时（Docker Hub auth/token 超时）

错误示例：
- `failed to fetch anonymous token ... auth.docker.io ... i/o timeout`
- `DeadlineExceeded` / `dial tcp ...:443: i/o timeout`

### 6.1 快速验证（WSL 内）
```bash
docker pull eclipse-temurin:25-alpine
```

### 6.2 解决方式（推荐）
在 Docker Desktop → **Settings → Docker Engine** 增加镜像加速器（示例）并重启 Docker Desktop：

```json
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://mirror.ccs.tencentyun.com",
    "https://hub-mirror.c.163.com"
  ]
}
```

如果你在公司网络：Docker Desktop → **Settings → Proxies** 填写公司 HTTP/HTTPS 代理，并重启。

完成后重新构建启动：
```bash
docker compose -f advanced/advanced-01-open-telemetry/docker-compose.yaml down -v --remove-orphans
docker builder prune -f
docker compose -f advanced/advanced-01-open-telemetry/docker-compose.yaml up -d --build
```

---

## 7. 端到端跑通（核心流程）

> 所有命令都在 `Samples` 根目录执行。  
> 若你不在根目录，会找不到 `@xxx.json` 文件。

### 7.1 Provider：创建 Asset
```bash
curl -H "X-Api-Key: password"   -d @transfer/transfer-01-negotiation/resources/create-asset.json   -H 'content-type: application/json'   http://localhost:19193/management/v3/assets   -s | jq
```

![image-20260304194329015](C:\Users\Think\AppData\Roaming\Typora\typora-user-images\image-20260304194329015.png)

### 7.2 Provider：创建 Policy

```bash
curl -H "X-Api-Key: password"   -d @transfer/transfer-01-negotiation/resources/create-policy.json   -H 'content-type: application/json'   http://localhost:19193/management/v3/policydefinitions   -s | jq
```

![image-20260304194644793](C:\Users\Think\AppData\Roaming\Typora\typora-user-images\image-20260304194644793.png)

### 7.3 Provider：创建 Contract Definition

```bash
curl -H "X-Api-Key: password"   -d @transfer/transfer-01-negotiation/resources/create-contract-definition.json   -H 'content-type: application/json'   http://localhost:19193/management/v3/contractdefinitions   -s | jq
```

![image-20260304194747405](C:\Users\Think\AppData\Roaming\Typora\typora-user-images\image-20260304194747405.png)

------

## 8. Consumer：请求 Dataset（获取 Offer ID）

```bash
curl -H "X-Api-Key: password"   -H "Content-Type: application/json"   -d @advanced/advanced-01-open-telemetry/resources/get-dataset.json   -X POST "http://localhost:29193/management/v3/catalog/dataset/request"   -s | jq
```

![image-20260304211912360](C:\Users\Think\AppData\Roaming\Typora\typora-user-images\image-20260304211912360.png)

![image-20260304211931896](C:\Users\Think\AppData\Roaming\Typora\typora-user-images\image-20260304211931896.png)

从输出中复制：

- `odrl:hasPolicy/@id`（后续协商需要用）
-  "@id": "MQ==:YXNzZXRJZA==:M2U1OWVlMTUtMTU0NC00Zjg2LWI2M2ItYzAxNDI2ZThlYWEx",

---

## 9. Consumer：发起 Contract Negotiation

### 9.1 把 Offer ID 填进 negotiate-contract.json
编辑：
- `advanced/advanced-01-open-telemetry/resources/negotiate-contract.json`

```bash
nano advanced/advanced-01-open-telemetry/resources/negotiate-contract.json
```

把文件里对应的 offer id 字段替换为第 8 步获得的 `odrl:hasPolicy/@id`。

![image-20260304212417582](C:\Users\Think\AppData\Roaming\Typora\typora-user-images\image-20260304212417582.png)

### 9.2 发起协商
```bash
curl -H "X-Api-Key: password"   -H "Content-Type: application/json"   -d @advanced/advanced-01-open-telemetry/resources/negotiate-contract.json   -X POST "http://localhost:29193/management/v3/contractnegotiations"   -s | jq
```

![image-20260305162511474](C:\Users\Think\AppData\Roaming\Typora\typora-user-images\image-20260305162511474.png)

记录返回的：

- `id`（记为 `NEG_ID`）
-  "@id": "f0b0d3fe-f15c-4b1e-86ac-018c89fdc163",

### 9.3 查询协商状态，直到出现 Agreement
```bash
NEG_ID='<把id填这里>'
curl -H 'X-Api-Key: password'   "http://localhost:29193/management/v3/contractnegotiations/$NEG_ID"   -s | jq

curl -H 'X-Api-Key: password' \
  "http://localhost:29193/management/v3/contractnegotiations/f0b0d3fe-f15c-4b1e-86ac-018c89fdc163" \
  -s | jq

```

![image-20260305162616983](C:\Users\Think\AppData\Roaming\Typora\typora-user-images\image-20260305162616983.png)

从返回里找到 **contract agreement id**（字段名可能略有差异，但会出现 agreement 的 id）。

35516229-d144-4d5c-8b71-14139501dfdc

---

## 10. Consumer：发起 TransferProcess（数据传输）

### 10.1 把 Agreement ID 填进 start-transfer.json
编辑：
- `advanced/advanced-01-open-telemetry/resources/start-transfer.json`

```bash
nano advanced/advanced-01-open-telemetry/resources/start-transfer.json
```

把其中的 `contractAgreementId`（或类似字段）改为第 9 步得到的 agreement id。

![image-20260305164319510](C:\Users\Think\AppData\Roaming\Typora\typora-user-images\image-20260305164319510.png)

### 10.2 发起 Transfer
```bash
curl -H "X-Api-Key: password"   -H "Content-Type: application/json"   -d @advanced/advanced-01-open-telemetry/resources/start-transfer.json   -X POST "http://localhost:29193/management/v3/transferprocesses"   -s | jq
```

返回 transfer process id 且状态推进，即表示跑通。

![image-20260305164458389](C:\Users\Think\AppData\Roaming\Typora\typora-user-images\image-20260305164458389.png)

---

## 11. 可视化验证（可选，但很建议）

- Jaeger UI（链路追踪）：`http://localhost:16686`

![image-20260305165107146](C:\Users\Think\AppData\Roaming\Typora\typora-user-images\image-20260305165107146.png)

- Prometheus（指标）：`http://localhost:9090`

---

## 12. 停止与清理

停止并保留数据卷：
```bash
docker compose -f advanced/advanced-01-open-telemetry/docker-compose.yaml down
```

停止并清理数据卷：
```bash
docker compose -f advanced/advanced-01-open-telemetry/docker-compose.yaml down -v --remove-orphans
```

---

## 13. 交付给平台团队的最小接口信息（建议写入接口文档）

- Provider Management：`http://<你机器IP>:19193/management`
- Provider DSP/Protocol：`http://<你机器IP>:19194/protocol`
- Consumer Management：`http://<你机器IP>:29193/management`
- Consumer DSP/Protocol：`http://<你机器IP>:29194/protocol`
- Management API Header：`X-Api-Key: password`

> 注意：对方团队如果不跑 consumer，只对接你们 connector，则至少需要 Provider 的 Management + DSP/Protocol 两个地址。

---

## 14. 故障定位最短清单（出问题时直接贴这几段日志即可定位）

```bash
docker compose -f advanced/advanced-01-open-telemetry/docker-compose.yaml ps
docker logs -n 200 provider
docker logs -n 200 consumer
```

以及失败的那条 curl 的完整输出（含 HTTP 状态码）。

------

## 15.结构分析与功能扩展

### 1. **Docker Compose 配置分析**

`docker-compose.yaml`这是运行整个系统的核心配置文件。具体来说，它包括了以下服务：

- **`provider`**：
  - 作为数据提供者，它负责暴露和共享数据。
  - 它会监听 `19193` 端口并提供管理 API 和协议接口（如 `POST /assets` 和 `POST /contractdefinitions`）。
- **`consumer`**：
  - 作为数据消费者，它请求并使用提供者的共享数据。
  - 监听 `29193` 端口，接收管理请求并发起数据流通。
- **`jaeger`** 和 **`prometheus`**（可选）：
  - 用于监控和追踪数据流通过程的工具，帮助你分析每个请求的延迟和性能。

这个文件的核心功能是配置不同服务之间的网络连接和数据流通路径

------

### 2. **Provider 与 Consumer 交互分析**

#### 1. **Provider**

- **提供数据**：Provider 创建和管理数据资产（如 `asset`），并暴露给 Consumer。通过 `POST /assets` 接口，Consumer 可以请求数据。
- **管理合约**：Provider 创建合约定义（`contract-definition`）并提供给 Consumer 进行协商。通过 `POST /contractdefinitions` 接口，Consumer 会创建合约并进行合约协商。
- **数据传输**：Provider 负责管理实际的数据流（TransferProcess），通过 `POST /transferprocesses` 实现数据的物理传输。

#### 2. **Consumer**

- **请求数据**：Consumer 向 Provider 请求共享数据（`GET /assets`），并通过合约进行数据协商（`POST /contractnegotiations`）。
- **合约协商**：Consumer 使用提供的合约模板和策略进行协商（`POST /contractnegotiations`），确保数据共享满足特定的合规要求。
- **数据接收**：当合约协商成功后，Consumer 发起数据传输过程（`POST /transferprocesses`），并监控传输状态。

------

### 3. **核心功能分析**

#### 1. **Asset、Policy 和 Contract Definition**

- 这些是 **Provider** 和 **Consumer** 之间进行数据共享和交换的核心对象。
- **Asset**：表示共享的数据或资源，Consumer 可以请求获取。
- **Policy**：定义对数据使用的规则，例如“数据只能在特定时间内使用”。
- **Contract Definition**：定义在特定协议下如何共享数据，确保所有参与方都达成一致并遵守约定。

#### 2. **Contract Negotiation 和 Transfer Process**

- **Contract Negotiation**：定义了如何与数据提供方协商合约（例如价格、数据访问规则等），包括如何达成合意并生成合约。
- **Transfer Process**：在合约达成后，触发数据的传输过程。这一过程需要确保数据在双方系统中正确、安全地传输。

#### 3. **监控与追踪（OpenTelemetry）**

- 通过集成 OpenTelemetry，你可以监控每一个请求的执行状态、延迟和性能，尤其是在多方协作的数据空间中，能够有效地帮助你了解每个请求的流动情况。

------

### 4. **如何执行流程**

#### 1. **启动容器并验证**

- 使用 `docker compose up -d` 启动容器并查看它们是否正常运行。
- 使用 `docker compose ps` 验证容器状态。

#### 2. **在 Provider 端创建资源**

- 通过 `POST /assets` 创建数据资产。
- 通过 `POST /policydefinitions` 创建策略。
- 通过 `POST /contractdefinitions` 创建合约定义。

#### 3. **在 Consumer 端请求数据并协商合约**

- 通过 `POST /catalog/dataset/request` 请求数据集。
- 通过 `POST /contractnegotiations` 发起合约协商。
- 通过 `POST /transferprocesses` 发起数据传输。

#### 4. **查看监控**

- 使用 **Jaeger** 或 **Prometheus** 来查看每个步骤的性能指标。

------

### 5.扩展功能代码模块

扩展代码的编写和部署通常会涉及以下几个部分：

- **自定义连接器实现**
- **协议和合约协商逻辑的修改**
- **集成其他外部服务或工具**

以下是扩展功能时需要关注的主要代码位置和模块：

#### 1 **`edc-connector` 代码库**

Eclipse EDC 的核心连接器功能会集中在 **`edc-connector`** 代码库中。这里包含了很多与数据共享和交换相关的核心逻辑，如：

- **合约协商**（Contract Negotiation）
- **数据传输流程**（Data Transfer）
- **资产管理和发布**（Asset Management）

##### 代码位置

1. **合约协商和策略引擎**
   - 你可以扩展合约协商的逻辑，修改协议、合约类型或增加自定义的协商步骤。
   - **相关代码**：`edc-connector` 的 `contract` 相关模块，主要在 `core/` 和 `spi/` 下。
2. **Asset 和 Policy 管理**
   - 资产（Asset）和策略（Policy）的创建、更新、查询等功能的代码也可以在此扩展。
   - **相关代码**：`edc-connector` 中的 `asset`、`policy` 模块，通常在 `management` 目录下。
3. **数据传输（Transfer Process）**
   - 如果你需要增加或修改数据传输的逻辑，例如添加新的协议、扩展传输的生命周期管理，你需要修改数据传输的代码。
   - **相关代码**：`edc-connector` 中的 `transfer` 模块。
4. **SPI 插件（可扩展点）**
   - Eclipse EDC 提供了 **SPI（Service Provider Interface）** 插件系统，允许你根据业务需求添加自定义模块（例如新的合约协商策略、数据传输协议等）。
   - **相关代码**：`edc-connector` 下的 `spi/` 目录。

#### 2 **Docker 服务与容器配置**

如果你希望修改连接器容器的配置（例如容器的启动参数、网络配置等），则需要调整 **`docker-compose.yaml`** 文件：

- 在 `docker-compose.yaml` 中，你可以定义不同服务的端口、网络、环境变量等。
- 例如，你可以在 **Provider** 或 **Consumer** 容器中增加自定义的环境变量来配置新的协议或策略。

一旦你编写了扩展的代码（如自定义的协议、合约、策略等），你需要将它们注册到连接器中。通常，Eclipse EDC 会在启动时通过 **SPI** 自动加载这些扩展。如果你在 `docker-compose.yaml` 中定义了额外的环境变量或配置，它们将被传递到连接器中。

例如，如果你扩展了合约协商，你可以在连接器的配置中指定你的自定义实现：

```
services:
  provider:
    image: edc-provider
    environment:
      - EDC_CONTRACT_NEGOTIATION_IMPL=org.example.CustomContractNegotiation
```

这样，连接器就会根据环境变量加载并使用你自定义的合约协商实现。

#### 3 **服务扩展与集成**

如果你需要集成其他外部服务（如数据库、身份认证、消息队列等），可以在容器配置中添加新的服务。例如：

- **添加 Kafka 或 Pulsar**：如果你需要实现实时数据流或消息传递，可以将消息队列服务（如 Kafka 或 Pulsar）添加到 Docker Compose 文件中，并且在连接器代码中集成消息队列的生产者/消费者。
- **集成身份认证系统**：例如，集成 OAuth2 或 DID（Decentralized Identity）来管理用户身份。

------

## 16. 如何扩展 `edc-connector` 代码

### 1. **`edc-connector` 模块的位置**

`edc-connector` 模块是 **Eclipse EDC 框架** 的一个重要组成部分，它通常位于 **Eclipse EDC 的 GitHub 仓库** 中。你可以通过以下方式访问和下载它：

- GitHub 仓库：[https://github.com/eclipse-edc/Connector](https://github.com/eclipse-edc/Connector?utm_source=chatgpt.com)

在该仓库中，你将找到多个重要模块，包括：

- **`core`**：实现了数据空间连接器的核心功能。
- **`spi`**：允许你扩展和定制连接器的插件接口（如协议、合约、策略等）。
- **`dataspace-protocol`**：定义了数据交换的协议及其实现。

------

### 2. **如何扩展 `edc-connector` 代码**

如果你想扩展 **`edc-connector`**，你需要在本地开发环境中进行以下操作：

#### 2.1 **克隆 Eclipse EDC 仓库**

首先，克隆 **Eclipse EDC 的 GitHub 仓库**：

```
git clone https://github.com/eclipse-edc/Connector.git
cd Connector
```

#### 2.2 **创建自定义模块**

在 `edc-connector` 中，你可以创建和修改以下几个部分来进行扩展：

- **合约协商（Contract Negotiation）**：扩展或修改合约协商的逻辑，增加新的验证、审批或决策步骤。
- **数据传输（Transfer Process）**：实现新的传输协议（例如，基于 gRPC 或 Kafka 的协议）。
- **策略（Policy）和策略引擎（Policy Engine）**：增加新的数据使用策略或扩展现有的 ODRL 策略。

#### 2.3 **修改和扩展代码**

1. **扩展合约协商逻辑**：
    在 `edc-connector` 中，找到与 **合约协商（Contract Negotiation）** 相关的模块，继承相关的接口并修改合约的逻辑。

   例如，假设你想自定义一个合约协商过程，你可以在 **contract negotiation** 部分编写扩展类。

   示例代码：

   ```
   public class CustomContractNegotiation extends AbstractContractNegotiation {
   
       @Override
       public void initiateNegotiation() {
           // 扩展合约协商的逻辑，增加新的策略
       }
   
       @Override
       public void validate() {
           // 验证新的合约条件或合规要求
       }
   }
   ```

2. **扩展数据传输协议**：
    `edc-connector` 支持自定义数据传输协议。如果你希望使用如 **gRPC** 或 **Kafka** 之类的协议进行数据传输，你需要实现协议的相关类，并在连接器中注册它。

   示例代码：

   ```
   public class GRPCTransferProcess extends AbstractTransferProcess {
       @Override
       public void execute() {
           // 实现基于 gRPC 的数据传输
       }
   }
   ```

3. **自定义策略引擎**：
    扩展 ODRL 或其他策略引擎，创建新的策略模型或者修改现有策略的执行逻辑。

   示例代码：

   ```
   public class CustomODRLPolicy extends ODRLPolicy {
       @Override
       public boolean evaluate() {
           // 根据自定义的策略规则执行验证
           return checkConditions();
       }
   }
   ```

------

## 17. **构建运行本地实例和调用自定义代码**

### 1. **连接器如何调用扩展的功能**

连接器会通过配置和接口调用扩展功能。Eclipse EDC 中的连接器通常会在以下几个地方调用你的扩展代码：

1. **启动时加载 SPI 插件**：

   - 在连接器启动时，系统会加载 **SPI 插件**，并将这些插件注册到框架中。你在扩展功能时需要通过 SPI 接口定义你自定义的服务。

   例如，假设你自定义了一个合约协商逻辑，你需要创建一个 SPI 接口实现类并注册到框架中，连接器在初始化时会自动加载并调用这些扩展。

2. **运行时调用扩展功能**：

   - 在合约协商、资产创建、数据传输等关键流程中，连接器会根据定义的逻辑触发对应的扩展功能。例如，当 **Consumer** 请求数据时，它会根据请求参数和策略来选择合适的合约模板。

3. **自定义协议和数据处理**：

   - 如果你扩展了数据传输协议或合约内容，连接器会根据你的自定义实现使用新的协议、策略或者数据处理方式。例如，假如你定义了一个新的协议类型（比如，基于 gRPC 的协议），连接器会根据配置使用你定义的协议。

------

### 2. **确保自定义功能已正确构建并启动**

1. **构建 `edc-connector`**：确保你已经成功构建了你的自定义功能。如果你做了修改，首先构建 `edc-connector`：

   ```
   ./gradlew build
   ```

2. **将修改后的代码包含到 Docker 镜像中**：
    如果你在 `edc-connector` 代码中进行了修改（例如更改了合约协商的代码），你需要确保这些修改被打包并部署到 Docker 镜像中。根据你的构建方式，确保自定义功能已被编译并包含在 Docker 镜像里。

   假设你用 Docker 构建镜像，你可以通过以下命令重新构建镜像：

   ```
   docker build -t custom-edc-connector .
   ```

3. **测试自定义功能**：在本地验证自定义功能是否能够正常运行。可以通过在本地运行 `edc-connector` 容器，调用 API 来测试自定义功能是否按预期工作。

   ```
   docker run -p 8080:8080 custom-edc-connector
   ```

   然后通过 Postman 或 `curl` 来测试你自定义的 API 是否能够正确响应。

------

### 3. **修改 `advanced` 示例中的 Docker Compose 配置**

现在你已经在 `edc-connector` 中做了自定义扩展，接下来你需要修改 `advanced-01-open-telemetry` 示例中的 **`docker-compose.yaml`** 配置文件，使其能够使用你修改后的连接器。

1. **更新 Docker Compose 配置**

   打开 `advanced-01-open-telemetry/docker-compose.yaml`，找到连接器服务部分，确保它使用你构建的 **自定义 Docker 镜像**。例如：

   ```
   services:
     provider:
       image: custom-edc-connector  # 这里使用你自定义的镜像
       container_name: edc-provider
       environment:
         - EDC_CONNECTOR_NAME=provider
         - EDC_VAULT_URL=http://vault:8200
         - EDC_DATASOURCE_DEFAULT_URL=jdbc:postgresql://postgres:5432/edc
       ports:
         - "19193:19193"
         - "19194:19194"
   
     consumer:
       image: custom-edc-connector  # 同样使用自定义的镜像
       container_name: edc-consumer
       environment:
         - EDC_CONNECTOR_NAME=consumer
         - EDC_VAULT_URL=http://vault:8200
         - EDC_DATASOURCE_DEFAULT_URL=jdbc:postgresql://postgres:5432/edc
       ports:
         - "29193:29193"
         - "29194:29194"
   ```

   这样，`provider` 和 `consumer` 容器都会使用你自定义的 Docker 镜像，并加载你修改后的连接器代码。

2. **暴露合约协商和数据传输功能**

   确保容器端口与 **`docker-compose.yaml`** 中的端口配置相匹配。你需要确保 `provider` 和 `consumer` 容器上的 **`protocol`** 和 **`management API`** 端口已暴露，供对接使用：

   - **Provider 端口：**
     - 管理 API（`/management`）
     - 协议 API（`/protocol`）
   - **Consumer 端口：**
     - 管理 API（`/management`）
     - 协议 API（`/protocol`）

   确保你使用正确的端口来进行 API 调用。
