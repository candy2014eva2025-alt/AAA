# ClearBank 渠道接入产品文档

> 本文档基于现有《The Currency Cloud 渠道接入产品文档》撰写，结构、章节顺序、字段命名习惯与 CC 渠道文档保持一致。ClearBank 与 Currency Cloud 同为"欧盟/英国"渠道（ClearBank 主体注册地为英国），本次为系统新增一条渠道，复用既有的渠道抽象（客户管理 / 账户管理 / 收款 / 付款 / 资金流）。
>
> 参考资料：
> - 现有 CC 接入需求：飞书《The Currency Cloud 渠道接入产品文档》
> - ClearBank Developer Portal：https://clearbank.github.io/uk/docs/

---

## 1. 版本信息

| 时间       | 版本号 | 变更人 | 变更说明                                                     |
| ---------- | ------ | ------ | ------------------------------------------------------------ |
| 2026/06/04 | v0.1   |        | 初稿：基于 CC 接入文档复制章节骨架，替换为 ClearBank 接入需求 |
|            |        |        | 1、客户认证流程新增 ClearBank 渠道分支                         |
|            |        |        | 2、新增 ClearBank 渠道-银行配置项                              |
|            |        |        | 3、新增 ClearBank Real / Virtual 账户申请、来账、付款、资金流  |

---

## 2. 产品方案

### 2.1 客户管理

#### 2.1.1 客户认证流程优化

##### 2.1.1.1 调整项

开拓客户在签约认证时的方式，挂接外币私募的认证方式（CC、ClearBank），业务审核认证方匹配的渠道并下推开户。已开放的 CC 渠道判断逻辑不动；本次新增 **ClearBank 分支**：

- 凡客户开户地区选择"英国（GB）"且开户币种含 GBP 时，系统在认证完成、KYC 通过后，**优先使用 ClearBank 通道开户**（优先级由"渠道优先级配置"控制，详见 2.4.2 通道筛选）。若客户已存在历史 CC 通道账户，按"渠道分配优先级 = 0"的现有逻辑兜底，不影响存量。
- 进件方式：仍走外包 KYC 渠道，材料齐备后由系统调用 ClearBank 开户接口；ClearBank 不接受外包 KYC，**法人/受益人/制裁筛查均由 AT 侧完成后**，再以"已通过尽调"的状态推送至 ClearBank，ClearBank 仅做账户层校验（详见 2.1.2）。
- 如果客户开户地区不在 ClearBank 支持范围（仅支持 UK、直布罗陀、根西岛、马恩岛、泽西岛，且币种必须为 GBP），则按现有 CC 渠道逻辑兜底处理，**不在 ClearBank 通道发起开户**。

##### 2.1.1.2 新增渠道 - 银行配置信息

以入审中台子任务节点维度，在基本信息里新增【企业经营地权属银行】，运营选填。新增字段如下（命名沿用 CC 文档习惯）：

| 字段                | 字段值                                  |
| ------------------- | --------------------------------------- |
| 渠道 ID             | acct\*\*\*\*\*                          |
| 业务标识            | APG 外贸收款                            |
| 渠道名称            | ClearBank                               |
| 渠道商户号          | （ClearBank 分配给 APG 的 InstitutionID） |
| 主体名称            | APG                                     |
| 关联银行 ID         | bk\*\*\*\*,bk\*\*\*\*                   |
| 是否需要渠道进件    | 否                                       |
| 是否需要入账解付    | 否                                       |
| 账户分配优先级      | 0                                       |
| 渠道类型            | CA 渠道                                  |
| 账户获取方式        | 实时                                     |
| **是否提前申请 VA 号** | **是**（ClearBank Real Account 先创建，再为每个客户开 vIBAN） |
| **是否支持多币种**     | **否（仅 GBP）**                          |
| **支持支付方式**       | **FPS、CHAPS、BACS、Cross-Border GBP**    |

#### 2.1.2 KYC 认证

逻辑沿用 CC 章节，区别在于 ClearBank 不提供外包 KYC：

**(1) EKYC 认证通过**

- 如果客户开户渠道命中 ClearBank，则跟 CC 逻辑相同：创建客户认证记录、更新业务子任务认证状态、进入后续 ClearBank 开户。
- 如果命中 CC 渠道，沿用原有逻辑（本次不变）。

**(2) EKYC 认证不通过**

调回 EKYC 审核节点，业务退人员进行重新发起 EKYC。提示【客户报错】，置入【系统报错】等待队列。ClearBank 不允许，**不再触发进件状态变更**，与 CC 一致。

#### 2.1.3 渠道进件审核

- ClearBank 不需要"渠道进件"（不下挂中介机构），因此进件审核节点对 ClearBank 渠道的处理为 **直接跳过**，状态机变更：
  - 渠道进件审核 → ClearBank：跳过（自动通过）
  - 渠道进件审核 → CC：现有逻辑不变
- 系统在渠道审核流转日志里写入一条"ClearBank 渠道无需进件，系统自动通过"的说明，便于追溯。

#### 2.1.4 创建客户的 VA 账户

ClearBank 的账户体系与 CC 不同：

- **Real Account（一级真实账户）**：APG 作为 Institution 在 ClearBank 持有的法人段隔离账户（FCA segregated GBP general account），全公司复用一个，**不为每个客户单独申请**，由运营事前在 ClearBank 平台开好后录入"渠道-银行配置"。
- **Virtual Account / vIBAN（二级虚拟账户）**：挂在 Real Account 之下，**每个客户对应 1 个或多个 vIBAN**。每个 vIBAN 自带独立 sort code + account number + IBAN，可独立收付。
- 创建子账户的判断条件：
  - 客户币种包含 GBP；
  - 客户开户地区在 ClearBank 支持范围内；
  - 客户 KYC 通过且渠道进件审核通过。
