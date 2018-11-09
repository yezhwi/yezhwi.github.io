---
layout:     post
title:      转：基于 GitLab 的 Code Review 教程
subtitle:   
date:       2018-11-09
author:     Yezhiwei
header-img: img/WechatIMG38.jpeg
catalog: true
category: java
tags:
    - Git
    - GitLab
---

> 本文由 ken.io 创作
> 
> 本文原文链接：https://ken.io/note/gitlab-code-review-tutorial

### 一、前言

#### 1、本文主要内容

* GitLab Code Review 机制说明
* Git Workflow 与 Git Code Review Workflow
* GitLab Code Review 配置说明
* GitLab Code Review 流程演示
* GitLab For IDE 插件介绍（JetBrains 等等）

#### 2、GitLab Code Review 机制

GitLab 可以在分支合并的时候支持两种方式：

* 在本地将源分支  (Source branch)代码合并到目标分支（Target branch）然后 Push 到目标分支（Target branch）
* 将源分支(Source branch)Push 到远端，然后在 GitLab 指定目标分支（Target branch）发起Merge Request，对目标分支（Target branch）拥有 Push 权限的用户执行 Merge 操作，完成合并。
也就是说，使用 GitLab 进行 Code Review 就是在分支合并环节发起 Merge Request，然后 Code Review 完成后将代码合并到目标分支。

#### 3、本教程适用环境信息

| 工具/环境	 | 版本
| ------ | ------ |
| GitLab | GitLab.com、GitLab 社区版皆可 |
| IDE | JetBrains（IntelliJ IDEA、PyCharm、PhpStorm、WebStorm、RubyMide、AppCode、CLion、GoLand、DataGrip、Rider、Android Studio 等等） |

虽然 Code Review 不一定非要结合 IDE 来做，但是也不得不感谢 JetBrains 开发了几乎覆盖所有主流编程语言的IDE

> JetBrains Tools 目前覆盖的主流语言有：C/C++、C#、DSL、F#、Go、Groovy、Java、JavaScript、TypeScript、Kotlin、Objective-C、PHP、Python、Ruby、Scala、SQL、Swift、VB.NET（排名不分先后）

### 二、GitLab Code Review 配置

#### 1、Code Review 工作流

* 通用 Git 工作流说明

