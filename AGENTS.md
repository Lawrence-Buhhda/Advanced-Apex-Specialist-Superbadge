# AGENTS.md

## 项目概览

本仓库保存的是 Salesforce **Advanced Apex Specialist Superbadge** 的分步骤 Apex/Visualforce 代码片段。它不是标准 Salesforce DX 项目：当前没有 `sfdx-project.json`、`force-app`、`package.xml` 或自动化测试脚本；目录按 Superbadge challenge step 拆分，每个目录里放对应步骤要创建或修改的 Salesforce 元数据源码。

根目录下的 `Read.md` 和 `Readmm.md` 目前只是占位内容，其中注释存在编码乱码，不包含可执行或关键项目信息。

## 目录结构

| 路径 | 作用 |
| --- | --- |
| `step2 Update the codebase to use best practices/` | 公共常量类 `Constants.cls`。 |
| `step3 Update the order trigger/` | Order 触发器及 helper，用于订单激活后汇总产品订购数量。 |
| `step4 Update the new product Visualforce page/` | 新建产品的 Visualforce 页面和扩展控制器。 |
| `step5 Create the Test Data Factory/` | 测试数据工厂初版。 |
| `step 6 Increase test coverage with unit tests/` | 测试数据工厂副本以及 `Product2Tests`、`OrderTests`。 |
| `step7  Automate internal announcements when inventory is low/` | 产品库存低于阈值时发布 Chatter Announcement 的触发器、helper 和 queueable。 |

## 核心业务对象与字段

代码依赖 Salesforce 标准对象和若干自定义字段/元数据：

- `Product2`
  - `Family`
  - `Initial_Inventory__c`
  - `Quantity_Ordered__c`
  - `Quantity_Remaining__c`
- `PricebookEntry`
- `Order`
- `OrderItem`
- `Account`
- `Contact`
- `CollaborationGroup`
- 自定义元数据 `Inventory_Setting__mdt`
  - `DeveloperName`
  - `Low_Quantity_Alert__c`

还依赖自定义标签：

- `Label.Select_One`
- `Label.Inventory_Level_Low`

## Apex 组件说明

### `Constants.cls`

集中保存业务常量：

- `DEFAULT_ROWS = 5`
- 订单状态：`Draft`、`Activated`
- Chatter 群组名：`Inventory Announcements`
- 通用错误消息
- `STANDARD_PRICEBOOK_ID = '01sIw000001SfKEIA0'`

注意：`STANDARD_PRICEBOOK_ID` 是硬编码 org Id，移植到其他 org 时很可能需要调整，或改为运行时查询/测试方法获取。

### `orderTrigger.tgr` 与 `OrderHelper.cls`

`orderTrigger` 在 `Order after update` 时调用 `OrderHelper.AfterUpdate`。

`OrderHelper.AfterUpdate` 会筛选状态刚变成 `Activated` 的订单，然后调用 `RollUpOrderItems`。`RollUpOrderItems` 查询相关 `OrderItem`，按 `Product2Id` 汇总所有相关订单项的 `Quantity`，并更新对应 `Product2.Quantity_Ordered__c`。

当前实现会根据传入订单中的产品集合，重新聚合这些产品的全部 `OrderItem` 数量，而不是只增量加本次订单数量。

### `Product2New` 与 `Product2Extension.cls`

`Product2New` 是 Visualforce 页面，用于批量新增产品并展示库存图表。

页面功能：

- 显示 `ChartHelper.GetInventory()` 返回的库存柱状图。
- 默认显示 `Constants.DEFAULT_ROWS` 行产品输入。
- 可通过 Add 按钮追加行。
- Save 按钮调用扩展控制器保存产品和标准价格手册条目。

`Product2Extension.Save` 的行为：

- 过滤掉必填信息不完整的 wrapper。
- 插入 `Product2`。
- 为成功插入的产品创建 `PricebookEntry`。
- 使用 `Constants.STANDARD_PRICEBOOK_ID` 作为标准价格手册 Id。
- 出错时回滚并显示通用错误消息。

注意：页面中 `UnitPrice` 列写成了 `<inputText>`，不是标准 `<apex:inputText>` 或 `<apex:inputField>`，部署或运行时需要确认 Visualforce 是否接受该标签。

### `TestDataFactory.cls`

测试数据工厂用于构造并插入 Superbadge 相关测试数据：

- `ConstructCollaborationGroup`
- `ConstructProducts`
- `ConstructPricebookEntries`
- `ConstructAccounts`
- `ConstructContacts`
- `ConstructOrders`
- `ConstructOrderItems`
- `InsertTestData`
- `VerifyQuantityOrdered`

`step5` 与 `step 6` 中各有一份 `TestDataFactory.cls`，内容当前相同。实际部署到 org 时只能有一个同名 Apex 类，需要选择最终版本或合并。

