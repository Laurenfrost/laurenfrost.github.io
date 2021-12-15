---
title: GitHub Actions 笔记
date: 2021-12-14 22:02:00
tags: DevOps
category: 日常踩坑
---

这段时间里总是在跟 GitHub Actions 打交道。GitHub Actions 这个功能本身也在不断的迭代中，而且它的文档还有很多不完备的地方，导致我踩了很多坑。

今日心血来潮，记录一下我能回忆起来的要点，以备日后使用。

![GitHub Actions 的执行逻辑](./github-actions-flowchart-Page-2.drawio.png)

## 踩坑记录

### 添加 workflows 文件要谨慎

GitHub Actions 只会执行相应 branch 里`.git/workflows/**`目录下的工作流文件。这也就意味着：

1. 每一个需要执行 Actions 的 branch 都必须添加 相应的 workflows 文件。

2. 一旦 workflows 需要变更，就必须变更所有 branch 里的每一个 workflows 文件，否则不同 branch 间的动作情况将会不一致，并且 Actions 的界面里触发的工作流将会分属不同的 workflows，进一步引发混乱。

3. workflows 文件中[`on.<push|pull_request>.<branches|tags>`](https://docs.github.com/en/actions/learn-github-actions/workflow-syntax-for-github-actions#onpushpull_requestbranchestags)虽说可以指定由哪些 branch 来触发这个工作流，但这本质是个 filter，你首先得把它放到相应的 branch 里，让它变得能被触发才行。

    就像这里 [Github Action: Trigger a workflow from another branch](https://github.community/t/github-action-trigger-a-workflow-from-another-branch/120770/3) 说的：
    > We have no method to trigger a workflow in a branch by the push event occurs in another branch.
    > 
    > Every workflow run has two important properties branch/tag ref (github.ref) and commit SHA (github.sha). Each workflow has only one branch/tag ref and only one commit SHA.
    > 
    > When you push a new commit to the feat branch in your repository, generally GitHub will do the following things before triggering workflows.
    > 
    > 1. In the feat branch, check if there is any workflow file in the **latest version (commit SHA)** of the repository files. If no any workflow file, no workflow run will be triggered.
    > 
    > 2. If find some existing workflow files, check the trigger events set in the workflow files. Whether the event names (github.event_name) have included push event, and the branches filter have includes feat branch. If no matched trigger events in the workflow files, no workflow run will be triggered.
    > 
    > 3. If have matched trigger events in the workflow files, trigger the workflows that have the matched trigger events.


## Trigger Events 归纳

记录一下不同操作所触发的事件。

|用户的操作|触发的事件|活动类型|备注|
|-|-|-|-|
|创建PR|`pull_request`|`opened`||
|关闭PR|`pull_request`|`closed`||
|重开PR|`pull_request`|`reopened`||
|分配任务|`pull_request`|`assigned`||
|指定reviewer|`pull_request`|`review_requested`||
|reviewer通过代码|``|``||
|reviewer拒绝通过|``|``||
|合并分支|1. `pull_request`<br>2. `push`|`closed`<br>n/a|先后触发了两个事件|
|创建分支|1. `create`<br>2. `push`|n/a<br>n/a|先后触发了两个事件|