![](https://img.ken.io/blog/git/workflow/master-develop-feature-release-kbrb.png)

1. 需求确认后，从 master 创建 develop 分支
1. 开发人员从 develop 分支创建自己的 feature 分支进行开发
1. master 分支发生变更，需要从 master 分支合并到 develop 分支、可以考虑定期合并一次
1. feature 分支合并到对应的 develop 分支之前，需要从 develop 分支合并到 feature 分支
1. feature 分支合并到对应的 develop 分支之后，发布到测试环境进行测试
1. develop 分支在测试环境测试通过之后，合并到 release 分支并发布到预发布环境进行测试
1. release 分支在预发布环境验证通过后，合并到 master 分支并发布到生产环境进行验证

分支名称约定：

| 分支类型 | 名称格式 | 说明 |
| ------ | ------ | ------ |
| Master | master | 有且只有一个| 
| Release | release-* | \*可以是班车发布日期也可以是需求名称缩写，也可以根据需要只用一个 release 分支 |
| Develop | develop-* | \*通常是班车发布日期或者需求名称缩写，也可以根据需要只用一个 develop 分支 |
| Feature | feature-{username}-* | 开个人员个人分支，*通常是班车发布日期或者需求名称缩写 |

* Code Review 环节选定

以上述 Git 工作流为例，开发人员在 Feature 分支进行开发，开发完成后 Merge 到 Develop 分支进行测试。

那么最适合做 Code Review 就是 Feature 分支合并到 Develop 分的环节。

![](https://img.ken.io/blog/git/workflow/master-develop-feature-release-codereview-kbrb.png)

#### 2、GitLab Repository 配置

GitLab 仓库相关配置以 gitlab.com 为例，本篇内容如果没有特别注明，也同样适用于私有化部署的GitLab CE 版本

* GitLab 新建仓库&创建分支

![](https://img.ken.io/blog/gitlab/codereview/gitlab-codereview-new-project-kbrb.png)

仓库地址：https://gitlab.com/ken-io/test

![](https://img.ken.io/blog/gitlab/codereview/gitlab-codereview-new-branch-kbrb.png)

新建分支：

release(from master)

develop-test(from master)、

feature-ken-test(from develop-test)

* Protected Branches 配置

为了保证必须以 Merge 的方式变更 develop 分支、release 分支、以及 master 分支，我们需要对 Push 以及 Merge 权限进行限制

菜单：Settings->Repository Settings 然后展开 Protected Branches 选项
https://gitlab.com/ken-io/test/settings/repository

![](https://img.ken.io/blog/gitlab/codereview/gitlab-codereview-settings-protected-branches-kbrb.png?v1)

这里，我们限制分支，所有的开发人员对 develop 分支、release 分支、以及 master 分支均无 Push 权限，只能以 Merge 方式合并到对应分支，而且只有 Maintainers（Masters） 组的用户有 Merge 权限。

对于通配符的分支保护，我们需要给 Maintainers（Masters）组的用户 Push 权限，不然无法新建对应格式的分支。对于固定名称的分支：master、release，可以限制所有角色都没有 Push 权限。

### 三、GitLab Code Review 示例

* 变更 Feature 分支

在线修改 feature-ken-test 分支 README.md 文件，为 Merge Request 提供基础

![](https://img.ken.io/blog/gitlab/codereview/gitlab-codereview-branch-feature-readme-update-kbrb.png)

这里随意更新一行内容，然后 Commit changes 即可。

* 创建 Merge Request

菜单：Merge Requests，然后点击：New Merge Request

![](https://img.ken.io/blog/gitlab/codereview/gitlab-codereview-new-merge-request-kbrb.png)

Source branch 选择：feature-ken-test

Target branch 选择：develop-test

然后：Compare branches and continue

![](https://img.ken.io/blog/gitlab/codereview/gitlab-codereview-merge-request-submit-kbrb.png)

操作项/填写项说明：

| 操作项/填写项	 | ken.io 的说明 |
| ------ | ------ |
| Title | 标题，没有特殊要求保持默认即可 |
| Description	 | 描述，需要将变更的需求描述清楚，最好附件 Code Review 要点 | 
| Assignee | 分配到的人，被分配到的人将会收到邮件通知，跟 Merge 权限没有必然关系，仍然是项目的Maintainers（Masters）角色拥有 Merge 权限 | 
| Milestone | 里程碑，如果没有可不选 | 
| Label | 标签，如果没有可不选 | 
| Approvers user	 | 批准人/审批人，必须为项目所在组成员，如果选择了批准人，那此次合并必须经由批准人批准 | 
| Approvers group | 批准人组，方便同时选择多个批准人 | 
| Approvals required | 最少批准个数，如果选了个3个批准人，Approvals required 设置为1，那么只需要1个批准人批准即可 | 
| Source branch | 源分支，跟上一步骤选择一致，这里主要用于确认 | 
| Target branch | 目标分支，跟上一步骤选择一致，这里主要用于确认 | 

Approvers 选项暂不适用于 Gitlab 的最新稳定版(11.1.4)，期望后续可以支持。
这里填写好 Description，选择 Assignee，然后 Submit merge request 即可。

* Merge Request 操作

![](https://img.ken.io/blog/gitlab/codereview/gitlab-codereview-merge-request-view-kbrb.png)

Merge Request 创建之后就会转到该页面，被分配到的人（Assignee）会收到邮件提醒，如果需要多个人进行 Code Review，只要将该页面的链接发给其他项目成员即可。项目成员可以查看变更并评论，只不过按照之前的配置，只有 Maintainers（Masters）角色的成员才有 Merge 的权限。

![](https://img.ken.io/blog/gitlab/codereview/gitlab-codereview-merge-request-changes-comment-kbrb.png)

在 Changes 选项卡中，我们可以看到所有的变更。将光标移动到行号处会出现评论按钮，我们可以点击评论按钮发起评论，这个评论是对项目成员可见的，大家可在讨论区进行讨论。最终讨论发起者有权将讨论标记为已解决 resolved

![](https://img.ken.io/blog/gitlab/codereview/gitlab-codereview-merge-request-discussion-kbrb.png)

当所有的问题已解决之后（如果选择了审批人也需要审批通过），Maintainers（Masters）成员点击 Merge 完成合并即可。

![](https://img.ken.io/blog/gitlab/codereview/gitlab-codereview-merge-request-merged-kbrb.png)

Merge 完成之后，可以选择 Remove Source Branch 等操作。

develop 分支合并到 release 分支，以及 release 分支合并到 master 是不需要经过 Code Review 的，直接 Merge 即可。这里就省略了。

### 四、IDE Merge Request 插件使用介绍

前面介绍了通过 GitLab 网页创建 Merge Request 并发起 Code Review，但作为开发人员，还是结合 IDE 来使用会更顺手，GitLab 提供了相关的 api，只要我们创建响应的 token，就可以供 IDE 插件来访问 GitLab，以便使用 IDE 代替在网页上操作。

#### 1、GitLab Access Token

菜单：User Settings->
Access Tokens 进入 Access Token 添加页面

![](https://img.ken.io/blog/gitlab/codereview/gitlab-codereview-personal-access-tokens-add-kbrb.png)

| 项 | 说明 |
| ------ | ------ |
| Name | 名称，根据自己喜好来即可 |
| Expires at | 过期时间，最远可以选择到10年后，根据自己需要填写即可 |
| Scopes | 范围，这里选择 api 就够用了 |

创建完成后，麻烦暂时保存 token 。因为一旦刷新或者重开页面，token 就不可见了。 

#### 2、JetBrains IDE GitLab 插件使用

JetBrains 提供了诸多 IDE：IntelliJ IDEA、PyCharm、PhpStorm、WebStorm、RubyMide、AppCode、CLion、GoLand、DataGrip、Rider、Android Studio 等等，如无意外，都适用 GitLab 插件。

安装以下两个插件即可：
Gitlab Projects：https://plugins.jetbrains.com/plugin/7975-gitlab-projects
Gitlab Integration：https://plugins.jetbrains.com/plugin/7319-gitlab-integration

* 安装 GitLab 插件

Settings->Plugins 进入 Plugins 管理页

![](https://img.ken.io/blog/jetbrains/settings/jetbrains-ide-plugins-kbrb.png)

点击 Browse repositories 并搜索 gitlab

![](https://img.ken.io/blog/jetbrains/settings/jetbrains-ide-plugins-browse-repositories-gitlab-kbrb.png)

安装 Gitlab Projects 以及 Gitlab Integration，然后重启 IDE 生效

* 配置 GitLab

在 Settings 界面搜索 GitLab Settings

![](https://img.ken.io/blog/jetbrains/settings/jetbrains-ide-gitlab-settings-kbrb.png)

填写 GitLab Server Url、Access Token，然后点击 Add New One 完成添加
如果是私有化部署的 GitLab，换成对面的域名或者 IP+Port 即可

* Create Merge Request

Clone 项目 feature-ken-test 分支到本地，变更后 push 到 origin。
然后在菜单中选择：VCS->Git->Git Lab-> Create Merge Request

![](https://img.ken.io/blog/jetbrains/settings/jetbrains-ide-gitlab-create-mergerequest-kbrb.png)

这里相当于我们在 GitLab 网页上进行创建操作，只不过少了一些选项，也暂不支持 Approvers 相关选项。
选择目标分支，被分配的人，填写好 Title、Description 然后点击 OK 即可。
Merge Request 创建完成后，插件会在右下角提示，点击链接即可跳转到 Merge Request 页面

> 如果提示冲突，请先将目标分支代码合并到当前分支

* Merge Request Manage

项目成员在菜单中选择：VCS->Git->Git Lab-> List Merge Request

![](https://img.ken.io/blog/jetbrains/settings/jetbrains-ide-gitlab-list-mergerequest-kbrb.png)

在这里可以看到待处理的 Merge Request，选中后点击 Code Review 就可以呼出 Merge Request 操作面板

![](https://img.ken.io/blog/jetbrains/settings/jetbrains-ide-gitlab-mergerequest-manage-kbrb.png)

| 按钮 | 说明 | 
| ------ | ------ |
| Diff | 查看所有变更文件及差异 |
| Comments | 查看、添加评论 |
| Assign to me | 将跟进人指给自己 |
| Merge | 执行 Merge |

* Merge Request Diff

Diff 界面说明：

![](https://img.ken.io/blog/jetbrains/settings/jetbrains-ide-gitlab-mergerequest-diff-control-kbrb.png)

左侧是本次合并的 commit 记录，右侧是本次合并的文件。双击对应文件即可查看差异明细

![](https://img.ken.io/blog/jetbrains/settings/jetbrains-ide-gitlab-mergerequest-diff-file-kbrb.png)

* Merge Request Comments

![](https://img.ken.io/blog/jetbrains/settings/jetbrains-ide-gitlab-mergerequest-comments-control-kbrb.png)

Comments 界面可以查看指定 Merge Reuqest 评论信息，也可以添加评论，双击可以查看完整评论内容。
但是不支持针对代码行发起讨论、对讨论标记为已解决等。

GitLab 插件还是更适用于 Create Merge Request、或者对于较为简单的提交进行 Code Review。如果需要讨论等功能，还是建议在 GitLab 页面上进行操作

#### 3、其他 IDE GitLab 插件使用

* Visual Studio

Visual Studio GitLab 插件：https://marketplace.visualstudio.com/items?itemName=MysticBoy.GitLabExtensionforVisualStudio

* Visual Studio Code

Visual Studio Code GitLab 插件：https://marketplace.visualstudio.com/items?itemName=jasonn-porch.gitlab-mr

* Atom

Atom GitLab 插件：https://atom.io/packages/gitlab

### 五、备注

* 延伸阅读

GitLab 安装部署教程：https://ken.io/note/centos7-gitlab-install-tutorial