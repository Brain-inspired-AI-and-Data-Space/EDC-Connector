

# sovity edc-ce 本地部署后完整操作文档（Local Demo）

> 适用范围：**sovity `edc-ce` Local Demo**
>
> 目标：在本地已经成功部署 `edc-ce` 后，完成一遍从 **Provider 提供数据** 到 **Consumer 浏览目录、协商合同并发起传输** 的完整演示流程。

---

## 1. 文档说明

本操作文档基于 sovity 官方 `edc-ce` 文档整理，适用于 **Local Demo** 场景。  
官方文档说明，Local Demo 会启动两套本地连接器，用于在同一台机器上演示完整的数据交换流程。  
默认入口包括：

- Provider UI：`http://localhost:11000`
- Consumer UI：`http://localhost:22000`

默认 Management API 地址包括：

- Provider Management API：`http://localhost:11000/api/management`
- Consumer Management API：`http://localhost:22000/api/management`

默认 Management API Key：

- `SomeOtherApiKey`

---

## 2. 前置条件（docker）

在执行本流程前，请先确认：

1. 已成功执行 `docker compose up -d`
2. 所有容器正常运行
3. 浏览器可以访问：
   - `http://localhost:11000`
   - `http://localhost:22000`

如果需要清楚全部数据重新启动，执行以下命令：

```
docker compose down -v --remove-orphans
```

确认没有残留容器：

```
docker compose ps
```

重新启动：

```
docker compose up -d
```

如果你想确保镜像和配置都重新加载，可以用：

```
docker compose up -d --build
```

---

## 3. 检查服务状态

在部署目录执行：

```bash
docker compose ps
docker compose logs -n 100
```

如果所有主要服务均处于 `Up` 状态，则可继续。

---

## 4. 完整业务流程概览

本次演示流程分为两部分：

### Provider 侧
1. 创建 Asset
2. 创建 Policy
3. 发布 Data Offer

### Consumer 侧
4. 浏览 Catalog
5. 发起 Contract Negotiation
6. 查看 Contract
7. 发起 Transfer

---

# 5. Provider 侧操作（`http://localhost:11000`）

## 5.1 登录 Provider UI

浏览器打开：

```text
http://localhost:11000
```

进入 Provider Connector 的 Dashboard 页面。

---

## 5.2 Create data offer

使用最新的 Connector 前端，无需像以前那样先创建 `<data_repository>` `Asset`，再创建 `<data_repository> `Policy，最后再`Data Offer`创建连接两者的 `<data_repository>`，而是可以使用一种全新的流程一次性完成所有New Data Offer操作。该流程首先定义资产背后的数据源，然后填写资产详细信息，例如资产名称和描述等元数据。最后，您可以选择资产在数据空间中的发布方式。填写完所有信息并Publish`点击按钮后，系统将创建一个数据发布请求，并根据指定的策略将资产发布到数据空间。根据连接的数据空间和 Connector 的版本，字段可能会有所不同，有时可能需要填写其他字段，或者会显示其他选项供您选择。

![](operationFig\8.png)

可以通过publishing条目选择policy的限制条件

| Publishing Mode                            | 含义                                                         |
| ------------------------------------------ | ------------------------------------------------------------ |
| **Publish unrestricted**                   | 数据可以被所有数据空间参与方访问，没有限制。                 |
| **Publish restricted**                     | 数据发布时附带限制规则，比如只有特定参与方才能访问。         |
| **Create asset only (without data offer)** | 只创建数据资产，不立即对外发布。你可以之后再创建 Data Offer。 |

## 5.2 Create Asset

使用 Eclipse 数据空间组件连接器 (EDC 连接器) 提供数据的第一步是创建资产。首先单击选项卡，**Assets**然后单击**New Asset**右上角的按钮。

![](operationFig\1.png)

您将被引导至`Create New Asset`流程对话框，您可以在其中定义资产本身的所有相关数据以及数据源的其他必要数据（如果数据及其数据源已存在）。为此，您可以选择两种报价类型，具体取决于数据源是否已存在，或者是否仅在请求时才执行。

![](operationFig\2.png)

首先，必须确定新数据方案的数据源是现有的还是需要请求才能获取。根据您的选择，后续对话框中将显示不同的字段。对于现有的数据源，必须输入相关数据，包括数据源类型、传输启动时使用的方法、URL 以及与数据源相关的其他必要机制。

可以通过提供 HTTP 方法和端点 URL 来连接来自任何 REST API 端点的数据。根据您的数据源，可能还需要进行其他设置：

- Query Params：用于定义 API 请求的查询参数（URL 中 `?key=value` 的部分），可以通过 “Add Pair” 添加键值对，这些参数会随请求发送。
- Additional Headers：用于添加 HTTP 请求头（Headers），例如 `Authorization`、`Content-Type` 等。
- **Enable Method Parameterization**：允许消费者动态指定 HTTP 方法（GET、POST、PUT 等）。
- **Enable Path Parameterization**：允许消费者在基础路径（Base Path）后追加路径。
- **Enable Query Params Parameterization**：允许消费者在请求时添加额外查询参数。
- **Enable Body Parameterization**：允许消费者在请求时提供请求体（Body）。
- Authentication：用于设置该资产请求的身份验证方式。

随后输入asset的总体信息，点击create后可以看到已发布的资产.

![](operationFig\3.png)



如果 UI 不支持您所需的数据地址类型，您可以提供自定义数据地址配置 JSON。

![](operationFig\4.png)

Example:

{% code title="JSON" overflow="wrap" lineNumbers="true" %}

```
{
  "properties": {
    "type": "HttpData",
    "baseUrl": "https://webhook.site/86b9b7e6-eb27-4c5f-b7e5-336d5f157f15"
  }
}
```

**说明**

Asset 表示要对外共享的数据资源。  
如果选择“Available (with data source)”，则需要同时配置实际数据源。

---

## 5.3 create a Policy

在左侧导航进入 **Policies** 页面，默认情况下，连接器启动时已创建一条策略，该策略始终有效，可用于向同一数据空间中的任何其他连接器提供数据服务，且无任何限制。要创建新策略，请单击`New policy`按钮。

有两种不同类型的保单可供选择：

- **消费者参与者 ID**：根据此策略，数据服务只能根据参与者 ID 提供给特定的连接器。此时必须知道此参与者 ID。
- **时间限制**：此策略可用于指定可以访问或协商数据服务的时间段。

政策之间也可以相互关联。换句话说，根据选择的不同，可能需要同时满足多项政策，以及相互独立地并行执行不同的政策.

![](operationFig\5.png)

**说明**

Policy 用于定义数据访问和使用规则。  
在后续发布 Data Offer 时，需要把 Policy 绑定到 Asset。

**预期结果**

创建成功后，该 Policy 会出现在 Policy 列表中。

![](operationFig\6.png)

---

## 5.4 Publish a Data Offer

要让您的数据对其他数据空间参与者可用，您必须定义一个`Data Offer`。`Data Offer`将策略链接到您的资产，然后在您的目录中提供这些资产，以便其他参与者可以查询这些资产，并根据设置的策略，在其目录浏览器中查看这些资产并启动合同谈判。

![](operationFig\7.png)

**填写内容**

- Data Offer ID：自定义唯一值，例如 `demo-offer-001`
- Asset：选择刚才创建的 `Test Asset`
- Access Policy：选择刚才创建的 Policy
- Contract Policy：选择刚才创建的 Policy（演示时可先复用同一个）

**预期结果**

发布成功后，该 Data Offer 会出现在当前 Connector 的 Offer 列表中。

---

# 6. Consumer 侧操作（`http://localhost:22000`）

