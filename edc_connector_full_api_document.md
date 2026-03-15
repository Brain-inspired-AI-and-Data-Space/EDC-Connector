
# Eclipse EDC 连接器接口文档

## 1. 概述

本文档描述了用于 **数据流通平台的 EDC 连接器** 的 API 接口。所有接口均要求 API 密钥认证。

## 2. 安全性

所有接口请求必须带上有效的 **API Key**。请求头中应包含：

- **X-Api-Key**: `password`（或根据实际生成的 API key）

------

## 3. `AssetApiV3` 接口概览

https://github.com/eclipse-edc/Connector/blob/main/extensions/control-plane/api/management-api/asset-api/src/main/java/org/eclipse/edc/connector/controlplane/api/management/asset/v3/AssetApiV3.java

https://github.com/eclipse-edc/Connector/blob/main/extensions/control-plane/api/management-api/asset-api/src/main/java/org/eclipse/edc/connector/controlplane/api/management/asset/v3/AssetApiV3Controller.java

### 3.1方法与请求逻辑：

| 方法              | 描述                         | 请求字段/参数              | 返回/HTTP 响应                                    |
| ----------------- | ---------------------------- | -------------------------- | ------------------------------------------------- |
| `createAssetV3`   | 创建资产（包括 dataAddress） | `JsonObject asset`         | `JsonObject`（包含 Asset Id + 时间戳）200/400/409 |
| `requestAssetsV3` | 根据条件查询资产             | `JsonObject querySpecJson` | `JsonArray` 200/400                               |
| `getAssetV3`      | 获取指定 ID 的资产           | `String id`                | `JsonObject` 200/400/404                          |
| `removeAssetV3`   | 删除指定资产（如果可删除）   | `String id`                | **void** 204/400/404/409                          |
| `updateAssetV3`   | 更新资产                     | `JsonObject asset`         | **void** 204/400/404                              |

| Java 方法                                   | REST 映射                            | 输入类型                 | 输出类型     |
| ------------------------------------------- | ------------------------------------ | ------------------------ | ------------ |
| `createAssetV3(JsonObject asset)`           | `POST /management/v3/assets`         | JSON body (AssetInputV3) | `JsonObject` |
| `requestAssetsV3(JsonObject querySpecJson)` | `POST /management/v3/assets/request` | JSON body (Query Spec)   | `JsonArray`  |
| `getAssetV3(String id)`                     | `GET /management/v3/assets/{id}`     | Path param               | `JsonObject` |
| `removeAssetV3(String id)`                  | `DELETE /management/v3/assets/{id}`  | Path param               | void         |
| `updateAssetV3(JsonObject asset)`           | `PUT /management/v3/assets`          | JSON body (AssetInputV3) | void         |

------

### 3.2 Asset 请求/响应模型

 输入模型：`AssetInputSchema`

