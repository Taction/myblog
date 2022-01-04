---
title: "DCO Rebase"
linkTitle: "DCO Rebase"
type: docs
date: 2021-12-18T19:27:34+08:00
draft: false
---

#### 使用rebase对提交进行签名

##### 背景

提交的一个开源项目增加了DCO要求，而且项目是通过merge来进行代码管理的。所以有些比较久的提交有比较多的commit没有被签名，且中间有很多次提交是从master merge进来的。

对于比较简单的，只有一次提交，或者全都是自己的提交（没有merge的提交），这种进行补签名很容易。通过查看[fix DCO文档](https://github.com/src-d/guide/blob/master/developer-community/fix-DCO.md)，可以很方便的找到解决方案。

有些提交这个时候进行rebase --signoff的时候，可能会需要解决很多冲突。如果你是这种情况，你可以继续向下看。

##### 解决方案

对于此类提交，我的处理方案是先rebase 签名并在这个过程中去掉所有merge过来的提交。然后如果跟master没有冲突就rebase master。如果有冲突就merge master进来（简单）。最后force push。

但是这个方式有个缺点，如果你之前合并的时候有冲突，那么你处理冲突的这个提交就被删掉了，你要自己重新处理下冲突，或者把文件考出来，等处理冲突的时候粘回去并检查是否有问题。

但是好处是，你不需要一个提交一个提交的解决冲突，只需要在最后一次进行冲突解决就可以（如果有的话）

##### 过程

> 在rebase进行过程中，你可以通过`git rebase --abort`来终止此次rebase

首先数出来这个分支有多少次提交需要处理，包含merge的提交，将`[N]`替换成实际的数量。

```shell
git rebase --signoff --interactive HEAD~[N]
```

如果是12次提交就是`git rebase --signoff --interactive HEAD~12`

如果你提交数量特别多，数起来很麻烦，那么你可以通过以下命令进行，其中`<commit hash>`为你第一次提交的前一次提交的hash前七位。

```shell
git rebase --signoff --interactive <commit hash>
```

然后会打开一个文本，内容是时间倒序的所有提交，首先对比查看第一条提交message，是不是自己这个pr的第一次提交。然后对照自己提交记录将所有不是你自己的提交行删除掉。

![image-20220104192650462](https://image-1255620078.cos.ap-nanjing.myqcloud.com/image-20220104192650462.png)

删除后：

![image-20220104194303661](https://image-1255620078.cos.ap-nanjing.myqcloud.com/image-20220104194303661.png)

保存文本，你应该就会发现控制台输出`Successfully rebased and updated refs/heads/<your branch name>.`

**注意**：如果你在rebase的过程中遇到问题，你需要解决冲突并执行

```
git commit --amend --no-edit --signoff
git rebase --continue
```



这时你的分支就会只剩下你自己的提交，且都已经被签名。假设你的分支要被合并到master上且本地master分支已经保持最新，你可以根据需求和你自己的喜好选择执行以下两个命令的任意一条

```
git rebase master
//or 
git merge --signoff master
```

如果有冲突就根据提示解决冲突。如果有冲突解决完成后需要根据你上一次执行方式选择`git rebase --continue`或者`git merge --continue`来继续进程）,如果你是用merge遇到了冲突，你需要执行`git commit --amend --no-edit --signoff`补提交签名。

最后执行：

```git push --force-with-lease origin <your branch name>```

查看你的所有提交,此时应该都已是补全DCO的状态了。

