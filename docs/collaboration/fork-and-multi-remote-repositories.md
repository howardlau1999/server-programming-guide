# fork 与多远端仓库下的协作

在以 GitHub 为代表的开源社区平台中，存在着一个较为特别的概念：fork。它的作用是对任意用户的仓库进行“复制”，得到一个属于自己的仓库。在 GitHub 上的开源项目中，fork 是常见的协作方法之一。

根据 GitHub 官方文档[《About forks》](https://docs.github.com/en/github/collaborating-with-pull-requests/working-with-forks/about-forks)的介绍，fork 的一个突出用途在于：对于很多开源项目来说，人们可以使用 fork 来得到一个新仓库，并在新仓库里根据自己的想法提交许多修改，然后在本地进行充分的测试之后通过 pull request 来将自己的修改提交到原来的代码仓库中。

为什么需要一个新仓库呢？这是因为原仓库并不公开提供 push 权限，因此即使没有权限的人们可以把仓库克隆下来，在本地创建分支并提交修改，但他们并不能进行 `push` 操作，自然也就无法直接通过这种方式来提交代码，以及将自己的分支直接合进主分支里。

???+info
    根据[《About collaborative development models》](https://docs.github.com/en/github/collaborating-with-pull-requests/getting-started/about-collaborative-development-models)，上述的模式也被称为 Fork and pull model，在开源项目中非常流行。也有一种 Shared repository model，它允许贡献者直接对仓库进行 push。这种模式在私有项目中较为常见。

fork 的功能并不局限于此。一些开源项目有时候会因为各种原因（人力不足、法律约束、理念冲突等），迟迟不能提供一些新功能。这时对这些功能有强烈需求的人们会为此开辟一个 fork，专门在这个 fork 上编写新功能的代码。同时，**fork 提供从源仓库拉取新修改的功能**，让 fork 出来的项目得以与源项目保持同步。例如，[MultiMC5-Cracked](https://github.com/AfoninZ/MultiMC5-Cracked) 是基于著名 Minecraft（我的世界）游戏启动器 [MultiMC5](https://github.com/MultiMC/MultiMC5) 的 fork 版本。因为后者不支持在不提供正版账号的情况下启动游戏，所以前者诞生并提供了新的功能，让用户能够使用非正版账号启动游戏。

## 添加远端仓库并 checkout

现在，试想我们就是 MultiMC5-Cracked 的作者，那么又该如何在 fork 出来的仓库里从源仓库拉取最新的修改呢？首先，我们需要在本地仓库执行以下命令：

```
$ git remote -v
origin	git@github.com:AfoninZ/MultiMC5-Cracked.git (fetch)
origin	git@github.com:AfoninZ/MultiMC5-Cracked.git (push)
```

我们使用此命令列出了当前仓库的远端仓库地址，其中的 `(fetch)` 和 `(push)` 意味着对应的远端仓库可以用来进行拉取和推送 commit。然后我们使用 `git remote add` 命令来添加新的远端仓库：

```
$ git remote add upstream git@github.com:MultiMC/MultiMC5.git
$ git remote -v
origin	git@github.com:AfoninZ/MultiMC5-Cracked.git (fetch)
origin	git@github.com:AfoninZ/MultiMC5-Cracked.git (push)
upstream	git@github.com:MultiMC/MultiMC5.git (fetch)
upstream	git@github.com:MultiMC/MultiMC5.git (push)
```

我们将源仓库 MultiMC5 添加为当前仓库的另一个远端仓库。虽然它同样有一个 `(push)` 标识，但我们并不能直接将 commit 推送到源仓库，因为我们没有权限。之后我们可以通过 `git fetch upstream` 来拉取源仓库的分支信息：

```
remote: Enumerating objects: 365, done.
remote: Counting objects: 100% (333/333), done.
remote: Compressing objects: 100% (99/99), done.
remote: Total 365 (delta 264), reused 303 (delta 234), pack-reused 32
Receiving objects: 100% (365/365), 131.51 KiB | 5.98 MiB/s, done.
Resolving deltas: 100% (266/266), completed with 121 local objects.
From github.com:MultiMC/MultiMC5
 * [new branch]        beature/debrand                           -> upstream/beature/debrand
 * [new branch]        develop                                   -> upstream/develop
// 这里省略了一些无关紧要的分支
 * [new branch]        stable                                    -> upstream/stable
 * [new tag]           0.6.12                                    -> 0.6.12
```

可以看到，Git 将源仓库的各种数据已经拉取了下来，包括各种分支的信息等。现在如果我们需要将源仓库的 `stable` 分支合并入我们自己的仓库的 `stable` 分支，只需运行 `git merge upstream/stable` 即可。

???+info
    你可能会在使用 `git checkout stable` 来切换到本地 `stable` 分支时遇到以下错误：
    
    ```
    $ git checkout stable
    hint: If you meant to check out a remote tracking branch on, e.g. 'origin',
    hint: you can do so by fully qualifying the name with the --track option:
    hint:
    hint:     git checkout --track origin/<name>
    hint:
    hint: If you'd like to always have checkouts of an ambiguous <name> prefer
    hint: one remote, e.g. the 'origin' remote, consider setting
    hint: checkout.defaultRemote=origin in your config.
    fatal: 'stable' matched multiple (2) remote tracking branches
    ```

    这里 Git 是在说：`origin` 和 `upstream` 上都有名为 `stable` 的分支，因此我们要么通过 `--track` 参数手动指定远端仓库，要么通过 `git config` 来设置默认要选择的远端仓库。

## 检查他人的 Pull request

假如我们是开源项目的作者，有人向我们发来了一个 Pull request。在这个 PR 里，他声称帮助我们实现了一些新功能，或者修复了一些 bug。要验证他的说法，我们很有可能需要得到他修改的分支对应的源码，然后进行需要的一些步骤，例如编译运行，手动测试等。我们有两种方法来得到 Pull request 对应的源码。

## 直接 fetch 对应的 Pull request

像 GitHub 这样的平台支持直接在本地仓库运行 `git fetch origin pull/ID/head:BRANCHNAME` 来将编号为 `ID` 的 Pull request 对应的分支拉取到本地名为 `BRANCHNAME` 的分支上。

### 添加 Refspec

我们需要修改本地仓库的 `.git/config` 文件，在 `[remote "origin"]` 栏目下添加新的一行：

```
fetch = +refs/pull/*/head:refs/pull/origin/*
```

???+info
    这项配置被[官方文档](https://git-scm.com/book/en/v2/Git-Internals-The-Refspec)称为 Refspec，它指定了 Git 应该将远端仓库的哪些 reference 拉取到本地。默认情况下，它的配置为：

    ```
    fetch = +refs/heads/*:refs/remotes/origin/*
    ```

    它意味着在进行 `git fetch` 时，Git 会将远端仓库上 `refs/heads` 文件夹下的所有 reference 文件拉取到本地的 `refs/remotes/origin/` 文件夹里。为了能够拉取 Pull request，我们可以添加上述的新的一行配置。更多细节请见官方文档。

之后我们运行 `git fetch origin`，得到了以下输出。我们这里仍以上文提到的 MultiMC5-Cracked 在 GitHub 上的仓库为例：

```
$ git fetch origin
remote: Enumerating objects: 772, done.
remote: Counting objects: 100% (406/406), done.
remote: Compressing objects: 100% (88/88), done.
remote: Total 772 (delta 289), reused 394 (delta 283), pack-reused 366
Receiving objects: 100% (772/772), 140.63 KiB | 249.00 KiB/s, done.
Resolving deltas: 100% (352/352), completed with 59 local objects.
From github.com:AfoninZ/MultiMC5-Cracked
 * [new ref]           refs/pull/1/head  -> refs/pull/origin/1
 * [new ref]           refs/pull/10/head -> refs/pull/origin/10
 * [new ref]           refs/pull/11/head -> refs/pull/origin/11
 // 以下省略
```

我们拉取了此仓库的所有 Pull request 对应的分支，其中的数字就是 Pull request 的编号。如果我们想要检查编号为 `11` 的 Pull request，可以运行以下命令：`git checkout -b pr-11 pull/origin/11` 来切换到对应的分支。

???+warning
    `refs/pull` 目录下的文件都是只读的，这意味着无论我们使用的是方法一还是方法二，在切换到 `pr-11` 分支之后都无法直接进行 `push`。不过我们仍然可以通过 `git push origin NAME` 来将当前分支推送到远端仓库里一个新的叫做 `NAME` 的分支上。