- 调用 `POST /v2/Accounts/{accountId}/Virtual`（其中 `accountId` 为 Real Account ID）创建 vIBAN，**一次仅创建一个 GBP vIBAN**（CC 是一次开 5 个 VA，ClearBank 不支持批量、不涉及多币种，只开 1 个 GBP vIBAN）。
- 创建成功后将 vIBAN 信息（sort code、account number、IBAN、accountId、ownerName）落库到"账户中心"。

### 2.2 账户管理

#### 2.2.1 配置变更

新增渠道及通道配置：

- 新增渠道：ClearBank（渠道 ID、关联银行、商户号、API 公私钥对、回调地址 等）
- 新增通道：参见 2.4.2 通道筛选 中"英国本地 GBP-FPS / CHAPS / BACS / Cross-Border"四条通道
- ClearBank 渠道下账户类型仅一种：本地收款账户（UK，GBP）。原 CC 渠道下"USD/EUR/HKD/CAD/全球"账户类型不复制到 ClearBank。

#### 2.2.2 账户申请流程

##### 业务流程图

```
客户                AT                  ClearBank
 │                  │                       │
 │─发起创建账户──▶  │                       │
 │                  │─校验是否命中 CB────  │
 │                  │─创建 vIBAN──────────▶│
 │                  │                       │─创建 vIBAN 成功
 │                  │◀──返回 vIBAN──────────│
 │                  │─落库 vIBAN 信息       │
 │                  │  (sort/acct no/IBAN)  │
 │◀─创建账户成功────│                       │
```

ClearBank 单次开户成功后，仅生成 **1 个本地 GBP vIBAN**，不存在 CC 那种"5 个 VA"概念。

##### 2.2.2.1 vIBAN 账户申请

**（1）收款账户申请**

**1. 产品原型**

本地收款账户 - 英国 - ClearBank Limited

> 在前端"开户银行"下拉中新增"ClearBank Limited"。
>
> - 账户状态：未开通 / 已开通
> - 账户币种：GBP 英镑
> - 支持收款币种：GBP 英镑
> - 收款方式：Faster Payment (FPS) / CHAPS / BACS

**2. 创建账户**

创建 vIBAN URL：`POST /v2/Accounts/{realAccountId}/Virtual`
查询 vIBAN URL：`GET /v2/Accounts/{accountId}/Virtual`

**关联关系**：客户 ID ──1:N── vIBAN ──1:1── Owner（即 sub-account-equivalent），ClearBank 无独立 contact 实体，"持有人/Owner 名"直接挂在 vIBAN 创建参数中。

- 新增客户与 ClearBank 渠道关联表

  | 字段             | 字段说明                                                   |
  | ---------------- | ---------------------------------------------------------- |
  | 主键 ID          |                                                            |
  | 客户 ID          |                                                            |
  | 账户 ID          | ClearBank 返回的 vIBAN accountId                             |
  | 业务标识         |                                                            |
  | 渠道 ID          | ClearBank 渠道                                              |
  | 发起类型         | 系统发起 / 客户发起                                          |
  | 字段映射关系文本 |                                                            |
  | 子账户 ID        |                                                            |
  | 状态             | 详见状态机                                                  |

  **状态机**：

  ```
  待申请 ──▶ 创建账户 ──成功──▶ 创建账户成功 ──▶ 查询 vIBAN ──▶ 已分配 vIBAN
                    │
                    └──失败──▶ 创建账户失败（等待定时重试 / 人工介入）
  ```

  > 与 CC 不同：ClearBank 没有独立的"创建联系人"动作。Owner 信息（姓名、地址、出生日期或公司编号）随 vIBAN 创建请求一并提交。

**产品细节**：

1. 渠道账户类型 + 开户地区（本地账户）+ 开户银行下拉选择，如果是 ClearBank 渠道，则不允许其它地区。
2. 触发 ClearBank 开户后，先校验"客户主体 - 渠道关联表"是否已有 ClearBank 通道、客户类型 (`本地收款账户 + 客户 ID + 开户银行 + 客户 ID + 渠道商户号 ID`) 的记录，若已存在直接展示历史 vIBAN。
3. 如果"客户 - 渠道关联表"中不存在记录，则发起新建 vIBAN：将 `house-account` 模型映射成 ClearBank Real Account，并在其下创建客户级 vIBAN。
4. 如果 ClearBank 返回失败，则更新状态为【创建 vIBAN 失败】，等待定时任务重推。

**接口参数（创建 vIBAN）**：

