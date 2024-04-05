# zim多端登录&zpns策略

## 需求背景

https://zegocloud.feishu.cn/docx/ERU8dFsgDoJUVkx16UZczScynDf

## 业务流程

1. 离线配置自定义策略
- 只需展示已配置了离线推送证书的厂商
    - 前端根据/zpns/provider_list接口返回的厂商列表，配置时，只展示对应的厂商列表
    - 其中谷歌需要特殊转换一下，/zpns/provider_list返回的是google，策略列表返回的是fcm
- 策略列表查询
- 策略保存
- 策略删除
1. zim多端登录参数
- 仅限专业版、旗舰版可配置
    - 前端通过/v2/get_im_config接口中的ver字段判断（1 免费版； 2 专业版； 3 旗舰版）
- 多端登录配置保存
- 多端登录配置查询

## 方案设计

需求的配置依赖noah系统提供，所以控制台实现权限的校验和接口转发即可。以下为接口列表

1. 接口列表

| 名称 | 描述 | noah接口文档 | 前端文档 |
| --- | --- | --- | --- |
| 离线策略列表查询 | 接口透传，对appid权限校验 | http://api-mgr.zego.cloud/project/388/interface/api/32010 | https://zegocloud.feishu.cn/wiki/AaCFw50FZiSw47klbOacvOoIn5f#part-SQ1cdXsq5oFgBwxu3tdcXv9cnnf |
| 离线策略新建和编辑 | 接口透传，对appid权限校验 | http://api-mgr.zego.cloud/project/388/interface/api/32017 | https://zegocloud.feishu.cn/wiki/AaCFw50FZiSw47klbOacvOoIn5f#part-KsCsdkJGyozhSwxNsnQcBQzHnof |
| 离线策略删除 | 接口透传，对appid权限校验 | http://api-mgr.zego.cloud/project/388/interface/api/32016 | https://zegocloud.feishu.cn/wiki/AaCFw50FZiSw47klbOacvOoIn5f#part-V64AdX3TWoGSY6xOPHIcfa82nKc |
| 多端登录配置新建和编辑 | 接口透传，对appid权限校验 | http://api-mgr.zego.cloud/project/388/interface/api/32023 | http://api-mgr.zego.cloud/project/474/interface/api/32096 |
| 多端登录配置查询 | 接口透传，对appid权限校验 | http://api-mgr.zego.cloud/project/388/interface/api/18066 | http://api-mgr.zego.cloud/project/474/interface/api/32097 |

## 工时评估

| 序号 | 功能模块 | 开发人员 | 评估工作量 |
| --- | --- | --- | --- |
| 1 | 策略接口 | 文锋 | 0.5d |
| 2 | 多端登录接口 | 文锋 | 0.5d |
| 3 | 与manage系统联调 | 文锋，万骏 | 0.5d |