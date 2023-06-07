我们是网易云音乐的云原生团队，负责运维基于 GitOps 的 CICD 平台 Horizon。在使用 Horizon 部署应用之前，云音乐采用传统的云主机部署模式，运维操作基于 Ansible ，有着以下问题
1. **版本控制不足**：Ansible 很难做到对整个基础架构的完整版本控制，导致问题排查和回滚困难。
2. **协作困难**：团队成员之间的协作不够顺畅，缺乏一致性和可追溯性。
3. **依赖多、复杂**： Ansible 运行的依赖项比较多，需要在本地或远程服务器上安装一些必要的依赖，这也增加了配置和安装的复杂性。
为解决以上问题，我们需要其它的更加先进、优秀的运维方法。
 
## 什么是GitOps

GitOps 是一种 DevOps 方法，通过使用 Git 作为单一的事实来源，将应用程序的生命周期管理自动化。GitOps 可以实现基础设施即代码、自动化部署、自动化测试和自动化监控，从而提高软件开发的速度和质量。GitOps 的核心理念是将所有的代码、配置和基础设施的变更都提交到 Git 仓库中，通过工具将这些变更应用到生产环境中。这种方法可以减少手动干预和错误，并提高开发团队的可见性和可控性。


## GitOps 的优势

通过以上描述，我们可以直观地感受到 GitOps 相对于传统手工运维有着许多优势：

1. **基础设施管理**：传统运维通常需要手动管理基础设施和应用程序的部署和配置，而GitOps使用版本控制系统管理基础设施和应用程序的配置，实现自动化管理。
2. **部署一致性**：传统运维流程中，不同环境中的应用程序版本可能不一致。GitOps通过使用版本控制系统管理应用程序配置，确保在不同环境中部署的应用程序版本一致。
3. **可视化和自动化**：GitOps提供了一种可视化和自动化的方法来管理基础设施和应用程序的部署，而传统运维通常需要手动执行这些操作。
4. **故障恢复和回滚**：GitOps允许团队使用版本控制系统来管理基础设施和应用程序的配置，这意味着可以轻松地回滚到以前的版本来恢复系统，以及快速恢复故障。传统运维通常需要手动回滚和恢复故障。
5. **多环境部署和多团队协作**：GitOps允许团队使用Git来管理不同环境中的配置，例如开发、测试和生产环境。它还支持多团队协作，团队可以在同一个Git仓库中进行合作管理。传统运维通常需要使用不同的工具来管理不同环境中的配置，团队之间的协作也可能需要额外的沟通。

## 内部实践

### 管控面运维

云原生团队已经实现了一套基于 Gitlab CI 的 GitOps 部署规范流程，用于管控组件的部署。每个运行中的组件都有一个对应的代码仓库和配置仓库。开发人员将修改推送到代码仓库后，经过审核后合并到主分支并打出对应的镜像。然后，只需要修改 GitOps 部署仓库，就可以将修改部署到线上。

1. 首先，开发人员需要 Fork 一份配置仓库的代码，通常是一个基于 Helm 的 Chart。在该仓库中，只需修改 values.yaml 文件，通常就可以满足开发人员的需求。而 templates 文件则打包放在 Harbor 中。在 Chart.yaml 文件的 dependency 字段中引用这些 templates，这样我们就可以在多个环境中部署相同的组件。通过将 templates 抽象出来，我们可以将统一的配置放在 templates 中，而在各自的配置仓库中，则可以放置不同的配置，兼顾统一和灵活。

2. 开发者在修改配置仓库后，需要将修改推到仓库中，并向 master 分支提交 MR。这时会触发流水线，验证修改是否正确。

3. 通过验证后，开发者需要请求团队中的其他人帮忙 review，通过检查后，合入主分支。内部 Gitlab 会有一个机器人，监听到 MR 后，会在 MR下方进行评论，开发者和评审者通过评论与机器人交流，输入`+1`，代表同意，当获得足够的票数之后，机器人会自动合入 MR。这个过程中，开发者不需要拥有该仓库的写权限，能有效防止误操作。![[Screenshot_20230607_182144.png]]

