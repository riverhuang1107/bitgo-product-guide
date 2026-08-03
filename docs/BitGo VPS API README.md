# BitGo VPS API

BitGo VPS API 用于查询可用 VPS 资源、管理 SSH 公钥，以及创建、查询、启动、停止和删除 VPS 实例。

## 基本信息

- Base URL：`https://bitgo-vps.expvent.com.cn:1111`
- API 风格：REST
- 请求格式：`application/json`
- 响应格式：`application/json`
- Swagger 版本：2.0
- Swagger 文件：[`ai-token-billing.swagger.json`](./ai-token-billing.swagger.json)、[`ai-vps-billing.swagger.json`](./ai-vps-billing.swagger.json)

> Swagger 文档没有声明 `securityDefinitions`、认证请求头或统一错误响应模型。如果服务实际需要认证，请以服务端提供的认证规范为准。
>
> 当前 Swagger 文件中的部分中文描述存在编码乱码，并且部分字符串缺少闭合引号，无法作为严格合法的 JSON 直接导入 Swagger UI。本 README 根据其中可可靠识别的路径、参数和数据结构整理。

## 快速开始

下面的示例使用 `curl`。如果服务端证书无法被本机信任，可仅在开发环境临时添加 `-k`；生产环境应配置并校验有效证书。

所有发送 JSON 的请求都应设置：

```text
Content-Type: application/json
```

## 签名认证

除 `GET /vps/v1/resources` 外，所有接口都必须同时携带钱包业务签名和接口级 ECDSA 签名。签名材料只能在运行时生成，禁止将真实私钥、完整签名或完整认证 Header 写入源码、仓库、日志或文档。

### 1. 生成 `X-Params`

使用自持 Web3 钱包对以下消息签名：

```text
${wallet_address}${money}${money_id}
```

按链和地址类型选择签名方式：`btc`/`ltc` 普通地址使用 compact ECDSA，Taproot 地址先进行 Taproot tweak 后使用 Schnorr；`eth` 使用 Ethereum hex 私钥的 ECDSA 签名。对于 compact 签名，将原始签名字节直接 Base64 编码；不要先转为 HEX 再 Base64。将该 Base64 结果写入 JSON：

```json
{
  "wallet_address": "WALLET_ADDRESS",
  "money": "AUTHORIZED_AMOUNT",
  "money_id": "MONEY_ID",
  "signature": "BASE64_WALLET_SIGNATURE"
}
```

再将该 JSON 按 UTF-8 整体 Base64 编码，作为 `X-Params`。

### 2. 生成接口级签名

接口级 ECDSA 密钥对必须稳定：首次生成后安全持久化，后续请求始终使用同一私钥及其对应的 `X-Public-Key`，以便验证本次 HTTP 请求未被替换。每次请求只重新生成随机 `X-Nonce`，并针对新的 `X-Params + X-Nonce` 重新计算 `X-Signature`。严格拼接 `message = X-Params + X-Nonce`，计算 SHA-256，再用独立的本地 ECDSA 私钥对摘要生成 ASN.1/DER 签名，最后将签名字节编码为 HEX，作为 `X-Signature`。

本项目的命令行工具默认将接口 ECDSA 私钥保存在用户配置目录的 `bitgo-vps/interface-ecdsa-p256.pem`，不会写入仓库。可通过环境变量 `BITGO_INTERFACE_ECDSA_KEY_FILE` 指定受保护的私钥文件路径。该私钥、完整签名和完整认证 Header 均不得提交或记录。

签名字段的编码规则如下：

| 字段 | 生成规则 |
| --- | --- |
| `signature`（钱包 JSON 字段） | compact 签名的原始字节直接 Base64 编码 |
| `X-Params` | 包含钱包业务字段与 `signature` 的 JSON 字符串整体 Base64 编码 |
| `X-Nonce` | 每次请求生成的随机字符串 |
| `X-Signature` | 对 `SHA-256(X-Params + X-Nonce)` 生成的 ECDSA ASN.1/DER 签名字节进行 HEX 编码 |

最终请求 Header：

```text
X-Params: BASE64_ENCODED_WALLET_PARAMS
X-Nonce: RANDOM_NONCE
X-Signature: ECDSA_SIGNATURE_HEX
X-Public-Key: ECDSA_PUBLIC_KEY
```

以下文档中的接口请求，除资源查询接口外，均需在 `curl` 中加入上述四个运行时生成的 Header：