| 字段               | 字段说明                                                                                                                                | 字段类型 | 要求 | 取值                                                                                          |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------------- | -------- | ---- | --------------------------------------------------------------------------------------------- |
| owner_name         | vIBAN 持有人名称（公司全称 / 个人姓名），不支持中文                                                                                          | string   | M    | 取企业认证营业执照名 / 个人证件姓名                                                              |
| kind               | `YourFunds` 客户自有资金 / `Customer` 客户委托资金（默认值取 `YourFunds`，APG 持有客户名义资金）                                              | string   | M    | YourFunds                                                                                     |
| account_holder_label | vIBAN 在客户端 / 对账单上的显示标签                                                                                                       | string   | O    | 默认取 owner_name                                                                             |
| sort_code          | 6 位 UK sort code，**ClearBank v3 起允许 APG 指定**；不传则由 ClearBank 自动分配                                                              | string   | O    | 不传，由 ClearBank 分配                                                                         |
| account_number     | 8 位 UK account number；不传则由 ClearBank 分配                                                                                          | string   | O    | 不传                                                                                          |
| iban               | 由 ClearBank 自动根据 sort code + account number 生成                                                                                     | string   | -    | 返回时落库                                                                                    |
| ownerType          | `Individual` / `Company` / `Other`                                                                                                      | string   | M    | 大陆/香港企业取 Company；个人取 Individual                                                       |
| countryOfIncorporation | 公司注册地（ownerType=Company 时必填）                                                                                                  | string   | C    | 取企业证件所在地国家代码（ISO 3166-1 alpha-2）                                                  |
| companyRegistrationNumber | 公司注册号（ownerType=Company 时必填）                                                                                              | string   | C    | 大陆取统一社会信用代码，香港取商业登记号                                                          |
| dateOfBirth        | 出生日期（ownerType=Individual 时必填，YYYY-MM-DD）                                                                                       | string   | C    | 取证件出生日                                                                                  |
| nationality        | 国籍（ownerType=Individual 时必填）                                                                                                       | string   | C    | ISO 3166-1 alpha-2                                                                            |
| address.line1      | 注册地址第一行                                                                                                                            | string   | M    | 英文地址，前 35 字符                                                                          |
| address.city       | 城市                                                                                                                                    | string   | M    |                                                                                               |
| address.postalCode | 邮编                                                                                                                                    | string   | M    |                                                                                               |
| address.country    | 国家                                                                                                                                    | string   | M    | ISO 3166-1 alpha-2                                                                            |
| your_reference     | APG 内部请求流水号，幂等键                                                                                                                | string   | M    | 系统生成 UUID                                                                                  |

> 注：
> - ClearBank vIBAN 创建是 **强幂等** 接口，必须传 `X-Request-Id` (= your_reference)，否则重复请求会被风控拒绝。
> - ClearBank 不支持中文，所有名称/地址必须为拉丁字符。
> - 如果命中"客户已在 ClearBank 渠道存在 vIBAN"，**不重复创建**，仅在前端展示。

**（2）收款账户展示**

**产品原型**

- ClearBank | 本地收款账户

  ```
  ClearBank Limited | 本地收款账户
  账户状态：已开通
  账户信息：
    Account Name:  <vIBAN 持有人名>
    Account No:    <8 位 account_number>
    Bank Name:     ClearBank Limited
    Country/Region: UK
    Bank Address:  Borough High Street, London, SE1 1LB
    Clearing System: Faster Payment / CHAPS / BACS
    SORT CODE:     <6 位 sort_code>
    IBAN:          <GB.. ..>
    BIC/SWIFT:     CLRBGB22 (UK 内不强制需要)
    Remarks:       Please include the following contents in your remarks
                   when making a payment: [Buyer Name] [Invoice/Contract Number]
                   [Product]
  账户币种：GBP 英镑
  支持收款币种：GBP 英镑
  入金渠道：FPS / CHAPS / BACS / Cross-Border (GBP)
  ```

> 注：在客户发起收款申请时，如果客户与 ClearBank 渠道关联表中已存在发起记录，则更新发起类型为客户发起。

##### 2.2.2.2 vIBAN 账户管理

vIBAN 一经分配，**sort code + account number 不可变更**（ClearBank 设计如此）。若 vIBAN 持有人信息（地址、负责人）变更，需调用 `PATCH /v2/Accounts/Virtual/{accountId}` 更新 owner 信息。状态变更：

- 启用：`enabled`
- 暂停：`suspended`（暂停后不再接收来账，挂账退回）
- 关闭：`closed`（不可恢复）

##### 2.2.2.3 关联负责人 / Owner

ClearBank 不存在独立的"联系人"实体，因此 **不复用 CC 的"创建联系人"接口**。当我们的客户主体信息变更时，仅触发 vIBAN owner 信息的更新（见上）。无需 `/v2/contacts/create`、`/v2/contacts/find` 这两类接口对接。

### 2.3 收款

#### 2.3.1 收款流程

```
客户             AT                  ClearBank             银行（发送方）
 │                │                       │◀────FPS/CHAPS/BACS 入金──│
 │                │◀── TransactionSettled │
 │                │   webhook  ──────────│
 │                │ 关闭幂等              │
 │                │ 校验签名 + Nonce      │
 │                │ 校验 vIBAN 是否本平台 │
 │                │ 命中客户              │
 │                │ 校验来账金额、币种    │
 │                │  ─ 已分配 vIBAN ?     │
 │                │   ├─是 ▶ 落库         │
 │                │   └─否 ▶ 退账         │
 │                │ 入账审核 (人工/自动)   │
 │                │ 资金入账到 sub-account │
 │◀──通知客户─────│                       │
```

##### 2.3.1.1 来账通知

ClearBank 来账通知（webhook，需提前在 ClearBank Portal 订阅）：

| Webhook 名称                        | 触发时机                                                                                            | APG 处理                                                                                                              |
| ----------------------------------- | --------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `TransactionSettled`                | 入账资金已结算到 vIBAN，资金可用。FPS：秒级；CHAPS：实时；BACS：第 3 个工作日 (D+3) 发出此通知       | 同步落"待补充付款人信息"或"已认领"的来账登记记录，触发后续入账审核                                                       |
| `OutboundHeldTransaction`           | ClearBank 因合规暂扣某笔出账                                                                          | 仅付款流程使用，本节不涉及                                                                                            |
| `TransactionRejected`               | ClearBank 拒绝并退回                                                                                | 同步标记"已退回"，对应来账记录置为 `failed`                                                                            |
| `AccountCreated`                    | vIBAN 异步创建完成                                                                                  | 见 2.2.2.1 异步落库                                                                                                   |
| `BalanceUpdate`（可选）              | vIBAN 余额变更                                                                                       | 用于内部对账校验                                                                                                      |

