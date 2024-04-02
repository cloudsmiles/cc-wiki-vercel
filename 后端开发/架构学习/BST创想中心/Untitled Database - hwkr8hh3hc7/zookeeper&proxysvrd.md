# zookeeper&proxysvrd

不足: 私有协议，不统一，自己维护名字服务
优势: 业务自主性强
描述: 强化目前后台zookeeper的功能逻辑，将所有业务服务注册到zookeeper节点，通过zookeeper提供统一的名字服务。业务服务之间TCP网状直连或Proxy中转。