```bash
curl "https://bitgo-vps.expvent.com.cn:1111/vps/v1/ssh-keys" \
  -H "X-Params: BASE64_ENCODED_WALLET_PARAMS" \
  -H "X-Nonce: RANDOM_NONCE" \
  -H "X-Signature: ECDSA_SIGNATURE_HEX" \
  -H "X-Public-Key: ECDSA_PUBLIC_KEY"
```

### 签名适用范围

`GET /vps/v1/resources` 是唯一免签接口。下表其余全部接口都必须使用本节定义的同一套签名逻辑：接口 ECDSA 密钥对保持不变；每次请求重新生成 `X-Nonce` 和 `X-Signature`：

| 方法 | 路径 | 是否签名 |
| --- | --- | --- |
| `GET` | `/vps/v1/resources` | 否 |
| `GET` | `/vps/v1/billings` | 是 |
| `GET` | `/vps/v1/ssh-keys` | 是 |
| `POST` | `/vps/v1/ssh-keys` | 是 |
| `GET` | `/vps/v1/vps` | 是 |
| `POST` | `/vps/v1/vps` | 是 |
| `GET` | `/vps/v1/vps/{instanceId}` | 是 |
| `DELETE` | `/vps/v1/vps/{instanceId}` | 是 |
| `POST` | `/vps/v1/vps/{instanceId}/start` | 是 |
| `POST` | `/vps/v1/vps/{instanceId}/stop` | 是 |

## 接口概览

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/vps/v1/resources` | 列出所有可用 VPS 区域、规格及镜像（`zone → instance_type → images`） |
| `GET` | `/vps/v1/billings` | 分页查询实例计费汇总信息，可按实例筛选 |
| `GET` | `/vps/v1/ssh-keys` | 查询 SSH Key 列表 |
| `POST` | `/vps/v1/ssh-keys` | 创建 SSH 密钥 |
| `GET` | `/vps/v1/vps` | 分页查询 VPS 实例 |
| `POST` | `/vps/v1/vps` | 创建 VPS 实例 |
| `GET` | `/vps/v1/vps/{instanceId}` | 查询单个 VPS 实例 |
| `DELETE` | `/vps/v1/vps/{instanceId}` | 删除 VPS 实例 |
| `POST` | `/vps/v1/vps/{instanceId}/start` | 启动 VPS 实例 |
| `POST` | `/vps/v1/vps/{instanceId}/stop` | 停止 VPS 实例 |

## 推荐调用流程

1. 调用 `/vps/v1/resources` 获取有效的 `zoneId`、`instanceTypeId` 和 `imageId`（该接口无需签名）。
2. 按签名认证规则调用 `/vps/v1/ssh-keys` 查询或创建 SSH Key。
3. 按签名认证规则调用 `POST /vps/v1/vps` 创建实例。
4. 按签名认证规则使用返回的 `instanceId` 查询实例，直到实例进入可用状态。
5. 按签名认证规则按需启动、停止或删除实例。

## 接口说明

### 查询可用 VPS 资源

```http
GET /vps/v1/resources
```

列出所有可用 VPS 区域、规格及镜像，返回按“区域 → 实例规格 → 镜像”组织的资源：

```text
zones[zoneId]
└── zone
└── instanceTypes[instanceTypeId]
    ├── instanceType
    └── images[]
```

实例规格主要包含 CPU、内存、磁盘、带宽、库存、网络计费类型、月价、时价和流量等字段。价格和流量字段在 Swagger 中定义为字符串。

```bash
curl "https://bitgo-vps.expvent.com.cn:1111/vps/v1/resources"
```

### 查询实例计费汇总

```http
GET /vps/v1/billings
```

分页查询 VPS 实例的累计计费记录。此接口必须按[签名认证](#签名认证)携带四个认证 Header。

查询参数：

| 参数 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `pageSize` | int64 | 否 | 每页记录数 |
| `pageNum` | int64 | 否 | 页码 |
| `instanceId` | string | 否 | 按 VPS 实例 ID 筛选 |

```bash
curl "https://bitgo-vps.expvent.com.cn:1111/vps/v1/billings?pageSize=20&pageNum=1&instanceId=instance-id" \
  -H "X-Params: BASE64_ENCODED_WALLET_PARAMS" \
  -H "X-Nonce: RANDOM_NONCE" \
  -H "X-Signature: ECDSA_SIGNATURE_HEX" \
  -H "X-Public-Key: ECDSA_PUBLIC_KEY"
