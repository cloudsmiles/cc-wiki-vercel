# app迁移不同sid 技术方案

## 背景

[【控制台运营】appID在不同sid主体间迁移（12月）](https://zegocloud.feishu.cn/docx/TXLVd9MyrofgqzxAqb4c2gtznPc)

## 流程设计

迁移app到不同sid

![Untitled](后端开发/工作/Design/app迁移不同sid%20技术方案/Untitled.png)

![Untitled](后端开发/工作/Design/app迁移不同sid%20技术方案/Untitled%201.png)

## 接口设计

迁移检查app接口

http://api-mgr.zego.cloud/project/1116/interface/api/32663

迁移app接口

http://api-mgr.zego.cloud/project/1116/interface/api/32657

查询迁移记录接口

http://api-mgr.zego.cloud/project/1116/interface/api/32660

## 数据库设计

迁移记录表

migrate_app_history

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| id | int | 自增主键 |
| content | text | 操作内容 |
| created_by | varchar(32) | 创建人 |
| created_at | bigint | 创建时间 |

## 工时评估

| 任务 | 操作人 | 工时 |
| --- | --- | --- |
| 迁移app接口 | 文锋 | 1.5d |
| 迁移记录接口 | 文锋 | 0.5d |
| 对接先智app迁移接口 | 文锋 | 0.5d |
| 前端联调 | 文锋 | 1.5d |