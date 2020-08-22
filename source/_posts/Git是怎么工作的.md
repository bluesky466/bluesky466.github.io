title: Git是怎么工作的
date: 2020-08-21 20:42:31
tags:
  - 技术相关
  - Git
---

相信大部分的程序员都会使用Git，可能使用IDE集成的可视化界面，也可能是直接用的命令行。但是可能很多人对Git的一些原理性的概念并没有什么了解，这篇博客就从Git的原理出发，讲解Git的一些命令操作的底层意义，可能会让大家使用Git的时候更加得心应手。

PS: 这篇博客是对部门技术分享的整理，绝大多数知识点可以在[*Pro* *Git*](https://www.progit.cn/)这本书里面找到，也有部分篇幅摘抄自这本书，我也强推大家去通读一遍这本书。


# Git与其它版本控制系统的差异

Git 和其它版本控制系统（包括 Subversion 和近似工具）的主要差别在于 Git 对待数据的方法。 概念上来区分，其它大部分系统以文件变更列表的方式存储信息。 这类系统（CVS、Subversion、Perforce、Bazaar 等等）将它们保存的信息看作是一组基本文件和每个文件随时间逐步累积的差异。

{% img /Git是怎么工作的/deltas.png %}

Git 不按照以上方式对待或保存数据。 反之，Git 更像是把数据看作是对小型文件系统的一组快照。 每次你提交更新，或在 Git 中保存项目状态时，它主要对当时的全部文件制作一个快照并保存这个快照的索引。 为了高效，如果文件没有修改，Git 不再重新存储该文件，而是只保留一个链接指向之前存储的文件。 Git 对待数据更像是一个 **快照流**

{% img /Git是怎么工作的/snapshots.png %}

也就是说Git实际是实现了一个小型的文件系统，把需要托管的文件的所有版本都备份到了这个系统里面，然后在需要的时候找到对应的版本拿出来使用。

大家都知道Git是Linus开发的，写操作系统内核的人思路的确就是和其他人的不一样，直接搞了个文件系统，而且还继承了Linux一切皆文件的思想。

# object(对象)

这个文件系统其实就在.git/objects目录里面，git把所有的提交文件、提交历史等都保存成一个object保存到这个目录里面。

当clone了一个空的项目的时候.git/objects目录里面会有几个子目录，但是并没有实际的文件:

```shell
tree .git/objects
.git/objects
├── info
└── pack
```

当我们做了添加了一些目录和文件:

```shell
tree .
.
├── dir_a
│   └── file_1
└── dir_b
    └── file_2
```

然后将它们提交到Git之后，Git就会对这些目录和文件原内容加上特定头部信息一起做SHA-1散列得到一个校验和，并且将它们保存到objects目录。散列值前两字符用于命名子目录，余下的38个字符则用作文件名，这个文件就是Git的存储对象(obejct):

```shell
tree .git/objects
.git/objects
├── 26
│   └── f76fe87ba045f5a6b40d93598ca96e5a1fab39
├── 31
│   └── 2f01ad223d5eb1e959122ac2829744b59fd7f2
├── 75
│   └── 8902e9f92ad80673cb0f1da4b8d34fbfe47544
├── 8a
│   └── 776b73f34c3b546a66fe6173dfa0a53a142b9b
├── b8
│   └── 947b77094228836f18792dc5fac15dfa9de11e
├── c5
│   └── 0a174948973a2dbe8a43fd9282d24a7a6074c4
├── info
└── pack
```

这些文件都是压缩过的，我们可以用`git cat-file`名去查看内容，由于这些文件都是有不同类型区分的，所以可以用-p参数自动识别文件类型，例如我们查看26f76fe87ba045f5a6b40d93598ca96e5a1fab39这个object:

```shell
git cat-file -p 26f76fe87ba045f5a6b40d93598ca96e5a1fab39
040000 tree 758902e9f92ad80673cb0f1da4b8d34fbfe47544    dir_a
040000 tree 8a776b73f34c3b546a66fe6173dfa0a53a142b9b    dir_b
```

它是就是我们的根目录，可以看到有两个tree类型的object，分别对应子目录dir_a和dir_b，我们进去dir_a看看:

```shell
git cat-file -p 758902e9f92ad80673cb0f1da4b8d34fbfe47544
100644 blob c50a174948973a2dbe8a43fd9282d24a7a6074c4    file_1
```

这里有个blob类型的object file_1，blob类型的object就用来文件快照，可以看到它保存了file\_1的所有内容:

```shell
git cat-file -p c50a174948973a2dbe8a43fd9282d24a7a6074c4
file 1 content
```

除了这些目录、文件的object之外我们的commit也是会被保存成一个obejct:

```shell
git cat-file -p 312f01ad223d5eb1e959122ac2829744b59fd7f2
tree 26f76fe87ba045f5a6b40d93598ca96e5a1fab39
author linjw <bluesky466@qq.com> 1597981085 +0800
committer linjw <bluesky466@qq.com> 1597981085 +0800

first commit
```

可以看到它有个tree字段指向了根目录的object(26f76fe87ba045f5a6b40d93598ca96e5a1fab39)

这个commit的object的校验和其实就是commit id:

```shell
commit 312f01ad223d5eb1e959122ac2829744b59fd7f2 (HEAD -> master)
Author: linjw <bluesky466@qq.com>
Date:   Fri Aug 21 11:38:05 2020 +0800

    first commit
```

所以通过这个commit我们就能构建起整个目录:

{% img /Git是怎么工作的/1.png %}

## 对象树

然后我们再修改下file\_1，提交个commit，commit id 是da64cc3756675914aab6df4c01b81539ae6ef39f，我们查看它的内容，发现它对比第一个commit多了个parent指向第一个commit:

```shell
git cat-file -p da64cc3756675914aab6df4c01b81539ae6ef39f
tree 8cbab3b672a04bc8bf5fa63af6e06b0d92bd4126
parent 312f01ad223d5eb1e959122ac2829744b59fd7f2
author linjw <bluesky466@qq.com> 1597982034 +0800
committer linjw <bluesky466@qq.com> 1597982034 +0800

modifying file_1
```

然后现在整个objects目录是这样的:

```shell
tree .git/objects
.git/objects
├── 1e
│   └── a40765c24dc3a9109c06d2e2c1408ea40568be
├── 26
│   └── f76fe87ba045f5a6b40d93598ca96e5a1fab39
├── 31
│   └── 2f01ad223d5eb1e959122ac2829744b59fd7f2
├── 70
│   └── d733623d440ad95d53272528ae900295855665
├── 75
│   └── 8902e9f92ad80673cb0f1da4b8d34fbfe47544
├── 8a
│   └── 776b73f34c3b546a66fe6173dfa0a53a142b9b
├── 8c
│   └── bab3b672a04bc8bf5fa63af6e06b0d92bd4126
├── b8
│   └── 947b77094228836f18792dc5fac15dfa9de11e
├── c5
│   └── 0a174948973a2dbe8a43fd9282d24a7a6074c4
├── da
│   └── 64cc3756675914aab6df4c01b81539ae6ef39f
├── info
└── pack
```

我们用下面这张图表示各个object 的关系:

{% img /Git是怎么工作的/2.png %}

这些object是以树形结构组织起来的，而且每个commit都能遍历找到那个版本的所有文件，所以当使用reset命令的时候只需要找到commit的object然后遍历对象树将object里面的内容解压出来替换工作区的文件就可以了。



# 引用

在理解了commit本质上其实是一个object之后，我们就能很容易理解引用这个概念了。

引用在Git里面其实本质也是一个文件，它们存放在.git/refs/下的子目录里面，例如本地引用的路径在.git/refs/heads目录里面:

```shell
tree .git/refs/heads
.git/refs/heads
└── master
```

我们可以看到现在里面只有个master文件，原因是我们只有一个master分支。让我们打印一下这个master文件的内容:

```shell
cat .git/refs/heads/master
da64cc3756675914aab6df4c01b81539ae6ef39f
```

它的内容其实就是object的检验和。让我们创建多一个develop分支，可以看到这个目录就多了个develop文件

```shell
tree .git/refs/heads
.git/refs/heads
├── develop
└── master
```

它的内容和master的一样:

```shell
cat .git/refs/heads/develop
da64cc3756675914aab6df4c01b81539ae6ef39f
```

让我们回滚这个分支到第一个提交，可以发现它的内容就变成了第一个commit的校验和:

```shell
cat .git/refs/heads/develop
312f01ad223d5eb1e959122ac2829744b59fd7f2
```

其实除了分支之外，我们打的tag也是一样的原理，例如我们此时在develop上一个v1.0的tag，就会发现.git/refs/tags/目录下多了一个v1.0的文件：

```shell
tree .git/refs/tags
.git/refs/tags
└── v1.0
```

它的内容也是第一个commit:

```shell
cat .git/refs/tags/v1.0
312f01ad223d5eb1e959122ac2829744b59fd7f2
```

现在引用的情况如下图:

{% img /Git是怎么工作的/3.png %}

所以我们的分支也好，tag也好，其实都是一个引用，它们本质上是一个文件，里面的内容就是指向的object的校验和，而我们的回滚代码，其实就是将各个分支的引用指向了不同的commit而已，如果我们在master分支将代码reset到commit 312f01，就会发现master的引用指向了这个commit。



## HEAD引用

那Git又是如何知道我们当前是在哪个分支的呢？

其实在Git里面还有个特殊的引用HEAD引用，它就在.git目录下面。我们可以打印下它的内容:

```shell
cat HEAD
ref: refs/heads/develop
```

可以发现它指向了我们的develop引用， 这就表示我们当前正在develop分支。

Git就是靠这个HEAD引用找到我们当前位于哪个commit:

{% img /Git是怎么工作的/4.png %}

当然HEAD的内容也可能直接指向某个commit号，例如我们checkout到某个tag的时候:

```shell
cat .git/HEAD
312f01ad223d5eb1e959122ac2829744b59fd7f2
```

{% img /Git是怎么工作的/5.png %}

这是因为tag是固定的，我们并不能直接修改tag指向的commit。

## reflog

我们都知道可以用`git log`去查看commit的日志，其实类似的我们可以用`git reflog`去查看引用的操作日志，它会的打印如下:

```shell
3cb383a (HEAD -> develop) HEAD@{0}: commit: difying file_2
312f01a (tag: v1.0) HEAD@{1}: checkout: moving from 312f01ad223d5eb1e959122ac2829744b59fd7f2 to develop
312f01a (tag: v1.0) HEAD@{2}: checkout: moving from develop to v1.0
312f01a (tag: v1.0) HEAD@{3}: reset: moving to 312f01ad223d5eb1e959122ac2829744b59fd7f2
da64cc3 (master) HEAD@{4}: checkout: moving from master to develop
da64cc3 (master) HEAD@{5}: commit: modifying file_1
312f01a (tag: v1.0) HEAD@{6}: commit (initial): first commit
```

这个东西有什么用呢？举个例子，假设我现在在develop修改了file_2提交了一个commit，没有推到服务器上，然后就reset --hard回到了上一个commit。这个时候突然反悔了想找到之前那个commit要怎么办？

对的，就是用reflog:

```shell
312f01a (HEAD -> develop, tag: v1.0) HEAD@{0}: reset: moving to 312f01ad223d5eb1e959122ac2829744b59fd7f2
3cb383a HEAD@{1}: commit: difying file_2
312f01a (HEAD -> develop, tag: v1.0) HEAD@{2}: checkout: moving from 312f01ad223d5eb1e959122ac2829744b59fd7f2 to develop
312f01a (HEAD -> develop, tag: v1.0) HEAD@{3}: checkout: moving from develop to v1.0
312f01a (HEAD -> develop, tag: v1.0) HEAD@{4}: reset: moving to 312f01ad223d5eb1e959122ac2829744b59fd7f2
da64cc3 (master) HEAD@{5}: checkout: moving from master to develop
da64cc3 (master) HEAD@{6}: commit: modifying file_1
312f01a (HEAD -> develop, tag: v1.0) HEAD@{7}: commit (initial): first commit
```

从下往上,可以看到HEAD引用的操作历史:

提交了第一个commit(312f01a) 

---> 提交了第二个commit(da64cc3) 

---> 从master切换到了develop分支，当前所处的commit号依然是da64cc3 

---> 移动回了commit 312f01a  

---> 从develop分支切换 到了v1.0这个tag，当前所处的commit号依然是312f01a

---> 切换回了develop分支，当前所处的commit号依然是312f01a

--->  修改了file_2提交了commit 3cb383a

----> reset 回到了commit 312f01a

所以我们就能找到丢失了的commit 3cb383a，此时只需要用reset --hard 3cb383a就能回到那个commit了。



# 远程分支

远程引用是对远程仓库的引用。我们从服务器拉取代码的时候就会将服务器的分支引用拉到本地，它们的文件在.git/refs/remotes/目录下的远程仓库对应的子目录里。例如我们在`git clone`的时候，Git会默认帮我们将远程仓库命名为origin，所以它的分支引用文件就在.git/refs/remotes/origin/目录下面。

这些远程分支以\<remote\>/\<branch\>的形式命名，例如origin仓库的master分支的名字就叫origin/master，所以我们可以用checkout命令直接切到远程分支:

```shell
git checkout origin/master
```

当有其他人往服务器推代码之后，我们需要用`git fetch`命令来抓取远程仓库有，而本地没有的数据:

```shell
git fetch origin
```

抓取完之后远程分支就更新了:

{% img /Git是怎么工作的/remote-branches-3.png %}

这个时候就可以用`git merge`命令将远程分支的代码合并到本地:

```shell
git merge origin/master
```

而我们工作中常用的`git pull` 在大多数情况下它的含义是一个 `git fetch` 紧接着一个 `git merge` 命令

顺便一讲，之前我们讲到commit是有parent概念的，而第一个commit由于之前已经没有提交了，所以它没有parent，普通的commit会有一个parent。

而`git merge`命令由于需要合并两个分支的修改，所以它会生成一个新的commit，它有两个parent:

{% img /Git是怎么工作的/6.png %}

例如我们上面的C6这commit就有两个parent C4和从C5

# pack机制

从上面我们可以看到Git向磁盘中存储对象使用松散对象格式，一个文件、目录、commit等对应一个文件，这样的操作可能会比较简单，但是其实是比较浪费磁盘空间的。而且在需要推送到远程仓库的时候需要一个个文件上传效率也比较低。所以Git会时不时将这些文件打包在一起以节省空间提高网络传输效率。

我们可以收到调用`git gc`命令让Git进行打包并清理一些不需要的对象。打包完成之后.git/objects里面的文件就会变小，并且在.git/objects/pack下面多出打包文件:

```shell
tree .git/objects/pack
.git/objects/pack
├── pack-20f597c6ab0c05f5c907023edf4e282be00ad6fe.idx
└── pack-20f597c6ab0c05f5c907023edf4e282be00ad6fe.pack
```

.idx文件是索引文件，而.pack就是将object对象打包成的二进制包，我们可以用`git verify-pack -v`命令查看包里的信息:

```shell
git verify-pack -v .git/objects/pack/pack-20f597c6ab0c05f5c907023edf4e282be00ad6fe.idx
1dcac3d26ede1fe64eddd3511a74d99a96ef97e2 commit 202 144 12
3fbc341a1b669518f3ef7aa038d71f2f7d68a5f0 commit 207 150 156
34ffe485561f71337d0b64fca8dc9c8d57d86027 commit 78 88 306 1 3fbc341a1b669518f3ef7aa038d71f2f7d68a5f0
58d65a51d2e0288b577c73a660f467bd548033f8 commit 231 166 394
ace7ce09e72571eb0474221f8e52b1767cebb0db commit 269 190 560
feafba560a569ff272319bb98dba0869aa2b242c commit 48 59 750 1 ace7ce09e72571eb0474221f8e52b1767cebb0db
9aaa265d2221fd96e5d3dd420fe9d93e7b99726f commit 13 24 809 1 58d65a51d2e0288b577c73a660f467bd548033f8
2cb88bd69545620211e77a7865b9e9990d9a0c20 commit 212 150 833
9750080a17990f284f5045e4eef885605f2eb6d8 commit 80 91 983 1 2cb88bd69545620211e77a7865b9e9990d9a0c20
69f9a2ec2f55ceed476a68190d32f543d89c2f78 commit 70 82 1074 1 3fbc341a1b669518f3ef7aa038d71f2f7d68a5f0
da5e1a12ab4c5fdca2a808e4b85c1c54dc780642 commit 81 91 1156 1 3fbc341a1b669518f3ef7aa038d71f2f7d68a5f0
c756597a8071fbf0b26fda95f0cd99edb68f7759 commit 79 90 1247 1 2cb88bd69545620211e77a7865b9e9990d9a0c20
3cb383a390cbd7cbf4872412b828987e0cdc1b13 commit 213 153 1337
da64cc3756675914aab6df4c01b81539ae6ef39f commit 64 76 1490 1 58d65a51d2e0288b577c73a660f467bd548033f8
312f01ad223d5eb1e959122ac2829744b59fd7f2 commit 163 119 1566
e4e3854198486f045f95c16d886a7ecec076799d tree   64 67 1685
758902e9f92ad80673cb0f1da4b8d34fbfe47544 tree   34 45 1752
593e3181013a603a1f273034ad4d2b30ebed201a tree   34 45 1797
70d733623d440ad95d53272528ae900295855665 tree   34 45 1842
cfb98c1ce98de4290f310e81cf0b9128749df334 tree   130 129 1887
08df14d61d050a7235a5bab256e63432d422ab61 tree   64 67 2016
67ed7f289e73c67217c5bcdbe18f8f55c2d3699f tree   34 45 2083
8205fc9052e9544796e762459e48909f661145dc tree   64 67 2128
4c1ae63f5abfcb9107241f03a86df80aafbc6780 tree   34 45 2195
a8c1a9640eb8e53987cd5684535769a9f86622cf tree   64 67 2240
b7391b72f9c25ce53cf9a3679f9157d19897fd5c tree   34 45 2307
022d4f657214d39e3d540d214aa7affdd44cf489 tree   96 100 2352
69f3e7095dc70df5693bb6c213bdc77d3665be3d tree   96 100 2452
8a776b73f34c3b546a66fe6173dfa0a53a142b9b tree   34 45 2552
26f76fe87ba045f5a6b40d93598ca96e5a1fab39 tree   64 67 2597
c50a174948973a2dbe8a43fd9282d24a7a6074c4 blob   15 24 2664
9d791ce077105e227ffae01975a81e980a06a9a2 blob   24 34 2688
6d1f7671b90551cb98157a48a7b26b1183dfb821 tree   27 40 2722 1 cfb98c1ce98de4290f310e81cf0b9128749df334
1ea40765c24dc3a9109c06d2e2c1408ea40568be blob   25 35 2762
e69de29bb2d1d6434b8b29ae775ad8c2e48c5391 blob   0 9 2797
d00491fd7e5bb6fa28c517a0bb32b8b506539d4d blob   2 11 2806
41ba6a00583e910f220ecd34021d4d29de0feeda blob   3 12 2817
896691a3909edace46e4b587e1f1496af22f36d7 tree   4 15 2829 1 022d4f657214d39e3d540d214aa7affdd44cf489
ce7499ae1967bd055f42c5458a60be5131f85da2 blob   21 31 2844
258be8efcad31cc0c6c8dc6bc7d15a2c6910cd1a blob   22 32 2875
840bbdd49116cbffa896bb7f5ab011f2ddf8d446 blob   19 29 2907
b8947b77094228836f18792dc5fac15dfa9de11e blob   15 24 2936
8cbab3b672a04bc8bf5fa63af6e06b0d92bd4126 tree   4 15 2960 1 69f3e7095dc70df5693bb6c213bdc77d3665be3d
非 delta：32 个对象
链长 = 1: 11 对象
.git/objects/pack/pack-20f597c6ab0c05f5c907023edf4e282be00ad6fe.pack: ok
```

同样的.git/refs/下面的引用文件也会被打包，这里可以看到该目录已经清空了:

```
tree .git/refs/heads
.git/refs/heads

0 directories, 0 files
```

它们会被打包到.git/packed-refs文件中,可以直接用`cat`命令查看:

```shell
cat .git/packed-refs
# pack-refs with: peeled fully-peeled sorted
3cb383a390cbd7cbf4872412b828987e0cdc1b13 refs/heads/develop
3cb383a390cbd7cbf4872412b828987e0cdc1b13 refs/heads/master
312f01ad223d5eb1e959122ac2829744b59fd7f2 refs/tags/v1.0
```