```

响应字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `instanceBilling` | array | 实例计费汇总记录列表 |
| `instanceBilling[].id` | uint64（字符串） | 计费记录主键 |
| `instanceBilling[].createdAt` | string | 记录创建时间 |
| `instanceBilling[].instanceId` | string | VPS 实例 ID |
| `instanceBilling[].instanceName` | string | VPS 实例名称 |
| `instanceBilling[].userid` | string | 所属用户 ID |
| `instanceBilling[].charge` | string | 累计费用，使用 decimal 字符串表示 |
| `totalCount` | int64 | 符合筛选条件的总记录数 |

示例响应：

```json
{
  "instanceBilling": [
    {
      "id": "1",
      "createdAt": "2026-07-23T08:00:00Z",
      "instanceId": "instance-id",
      "instanceName": "dev-vps-01",
      "userid": "user-id",
      "charge": "0.01488"
    }
  ],
  "totalCount": 1
}
```

命令行工具调用（在 `tools/provision-vps` 目录执行）：

```powershell
go run . -action list-billings -page-size 20 -page-num 1
go run . -action list-billings -instance-id instance-id
```

为排查签名接口的请求超时，可为任意签名操作加上 `-print-curl`。工具会将方法、完整 endpoint 和请求体输出为 `curl` 模板；钱包参数、签名和接口公钥会始终脱敏，需由工具在运行时生成：

```powershell
go run . -action create-vps -print-curl -display-name dev-vps-01 -zone-id zone-id -instance-type-id instance-type-id -image-id image-id -ssh-key-id ssh-key-id
```

### 查询 SSH Key

```http
GET /vps/v1/ssh-keys
```

响应字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `sshKeys` | array | SSH Key 列表 |
| `sshKeys[].sshKeyId` | string | SSH Key ID |
| `sshKeys[].displayName` | string | 显示名称 |
| `sshKeys[].description` | string | 描述 |
| `sshKeys[].publicKey` | string | SSH 公钥 |
| `sshKeys[].createdAt` | date-time | 创建时间 |
| `sshKeys[].updatedAt` | date-time | 更新时间 |

```bash
curl "https://bitgo-vps.expvent.com.cn:1111/vps/v1/ssh-keys"
```

### 创建 SSH Key

```http
POST /vps/v1/ssh-keys
```

创建 SSH 密钥。

请求体字段均为必填：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `displayName` | string | 显示名称 |
| `description` | string | 描述 |
| `publicKey` | string | OpenSSH 格式的公钥 |

```bash
curl -X POST "https://bitgo-vps.expvent.com.cn:1111/vps/v1/ssh-keys" \
  -H "Content-Type: application/json" \
  -d '{
    "displayName": "dev-key",
    "description": "Development SSH key",
    "publicKey": "ssh-ed25519 AAAA... user@example.com"
  }'
```

成功响应：

```json
{
  "sshKey": {
    "sshKeyId": "key-id",
    "displayName": "dev-key",
    "description": "Development SSH key",
    "publicKey": "ssh-ed25519 AAAA... user@example.com",
    "createdAt": "2026-07-16T08:00:00Z",
    "updatedAt": "2026-07-16T08:00:00Z"
  }
}
```

> 只提交公钥，不要提交 SSH 私钥，也不要将私钥写入仓库。

### 查询 VPS 列表

```http
GET /vps/v1/vps
```

查询参数：

| 参数 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `pageSize` | int32 | 否 | 每页数量；文档描述默认 20、最大 100 |
| `pageNumber` | int32 | 否 | 页码，从 1 开始 |
| `status` | string | 否 | 按实例状态过滤 |

```bash
curl "https://bitgo-vps.expvent.com.cn:1111/vps/v1/vps?pageSize=20&pageNumber=1&status=InstanceStatusRunning"
```

响应包含：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `instances` | array | 实例列表 |
| `total` | int32 | 总记录数 |
| `pageSize` | int32 | 当前每页数量 |
| `pageNumber` | int32 | 当前页码 |

### 创建 VPS

```http
POST /vps/v1/vps
```

请求字段：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `displayName` | string | 是 | 实例显示名称 |
| `zoneId` | string | 是 | 区域 ID |
| `instanceTypeId` | string | 是 | 实例规格 ID |
| `imageId` | string | 是 | 镜像 ID |
| `sshKey` | string | 否 | SSH Key；Swagger 未明确这是 Key ID 还是公钥内容 |
| `password` | string | 否 | 登录密码 |
| `command` | string | 否 | 实例初始化命令 |

使用 SSH Key：

```bash
curl -X POST "https://bitgo-vps.expvent.com.cn:1111/vps/v1/vps" \
  -H "Content-Type: application/json" \
  -d '{
    "displayName": "dev-vps-01",
    "zoneId": "zone-id",
    "instanceTypeId": "instance-type-id",
    "imageId": "image-id",
    "sshKey": "ssh-key-id",
    "command": "echo ready"
  }'