> 与 CC 不同：ClearBank **没有 `pending`（待批准/合规检查）这种中间态来账通知**。CC 中 `pending` 来源于"等待 compliance check"，ClearBank 的对应行为是 `OutboundHeldTransaction`（仅出账方向）和 `TransactionRejected`（直接拒绝）。

ClearBank 来账 webhook 关键字段（payload）：

| 字段名                  | 字段描述                                            | 字段值                                              | 是否落库存储 |
| ----------------------- | --------------------------------------------------- | --------------------------------------------------- | ------------ |
| 创建时间                | 发文报文创建时间戳                                  |                                                     | 是           |
| 更新时间                | 该来账最后一次状态更新时间                          |                                                     | 是           |
| 来账渠道                | 固定为 ClearBank                                    |                                                     | 是           |
| 渠道交易 ID             | ClearBank 交易 ID（`endToEndId` / `transactionId`） |                                                     | 是           |
| 账户 ID                 | ClearBank 的账户 ID                                 | 本笔交易归属的 vIBAN accountId                       | 是           |
| 客户 ID                 | 客户主体的 APG ID                                   | 通过 account_id 关联客户主体                        | 是           |
| 客户名称                | 客户主体名称                                        |                                                     | 是           |
| vIBAN 号                | 收款 vIBAN 账号                                     | 通过 account_id、更新到客户主体下                    | 是           |
| 状态                    | 来账状态                                            | 待补充付款人信息 / 已完成 / 已退回                   | 是           |
| 交易对手类型            | 公账 / 私账                                         | 报文中带 `debtor.organisationIdentification` 则为公账 | 是           |
| 付款人名称              | 通过 `debtor.name` 取                               |                                                     | 是           |
| 付款人地址              |                                                     | `debtorAgentAddress`                                | 是           |
| 付款人国家/地区         |                                                     | `debtorCountry`                                     | 是           |
| 付款人银行              | sender bic（可能为空）                              | `debtorAgent.bic` / `debtorAgent.name`              | 是           |
| 付款行 SWIFTCODE        | sender bic                                          | `debtorAgent.bic`                                   | 是           |
| 付款行清算号            | sort code / 国家清算号                              | `debtorAgent.clearingSystemMemberId.id`             | 是           |
| sender 国家清算系统     | 来源国家清算系统标识                                | `clearingSystemId` 例如 GBDSC                       | 是           |
| 收款币种                | 固定 GBP                                            | GBP                                                 | 是           |
| 收款金额                | 报文中入账金额                                      | `amount`                                            | 是           |
| 到账币种                | 固定 GBP                                            | GBP                                                 | 是           |
| 到账金额                | 与"收款金额"一致（ClearBank 不参与换汇）             | `amount`                                            | 是           |
| 流水附言                | 报文 remittance 信息                                | `remittanceInformation`                             | 是           |
| 渠道通知                | 来账通知报文体                                      | 完整 webhook payload 落库归档                       | 是           |
| 回单时间                | 报文中 `endToEndId.creationDateTime`                | 或 `settlementDateTime`                             | 是           |
| 结算时间                | 报文中 `settlementDateTime`                         | 视为最终入账时间                                    | 是           |
| 回单影像                | ClearBank 暂不提供影像，本字段置空                  |                                                     | 否           |

中台页面展示同 CC 现有"来账管理"列表，仅新增"渠道 = ClearBank"过滤项。

##### 2.3.1.2 来账通知校验

```
                                ┌───────────────────────────┐
TransactionSettled webhook ──▶ │ 校验 DigitalSignature      │
                                │ + Nonce 去重                │
                                │ + vIBAN 是否本平台命中     │
                                │ + 金额/币种是否一致         │
                                └───────────────────────────┘
                                              │
                                ┌─────────────┴────────────┐
                                │ 命中 vIBAN？              │
                            是 │                          │ 否
                                ▼                          ▼
                          落库"待补充付款人        发起 Return（退账）
                          信息"或"已认领"           调用退账接口
```

**校验规则**：

1. ClearBank webhook 每个请求都带 `DigitalSignature` Header（基于 ClearBank 公钥的 RSA 签名），系统必须验签通过才入库；
2. 校验 `Nonce` 在最近 24 小时内未重复，重复直接 200 OK 但不重复处理（幂等）；
3. 校验 vIBAN `accountId` 在本系统"客户-ClearBank 渠道关联表"内可找到，且状态为"已分配 vIBAN"；
4. 若校验失败：调用 `POST /v3/Payments/Return` 直接退回付款人，并以 `failed` 状态落库；
5. 如果到账金额超过该客户单笔限额 / 日累计限额（详见 2.4.2 通道限额），进入"待审核"等待人工处理。

##### 2.3.1.3 入账审核

判断来账是否命中客户：

1. 来账 vIBAN 已分配 → 命中客户 ID → 直接进入"已认领"。
2. 来账 vIBAN 不在分配表中（理论上不会发生，做兜底）→ 进入"待补充"，由运营核实。

入账审核状态机：

```
待认领 ──▶ 已认领 ──▶ 已入账 ──▶ 已完成
              │
              └──失败──▶ 待补充付款人信息 ──▶ 退回
```

##### 2.3.1.4 退款

**产品要求**：与 CC 章节一致。运营在中台"来账管理 → 详情"操作【退款】。

**接口**：`POST /v3/Payments/Return`，路径取来账原 `endToEndId`，金额可全额或部分（ClearBank 支持部分退款）。

**支持退款的入金类型**：FPS / CHAPS / BACS / Cross-Border（GBP）入账，**ClearBank 原币原路退回**，不允许换币（单币种渠道）。

**退款状态机**：

