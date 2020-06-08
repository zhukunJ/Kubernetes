# Kubernetes技术实践


- **搭建生产环境可用的Kubernetes集群**
      Kubernetes的主要组件以及一些基本概念，部署一个生产环境可用的Kubernetes集群，安装Kubernetes常用的插件，在生成环境如何去升级已有的Kubernetes集群，如何轻松管理多个Kubernetes集群；

- **Kubernetes编排基础** 
      Pod基础与进阶，包括Pod的原理，API对象、Pod资源限制、Pod健康检查、生命周期管理、共享卷、Pod调度相关知识及生产环境建议及操作实践；Kubernetes常用资源对象，主要包括ReplicaSet、Deployment、StatefulSet、DeamonSet、Job等等；权限管理，包括ServiceAccount介绍、RBAC基于角色的访问控制等等；

- **Kubernetes容器持久化存储** 
      PV、PVC、StorageClass以及本地化持久存储相关的知识，使用Ceph分布式存储和阿里云云盘作为Kubernetes容器持久化存储；
- **Kubernetes网络选型** 
      容器跨主机网络、Kubernetes网络模型与CNI网络插件、常用的三层网络方案、以及Kubernetes服务发现，包括集群内部服务发现(Servcie)、集群外部服务访问(Ingress)、Headlss服务、CoreDNS以及4/7层服务发现实践；

- **Kubernetes包管理工具Helm** 
      搭建私有Helm仓库、如何使用Helm仓库、如何编写Helm Charts，为我们网站的微服务编写charts，并上传等到私有Helm仓库；

- **部署企业级私有仓库** 
      部署企业级代码管理平台Gitlab和容器私有仓库Harbor，学完这部分内容你将掌握如何通过容器化的方式部署Gitlab和Harbor，以及Gitlab和Harbor的基本使用；

- **CI/CD流水线** 
      CI/CD流水线技术，认识几种CI/CD产品，介绍他们如何配置、使用，使用CI/CD流水线将我们的网站部署到Kubernetes集群上，通过使用CI/CD流水线来提升我们的开发、测试、部署流程的自动化水平；

- **Kubernetes日志管理** Kubernetes日志处理的基本原理，以及如何通过EFK收集Kubernetes集群的日志及微服务产生的日志；

- **Kubernetes监控** Kubernetes监控原理，如何部署Prometheus，并使用它进行集群及应用监控，结合AlertManager实现故障告警；

- **通过Istio实现服务治理（进行中）** 如何使用Istio实现微服务服务的蓝绿部署、金丝雀部署，如何对服务进行限流、熔断和链路追踪;
;
- **Kubenetes基于ingress实现蓝绿发布**  灰度及蓝绿发布是为新版本创建一个与老版本完全一致的生产环境，在不影响老版本的前提下，按照一定的规则把部分流量切换到新版本，当新版本试运行一段时间没有问题后，将用户的全量流量从老版本迁移至新版本;


