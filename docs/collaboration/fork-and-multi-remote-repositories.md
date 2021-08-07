# fork 与多远端仓库下的协作

在以 GitHub 为代表的开源社区平台中，存在着一个较为特别的概念：fork。它的作用是对任意用户的仓库进行“复制”，得到一个属于自己的仓库。在 GitHub 上的开源项目中，fork 是常见的协作方法之一。

根据 GitHub 官方文档[《About forks》](https://docs.github.com/en/github/collaborating-with-pull-requests/working-with-forks/about-forks)的介绍，fork 的一个突出用途在于：对于很多开源项目来说，人们可以使用 fork 来得到一个新仓库，并在新仓库里根据自己的想法提交许多修改，然后在本地进行充分的测试之后通过 pull request 来将自己的修改提交到原来的代码仓库中。

为什么需要一个新仓库呢？这是因为原仓库并不公开提供 push 权限，因此即使没有权限的人们可以把仓库克隆下来，在本地创建分支并提交修改，但他们并不能进行 `push` 操作，自然也就无法直接通过这种方式来提交代码，以及将自己的分支直接合进主分支里。

???+info
    根据[《About collaborative development models》](https://docs.github.com/en/github/collaborating-with-pull-requests/getting-started/about-collaborative-development-models)，上述的模式也被称为 Fork and pull model，在开源项目中非常流行。也有一种 Shared repository model，它允许贡献者直接对仓库进行 push。这种模式在私有项目中较为常见。

fork 的功能并不局限于此。一些开源项目有时候会因为各种原因（人力不足、法律约束、理念冲突等），迟迟不能提供一些新功能。这时对这些功能有强烈需求的人们会为此开辟一个 fork，专门在这个 fork 上编写新功能的代码。同时，**fork 提供从源仓库拉取新修改的功能**，让 fork 出来的项目得以与源项目保持同步。例如，[MultiMC5-Cracked](https://github.com/AfoninZ/MultiMC5-Cracked) 是基于著名 Minecraft（我的世界）游戏启动器 [MultiMC5](https://github.com/MultiMC/MultiMC5) 的 fork 版本。因为后者不支持在不提供正版账号的情况下启动游戏，所以前者诞生并提供了新的功能，让用户能够使用非正版账号启动游戏。

## 添加远端仓库并 checkout

如果我们是开源项目的作者


