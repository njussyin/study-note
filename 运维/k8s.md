# Kubernetes是什么？
容器集群管理系统。

# Kubernetes 特点
可移植: 支持公有云，私有云，混合云，多重云（multi-cloud）<br>
可扩展: 模块化, 插件化, 可挂载, 可组合<br>
自动化: 自动部署，自动重启，自动复制，自动伸缩/扩展<br>

# 优势

# 架构
## master模块
### API Server
处理API操作，k8s中所有组件都和API Server进行连接，组件相互之间不进行独立连接。<br>
它本身是一个可水平扩展的部署组件。
### controller
控制器，管理集群状态。比如自动对容器进行修复、自动进行水平扩张。<br>
它虽然只有一个active，但是可以进行热备。
### scheduler
调度器，完成调度操作。比如将用户提交的container，依据其对CPU和memory的需求，找到一个合适节点放进去。<br>
同控制器一样可以热备。
### etcd
分布式存储系统，API Server需要的原信息都放置在这里，其本身是高可用的，它保证整个master组件的高可用性。

## node模块
