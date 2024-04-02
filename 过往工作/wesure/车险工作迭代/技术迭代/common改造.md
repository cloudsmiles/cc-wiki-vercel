# common改造

Status: Completed

同步工作

- 新的公共项目地址 [点击此处](https://gitlab.inwesure.com/AT-car-insurance/carcommon)
- common规范 [[]]
- 迁移方案是新旧共存，如果迁移到新的地址，需要找文宇添加gopath（看能不能先批量加）
- 3.42.0迭代之后，旧的common项目禁止提交，只能提交到新的项目
- 我写了个脚本，批量替换，使用方法就是在项目根目录运行。（但为了不误改，所以还有一些匹配不了，不过改动量经过测试已经很少了）

方案实施

- [x]  carcommon同步最新代码
- [x]  旧common停止更新
- [x]  各服务运行run.sh脚本
- [x]  各服务修改gopath