4. 合入主分支，会触发另一条流水线，将修改应用到 Kubernetes 中。

5. 开发者如果观察到本次修改存在任何问题，并且无法快速恢复，那么可以找到上一次成功的发布，重新跑一下对应的流水线，就可以快速回滚。
![[41648f9ff45cbd973fe9c8683a93808e.webp]]

以上流程，我们通过引入测试流水线和 Reviewer 降低了出错的概率，通过重新运行部署流水线完成了快速回滚，通过 commit 记录每次部署的操作者，通过自动化部署减少了人工介入，解决了绝大部分传统部署过程中的问题。

### Horizon 应用部署

经过上面的实践，我们已经见识到了 GitOps 的强大。所以我们将 GitOps 应用到了 Horizon 中。Horizon 中的 GitOps 逻辑与上述逻辑略有不同，它底层使用 Argo CD 作为 GitOps 引擎。
GitOps 引擎会监听 GitOps 仓库和Kubernetes资源。Horizon 会为其中的每个应用创建对应的 GitOps 仓库，GitOps 仓库有两个 branch，一个为 master 分支，通过 Argo CD与 Kubernetes 中的相关资源对应；另一个为 gitops 分支，gitops 分支中包含了用户修改之后还未生效的配置。

当用户发布或者构建发布时，Horizon 会将 gitops 分支合并到 master 分支中。因为 Argo CD 监听了 GitOps 仓库中的 master 分支，所以 Argo CD 会感知到变化。通过和 Kubernetes 中的相关资源比对，如果不一致则触发一次同步，将 GitOps 仓库中的配置，应用到 Kubernetes 中。完成一次部署。
![[4816fe01ac1a4194244b20898111b860.webp]]
Horizon在合并分支时，会将相关参数，比如操作人、时间、修改等信息记录到 description 中，方便审阅。
#### GitOps 仓库

Horizon 基于 Helm 部署应用，所以 GitOps 仓库约等于 Helm Chart + JsonSchema。Horizon 使用 JsonSchema 渲染表单，获取用户输入；使用用户输入值与 Helm Chart 一起渲染 Manifest，部署应用。
下图为 GitOps 仓库结构，`Chart.yaml` 文件是 Helm Chart 的标准，其中引用了真正的部署 Helm Chart，我们称之为模板
```yaml
apiVersion: v2  
name: demo  
version: 1.0.0  
dependencies:  
- name: deployment  
version: v0.0.1-ec06d596  
repository: https://horizon-harbor-core.horizon.svc.cluster.local/chartrepo/horizon-template
```
`application.yaml` 包含了用户通过 JsonSchemaForm 表单填写的数据
```yaml
deployment:
  app:
    envs:
    - name: test
      value: test
    spec:
      replicas: 1
      resource: x-small
```
`pipeline-output.yaml` 包含了 CI 阶段的输出，最主要的是其中包含了构建出的镜像名
```yaml
deployment:
  image: library/demo:v1
  git:
    branch: master
    commitID: 28992d8f35a6ef38d59181080b3728df9540d8d6
    url: https://github.com/horizoncd/springboot-source-demo.git
```
`sre.yaml` 是提供给 SRE 修改的，在一些特殊情况下，SRE 可以通过该文件重载 `values.yaml` 中的数据
```yaml
deployment: 
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: cloudnative/demo
            operator: In
            values:
            - "true"
```
system目录下的文件记录了部署的元信息， `env.yaml` 记录了部署环境相关的信息
```yaml
deployment:
  env:
    environment: local
    region: local
    namespace: local-1
    baseRegistry: horizon-harbor-core.horizon.svc.cluster.local
    ingressDomain: cloudnative.com
```

![[Pasted image 20230607175545.png]]

## Pull还是Push

以上两个部署例子，采用了 GitOps 的两种风格，Push 和 Pull。Push 的特点是通过 Git CI，将修改推送到目标系统中；Pull 则是需要一个 Agent，监听 Git 仓库变更，并将变更应用到目标系统中。

那我们在实际中应该选择哪种方式呢？

