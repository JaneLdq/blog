---
title: Git - Merging & Rebasing
date: 2018-09-10 20:36:54
categories: 工具
tags: Git
---
平时只用过最简单的git merge，还一般都是在GUI上点点按钮就结束了，今天正好遇到了一个比较不一样的操作，趁着记忆还新鲜，就顺便聊聊git merge与git rebase那些事吧～

简化版故事背景是这样的：
1. `dev`是主要开发分支
2. `master`一般不动，只在release时把`dev`的代码merge过来

结果，今天正好在`dev`改了些代码，而且又正好要往`master`merge，结果发现上一次merge到`master`的代码改动又出现在了这次的提交中，检查`master`发现代码确实已经过去了，但是git在对比时的时候还是会识别出提交过的代码，很奇怪。

又是一番检查，发现问题就出在这条命令上：
```
$ git merge --squash
```

<!--more-->

# $ git merge
我们先来看看git merge的一般情况。
如上所述，我们在开发新功能时一般都会切分支单独开发，开发结束后再把代码merge回`master`。

假设当前提交历史如下：

```
... - A - B - C - G         <== master
       \
        D - E - F           <== dev
```
在`master`分支上调用`$ git merge dev`就会基于`master`把`dev`上的commits重放（replay）到`master`上，并且新建一个**"merge commit"**记录merge后的变更，变成这样：

```
... - A - B - C - G - H     <== master
       \            /
         D - E - F          <== dev
```
`H`就是由merge操作创建的新的"merge commit"。

如果加上`--squash`参数，git不会帮我们创建新的commit，也不会移动HEAD，但是squash merge之后，所有来自`dev`分支的commit做的更改就像被复制到`master`上，好像所有的改动都是原本就是基于master做的，调用`$ git status`可以查看到是merge之后的变动尚未提交。这时，我们就可以把所有的变更打包成新的单个commit，这个commit的内容跟`dev`分支一样，但它完全独立于`dev`。
如下图，所示，H中包含了D-E-F中的所有改动，并且并不会如常规merge一般与F连接起来。
```
... - A - B - C - G - H     <== master
       \         
        D - E - F           <== dev
```
官方解释：
> --squash: Produce the working tree and index state as if a real merge happened (except for the merge information), but do not actually make a commit, move the HEAD, or record $GIT\_DIR/MERGE_HEAD (to cause the next git commit command to create a merge commit). This allows you to create a single commit on top of the current branch whose effect is the same as merging another branch

这样就能解释为什么会出现文章开头遇到的情况了。因为上一次把`dev`合并到`master`用的是squash merge，但是merge之后没有删除`dev`而是随后又接着在`dev`上开发I-J-K，下一次merge时，git找到的`dev`和`master`的共同祖先还是A，所以D-E-F-I-J-K都会被算作是新的commit，然后重复更改就出现了。

所以，如果使用squash merge某个分支后，以后还要基于该分支开发，那么就应该在merge后立即删除并且重新从`master`切出来，以保证提交历史的一致性。

---
# $ git rebase
由于我们遇到的情况是`dev`分支上已经有了新的提交，就不能直接删除了，所以想看看能不能用rebase抢救一下～

按我的理解，不同于merge会保持原始的分支记录，只在最后新建一个merge commit把二者合一；rebase则是在base分支上，把merge分支的所有commit重新提交了一遍，相当于重写了提交历史，使得提交历史更简洁清晰。

假设当前提交历史如下：
```
... - A - B - C - G         <== master
       \
        D - E - F           <== dev
```
在`dev`分支调用`$ git rebase master`后就变成了这样：
```
... - A - B - C - G - D' - E' - F'
                  ^             ^
                master         dev
```
git会找到`master`和`dev`提交历史的共同祖先A，然后把D-E-F的变更保存在临时文件中，把指针指向base分支的当前commit，也就是G，最后按顺序把D-E-F重新基于base操作一遍，变成D'-E'-F'。

这时再把`dev`merge到`master`就可以看到提交历史是线性的，就好像所有提交都是按顺序发生的，不同于直接merge保留的并行历史。

> Rebasing replays changes from one line of work onto another in the order they were introduced, whereas merging takes the endpoints and merges them together.

整洁的历史记录看起来令人心旷神怡，然而，在使用rebase时一定要记得遵守如下指南哦，否则后果可能就不那么令人愉快了～

重要的事情，说三遍。
**Do not rebase commits that exist outside your repository！**
**Do not rebase commits that exist outside your repository！**
**Do not rebase commits that exist outside your repository！**

为什么不要这样做？因为rebase操作实际上是把原有的commits都丢弃，重新创建commits，即使内容和顺序完全一样，有的东西就是不一样了，你懂的。

官网上有个看起来就很头大的例子，参观请移步[Git-Rebasing #The Perils of Rebasing][1]

---
# Merging or rebasing?
merge和rebase其实体现的是看待提交历史的不同角度。
如果把提交历史看做是**用来记录实际发生了些什么**，那么它就不应该被修改，而merge能原始的保留历史记录状态，但是要注意一点，每一次merge都会创建一条新的merge commit（这里指常规merge），这会导致历史记录中出现很多merge commit。

而如果把提交历史**用于讲述整个项目是如何创造出来的故事线**，那么你可能需要记录的是关键的里程碑节点和清晰的描述。就好比原始提交记录是草稿，那么通过rebase可以做一些整理、编辑操作，使得最终呈现出来的是更加完整而清晰的作品。

我们不能说merge和rebase一定谁更好，否则就不会有两者同时存在了，更多的时候你会需要结合所遇到的情形组合使用各种命令。

不过有一种常见的做法值得参考的是：在提交本地分支前可以rebase一下，以保持本地分支的历史记录的整洁。但是，千万记得不要轻易rebase公共branch哦！


---
# 参考资料
* [Git-Basic Merging][2]
* [Git-Rebasing][3]
* [Merging vs. Rebasing][4]
* [StackOverflow-How to squash commits in Git][5]


  [1]: https://git-scm.com/book/en/v2/Git-Branching-Rebasing
  [2]: https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging
  [3]: https://git-scm.com/book/en/v2/Git-Branching-Rebasing
  [4]: https://www.atlassian.com/git/tutorials/merging-vs-rebasing
  [5]: https://stackoverflow.com/questions/44021102/how-to-squash-commits-in-git