```
待补充受益人信息 ──▶ 审核中 ──▶ 退款中 ──▶ 已退款 / 退款失败
                                              │
                                              ▼
                                       【返回】（撤销）
```

**受益人展示字段（退款）**：

ClearBank 退款 = 原路返回，受益人信息直接取来账 sender 报文的字段，不允许运营手动修改受益人主账户。仅可修改"附言"和"金额"。

| 字段                    | 字段说明                              | 是否必填 | 赋值                                                  |
| ----------------------- | ------------------------------------- | -------- | ----------------------------------------------------- |
| 受益人名称              | 取来账 sender.name                    | 必填     | sender.name                                            |
| 受益人国家/地区         | 取来账 sender.country                 | 必填     | sender.country                                         |
| 受益人详细地址          | 取来账 sender.address                 | 必填     | sender.address                                         |
| 受益人账户币种          | 固定 GBP                              | 必填     | GBP                                                    |
| 受益人 sort code        | 取来账 debtorAgent.sortCode           | 必填     | debtorAgent.clearingSystemMemberId                     |
| 受益人 account_number   | 取来账 debtorAccount                  | 必填     | debtorAccount.identification                           |
| 退款金额                | 默认取来账原金额，可改成部分          | 必填     | amount ≤ 原入账金额                                    |
| 退款附言                | 退款原因                              | 必填     | 运营填写                                              |

**接口字段（POST /v3/Payments/Return）**：

| 接口字段             | 字段说明                  | 是否必填 | 字段赋值                                              |
| -------------------- | ------------------------- | -------- | ----------------------------------------------------- |
| originalEndToEndId   | 原入账 endToEndId         | 必填     | 取来账落库的 endToEndId                                |
| amount               | 退款金额                  | 必填     | 客户填写                                              |
| currency             | 固定 GBP                  | 必填     | GBP                                                    |
| reasonCode           | 退款原因代码（ISO 20022） | 必填     | 默认 `RETN`，运营可改                                  |
| reasonInformation    | 退款原因附言              | 必填     | 运营填写                                              |
| accountId            | 退款资金来源 vIBAN        | 必填     | 命中客户的 vIBAN accountId                             |

### 2.4 付款 / 提现

账户维护中，提现账户和供应商账户中针对大陆 CNY 相关账户维护逻辑不变，付款流程中，提现结汇和付款结汇流程逻辑不变；ClearBank **不支持换汇**（单币种渠道），因此所有"结汇"动作走 CC 渠道或原有路径，本节仅描述 **GBP 出境的非结汇付款 / 提现**。

#### 2.4.1 供应商账户 / 提现账户维护

- 本地 local 付款：ClearBank 渠道仅对客开放 **英国（GBP）** 一种 local 付款。CC 原有的"美国/欧盟/香港/加拿大"四地 local 不复制到 ClearBank。
- 当 ClearBank 渠道开放时，账户地区下拉额外屏蔽以上四个国家，账户币种额外屏蔽以上四个币种（USD、EUR、HKD、CAD）。
- 发起付款时，到账币种在现有逻辑下，屏蔽以上四个币种，仅保留 GBP。

##### 2.4.1.2 供应商账户维护

**（1）客户录入 → 2024-12-11 复用 CC 已有调整：调整英国地区 GBP 和非 GBP 的银行账户信息**

| 模块             | 字段类型       | 字段             | 字段说明                                                                                                                                                                                                                                                                                                                                                                                                                  |
| ---------------- | -------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 供应商账户信息   | 通用字段       | 供应商名称       | 交易相对方资质管理                                                                                                                                                                                                                                                                                                                                                                                                       |
|                  |                | 账户地区         | 英国（GB）。其它地区屏蔽 ClearBank，由 CC 兜底                                                                                                                                                                                                                                                                                                                                                                            |
|                  |                | 账户币种         | 仅 GBP                                                                                                                                                                                                                                                                                                                                                                                                                  |
|                  |                | 账户类型         | 公账 / 私账                                                                                                                                                                                                                                                                                                                                                                                                              |
|                  | 条件字段       | 银行卡号         | 1) 账户为公账时录入：8 位 account_number，UK BACS-style，无 IBAN；<br>2) 账户为私账时录入：同上；<br>3) 跨境（payment_type=Cross-Border）时录入 IBAN（GBxxxxxxxx）<br>4) 账户为公账时，结构化字段为 `BeneficiaryAccountNumber`；账户为私账时同；如果是国际 IBAN，则使用 `BeneficiaryIban`，不再使用 `account_number` |
|                  | 银行账户信息   | 通用字段         | 开户银行 SWIFTCODE：可选填，国内 UK 间 FPS / CHAPS / BACS 不强依赖，BIC 不传走 sort_code + account_number。                                                                                                                                                                                                                                                                                                                |
|                  |                | 开户行           | 1) 账户币种为 GBP，且 ClearBank 渠道下，无需开户行的国家、地区、城市等地址（FPS/CHAPS/BACS 仅靠 sort_code 寻址）<br>2) 账户币种为 GBP，且选择 Cross-Border（GBP 出境）渠道下，需要开户行的国家、地区、城市、地址                                                                                                                                                                                                            |
|                  | 条件字段       | 银行清算号        | 1) 账户为公账时，sort_code（6 位）必填，账户币种为 GBP，账户类型为 FPS / CHAPS / BACS<br>2) 账户为私账时，sort_code（6 位）必填，账户类型为 FPS / CHAPS / BACS<br>3) Cross-Border（GBP 出境）时不需要 sort_code，仅需要 IBAN + 收款行 BIC                                                                                                                                                                                  |
|                  |                | 银行卡号（Bank Code）|                                                                                                                                                                                                                                                                                                                                                                                                                          |
| 其他犯罪信息（含险） | 业务字段       | -                | 结构化 -                                                                                                                                                                                                                                                                                                                                                                                                                  |