采用 Push 风格会有如下特点：
- 简单灵活。这种部署方式已经被众多知名的 CI/CD 工具所采用，例如 Jenkins CI，AWS Code系列。易于同其他的脚本或工具集成。
- 需要权限。需要获取部署环境的凭证，确保自己 CD 流水线的权限最小化。防止被攻击时，暴露太高的权限。
- 无法感知部署状态。如果开发者手动修改 Kubernetes 资源，造成实际资源与 Git 仓库配置存在差异，Push 风格的 Gitops 无法感知。

采用 Pull 部署风格会有如下特点：
- 安全。因为 GitOps 代理运行在 Kubernetes 集群中，因此仅需要最小的权限用于部署。
- 一致性。Pull 风格的 GitOps 会监听 Kubernetes 资源和 Git 仓库，Kubernetes 资源发生变更时与 Git 仓库进行比对，保证所有的修改都以 Git 仓库为准。

所以在实际生产中我们应该尽量选择 Pull 风格的 GitOps。


## GitOps是银弹吗？

上面论述了许多 GitOps 的优势，那么 GitOps 真的是银弹，可以解决所有问题吗？

1. 无法自动更新：上面我们提到，代码仓库中打出镜像后，手动修改配置仓库，运行 CI 部署到环境中。那是否可以通过代码仓库的 CI，自动更新配置仓库？可以但是不推荐，因为 Git 遇到冲突时需要手动解决，设想如果有多个 CI 进程同时修改配置仓库，那么最终必然会导致冲突出现。
2. 修改不便：设想，如果我们有多个环境，每个环境都对应一个配置仓库，那么一旦需要修改一个统一的值，那么需要修改所有仓库。
3. 密码管理：Git 仓库中数据都是明文显示，并且 Git 仓库会记住所有的历史修改，所以放在 Git 仓库中的明文信息应该加密。虽然社区里面开发了一些类似于 [git-secret](https://github.com/sobolevn/git-secret) 的工具，但使用起来还是不太方便。
4. 回滚没有

所以 GitOps 并不是银弹，开发者需要基于自身情况判断，选择最适合自己的方案。

## 结语
GitOps 作为一种 DevOps 方法，还在快速发展中。我们也会持续关注、尝试 GitOps 领域的最新技术。如果你也想使用 GitOps 或者对其感兴趣，可以[加入我们](https://github.com/horizoncd/horizon#contact-us)，和我们一起讨论。

## FAQ

我们在与外部用户的交流中发现，许多用户使用 istio 与 deployment 实现流量灰度。如下图所示，流量通过 virtual services 中的规则找到对应的 pods。
在下图所示的情况下，所有的 deployment 都对应同一个 service。如果将 deployment 和 service 放在同一个部署仓库中，因为 istio 的金丝雀发布要求创建一个新版本的 deployment，所以会同时创建新的 service 和 deployment。但是因为 virtual service 中的配置，所以不能创建新的service。这样一来就产生了冲突。
![[6847c91db7e58bfa4887ed27d8abffb3.png]]
这里我们给出几种解法。

### 多Host
问题的关键在于 virtual service 中写死了 service 地址，我们可以通过修改 virtual service 中的 host，来达成每个 deployment 与一个 service 对应。可以在同一条流水线中，创建新的 deployment 和 service，等待 pod 全部启动后，修改 virtual service，将新的 service 配置到 virtual service 中，并配置流量比例。

### 单“Deployment”
deployment 的升级过程中需要创建新的 deployment，如果我们升级时不需要创建新的 deployment，也可以解决这个问题。这里我们引入 argo rollout，argo rollouts是一个Kubernetes控制器和一套CRD，为Kubernetes提供了先进的部署能力，如蓝绿、金丝雀、金丝雀分析、实验和渐进式交付功能。argo rollout的升级过程中，只会有一个 rollout，所以我们可以保证只有一个 service 存在。argo rollout 对 istio 流量灰度有完整的[支持](https://argo-rollouts.readthedocs.io/en/stable/features/traffic-management/istio/)。
argo Rollout 相当成熟，已经在 Horizon 内部进行了大规模实践，我们推荐使用。


