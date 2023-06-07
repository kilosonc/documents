我们是网易云音乐的云原生团队，负责运维基于 GitOps 的 CICD 平台 Horizon。该平台涵盖了多个管控组件，而这些组件的运维过程也存在许多挑战，运维容易出错、没有版本控制、回滚难、协作困难、自动化程度不高。我们需要解决这些问题，以最大限度地减少运维错误，并保证出现故障时能够快速恢复。此外，我们也需要满足多人协作的需求，提高自动化水平等。

## 什么是GitOps

## 管控面运维

为了解决上述问题，我们的云原生团队已经实现了一套基于 Gitlab CI 的 GitOps 部署规范流程，用于管控组件的部署。每个运行中的组件都有一个对应的代码仓库和配置仓库。开发人员将修改推送到代码仓库后，经过审核后合并到主分支并打出对应的镜像。然后，只需要修改 GitOps 部署仓库，就可以将修改部署到线上。

1. 首先，开发人员需要 Fork 一份配置仓库的代码，通常是一个基于 Helm 的 Chart。在该仓库中，只需修改 values.yaml 文件，通常就可以满足开发人员的需求。而 templates 文件则打包放在 Harbor 中。在 Chart.yaml 文件的 dependency 字段中引用这些 templates，这样我们就可以在多个环境中部署相同的组件。通过将 templates 抽象出来，我们可以将统一的配置放在 templates 中，而在各自的配置仓库中，则可以放置不同的配置，兼顾统一和灵活。

2. 开发者在修改配置仓库后，需要将修改推到仓库中，并向 master 分支提交 MR。这时会触发流水线，验证修改是否正确。

3. 通过验证后，开发者需要请求团队中的其他人帮忙 review，通过检查后，合入主分支。

4. 合入主分支，会触发另一条流水线，将修改应用到 Kubernetes 中。

5. 开发者如果观察到本次修改存在任何问题，并且无法快速恢复，那么可以找到上一次成功的发布，重新跑一下对应的流水线，就可以快速回滚。

![[Pasted image 20230606193220.jpg]]

以上流程，我们通过引入测试流水线和 Reviewer 降低了出错的概率，通过重新运行部署流水线完成了快速回滚，通过 commit 记录每次部署的操作者，通过自动化部署减少了人工介入，解决了绝大部分传统部署过程中的问题。

## Horizon 应用部署

经过上面的实践，我们已经见识到了 GitOps 的强大。所以我们将 GitOps 应用到了 Horizon 中。Horizon 中的 GitOps 逻辑与上述逻辑略有不同，它底层使用 Argo CD 作为 GitOps 引擎。
GitOps 引擎会监听 GitOps 仓库和Kubernetes资源。Horizon 会为其中的每个应用创建对应的 GitOps 仓库，GitOps 仓库有两个 branch，一个为 master 分支，通过 Argo CD与 Kubernetes 中的相关资源对应；另一个为 gitops 分支，gitops 分支中包含了用户修改之后还未生效的配置。

当用户发布或者构建发布时，Horizon 会将 gitops 分支合并到 master 分支中。因为 Argo CD 监听了 GitOps 仓库中的 master 分支，所以 Argo CD 会感知到变化。通过和 Kubernetes 中的相关资源比对，如果不一致则触发一次同步，将 GitOps 仓库中的配置，应用到 Kubernetes 中。完成一次部署。

![[Pasted image 20230606201754.jpg]]

Horizon在合并分支时，会将相关参数，比如操作人、时间、修改等信息记录到 description 中，方便审阅。
Horizon中的回滚，是将对应的修改再合并到 master 分支。

## GitOps 的优势

经过以上两个例子，我们可以直观地感受到 GitOps 相对于传统手工运维有着许多优势：

1. 基础设施管理：传统运维通常需要手动管理基础设施和应用程序的部署和配置，而GitOps使用版本控制系统管理基础设施和应用程序的配置，实现自动化管理。
    
2. 部署一致性：传统运维流程中，不同环境中的应用程序版本可能不一致。GitOps通过使用版本控制系统管理应用程序配置，确保在不同环境中部署的应用程序版本一致。
    
3. 可视化和自动化：GitOps提供了一种可视化和自动化的方法来管理基础设施和应用程序的部署，而传统运维通常需要手动执行这些操作。
    
4. 故障恢复和回滚：GitOps允许团队使用版本控制系统来管理基础设施和应用程序的配置，这意味着可以轻松地回滚到以前的版本来恢复系统，以及快速恢复故障。传统运维通常需要手动回滚和恢复故障。
    
5. 多环境部署和多团队协作：GitOps允许团队使用Git来管理不同环境中的配置，例如开发、测试和生产环境。它还支持多团队协作，团队可以在同一个Git仓库中进行合作管理。传统运维通常需要使用不同的工具来管理不同环境中的配置，团队之间的协作也可能需要额外的沟通。

## GitOps是银弹吗？

现在 Horizon 已经在 Github 上开源，如果你对 GitOps 或者 Horizon 感兴趣，可以加入我们的[微信群](https://github.com/horizoncd/horizon#contact-us)与我们进一步交流

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