> 子表头："国家地区"（英国、加拿大、墨西哥）

**（2）渠道筛选（+ 寻找付款行）**

如果产品中的"账户清算行 / 路由方"中带有"ClearBank Limited"，则自付款指令 → APG `realaccount` 子账户 → 客户 sub-account 抵扣。

只需要报备客户的 vIBAN（sub-account 对应）即可。

#### 2.4.2 发起付款 / 提现（非结汇）

##### 1、产品流程和状态机

```
客户                AT                                ClearBank
 │  确定付款       │                                       │
 │ 查询付款账户   │ 返回可选择币种名单                      │
 │ 选择付款账户   │                                       │
 │ 选择到账币种   │ 查询客户应商账户                        │
 │                │ 是否绑定供应商账户                      │
 │                │  ─ 否 ─▶ 跳转绑定供应商账户流程         │
 │                │  ─ 是 ─▶ 选择供应商账户                 │
 │ 点击下一步     │                                       │
 │                │ 通道筛选（命中 ClearBank-GBP-FPS/CHAPS/BACS/Cross-Border）│
 │                │ 是否涉及汇兑                            │
 │                │ ─ 是 ─▶ 走 CC 通道或退回（CB 不参与换汇）│
 │                │ ─ 否 ─▶ 进入付款准备                    │
 │                │ 风控审核+客户白名单                     │
 │                │ 选择安全验证方式（短信 + 邮件）          │
 │ 提交付款       │ 系统按产品类型支付到客户 vIBAN          │
 │                │ 查询 sub-account（vIBAN）付款币种下的余额│
 │                │ 不足 ─▶ 拒绝；足额 ─▶ 调用 ClearBank 付款│
 │                │                            ──────────▶│ 调用 POST /v3/Payments/{FPS|CHAPS|BACS}
 │                │                            ◀──同步 ACK│
 │                │ 等异步 webhook 落账                     │
 │◀──付款回执─────│                                       │
```

**状态机**：付款下单 → 待支付 → 已发起 → 待结算 → 已结算 / 已退回 / 已失败

##### 通道字段（通道维护）

按照 CC 文档的"通道信息字段"结构，新增 4 条 ClearBank 通道：

**通道 1：英国本地 GBP FPS 通道**

| 通道信息字段          | 字段说明                          |
| --------------------- | --------------------------------- |
| 通道 ID               | 系统生成                          |
| 通道性质              | 本地                              |
| 渠道 ID               | ClearBank 的渠道 ID                |
| 渠道名称              | ClearBank                          |
| 收款账户类别          | /                                 |
| 收款机构 list         | /                                 |
| 通道清算币种          | GBP 英镑                          |
| 通道收付款支持币种    | GBP 英镑                          |
| 通道假期日历          | 英国假期（包含 Bank Holiday）     |
| 限额                  | 单笔 1,000,000 GBP（FPS 上限受 FPS 网络上限约束，2024 起为 £1m）|
| 通道清算国家 / 地区   | 英国、直布罗陀、根西岛、马恩岛、泽西岛 |
| 通道优先级            | 0                                 |

**通道 2：英国本地 GBP CHAPS 通道**

| 通道信息字段          | 字段说明                          |
| --------------------- | --------------------------------- |
| 通道 ID               | 系统生成                          |
| 通道性质              | 本地                              |
| 渠道 ID               | ClearBank 的渠道 ID                |
| 渠道名称              | ClearBank                          |
| 通道清算币种          | GBP 英镑                          |
| 通道收付款支持币种    | GBP 英镑                          |
| 通道假期日历          | 英国 Bank Holiday + CHAPS 营业日（Mon-Fri 06:00-18:00）|
| 限额                  | 无单笔上限                        |
| 通道清算国家 / 地区   | 英国                              |
| 通道优先级            | 1（CHAPS 费用高，单笔 > FPS 限额时启用）|

**通道 3：英国本地 GBP BACS 通道**

| 通道信息字段          | 字段说明                          |
| --------------------- | --------------------------------- |
| 通道 ID               | 系统生成                          |
| 通道性质              | 本地                              |
| 渠道 ID               | ClearBank 的渠道 ID                |
| 渠道名称              | ClearBank                          |
| 通道清算币种          | GBP 英镑                          |
| 通道收付款支持币种    | GBP 英镑                          |
| 通道假期日历          | 英国 Bank Holiday + BACS 3 日处理周期 |
| 限额                  | 单笔 20,000,000 GBP                 |
| 通道清算国家 / 地区   | 英国                              |
| 通道优先级            | 2（结算 T+3，仅用于批量付薪等场景）  |

**通道 4：ClearBank GBP Cross-Border 通道**

| 通道信息字段          | 字段说明                          |
| --------------------- | --------------------------------- |
| 通道 ID               | 系统生成                          |
| 通道性质              | 跨境                              |
| 渠道 ID               | ClearBank 的渠道 ID                |
| 渠道名称              | ClearBank                          |
| 通道清算币种          | GBP 英镑                          |
| 通道收付款支持币种    | GBP 英镑（出境到非 UK 国家）     |
| 通道假期日历          | SWIFT / Target2 假期              |
| 限额                  | 单笔 1,000,000 GBP                   |
| 通道清算国家 / 地区   | 除英国以外的国家                  |
| 通道优先级            | 1                                 |

##### 通道筛选

CC 文档的通道筛选规则保持不变，本次仅新增"如果到账币种为 GBP 且收款行在英国境内，优先 ClearBank-FPS（金额 ≤ £1m 时）；> £1m 时优先 CHAPS；批量付薪场景走 BACS；出境（收款行不在 UK）时走 Cross-Border GBP"。其余币种仍由 CC 兜底。