## 6.1 登录 Consumer UI

浏览器打开：

```text
http://localhost:22000
```

进入 Consumer Connector 的 Dashboard 页面。

---

## 6.2 Finding Data Offers Using the Catalog Browser

要查找和使用来自其他连接器的数据，您首先需要获取要探索其产品的连接器的连接器端点。请确保两个连接器都注册在同一数据空间身份提供商

找到要探索的连接器的端点后，点击选项卡`Catalog Browser`，然后在搜索栏中输入 URL `Connector Endpoint`。您的连接器将自动显示其他连接器中所有可用的产品/服务，前提是您的连接器有权查看这些产品/服务。此评估会`Access Policy`在您请求目录时，由请求的连接器进行。

![](operationFig\9.png)

**预期结果**

如果 Access Policy 配置允许，则可以看到 Provider 刚才发布的 Data Offer。

![](operationFig\10.png)

一旦您确定了感兴趣的资产，您可以单击其所在行，将显示如下所示的详细信息对话框。

![](operationFig\11.png)

如果某个资产有多个不同的数据报价，它们将显示在选项卡下的详细信息视图中`Contract Offers`。

点击`Negotiate`即可开始协商所需数据套餐的合同。

![](operationFig\12.png)

勾选复选框后，点击按钮`Confirm`，已正式启动合同谈判流程。

![](operationFig\13.png)

---

## 6.3 Data Transfer Process

要将数据传输到所需的数据接收器，请导航至该`Contracts`页面。此页面显示您的所有合同协议，包括数据消费合同和数据提供合同。

![](operationFig\14.png)



![](operationFig\15.png)

点击`Transfer`定义数据接收器属性。

![](operationFig\16.png)

支持两种数据接收器类型：

1. `Custom Datasink Config (JSON)`
2. `REST-API Endpoint`

1. 转移到 REST API 端点

要将数据传输到 REST API 端点，请选择 HTTP 方法并提供数据接收器的 URL。您还可以添加其他标头，例如用于身份验证的标头。

2. 将数据传输到自定义数据接收器配置（JSON）

对于 UI 未直接支持的更高级的数据接收器端点，您可以以 JSON 格式输入数据接收器属性。

**示例：** {% code title="JSON" overflow="wrap" lineNumbers="true" %}

```
{
  "properties": {
    "type": "HttpData",
    "baseUrl": "https://webhook.site/86b9b7e6-eb27-4c5f-b7e5-336d5f157f15"
  }
}
```

点击**Initiate transfer** 开始传输数据

![](operationFig\17.png)

## 6.4 合同终止

服务提供商和消费者均可选择终止现有合同。要发起终止，只需在“合同”区域点击现有合同（无论其是服务提供商还是消费者），然后在“合同详情”选项卡的左下角点击红色按钮`Terminate`即可

![](operationFig\18.png)

点击按钮后`Terminate`，会弹出一个新对话框，您必须在其中`Detailed Reason`指定终止原因。此外，您还必须勾选一个复选框，以确认您了解终止合同的后果`I understand the consequences of terminating a contract`。此时，合同尚未终止；只有当您填写完所有必填字段并`Terminate`点击此对话框右下角的按钮后，终止流程才会正式启动。

如果合同已终止，无论终止方是谁，双方都可以在`Contracts`状态图标旁边的区域清楚地看到这一点，该图标会被红色划掉。此外，`Contracts`通过`Terminated`标签页还可以查看一个单独的子区域，该子区域仅列出已终止的合同。

---

# 7. Vault Secrets UI

**Vault Secrets ** 的作用主要是集中管理和安全存储敏感信息（Secrets），例如：API Key、数据库密码、证书、令牌等。它本质上就是一个 **可视化的秘密管理界面**。

