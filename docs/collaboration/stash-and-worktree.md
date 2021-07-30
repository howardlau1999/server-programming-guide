# Git stash 与 working trees 的使用

本文会介绍 Git 中的两条非常有用的命令：`git stash` 和 `git worktree`。它们能够帮助你高效地使用 Git 管理自己编写的各种修改，快速地在分支间切换而不影响你在上一个分支里还没有提交的修改。上面的文字可能看上去有些费解，请继续阅读下文，我们会结合例子来介绍这两个强大的命令的基本使用。

## stash

有的时候，我们可能在自己的分支上进行着工作，暂存区和工作区里含有大量修改。而这时 leader 突然让我们立刻修复一个严重的 bug，而这需要我们切换到主分支进行修改。

我们不希望在切换分支之后暂存区和工作区里还留着我们之前的修改，因为我们之前的修改可能还不完善，会让程序无法运行。而且，有时候我们还没提交的修改会和其他分支的提交产生冲突，这时 Git 并不允许我们切换分支，就像下面的提示一样：

``` hl_lines="4"
$ git checkout a
error: Your local changes to the following files would be overwritten by checkout:
	example.txt
Please commit your changes or stash them before you switch branches.
Aborting
```

仔细一看，Git 提示我们要么提交修改，要么使用 `stash`。提交修改大多数情况下是不可接受的，我们不想提交一些还没有真正完成的修改。这时 `git stash` 命令就能派上用场了。

stash 在英语里的意思是藏匿，也就是说我们可以将暂存区和工作区里的修改暂时藏起来，然后 Git 会认为我们什么修改也没做，我们因此可以顺利地切换到新的分支。等到 bug 修复完成，我们切回原来的分支之后，我们可以让原来被藏起来的修改重新“现形”，出现在暂存区或工作区中。

我们以一个只有 `example.txt` 这个文件的仓库为例子，假如我们此时在 `a` 分支上向它末尾添加了新的一行 `new line`，但还没来得及进行 `git add` 或者是 `git commit` 等操作。这时候 leader 需要我们从 `master` 分支上切出新的 `bug-fix` 分支来修复一个 bug，那么这时候我们可以运行 `git stash`，得到以下输出：

```
$ git stash
Saved working directory and index state WIP on a: 4404526 commit a 1
```

Git 提示说已经保存了当前的修改。这时我们运行 `git status`，可以得到以下输出：

```
$ git status
On branch a
Your branch is up to date with 'origin/a'.

nothing to commit, working tree clean
```

我们的工作区和暂存区都为空，就好像我们从来没有做出那行修改一样！这时我们就能切换到其他分支，进行别的工作了。当我们回到 `a` 分支时，可以使用 `git stash pop` 来恢复之前的修改：

```
On branch a
Your branch is up to date with 'origin/a'.

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   example.txt

no changes added to commit (use "git add" and/or "git commit -a")
Dropped refs/stash@{0} (6349d71a1c2d7647b38e917b5e4506afd4ae5a3a)
```

可以看到，我们之前对 `example.txt` 的修改已经重新出现。以上就是 `git stash` 的基本用法介绍。

???+info
    `git stash` 命令其实是 `git stash push` 的缩写。从 `push` 和 `pop` 的关系可以看出，Git 保存我们修改的数据结构是一个栈，这意味着我们可以多次使用 `git stash`，并通过 `push` 和 `pop` 来存取我们的修改。

    特别地，进行 `push` 和 `pop` 操作并不是必须在同一个分支上进行，我们可以暂存在一个分支上的修改，然后将它释放到另一个分支上。

    使用 `git stash list` 命令可以查看当前暂存的所有批次的修改。其他命令及更多参数请见[官方文档](https://git-scm.com/docs/git-stash)。

## working trees

上述的 `stash` 命令使用起来非常简单，但在一些特定的场景下却并不能给我们带来便利。如果我们正在修改的源代码正在被一些外部程序所依赖，那么在我们进行 `stash` 之后这些外部程序的运行将会受到影响，但这些影响往往是不必要或者会拖累我们工作的。

一个简单的例子是：我们启动了一个 HTTP 服务器来让我们可以访问 `localhost` 来实时查看我们正在编辑的前端项目，每当我们修改源文件，我们的浏览器就会看到最新的结果。这时如果我们运行 `stash` 并切换到其他分支进行工作，我们的浏览器也会自动展示对应的修改。问题在于对于一些复杂的前端项目来说，对源码的修改会需要经过一系列复杂的编译步骤才能最终输出给浏览器，而编译的过程会消耗大量的 CPU 资源和内存，拖累我们电脑的性能。另一个例子是，我们在原来的分支上开发时启动了一些辅助工具，这些工具依赖当前的代码，如果这时突然切换到另一个分支，那么另一个分支上一些截然不同的代码会让这些工具崩溃。

**如果能够在保留当前修改的同时，让当前的源文件都不变，并且我们还能切换到其他分支进行工作就好了。**这看起来有些匪夷所思，Git 切换一个分支之后，仓库目录里的所有源文件不是会变成相应分支的文件吗？的确如此，除非我们使用强大的 `git worktree` 命令。

Git working trees 实现上述目标的方法是**创建一个新的文件夹来放置要切换到的分支对应的所有源文件**。仍以上文的 `example.txt` 为例，假如我们此时在 `a` 分支做了一些修改，但是还没有运行像 `git add` 或者 `git commit` 这样的命令。此时我们被要求切换到 `master` 分支上修改一个 bug。我们可以运行 `git worktree add ../fix-bug-on-master master`，得到以下输出：

```
$ git worktree add ../fix-bug-on-master master 
Preparing worktree (checking out 'master')
HEAD is now at 554a828 commit master 2
```

???+info
    命令中的 `../fix-bug-on-master` 代表一个路径，告诉 Git 应该创建哪个文件夹。`master` 表示要切换的分支名，可以省略这个参数表示直接使用当前的分支。更多参数请见[官方文档](https://git-scm.com/docs/git-worktree)。

此时 Git 会将当前 `master` 分支的源文件释放到 `../fix-bug-on-master` 文件夹里，而我们当前分支以及刚刚做的所有修改都不变。相当于我们手动切换到了 `..` 文件夹，然后将仓库克隆到了 `fix-bug-on-master` 文件夹里。

此时我们可以通过 `cd ../fix-bug-on-master` 来切换文件夹，开始 bug 的修复工作，最后进行 commit。当我们完成工作后，我们可以直接删除此文件夹，并运行 `git worktree remove ./fix-bug-on-master` 来移除这个 working tree。我们可以在一个仓库的文件夹里通过 `git worktree list` 来查看当前仓库以及相关的 working tree 列表。

对比使用 `stash`，虽然 `worktree` 在使用上更加麻烦，但是它为我们在不同分支和修改之间提供了良好的隔离性，让我们可以同时在不同分支对应的仓库目录里进行工作。究竟应该使用哪个应当取决于我们遇到的实际问题。如果在切换分支时不希望对当前的源文件进行改变，那么我们往往需要使用 `worktree`，否则使用简单的 `stash` 就能满足我们的需要。