##### 4、客户出账拆账演算

- POBO（即从客户 vIBAN 出账）
  - 步骤 1：付款准备 - 系统从客户 vIBAN（sub-account 等价物）扣款到 Real Account house pool；
  - 步骤 2：调用 ClearBank `POST /v3/Payments/FPS|CHAPS|BACS` 发起出账；
  - 步骤 3：webhook `TransactionSettled` 回调，系统标记"已结算"，扣完手续费；
  - 步骤 4：如出账失败，`TransactionRejected` 或 `OutboundHeldTransaction` 回调，资金原路回到 vIBAN，置"已退回"状态。

- 不 POBO（house account 名义出账）
  - 步骤 1：APG 根据扣款金额从客户各个 CA 子户扣款到 payout 中间户（同 CC 现有逻辑）；
  - 步骤 2：判断 APG 的 Real Account 余额是否充足，不足挂起预警等运营调拨；
  - 步骤 3：充足直接按照对客承诺的最终汇出金额发起付款 / 提现，需要调整的（GBP 单币种本身无汇率，仅金额）发起。

##### 渠道调用（ClearBank）

**付款接口 URL**：

- FPS：`POST /v3/Payments/FPS`
- CHAPS：`POST /v3/Payments/CHAPS`
- BACS：`POST /v3/Payments/Bacs`（注意 ClearBank 用 `Bacs`，混用首字母大小写）
- Cross-Border（GBP 出境）：`POST /v3/Payments/CrossBorder/GBP`

**关键字段（以 FPS 为例）**：

| 接口字段                            | 字段说明                                            | 字段赋值                                          |
| ----------------------------------- | --------------------------------------------------- | ------------------------------------------------- |
| accountId                           | 付款 vIBAN（POBO）/ Real Account（非 POBO）         | 取客户 vIBAN id 或 APG Real Account id            |
| accountIdentifier.sortCode          | 付款方 sort code（由 accountId 衍生，通常不需传）   |                                                   |
| amount                              | 付款金额（GBP）                                     | 最终汇出金额                                      |
| reference                           | 付款附言                                            | 取付款附言                                        |
| endToEndId                          | APG 端到端 ID，幂等键                                | 系统生成 UUID                                     |
| paymentScheme                       | `FPS` / `CHAPS` / `BACS` / `CrossBorderGBP`         | 通道筛选决定                                      |
| creditor.entityType                 | `Individual` / `Company`                            | 由供应商账户类型决定                              |
| creditor.companyName                | 收款公司名（公账时必填）                            | 供应商名称                                        |
| creditor.firstName                  | 名（私账时必填）                                    | 供应商名称的名字部分                              |
| creditor.lastName                   | 姓（私账时必填）                                    | 供应商名称的姓                                    |
| creditor.address.line1              | 详细地址（CHAPS / Cross-Border 必填）               | 供应商地址                                        |
| creditor.address.city               | 城市                                                |                                                   |
| creditor.address.country            | 国家                                                | 收款国 ISO 3166-1 alpha-2                          |
| creditor.address.postalCode         | 邮编                                                |                                                   |
| creditorAccount.sortCode            | 收款方 sort code                                    | FPS / CHAPS / BACS 必填                            |
| creditorAccount.accountNumber       | 收款方 account number                               | FPS / CHAPS / BACS 必填                            |
| creditorAccount.iban                | 收款方 IBAN                                         | Cross-Border 必填                                  |
| creditorAccount.bic                 | 收款行 BIC                                          | Cross-Border 必填，UK 内可选                       |
| chargeBearer                        | `SHA` / `OUR` / `BEN`                               | UK 内 N/A；Cross-Border 默认 `SHA`                 |
| purposeCode                         | ISO 20022 purpose code                              | 按付款指令映射                                    |
| ultimateDebtor.name                 | 终极付款人（POBO 时填客户名，非 POBO 时填 APG）     |                                                   |
| ultimateCreditor.name               | 终极收款人                                          | 供应商名称                                        |

> 与 CC 不同：ClearBank 不需要"创建受益人"前置接口。付款指令直接带受益人信息一次性下发，**无受益人持久化实体**。如果业务侧希望沉淀受益人，请保留在 APG 内部"供应商账户"表中。

##### 5、付款回调处理

- 现金提示：ClearBank 提交 `POST /v3/Payments/*` 接口同步返回 `202 Accepted`，最终结果走 webhook 异步推送。
  - `TransactionSettled`：付款已落地，更新状态 `completed`；
  - `TransactionRejected`：付款被拒，更新状态 `failed`，资金回到 vIBAN，触发回滚通知；
  - `OutboundHeldTransaction`：因合规暂扣，更新状态 `held`，运营介入。
- 来账重发机制：完成提交后定时任务每天一次（默认晚 22:30）扫描 `held / submitted_no_callback` 状态付款，调用 `GET /v3/Payments/{endToEndId}` 主动查询，覆盖 webhook 漏推场景。
- 流转日志：每一次状态变更落"付款状态流转表"，便于客服查询。

##### 4、来账复发现表

中台显示：付款 - 付款指令查询 - 查看详情，新增【渠道流转明细】显示子菜单。数字描述如下：

1. 付款发起、事项处理中、扣款失败 4、付款发起中 5、内部渠道发送中 6、失败-内部渠道未送达 7、失败 8、成功-中台缺 10、成功-渠道已送 11、成功-内部渠道送达 12、成功

#### 2.4.3 资金流

付款 / 提现（非结汇）内外部资金流推演

ClearBank 单币种，不涉及换汇环节，资金流比 CC 简洁：