### `Product2Tests.cls`

测试 `Product2Extension`：

- 设置 `Page.product2New` 为当前页面。
- 创建扩展控制器。
- 调用 `addRows` 并断言行数。
- 填充 5 个产品 wrapper。
- 调用 `save`、`GetFamilyOptions`、`GetInventory`。
- 查询确认 `Test Product1` 被创建。

注意：该测试使用 `@isTest(seeAllData=true)`，对 org 中现有数据、标准价格手册和页面/图表依赖更强。

### `OrderTests.cls`

测试订单激活后的产品订购数量汇总：

- `@testSetup` 调用 `TestDataFactory.InsertTestData(1)`。
- 查询一个 `Order` 和一个 `Product2`。
- 将订单状态改为 `Activated`。
- 重新查询产品并调用 `VerifyQuantityOrdered` 断言 `Quantity_Ordered__c` 增加 `Constants.DEFAULT_ROWS`。

### `Product2Trigger.cls`、`Product2Helper.cls` 与 `AnnouncementQueueable.cls`

`product2Trigger` 在 `Product2 after update` 时调用 `Product2Helper.AfterUpdate`。

`Product2Helper.AfterUpdate`：

- 查询所有 `Inventory_Setting__mdt`。
- 用 `DeveloperName -> Low_Quantity_Alert__c` 建立 map。
- 当产品 `Quantity_Remaining__c` 低于对应 family 阈值时，加入公告列表。
- 调用 `PostAlerts`。

`Product2Helper.PostAlerts`：

- 为每个低库存产品构造 `ConnectApi.AnnouncementInput`。
- 公告过期日期为明天。
- 不发送邮件。
- 文本为产品名加 `Constants.INVENTORY_LEVEL_LOW`。
- 将公告列表交给 `AnnouncementQueueable` 异步发布。

`AnnouncementQueueable`：

- 实现 `Queueable`。
- `execute` 调用 `PostAnnouncements`。
- `PostAnnouncements` 在非测试上下文中调用 `ConnectApi.Announcements.postAnnouncement('Internal', a)`。
- DML 限制不足时会重新 enqueue 剩余公告。

注意：`Product2Helper.COLLABORATION_GROUP` 查询了 `Inventory Announcements` 或测试群组，但当前发布公告时没有使用该查询结果，而是固定传入 `'Internal'`。

## 外部/缺失依赖

当前仓库没有包含以下元数据源码，但代码引用了它们：

- `ChartHelper` Apex 类。
- Visualforce 页面元数据包装文件，例如 `.page` 或 `.page-meta.xml`。
- Apex 类/触发器的 `.cls-meta.xml`、`.trigger-meta.xml`。
- 自定义字段定义。
- 自定义标签定义。
- `Inventory_Setting__mdt` 自定义元数据类型和记录。
- Chatter 群组 `Inventory Announcements`。
- 标准价格手册 Id 对应的 org 数据。

如果要部署到新 org，需要补齐这些元数据，或从目标 Trailhead org 中手动创建/确认。

## 测试与验证

仓库本身没有本地可运行的测试命令。验证通常需要在 Salesforce org 中运行 Apex 测试：

- `Product2Tests`
- `OrderTests`

如果改造成 SFDX 项目后，可使用 Salesforce CLI 执行类似命令：

```bash
sf apex run test --tests Product2Tests,OrderTests --result-format human --code-coverage
```

当前代码中的测试和实现对具体 org 数据存在依赖，尤其是：

- `Constants.STANDARD_PRICEBOOK_ID`
- `Product2Tests` 的 `seeAllData=true`
- `ChartHelper`
- 自定义标签、字段和 metadata records

## 维护注意事项

- 这是分步骤答案仓库，不是单一可部署源码树；同名类在不同步骤目录出现时，后续步骤通常代表更完整版本。
- 修改时优先保持 Superbadge 要求的类名、方法名和注释结构，因为校验可能依赖这些约定。
- 避免在同一个 Salesforce org 中同时部署两个目录里的同名 Apex 类，例如两份 `TestDataFactory.cls`。
- 对价格手册相关逻辑做测试时，最好改用 `Test.getStandardPricebookId()` 或运行时查询，减少 org Id 硬编码风险。
- 对低库存公告逻辑做增强时，重点检查 `Inventory_Setting__mdt.DeveloperName` 是否一定等于 `Product2.Family` 的 picklist value；当前代码直接用 family 值取 map，缺少空值保护。
- `OrderHelper.RollUpOrderItems` 当前未显式过滤 activated orders，只根据传入 activated order 中涉及的产品重新聚合所有相关 `OrderItem`；如业务要求只统计 activated order，需要调整查询条件。

