# Proxysvrd转发方式（QQGame）

不足: 私有协议，不统一，proxysvrd之间需要同步路由
优势: 业务自主性强
描述: 后台服务以Proxysvrd为中心，服务首先注册自身到Proxysvrd，业务请求都通过Proxysvrd转发（TCP），Proxysvrd管理路由信息