**有垫资情况**（仍按 CC 现有规则；ClearBank 不参与垫资判断，仅做出账动作）：

```
Step 1：发起付款申请
  扣款金额 1000 GBP
  手续费 0 GBP（UK FPS 渠道，0 手续费的情况下；CHAPS 按合同收）
  实际汇出金额 1000 GBP

Step 2：扣款
  客户 GBP 收款 vIBAN (CA 账户) ─1000 GBP─▶ 客户 GBP 收款 vIBAN（CB 渠道）
                                            └─1000 GBP─▶ payout 中间户

Step 3.1：付款失败
  payout 中间户 ─1000 GBP─▶ 客户 GBP vIBAN (CA 账户) ─1000 GBP─▶ 客户 GBP 收款账户

Step 3.2：付款成功
  payout 中间户 ─1000 GBP─▶ APG GBP CB 渠道户 ─1000 GBP─▶ APG GBP CB 收款户
                                                          └─1000 GBP─▶ 受益人 (ClearBank 出境)
  手续费：APG 手续费 CA 户 + N GBP（CHAPS 时记录）
```

**无垫资情况**：

```
Step 1：发起付款申请
  扣款金额 1000 GBP
  手续费 0 GBP（FPS 0 手续费）
  实际汇出金额 1000 GBP

Step 2.1：付款失败
  payout 中间户 ─1000 GBP─▶ 客户 GBP vIBAN (CA 账户)（资金原路退回）

Step 2.2：付款成功
  payout 中间户 ─1000 GBP─▶ APG GBP CB house pool ─1000 GBP─▶ 受益人
  手续费 CA 户 + 0 GBP（FPS 渠道）

CB
└─ APG CB GBP 账户 +1000 GBP（house）
└─ 客户 CB GBP vIBAN −1000 GBP（POBO 模式下）
```

> 注：
> 1. ClearBank 单一 GBP 币种，**不存在跨币种垫资问题**，垫资仅出现在多币种 CC 渠道。
> 2. 如果业务侧未来扩展 ClearBank 多币种（ClearBank Multi-Currency Accounts，已对部分客户开放 USD/EUR），需要追加章节。

---

## 3. 中台页面修改

中台页面：

(1) 付款 → 付款指令查询 → 查看详情，新增【渠道流转明细】Tab 显示子菜单。数字描述如下：

1. 付款发起、2. 事项处理中、3. 扣款失败、4. 付款发起中、5. 内部渠道发送中、6. 失败 - 内部渠道未送达、7. 失败、8. 成功 - 中台缺、10. 成功 - 渠道已送、11. 成功 - 内部渠道送达、12. 成功

(2) 收款 → 来账管理：新增"渠道筛选"下拉值 = ClearBank。

(3) 渠道配置：新增"ClearBank"渠道下属四条通道（FPS / CHAPS / BACS / Cross-Border-GBP）。

(4) 退款管理：在状态机的"待补充受益人信息"列上保留【返回】按钮（ClearBank 退款不允许编辑受益人，但允许金额 / 附言修改、可【返回】撤销）。

---

## 4. 待办 / 与 CC 的关键差异速查

| 维度                | Currency Cloud                            | ClearBank                                                  |
| ------------------- | ----------------------------------------- | ---------------------------------------------------------- |
| 渠道性质            | 多币种、全球                              | 单币种 GBP、UK 本地 + UK 跨境出境                            |
| 一次开户产出账户数  | 5 个 VA（USD / EUR / GBP / CAD / Global） | 1 个 GBP vIBAN                                              |
| 联系人体系          | 必须 `/v2/contacts/create`              | 无独立 contact，owner 信息随 vIBAN 一并提交                  |
| 是否外包 KYC        | 支持                                      | 不支持，APG 侧完成所有合规筛查                              |
| 换汇                | 支持，`/v2/conversions/create`            | 不支持                                                     |
| 来账通知            | `pending` → `completed`，存在 compliance 中间态 | 仅 `TransactionSettled` / `TransactionRejected`；出账方向才有 `OutboundHeldTransaction` |
| 退款                | `/v2/payments/...`（按原路）              | `POST /v3/Payments/Return`，支持部分退款                    |
| 付款接口            | `/v2/payments/create`                     | `/v3/Payments/{FPS|CHAPS|Bacs|CrossBorder/GBP}` 共 4 个     |
| 鉴权                | username + api_key                        | mTLS + RSA `DigitalSignature` Header + `X-Request-Id` 幂等  |
| 受益人持久化        | 必须先 `/v2/beneficiaries/create`         | 不需要，付款指令一次性带受益人                              |
| 限额                | 见 CC 通道                                | FPS £1m、CHAPS 无上限、BACS £20m、Cross-Border £1m         |

---

## 5. 待澄清问题（TODO）

1. ClearBank 开户主体 APG 在 ClearBank 是签约 "Embedded Banking" 还是 "Agency Banking"？涉及到 Real Account 是否可以直接用于 POBO（Embedded 可以，Agency 需走代理）。
2. ClearBank Multi-Currency Accounts（USD/EUR 段）是否在本期开放给客户？本期文档默认不开放。
3. ClearBank 出境 GBP（CrossBorder/GBP）的费率与代理行（哪家 correspondent bank）是否已合同确定？影响"对客手续费"计算。
4. ClearBank webhook 重试规则：ClearBank 默认 5 次指数回退后停止推送。是否需要 APG 端在 D+1 跑批主动 `GET /v3/Payments/{endToEndId}` 兜底？
5. Real Account 与 vIBAN 命名是否要在客户端展示？建议仅展示 vIBAN 信息，Real Account 内部使用。
6. ClearBank 平台对接的 Sandbox 环境与 Production 环境的切换流程，是否需要新增运营配置开关？