| 字段                | 类型   | 必需 | 说明                                                         |
| ------------------- | ------ | ---- | ------------------------------------------------------------ |
| `@context`          | Object | 必需 | JSON‑LD 上下文                                               |
| `id`                | String | 可选 | 资产 ID                                                      |
| `type`              | String | 可选 | 资产类型（默认由 EDC 定义）                                  |
| `properties`        | Map    | 必需 | 资产属性                                                     |
| `privateProperties` | Map    | 可选 | 私有属性（一般平台不使用）                                   |
| `dataAddress`       | Object | 必需 | 数据地址（见下） ([GitHub](https://raw.githubusercontent.com/eclipse-edc/Connector/main/extensions/control-plane/api/management-api/asset-api/src/main/java/org/eclipse/edc/connector/controlplane/api/management/asset/v3/AssetApiV3.java)) |

示例（源码给出）：

```
{
  "@context": { "@vocab": "https://w3id.org/edc/v0.0.1/ns/" },
  "@id": "asset-id",
  "properties": {
    "key": "value"
  },
  "privateProperties": {
    "privateKey": "privateValue"
  },
  "dataAddress": {
    "type": "HttpData",
    "baseUrl": "https://jsonplaceholder.typicode.com/todos"
  }
}
```

输出模型：`AssetOutputSchema`

Asset 查询返回内容：

| 字段                | 类型   | 说明                                             |
| ------------------- | ------ | ------------------------------------------------ |
| `id`                | String | 资产 ID                                          |
| `type`              | String | 资产类型                                         |
| `properties`        | Map    | 公开属性                                         |
| `privateProperties` | Map    | 私有属性                                         |
| `dataAddress`       | Object | 数据地址                                         |
| `createdAt`         | Number | 创建时间戳 :contentReference[oaicite:5]{index=5} |

示例：

```
 {
    "@context": { "@vocab": "https://w3id.org/edc/v0.0.1/ns/" },
    "@id": "asset-id",
    "properties": {
         "key": "value"
         },
         "privateProperties": {
            "privateKey": "privateValue"
         },
          "dataAddress": {
              "type": "HttpData",
              "baseUrl": "https://jsonplaceholder.typicode.com/todos"
         },
         "createdAt": 1688465655
                }
                """;
    }
```

------

### 3.3 资产管理接口

#### 3.3.1 创建资产 (Create Asset)

**请求方法**: `POST`

**请求路径**: `/management/v3/assets`

**请求体**:

```json
{
  "@context": { "@vocab": "https://w3id.org/edc/v0.0.1/ns/" },
  "@id": "assetId",
  "properties": {
    "name": "product description",
    "contenttype": "application/json"
  },
  "dataAddress": {
    "type": "HttpData",
    "name": "Test asset",
    "baseUrl": "https://jsonplaceholder.typicode.com/users",
    "proxyPath": "true"
  }
}
```

- **@context**: 指定数据模型的命名空间。
- **@id**: 资产的唯一标识符（例如 `assetId`）。
- **properties**: 资产的属性信息，其中包含：
  - **name**: 资产的描述或名称（例如 `product description`）。
  - **contenttype**: 资产的内容类型（例如 `application/json`）。
- **dataAddress**: 资产的数据地址信息，指定数据源的访问路径。
  - **type**: 数据类型（例如 `HttpData`）。
  - **name**: 数据源名称（例如 `Test asset`）。
  - **baseUrl**: 数据源的基本 URL 地址（例如 `https://jsonplaceholder.typicode.com/users`）。
  - **proxyPath**: 是否启用代理路径（例如 `true`）。

**响应**:

- **状态码**: `200 "Asset was created successfully. Returns the asset Id and created timestamp"`

- **状态码**: `400 "Request body was malformed"`
- **状态码**: `409 "Could not create asset, because an asset with that ID already exists"`

------

#### 3.3.2 根据条件查询资产 (Query Assets)

**请求方法**: `POST`

**请求路径**: `/management/v3/assets/request`

**请求体**: QuerySpec（可带分页、过滤条件）

**响应**:

- **状态码**: `200 "The assets matching the query"`

  输出模型：`AssetOutputSchema`

- **状态码**: `400 "Request body was malformed"`

---

#### 3.3.3 根据ID查询资产 (Query Assets)

**请求方法**: `GET`

**请求路径**: `/management/v3/assets/{id}`

**请求参数**:

- `id`: 资产的唯一标识符（路径参数）

**响应**:

- **状态码**: `200 "The asset" `

  输出模型：`AssetOutputSchema`

- **状态码**: `400 "Request was malformed, e.g. id was null"`

- **状态码**: `404 "An asset with the given ID does not exist"`

------

#### 3.3.4 删除资产 (Delete Asset)

**请求方法**: `DELETE`

**请求路径**: `/management/v3/assets/{id}`

**请求参数**:

- `id`: 资产的唯一标识符（路径参数）

**响应**:

- **状态码**: `204 "Asset was deleted successfully"`
- **状态码**: `400 Request was malformed, e.g. id was null`
- **状态码**: `404 An asset with the given ID does not exist`
- **状态码**: `409 The asset cannot be deleted, because it is referenced by a contract agreement`

---

#### 4.3.2 更新资产 (Update Asset)

**请求方法**: `PUT`

**请求路径**: `/management/v3/assets`

**请求体**:

```json
{
  "@context": { "@vocab": "https://w3id.org/edc/v0.0.1/ns/" },
  "@id": "assetId",
  "properties": {
    "name": "updated product description",
    "contenttype": "application/json"
  },
  "dataAddress": {
    "type": "HttpData",
    "name": "Updated Test asset",
    "baseUrl": "https://jsonplaceholder.typicode.com/users",
    "proxyPath": "true"
  }
}
```

- **@context**: 指定数据模型的命名空间。
- **@id**: 资产的唯一标识符（例如 `assetId`）。
- **properties**: 资产的属性信息，其中包含：
  - **name**: 资产的描述或名称（例如 `updated product description`）。
  - **contenttype**: 资产的内容类型（例如 `application/json`）。
- **dataAddress**: 资产的数据地址信息，指定数据源的访问路径。

**响应**:

- **状态码**: `204 Asset was updated successfully`
- **状态码**: `404 Asset could not be updated, because it does not exist.`
- **状态码**: `400 Request was malformed, e.g. id was null.`

---

## 4. `CatalogApiV3` 接口概览

https://github.com/eclipse-edc/Connector/blob/main/extensions/control-plane/api/management-api/catalog-api/src/main/java/org/eclipse/edc/connector/controlplane/api/management/catalog/v3/CatalogApiV3.java

https://github.com/eclipse-edc/Connector/blob/main/extensions/control-plane/api/management-api/catalog-api/src/main/java/org/eclipse/edc/connector/controlplane/api/management/catalog/v3/CatalogApiV3Controller.java

### 4.1 获取 Catalog 目录接口

**请求方法**: `POST`

**请求路径**: `/management/v3/catalog`

**请求体**（CatalogRequestV3）

| 字段                  | 类型   | 是否必需 | 说明                                                         |
| --------------------- | ------ | -------- | ------------------------------------------------------------ |
| `@context`            | object | 是       | JSON‑LD 上下文                                               |
| `@type`               | string | 是       | 类型标记（固定为 `CatalogRequest`）                          |
| `counterPartyAddress` | string | 是       | 目标 Connector 的协议端 URL（例如 `http://provider-address`） |
| `counterPartyId`      | string | 否       | 对方 Connector 的标识                                        |
| `protocol`            | string | 是       | DSP 协议版本标识（例如 `dataspace-protocol-http:2025-1`）    |
| `additionalScopes`    | list   | 否       | 额外 scope 控制                                              |
| `querySpec`           | object | 否       | 查询规范（分页／过滤等）                                     |

示例请求：

```
{
  "@context": { "@vocab": "https://w3id.org/edc/v0.0.1/ns/" },
  "@type": "CatalogRequest",
  "counterPartyAddress": "http://provider-address",
  "counterPartyId": "provider-id",
  "protocol": "dataspace-protocol-http:2025-1",
  "additionalScopes": [
    "org.eclipse.edc.vc.type:SomeCredential:read"
  ],
  "querySpec": {
    "offset": 0,
    "limit": 50,
    "sortOrder": "DESC",
    "sortField": "fieldName",
    "filterExpression": []
  }
}
```

**响应**:  "Gets contract offers (=catalog) of a single connector"

​            **响应Body**：CatalogSchema

```
 {
       "@id": "7df65569-8c59-4013-b1c0-fa14f6641bf2",
       "@type": "dcat:Catalog",
       "dcat:dataset": {
              "@id": "bcca61be-e82e-4da6-bfec-9716a56cef35",
              "@type": "dcat:Dataset",
              "odrl:hasPolicy": {
                       "@id": "OGU0ZTMzMGMtODQ2ZS00ZWMxLThmOGQtNWQxNWM0NmI2NmY4:YmNjYTYxYmUtZTgyZS00ZGE2LWJmZWMtOTcxNmE1NmNlZjM1:NDY2ZTZhMmEtNjQ1Yy00ZGQ0LWFlZDktMjdjNGJkZTU4MDNj",
                       "@type": "odrl:Set",
                       "odrl:permission": {
                                "odrl:target": "bcca61be-e82e-4da6-bfec-9716a56cef35",
                                "odrl:action": {
                                    "odrl:type": "http://www.w3.org/ns/odrl/2/use"
                                },
                                "odrl:constraint": {
                                    "odrl:and": [
                                        {
                                            "odrl:leftOperand": "https://w3id.org/edc/v0.0.1/ns/inForceDate",
                                            "odrl:operator": {
                                                "@id": "odrl:gteq"
                                            },
                                            "odrl:rightOperand": "2023-07-07T07:19:58.585601395Z"
                                        },
                                        {
                                            "odrl:leftOperand": "https://w3id.org/edc/v0.0.1/ns/inForceDate",
                                            "odrl:operator": {
                                                "@id": "odrl:lteq"
                                            },
                                            "odrl:rightOperand": "2023-07-12T07:19:58.585601395Z"
                                        }
                                    ]
                                }
                            },
                            "odrl:prohibition": [],
                            "odrl:obligation": []
                        },
                        "dcat:distribution": [
                            {
                                "@type": "dcat:Distribution",
                                "dct:format": {
                                    "@id": "HttpData"
                                },
                                "dcat:accessService": "5e839777-d93e-4785-8972-1005f51cf367"
                            }
                        ],
                        "description": "description",
                        "id": "bcca61be-e82e-4da6-bfec-9716a56cef35"
                    },
                    "dcat:service": {
                        "@id": "5e839777-d93e-4785-8972-1005f51cf367",
                        "@type": "dcat:DataService",
                        "dct:terms": "connector",
                        "dcat:endpointURL": "http://localhost:16806/protocol"
                    },
                    "dspace:participantId": "urn:connector:provider",
                    "@context": {
                        "@vocab": "https://w3id.org/edc/v0.0.1/ns/",
                        "dct": "http://purl.org/dc/terms/",
                        "edc": "https://w3id.org/edc/v0.0.1/ns/",
                        "dcat": "http://www.w3.org/ns/dcat#",
                        "odrl": "http://www.w3.org/ns/odrl/2/",
                        "dspace": "https://w3id.org/dspace/2025/1/"
                    }
                }
                """;
    }

```

------

### 4.2 获取单个数据集（Dataset）接口

**请求方法**: `POST`

**请求路径**: `/management/v3/dataset`

#### 请求体（DatasetRequestV3）

| 字段                  | 类型   | 是否必需 | 说明                        |
| --------------------- | ------ | -------- | --------------------------- |
| `@type`               | string | 是       | 类型标记（DatasetRequest）  |
| `counterPartyAddress` | string | 是       | 目标 Connector 的协议端 URL |
| `counterPartyId`      | string | 否       | 对端 ID                     |
| `protocol`            | string | 是       | DSP 协议版本标识            |
| `querySpec`           | object | 否       | 查询规范                    |

**示例请求**：

```
{
  "@context": { "@vocab": "https://w3id.org/edc/v0.0.1/ns/" },
  "@type": "DatasetRequest",
  "@id": "dataset-id",
  "counterPartyAddress": "http://provider-address:protocolPort/protocol",
  "counterPartyId": "provider-id",
  "protocol": "dataspace-protocol-http:2025-1"
}
```

**响应**:  "Gets single dataset from a connector"

​            **响应Body**：DatasetSchema

```
  {
                    "@id": "bcca61be-e82e-4da6-bfec-9716a56cef35",
                    "@type": "dcat:Dataset",
                    "odrl:hasPolicy": {
                        "@id": "OGU0ZTMzMGMtODQ2ZS00ZWMxLThmOGQtNWQxNWM0NmI2NmY4:YmNjYTYxYmUtZTgyZS00ZGE2LWJmZWMtOTcxNmE1NmNlZjM1:NDY2ZTZhMmEtNjQ1Yy00ZGQ0LWFlZDktMjdjNGJkZTU4MDNj",
                        "@type": "odrl:Set",
                        "odrl:permission": {
                            "odrl:target": "bcca61be-e82e-4da6-bfec-9716a56cef35",
                            "odrl:action": {
                                "odrl:type": "http://www.w3.org/ns/odrl/2/use"
                            },
                            "odrl:constraint": {
                                "odrl:and": [
                                    {
                                        "odrl:leftOperand": "https://w3id.org/edc/v0.0.1/ns/inForceDate",
                                        "odrl:operator": {
                                            "@id": "odrl:gteq"
                                        },
                                        "odrl:rightOperand": "2023-07-07T07:19:58.585601395Z"
                                    },
                                    {
                                        "odrl:leftOperand": "https://w3id.org/edc/v0.0.1/ns/inForceDate",
                                        "odrl:operator": {
                                            "@id": "odrl:lteq"
                                        },
                                        "odrl:rightOperand": "2023-07-12T07:19:58.585601395Z"
                                    }
                                ]
                            }
                        },
                        "odrl:prohibition": [],
                        "odrl:obligation": [],
                        "odrl:target": "bcca61be-e82e-4da6-bfec-9716a56cef35"
                    },
                    "dcat:distribution": [
                        {
                            "@type": "dcat:Distribution",
                            "dct:format": {
                                "@id": "HttpData"
                            },
                            "dcat:accessService": "5e839777-d93e-4785-8972-1005f51cf367"
                        }
                    ],
                    "description": "description",
                    "id": "bcca61be-e82e-4da6-bfec-9716a56cef35",
                    "@context": {
                        "@vocab": "https://w3id.org/edc/v0.0.1/ns/",
                        "dct": "http://purl.org/dc/terms/",
                        "edc": "https://w3id.org/edc/v0.0.1/ns/",
                        "dcat": "http://www.w3.org/ns/dcat#",
                        "odrl": "http://www.w3.org/ns/odrl/2/",
                        "dspace": "https://w3id.org/dspace/2025/1/"
                    }
                }
                """;
    }
```

------

## 5.Policy DefinitionApiV3概览

### 5.1接口功能概览

| 功能             | 方法   | 路径                                                   |
| ---------------- | ------ | ------------------------------------------------------ |
| 查询策略定义     | POST   | `/management/v3/policydefinitions/request`             |
| 获取单个策略定义 | GET    | `/management/v3/policydefinitions/{id}`                |
| 创建策略定义     | POST   | `/management/v3/policydefinitions`                     |
| 删除策略定义     | DELETE | `/management/v3/policydefinitions/{id}`                |
| 更新策略定义     | PUT    | `/management/v3/policydefinitions/{id}`                |
| 验证策略定义     | POST   | `/management/v3/policydefinitions/{id}/validate`       |
| 策略评估计划     | POST   | `/management/v3/policydefinitions/{id}/evaluationplan` |

------

### 5.2**查询策略定义 (Query Policy Definitions)**

**请求方法**: `POST`

**请求路径**: `/management/v3/policydefinitions/request`

**请求体**：根据条件查询策略定义列表，支持过滤、分页等查询参数（QuerySpec）

**响应**：

- **状态码**: `200 The policy definitions matching the query`

  返回内容：PolicyDefinitionOutputSchema

  ```
    {
                      "@context": { "@vocab": "https://w3id.org/edc/v0.0.1/ns/" },
                      "@id": "definition-id",
                      "policy": {
                          "@context": "http://www.w3.org/ns/odrl.jsonld",
                          "@type": "Set",
                          "uid": "http://example.com/policy:1010",
                          "permission": [{
                              "target": "http://example.com/asset:9898.movie",
                              "action": "display",
                              "constraint": [{
                                  "leftOperand": "spatial",
                                  "operator": "eq",
                                  "rightOperand":  "https://www.wikidata.org/wiki/Q183",
                                  "comment": "i.e Germany"
                              }]
                          }]
                      },
                      "createdAt": 1688465655
                  }
              """;
  ```

- **状态码**: `400 Request was malformed`

------

### 5.3获取单个策略定义 (Get Policy Definition)

**请求方法**: `GET`
**请求路径**: `/management/v3/policydefinitions/{id}`

**请求参数**：id

**响应**：

- **状态码**：`200 The  policy definition`

   返回内容: PolicyDefinitionOutputSchema

- **状态码**：`400 Request was malformed, e.g. id was null`

- **状态码**：`404 An  policy definition with the given ID does not exist`

------

### 5.4创建策略定义 (Create Policy Definition)

**请求方法**: `POST`
**请求路径**: `/management/v3/policydefinitions`

**请求体示例**:PolicyDefinitionInputSchema

```
{
                    "@context": { "@vocab": "https://w3id.org/edc/v0.0.1/ns/" },
                    "@id": "definition-id",
                    "policy": {
                        "@context": "http://www.w3.org/ns/odrl.jsonld",
                        "@type": "Set",
                        "uid": "http://example.com/policy:1010",
                        "profile": "http://example.com/odrl:profile:02",
                        "permission": [{
                            "target": "http://example.com/asset:9898.movie",
                            "action": "display",
                            "constraint": [{
                                "leftOperand": "spatial",
                                "operator": "eq",
                                "rightOperand":  "https://www.wikidata.org/wiki/Q183",
                                "comment": "i.e Germany"
                            }]
                        }]
                    }
                }
```

**响应**:

- **状态码**: `200 "policy definition was created successfully. Returns the Policy Definition Id and created timestamp"`
- **状态码**: `400 "Request body was malformed"`、
- **状态码**: `409  "Could not create policy definition, because a contract definition with that ID already exists"`、

------

### 5.4删除策略定义 (Delete Policy Definition)

**请求方法**: `DELETE`
**请求路径**: `/management/v3/policydefinitions/{id}`

**响应**:

- **204 "Policy definition was deleted successfully"**
- **400 "Request was malformed, e.g. id was null",**: 
- **404 "An policy definition with the given ID does not exist"**
- **409  "The policy definition cannot be deleted, because it is referenced by a contract definition"**

------

### 5.5  更新策略定义 (Update Policy Definition)

**请求方法**: `PUT`
**请求路径**: `/management/v3/policydefinitions/{id}`

**请求体**： PolicyDefinitionInputSchema

**响应**:

- **204 "policy definition was updated successfully."**: 更新成功
- **400 "Request body was malformed"**
- **404 "policy definition could not be updated, because it does not exists"**: 指定 ID 不存在

------

### **5.6 验证策略定义 (Validate Policy Definition)**

**请求方法**: `POST`
 **请求路径**: `/management/v3/policydefinitions/{id}/validate`

**说明**: 检查策略定义是否符合语法规范和约束条件。[GitHub](https://raw.githubusercontent.com/eclipse-edc/Connector/main/extensions/control-plane/api/management-api/policy-definition-api/src/main/java/org/eclipse/edc/connector/controlplane/api/management/policy/v3/PolicyDefinitionApiV3.java)

**响应**:

- **200 "Returns the validation result"**

```
  {
                    "@context": { "@vocab": "https://w3id.org/edc/v0.0.1/ns/" },
                    "@type": "PolicyValidationResult",
                    "isValid": false,
                    "errors": [
                        "error1",
                        "error2"
                    ]
                }
                """;
```

- **404  "policy definition could not be validated, because it does not exists"**

------

### **4.7 策略评估计划 (Create Evaluation Plan)**

**请求方法**: `POST`
 **请求路径**: `/management/v3/policydefinitions/{id}/evaluationplan`

**说明**: 为指定策略创建评估计划，用于策略执行流程分析。

**请求体示例**: PolicyEvaluationPlanRequestSchema

```
 {
                    "@context": { "@vocab": "https://w3id.org/edc/v0.0.1/ns/" },
                    "@type": "PolicyEvaluationPlanRequest",
                    "policyScope": "catalog"
                }
                """;
```

**响应**:

- **200 "Returns the evaluation plan"**

   PolicyEvaluationPlanSchema

```
  {
                     "@type": "PolicyEvaluationPlan",
                     "preValidators": "DcpScopeExtractorFunction",
                     "permissionSteps": {
                         "@type": "PermissionStep",
                         "isFiltered": false,
                         "filteringReasons": [],
                         "ruleFunctions": [],
                         "constraintSteps": {
                             "@type": "AtomicConstraintStep",
                             "isFiltered": true,
                             "filteringReasons": [
                                 "leftOperand 'MembershipCredential' is not bound to scope 'request.catalog'",
                                 "leftOperand 'MembershipCredential' is not bound to any function within scope 'request.catalog'"
                             ],
                             "functionParams": [
                                 "'MembershipCredential'",
                                 "EQ",
                                 "'active'"
                             ]
                         },
                         "dutySteps": []
                     },
                     "prohibitionSteps": [],
                     "obligationSteps": [],
                     "postValidators": "DefaultScopeMappingFunction",
                     "@context": {
                         "@vocab": "https://w3id.org/edc/v0.0.1/ns/",
                         "edc": "https://w3id.org/edc/v0.0.1/ns/",
                         "odrl": "http://www.w3.org/ns/odrl/2/"
                     }
                 }
                """;
```

- **404 "An evaluation plan could not be created, because the policy definition does not exists"**

------

## 6.ContractDefinitionApiV3概览

https://github.com/eclipse-edc/Connector/blob/main/extensions/control-plane/api/management-api/contract-definition-api/src/main/java/org/eclipse/edc/connector/controlplane/api/management/contractdefinition/v3/ContractDefinitionApiV3.java

https://github.com/eclipse-edc/Connector/blob/main/extensions/control-plane/api/management-api/contract-definition-api/src/main/java/org/eclipse/edc/connector/controlplane/api/management/contractdefinition/v3/ContractDefinitionApiV3Controller.java

### 6.1 接口功能概览

| 功能             | 方法   | 路径                                         |
| ---------------- | ------ | -------------------------------------------- |
| 创建合同定义     | POST   | `/management/v3/contractdefinitions`         |
| 查询合同定义列表 | POST   | `/management/v3/contractdefinitions/request` |
| 获取单个合同定义 | GET    | `/management/v3/contractdefinitions/{id}`    |
| 更新合同定义     | PUT    | `/management/v3/contractdefinitions/{id}`    |
| 删除合同定义     | DELETE | `/management/v3/contractdefinitions/{id}`    |

------

### 6.2 创建合同定义

**请求方法**：`POST`
**请求路径**：`/management/v3/contractdefinitions`

**请求体示例**  ContractDefinitionInputSchema

```
{
                    "@context": { "@vocab": "https://w3id.org/edc/v0.0.1/ns/" },
                    "@id": "definition-id",
                    "accessPolicyId": "asset-policy-id",
                    "contractPolicyId": "contract-policy-id",
                    "assetsSelector": []
                }
                """;
```

**字段说明**

| 字段                            | 类型                      | 必填 | 说明               |
| ------------------------------- | ------------------------- | ---- | ------------------ |
| `@context`                      | object                    | 是   | JSON-LD 上下文     |
| `@id`                           | string                    | 是   | 合同定义 ID        |
| `accessPolicyId`                | string                    | 是   | 访问策略 ID        |
| `contractPolicyId`              | string                    | 是   | 合同策略 ID        |
| `assetsSelector`                | array                     | 是   | 资产选择条件列表   |
| `assetsSelector[].@type`        | string                    | 是   | 一般为 `Criterion` |
| `assetsSelector[].operandLeft`  | string                    | 是   | 过滤字段           |
| `assetsSelector[].operator`     | string                    | 是   | 比较操作符         |
| `assetsSelector[].operandRight` | string / number / boolean | 是   | 比较值             |

**响应：**

- **200 "contract definition was created successfully. Returns the Contract Definition Id and created timestamp"**
- **400 "Request body was malformed",**
- **409  "Could not create contract definition, because a contract definition with that ID already exists"**

------

### 6.3 查询合同定义列表

**请求方法**：`POST`
**请求路径**：`/management/v3/contractdefinitions/request`

**响应：**

- **200 "The contract definitions matching the query"**

  ContractDefinitionOutputSchema

```
 {
                    "@context": { "@vocab": "https://w3id.org/edc/v0.0.1/ns/" },
                    "@id": "definition-id",
                    "accessPolicyId": "asset-policy-id",
                    "contractPolicyId": "contract-policy-id",
                    "assetsSelector": [],
                    "createdAt": 1688465655
                }
```

- **400 "Request was malformed"**

------

### 6.4 获取单个合同定义

**请求方法**：`GET`
**请求路径**：`/management/v3/contractdefinitions/{id}`

**响应：**

- **200  "The contract definition"**  ContractDefinitionOutputSchema
- **400   "Request was malformed, e.g. id was null"**
- **404  "An contract agreement with the given ID does not exist"**

------

### 6.5  更新合同定义

**请求方法**：`PUT`
**请求路径**：`/management/v3/contractdefinitions/{id}`

**请求体**：ContractDefinitionInputSchema

**响应：**

- **204 "Contract definition was updated successfully"**
- **400   "Request was malformed, e.g. id was null"**
- **404  "A contract definition with the given ID does not exist"**

------

### 6.6 删除合同定义

**请求方法**：`DELETE`
**请求路径**：`/management/v3/contractdefinitions/{id}`

**响应：**

- **204  "Contract definition was deleted successfully"**
- **400  "Request was malformed, e.g. id was null"**
- **404  "A contract definition with the given ID does not exist"**

------

## 7.ContractNegotiationApiV3

https://github.com/eclipse-edc/Connector/blob/main/extensions/control-plane/api/management-api/contract-negotiation-api/src/main/java/org/eclipse/edc/connector/controlplane/api/management/contractnegotiation/v3/ContractNegotiationApiV3.java

https://github.com/eclipse-edc/Connector/blob/main/extensions/control-plane/api/management-api/contract-negotiation-api/src/main/java/org/eclipse/edc/connector/controlplane/api/management/contractnegotiation/v3/ContractNegotiationApiV3Controller.java

### 7.1 功能概览

| 功能             | 方法   | 路径                                                 |
| ---------------- | ------ | ---------------------------------------------------- |
| 查询协商列表     | POST   | `/management/v3/contractnegotiations/request`        |
| 获取单个协商     | GET    | `/management/v3/contractnegotiations/{id}`           |
| 获取协商状态     | GET    | `/management/v3/contractnegotiations/{id}/state`     |
| 获取协商对应协议 | GET    | `/management/v3/contractnegotiations/{id}/agreement` |
| 发起协商         | POST   | `/management/v3/contractnegotiations`                |
| 终止协商         | POST   | `/management/v3/contractnegotiations/{id}/terminate` |
| 删除协商         | DELETE | `/management/v3/contractnegotiations/{id}`           |

### 7.2 查询合同协商列表

**请求方法**：`POST`
**请求路径**：`/management/v3/contractnegotiations/request`

**响应：**

- **200 "The contract negotiations that match the query"**

  ContractNegotiationSchema

  ```
  [
    {
      "@id": "negotiation-id",
      "type": "CONSUMER",
      "protocol": "dataspace-protocol-http",
      "state": "REQUESTED",
      "counterPartyId": "provider",
      "counterPartyAddress": "http://provider:19194/protocol",
      "createdAt": 1772630629678,
      "assetId": "assetId"
    }
  ]
  
  ```

- **400 "Request was malformed"**

------

### 7.3 获取单个合同协商

**请求方法**：`GET`
**请求路径**：`/management/v3/contractnegotiations/{id}`

**响应：**

- **200 "The contract negotiation"**  ContractNegotiationSchema
- **400  "Request was malformed, e.g. id was null"**
- **404 "An contract negotiation with the given ID does not exist"**

------

### 7.4 获取合同协商状态

**请求方法**：`GET`
**请求路径**：`/management/v3/contractnegotiations/{id}/state`

**响应：**

- **200 "The contract negotiation's state"** NegotiationState
- **400  "Request was malformed, e.g. id was null"** 
- **404  "An contract negotiation with the given ID does not exist",**

------

### 7.5 获取协商对应的合同协议

**请求方法**：`GET`
**请求路径**：`/management/v3/contractnegotiations/{id}/agreement`

**响应：**

- **200 "The contract agreement that is attached to the negotiation, or null",**  ContractAgreementSchema
- **400 "Request was malformed, e.g. id was null**
- **404 "An contract negotiation with the given ID does not exist"**

------

### 7.6 发起合同协商

**请求方法**：`POST`
**请求路径**：`/management/v3/contractnegotiations`

请求体模型

源码中定义的请求模型为 `ContractRequestV3`

| 字段                  | 类型   | 必填 | 说明                                    |
| --------------------- | ------ | ---- | --------------------------------------- |
| `@context`            | object | 是   | JSON-LD 上下文                          |
| `@type`               | string | 否   | 类型，源码示例里是 ContractRequest 类型 |
| `protocol`            | string | 是   | 协议标识                                |
| `counterPartyAddress` | string | 是   | 对端协议地址                            |
| `policy`              | object | 是   | Offer 对象                              |
| `callbackAddresses`   | array  | 否   | 回调地址列表                            |

**请求体示例**

```
{
  "@context": {
    "@vocab": "https://w3id.org/edc/v0.0.1/ns/"
  },
  "@type": "https://w3id.org/edc/v0.0.1/ns/ContractRequest",
  "counterPartyAddress": "http://provider-address",
  "protocol": "dataspace-protocol-http:2025-1",
  "policy": {
    "@context": "http://www.w3.org/ns/odrl.jsonld",
    "@type": "odrl:Offer",
    "@id": "offer-id",
    "assigner": "providerId",
    "permission": [],
    "prohibition": [],
    "obligation": [],
    "target": "assetId"
  },
  "callbackAddresses": [
    {
      "transactional": false,
      "uri": "http://callback/url",
      "events": [
        "contract.negotiation",
        "transfer.process"
      ],
      "authKey": "auth-key",
      "authCodeId": "auth-code-id"
    }
  ]
}
```

**policy 字段中的 Offer 结构**

源码还单独定义了 `OfferV3`

| 字段       | 类型   | 必填 | 说明                  |
| ---------- | ------ | ---- | --------------------- |
| `@type`    | string | 否   | 一般为 `odrl:Offer`   |
| `@id`      | string | 是   | offer ID              |
| `assigner` | string | 是   | 提供方 participant ID |
| `target`   | string | 是   | 目标资产 ID           |

```
{
  "@type": "IdResponse",
  "@id": "4de26f9b-5bd3-4873-9022-b9a2402d62fd",
  "createdAt": 1772630629678,
  "@context": {
    "@vocab": "https://w3id.org/edc/v0.0.1/ns/",
    "edc": "https://w3id.org/edc/v0.0.1/ns/",
    "odrl": "http://www.w3.org/ns/odrl/2/"
  }
}
```

**响应：**

- **200 "The negotiation was successfully initiated. Returns the contract negotiation ID and created timestamp**
- **400 "Request body was malformed**

------

### 7.7 终止合同协商

**请求方法**：`POST`
**请求路径**：`/management/v3/contractnegotiations/{id}/terminate`

**请求体模型**

源码定义了 `TerminateNegotiationV3`

| 字段     | 类型   | 必填 | 说明                       |
| -------- | ------ | ---- | -------------------------- |
| `@type`  | string | 否   | 类型，TerminateNegotiation |
| `@id`    | string | 否   | 协商 ID                    |
| `reason` | string | 否   | 终止原因                   |

**请求体示例**

```
{
  "@context": {
    "@vocab": "https://w3id.org/edc/v0.0.1/ns/"
  },
  "@type": "https://w3id.org/edc/v0.0.1/ns/TerminateNegotiation",
  "@id": "negotiation-id",
  "reason": "a reason to terminate"
}
```

**响应：**

- **200 ContractNegotiation is terminating**
- **400 "Request was malformed"**
- **404 "A contract negotiation with the given ID does not exist",**

------

### 7.8 删除合同协商

**请求方法**：`DELETE`
**请求路径**：`/management/v3/contractnegotiations/{id}`

**响应**

- `204`："ContractNegotiation is deleted"
- `400`："Request was malformed, e.g. id was null"
- `404`："A contract negotiation with the given ID does not exist",
- `409`：The given contract negotiation cannot be deleted due to a wrong state or has existing contract agreement"

------

## 8.ContractAgreementApiV3概览

https://github.com/eclipse-edc/Connector/blob/main/extensions/control-plane/api/management-api/contract-agreement-api/src/main/java/org/eclipse/edc/connector/controlplane/api/management/contractagreement/v3/ContractAgreementApiV3.java

### 8.1 接口功能概览

| 功能                     | 方法 | 路径                                                 |
| ------------------------ | ---- | ---------------------------------------------------- |
| 查询合同协议列表         | POST | `/management/v3/contractagreements/request`          |
| 获取单个合同协议         | GET  | `/management/v3/contractagreements/{id}`             |
| 通过协议 ID 获取对应协商 | GET  | `/management/v3/contractagreements/{id}/negotiation` |

------

### 8.2查询合同协议列表

**请求方法**：`POST`
**请求路径**：`/management/v3/contractagreements/request`

**响应：**

- **200 "The contract agreements matching the query"**  ContractAgreementSchema
- **400 "Request body was malformed",**

------

### 8.3 获取单个合同协议

**请求方法**：`GET`
**请求路径**：`/management/v3/contractagreements/{id}`x

**响应：**

- `200`："The contract agreement"   ContractAgreementSchema
- `400`："Request was malformed, e.g. id was null",
- `404`："An contract agreement with the given ID does not exist"

------

### 8.4 通过合同协议 ID 获取对应协商

**请求方法**：`GET`
**请求路径**：`/management/v3/contractagreements/{id}/negotiation`

**响应：**

- `200`："The contract negotiation"  ContractNegotiationSchema
- `400`："Request was malformed, e.g. id was null",
- `404`："An contract agreement with the given ID does not exist

------

## 9 TransferProcessApiV3概览

https://github.com/eclipse-edc/Connector/blob/main/extensions/control-plane/api/management-api/transfer-process-api/src/main/java/org/eclipse/edc/connector/controlplane/api/management/transferprocess/v3/TransferProcessApiV3.java

### 9.1 接口功能概览

| 功能                        | 方法 | 路径                                                |
| --------------------------- | ---- | --------------------------------------------------- |
| 查询传输过程列表            | POST | `/management/v3/transferprocesses/request`          |
| 获取单个传输过程            | GET  | `/management/v3/transferprocesses/{id}`             |
| 获取传输状态                | GET  | `/management/v3/transferprocesses/{id}/state`       |
| 发起传输                    | POST | `/management/v3/transferprocesses`                  |
| 请求回收资源（deprovision） | POST | `/management/v3/transferprocesses/{id}/deprovision` |
| 终止传输                    | POST | `/management/v3/transferprocesses/{id}/terminate`   |
| 暂停传输                    | POST | `/management/v3/transferprocesses/{id}/suspend`     |
| 恢复传输                    | POST | `/management/v3/transferprocesses/{id}/resume`      |

### 9.2 查询传输过程列表

**请求方法**：`POST`
**请求路径**：`/management/v3/transferprocesses/request`

**响应：**

- **200 The transfer processes matching the query** 

  TransferProcessSchema

  | 字段                  | JSON 类型 | 说明                     |
  | --------------------- | --------- | ------------------------ |
  | `@type`               | string    | TransferProcess 类型     |
  | `@id`                 | string    | 传输过程 ID              |
  | `type`                | string    | `PROVIDER` 或 `CONSUMER` |
  | `protocol`            | string    | 协议类型                 |
  | `counterPartyId`      | string    | 对端 participant ID      |
  | `counterPartyAddress` | string    | 对端地址                 |
  | `state`               | string    | 当前状态                 |
  | `contractAgreementId` | string    | 合同协议 ID              |
  | `errorDetail`         | string    | 错误详情                 |
  | `dataDestination`     | object    | 数据目标                 |
  | `privateProperties`   | object    | 私有属性                 |
  | `callbackAddresses`   | array     | 回调配置                 |

```
 {
                    "@context": { "@vocab": "https://w3id.org/edc/v0.0.1/ns/" },
                    "@type": "https://w3id.org/edc/v0.0.1/ns/TransferProcess",
                    "@id": "process-id",
                    "correlationId": "correlation-id",
                    "type": "PROVIDER",
                    "state": "STARTED",
                    "stateTimestamp": 1688465655,
                    "assetId": "asset-id",
                    "contractId": "contractId",
                    "dataDestination": {
                        "type": "data-destination-type"
                    },
                    "privateProperties": {
                        "private-key": "private-value"
                    },
                    "errorDetail": "eventual-error-detail",
                    "createdAt": 1688465655,
                    "callbackAddresses": [{
                        "transactional": false,
                        "uri": "http://callback/url",
                        "events": ["contract.negotiation", "transfer.process"],
                        "authKey": "auth-key",
                        "authCodeId": "auth-code-id"
                    }]
                }
                """;
```

- **400 Request was malformed**

------

### 9.3 获取单个传输过程

**请求方法**：`GET`
**请求路径**：`/management/v3/transferprocesses/{id}`

**响应示例**

```
{
  "@type": "https://w3id.org/edc/v0.0.1/ns/TransferProcess",
  "@id": "process-id",
  "type": "PROVIDER",
  "protocol": "dataspace-protocol-http:2025-1",
  "counterPartyId": "provider",
  "counterPartyAddress": "http://provider-address",
  "state": "STARTED",
  "contractAgreementId": "agreement-id",
  "errorDetail": null,
  "dataDestination": {
    "type": "data-destination-type"
  },
  "privateProperties": {
    "private-key": "private-value"
  },
  "callbackAddresses": []
}
```

**响应码**

- `200`：The transfer process.  TransferProcessSchema
- `400`："Request was malformed, e.g. id was null"
- `404`：A transfer process with the given ID does not exist"

------

### 9.4 获取传输状态

**请求方法**：`GET`
**请求路径**：`/management/v3/transferprocesses/{id}/state`

**响应码**

- `200`：The  transfer process's state"  

  TransferStateSchema

  ```
   {
                      "@context": { "@vocab": "https://w3id.org/edc/v0.0.1/ns/" },
                      "@type": "https://w3id.org/edc/v0.0.1/ns/TransferState",
                      "state": "STARTED"
                  }
  ```

- `400`："Request was malformed, e.g. id was null"

- `404`："An  transfer process with the given ID does not exist"

------

### 9.5 发起传输

**请求方法**：`POST`
**请求路径**：`/management/v3/transferprocesses`

**请求体模型**

源码中的 `TransferRequestSchema` 包含以下字段：[GitHub](https://raw.githubusercontent.com/eclipse-edc/Connector/main/extensions/control-plane/api/management-api/transfer-process-api/src/main/java/org/eclipse/edc/connector/controlplane/api/management/transferprocess/v3/TransferProcessApiV3.java)

| 字段                  | 类型   | 必填       | 说明                     |
| --------------------- | ------ | ---------- | ------------------------ |
| `@context`            | object | 是         | JSON-LD 上下文           |
| `@type`               | string | 否         | 一般为 `TransferRequest` |
| `protocol`            | string | 是         | 协议版本                 |
| `counterPartyAddress` | string | 是         | 对端协议地址             |
| `contractId`          | string | 是         | 合同 ID                  |
| `assetId`             | string | 否，已废弃 | 资产 ID                  |
| `transferType`        | string | 是         | 传输类型                 |
| `dataDestination`     | object | 否         | 数据目标                 |
| `privateProperties`   | object | 否         | 私有属性                 |
| `callbackAddresses`   | array  | 否         | 回调地址列表             |

**请求体示例**

这个示例直接来自源码里的 `TRANSFER_REQUEST_EXAMPLE`：[GitHub](https://raw.githubusercontent.com/eclipse-edc/Connector/main/extensions/control-plane/api/management-api/transfer-process-api/src/main/java/org/eclipse/edc/connector/controlplane/api/management/transferprocess/v3/TransferProcessApiV3.java)

```
{
  "@context": {
    "@vocab": "https://w3id.org/edc/v0.0.1/ns/"
  },
  "@type": "https://w3id.org/edc/v0.0.1/ns/TransferRequest",
  "protocol": "dataspace-protocol-http:2025-1",
  "counterPartyAddress": "http://provider-address",
  "contractId": "contract-id",
  "transferType": "transferType",
  "dataDestination": {
    "type": "data-destination-type"
  },
  "privateProperties": {
    "private-key": "private-value"
  },
  "callbackAddresses": [
    {
      "transactional": false,
      "uri": "http://callback/url",
      "events": [
        "contract.negotiation",
        "transfer.process"
      ],
      "authKey": "auth-key",
      "authCodeId": "auth-code-id"
    }
  ]
}
```

**响应示例**

```
{
  "@type": "IdResponse",
  "@id": "11aaf6b7-8aed-4d31-9ca2-b68c2ec22a36",
  "createdAt": 1772700259635,
  "@context": {
    "@vocab": "https://w3id.org/edc/v0.0.1/ns/",
    "edc": "https://w3id.org/edc/v0.0.1/ns/",
    "odrl": "http://www.w3.org/ns/odrl/2/"
  }
}
```

**响应码**

- `200`："The transfer was successfully initiated. Returns the transfer process ID and created timestamp
- `400`："Request body was malformed"

------

### 9.6 请求回收资源（Deprovision）

**请求方法**：`POST`
**请求路径**：`/management/v3/transferprocesses/{id}/deprovision`

请求回收与某个传输过程关联的资源。源码方法名为：

- `deprovisionTransferProcessV3(String id)`

源码说明这同样是异步行为，请求成功只表示系统已接收 deprovision 请求。

**响应码**

- `204`："Request to deprovision the transfer process was successfully received
- `400`："Request was malformed, e.g. id was null"
- `404`："A transfer process with the given ID does not exist",

------

### 9.7 终止传输

**请求方法**：`POST`
**请求路径**：`/management/v3/transferprocesses/{id}/terminate`

**请求体模型**

源码中的 `TerminateTransferSchema` 示例为：[GitHub](https://raw.githubusercontent.com/eclipse-edc/Connector/main/extensions/control-plane/api/management-api/transfer-process-api/src/main/java/org/eclipse/edc/connector/controlplane/api/management/transferprocess/v3/TransferProcessApiV3.java)

```
{
  "@context": {
    "@vocab": "https://w3id.org/edc/v0.0.1/ns/"
  },
  "@type": "https://w3id.org/edc/v0.0.1/ns/TerminateTransfer",
  "reason": "a reason to terminate"
}
```

**响应码**

- `204`："Request to terminate the transfer process was successfully received"
- `400`："Request was malformed, e.g. id was null"
- `404`： "A transfer process with the given ID does not exist"
- `409`： "Could not terminate transfer process, because it is already completed or terminated."

------

### 9.8暂停传输

**请求方法**：`POST`
**请求路径**：`/management/v3/transferprocesses/{id}/suspend`

**说明**

请求暂停某个传输过程。源码方法名为：

- `suspendTransferProcessV3(String id, JsonObject suspendTransfer)`

**请求体模型**

源码中的 `SuspendTransferSchema` 示例为：[GitHub](https://raw.githubusercontent.com/eclipse-edc/Connector/main/extensions/control-plane/api/management-api/transfer-process-api/src/main/java/org/eclipse/edc/connector/controlplane/api/management/transferprocess/v3/TransferProcessApiV3.java)

```
{
  "@context": {
    "@vocab": "https://w3id.org/edc/v0.0.1/ns/"
  },
  "@type": "https://w3id.org/edc/v0.0.1/ns/SuspendTransfer",
  "reason": "a reason to suspend"
}
```

**响应码**

- `204`："Request to suspend the transfer process was successfully received"
- `400`：Request was malformed, e.g. id was null"
- `404`： "A transfer process with the given ID does not exist"
- `409`："Could not suspend the transfer process, because it is already completed or terminated.",

------

### 9.9  恢复传输

**请求方法**：`POST`
**请求路径**：`/management/v3/transferprocesses/{id}/resume`

**说明**

请求恢复一个已暂停的传输过程。源码方法名为：

- `resumeTransferProcessV3(String id)`

这同样是异步行为，请求成功只代表恢复请求被成功接收。[GitHub](https://raw.githubusercontent.com/eclipse-edc/Connector/main/extensions/control-plane/api/management-api/transfer-process-api/src/main/java/org/eclipse/edc/connector/controlplane/api/management/transferprocess/v3/TransferProcessApiV3.java)

**响应码**

- `204`："Request to resume the transfer process was successfully received"
- `400`："Request was malformed, e.g. id was null"
- `404`："A transfer process with the given ID does not exist"
