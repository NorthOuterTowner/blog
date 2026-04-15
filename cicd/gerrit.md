# 使用 Gerrit 进行代码提交

使用`Gerrit`首先需要确保配置了`Git`，从中获得了公钥，并且在本地有`Git bash`。

## 找到Git公钥

在设备上找到`.ssh`文件
```bash
ls ~/.ssh # Windows
ls -al ~/.ssh # Linux
```
默认的文件名为`id_rsa.pub`，将文件中的内容复制进行复制，打开`Gerrit`平台上的`settings`，找到`SSH Keys`粘贴。

## 克隆项目代码

此处项目提供了带`hooks`的`clone`方法，通过`git bash`直接`clone`到本地即可。

## 代码变更
按照标准的`git flow`，需要首先从`dev`分支签出到个人的分支，之后在本地的进行修改。
```bash
git checkout dev
git checkout -b your-branch
```
修改代码后正常提交。
```bash
git add .
git commit -m "feat: 完成了某项功能逻辑"
```
签回`dev`分支合并个人分支
```bash
git checkout master
git merge your-branch
```
把本地的更改内容推到`Gerrit`评审轨道。此处不能直接推送，否则无法进入`Gerrit`的评审轨道，因此必须用下面的方式进行推送。
```bash
git push origin HEAD:refs/for/master
```

完成代码变更的请求后邀请其他小组成员进行评审，评审分数达到项目要求后就能将变更同步到远程的仓库中了。

## 项目要求变更
在平台集成的`Gerrit`中，可能以及设置了一些对代码融合的分数要求，如下面要求必须有人提交了最高分，且没有人提交最低分。
```
Submit Condition:
label:Code-Review=MAX
-label:Code-Review=MIN
```
如果对这种默认的方式不满意，可以要求项目的负责人进行更改。

*Written By Ruize li on 4/12/2026* 