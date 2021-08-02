# fork 与多 origin

在以 GitHub 为代表的开源社区平台中，存在着一个较为特别的概念：fork。它的作用是对任意用户的仓库进行“复制”，得到一个属于自己的仓库，同时

根据官方文档[《About forks》](https://docs.github.com/en/github/collaborating-with-pull-requests/working-with-forks/about-forks)的介绍，它的一个突出的用途在于：对于很多开源项目来说，人们可以使用 fork 来得到一个新仓库，并在新仓库里根据自己的想法提交许多修改，然后在本地进行充分的测试之后通过 Pull request 来将自己的修改提交到原来的代码仓库中。为什么需要一个新仓库呢？这是因为原仓库并不公开提供 push 权限，因此即使没有权限的人们可以把仓库克隆下来，在本地创建分支并提交修改，但他们并不能进行 `push` 操作，自然也就无法给自己对分支创建相应的 Pull request。