```

成功响应：

```json
{
  "instanceId": "instance-id"
}
```

> 不要把真实密码、私钥或其他敏感认证材料提交到 Git。建议优先使用 SSH 公钥认证，并通过环境变量或 Secret 管理器注入敏感配置。

### 查询单个 VPS

```http
GET /vps/v1/vps/{instanceId}
```

```bash
curl "https://bitgo-vps.expvent.com.cn:1111/vps/v1/vps/instance-id"
```

响应的 `instance` 对象主要包含：

- 基本信息：`instanceId`、`instanceName`、`zoneId`、`instanceType`
- 计算资源：`cpu`、`memory`
- 镜像信息：`imageId`、`imageName`
- 状态与时间：`status`、`instanceCreateTime`、`expiredTime`
- 网络信息：`publicIpAddresses`、`privateIpAddresses`、`nics`
- 磁盘信息：`systemDisk`、`dataDisks`
- SSH 与计费关联：`keyId`、`moneyId`、`publicKey`
- 记录时间：`createdAt`、`updatedAt`

### 启动 VPS

```http
POST /vps/v1/vps/{instanceId}/start
```

Swagger 将空 JSON 对象定义为必填请求体：

```bash
curl -X POST "https://bitgo-vps.expvent.com.cn:1111/vps/v1/vps/instance-id/start" \
  -H "Content-Type: application/json" \
  -d '{}'
```

成功时返回 HTTP `200` 和一个空 JSON 对象。

### 停止 VPS

```http
POST /vps/v1/vps/{instanceId}/stop
```

请求字段：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `force` | boolean | 否 | `true` 表示强制关机；默认正常关机 |

正常关机：

```bash
curl -X POST "https://bitgo-vps.expvent.com.cn:1111/vps/v1/vps/instance-id/stop" \
  -H "Content-Type: application/json" \
  -d '{"force": false}'
```

强制关机可能造成未保存数据丢失，仅在正常关机失败时使用：

```bash
curl -X POST "https://bitgo-vps.expvent.com.cn:1111/vps/v1/vps/instance-id/stop" \
  -H "Content-Type: application/json" \
  -d '{"force": true}'
```

### 删除 VPS

```http
DELETE /vps/v1/vps/{instanceId}
```

```bash
curl -X DELETE "https://bitgo-vps.expvent.com.cn:1111/vps/v1/vps/instance-id"
```

成功时返回 HTTP `200` 和一个空 JSON 对象。

> 删除操作通常不可逆。调用前应核对 `instanceId`，并确认实例中的数据已经备份。

## 实例状态

Swagger 定义了以下状态值：

| 状态 | 含义 |
| --- | --- |
| `InstanceStatusUnknown` | 未知 |
| `InstanceStatusNew` | 新建 |
| `InstanceStatusActive` | 活跃 |
| `InstanceStatusOff` | 关闭 |
| `InstanceStatusArchive` | 已归档 |
| `InstanceStatusResizing` | 调整规格中 |
| `InstanceStatusDeploying` | 部署中 |
| `InstanceStatusRebuilding` | 重装中 |
| `InstanceStatusReboot` | 重启中 |
| `InstanceStatusRunning` | 运行中 |
| `InstanceStatusStopped` | 已停止 |
| `InstanceStatusBooting` | 启动中 |
| `InstanceStatusReleasing` | 释放中 |
| `InstanceStatusStopping` | 停止中 |
| `InstanceStatusRecycle` | 位于回收站 |
| `InstanceStatusRecycling` | 回收中 |
| `InstanceStatusCreateFailed` | 创建失败 |
| `InstanceStatusImaging` | 制作镜像中 |

## 错误处理

Swagger 只声明了成功响应 `200`，没有描述错误响应结构。客户端应至少：

- 检查 HTTP 状态码，不假设所有响应都是成功结果。
- 为网络超时、TLS 错误和 `5xx` 响应实现有限次数的退避重试。
- 对创建、启停和删除等写操作谨慎重试，避免重复执行。
- 在日志中保留请求时间、路径、HTTP 状态码和服务端错误信息，但不要记录密码、私钥或完整认证材料。
- 创建实例后通过查询接口轮询实例状态，不要仅凭创建请求返回 `200` 判断实例已经可用。

## Git 管理

项目文件由 Git 管理。常用命令：

```bash
git status
git add README.md ai-token-billing.swagger.json
git commit -m "docs: add BitGo VPS API README"
```
