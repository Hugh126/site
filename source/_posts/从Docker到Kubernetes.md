---
title: 从Docker到Kubernetes
date: 2023-09-26 14:58:33
tags:
---
docker作为容器虚拟化技术，用起来是真的香。谁用谁知道...  
知道什么是趋势，我们就知道到底要不要去学一样东西。
<!--more-->

# 一 认识Docker
Docker 是一种开源的软件平台，用于创建、运行和管理容器化的应用程序。容器是一种轻量级的虚拟化技术，可以在隔离的环境中运行应用程序和其依赖项，从而提高可移植性、安全性和效率。要了解 Docker 的工作原理，你需要了解以下几个概念：

- **镜像**：镜像是一个包含应用程序代码、库、环境变量和配置文件的只读模板。你可以使用镜像来创建和启动容器。
- **容器**：容器是一个镜像的运行实例。你可以创建、启动、停止、移动或删除容器，使用 Docker API 或 CLI 进行操作。容器可以在本地机器、虚拟机或云端运行。
- **仓库**：仓库是一个存储和共享镜像的地方。你可以使用公共或私有仓库来推送或拉取镜像。Docker Hub 是一个流行的公共仓库，提供了数百万的镜像。
- **Dockerfile**：Dockerfile 是一个文本文件，用于定义如何从一个基础镜像构建一个自定义镜像。你可以在 Dockerfile 中指定要添加的文件、要执行的命令、要暴露的端口等。
- **Docker Compose**：Docker Compose 是一个工具，用于定义和运行由多个容器组成的应用程序。你可以使用 YAML 文件来配置应用程序的服务、网络和卷，然后使用一个命令来启动所有的容器。

如果你想学习如何使用 Docker 来构建和运行容器化的应用程序，你可以参考以下一些教程：

- [Docker 101 Tutorial](https://www.docker.com/101-tutorial/)(^1^)：这是一个自学的教程，教你如何构建镜像、运行容器、使用卷来持久化数据和挂载源代码，以及使用 Docker Compose 来定义应用程序。你还可以学习一些高级主题，如网络和镜像构建最佳实践。
- [Overview of the get started guide](https://docs.docker.com/get-started/)(^2^)：这是一个分步的指南，教你如何开始使用 Docker。这个指南向你展示如何构建和运行一个镜像作为一个容器，如何使用 Docker Hub 来共享镜像，如何使用多个容器和数据库来部署 Docker 应用程序，以及如何使用 Docker Compose 来运行应用程序。
- [Learn Docker Tutorial](https://www.javatpoint.com/docker-tutorial)(^3^)：这是一个涵盖了 Docker 的基本概念、特性、架构、安装、镜像和容器等方面的教程。你还可以学习如何使用 Dockerfile 和 Docker Compose 来创建和管理自己的应用程序，并查看一些不同编程语言（如 Java, PHP, Python, Scala, Perl, Ruby, Swift, Ubuntu 等）的示例。

源: 与必应的对话， 2023/9/26
(1) Docker 101 Tutorial | Docker. https://www.docker.com/101-tutorial/.
(2) Overview of the get started guide | Docker Docs. https://docs.docker.com/get-started/.
(3) Learn Docker Tutorial - javatpoint. https://www.javatpoint.com/docker-tutorial.
(4) play-with-docker. https://labs.play-with-docker.com/.

# 二 Docker运行原理
Docker运行原理是利用Linux内核的特性，如cgroups、namespaces和AUFS等，来实现应用程序和系统资源的隔离和虚拟化。Docker的工作过程大致如下：

- 用户通过Docker客户端向Docker守护进程发送命令，如创建、运行、停止、删除等。
- Docker守护进程接收到命令后，会根据用户指定的镜像或者Dockerfile来创建或启动一个或多个容器。
- 容器是一个运行时环境，包含了应用程序和其依赖的文件系统、网络和设备等。容器之间是相互隔离的，不会互相影响。
- 容器可以访问宿主机上的资源，如文件系统、网络和设备等。容器也可以通过端口映射或者网络连接来与其他容器或外部系统进行通信。
- 容器的运行状态可以通过Docker命令进行管理，如查看、停止、重启、删除等。容器的数据可以通过数据卷或者镜像提交来进行持久化或共享。

如果你想了解更多关于Docker的原理和工作过程的细节，你可以参考以下一些网页：

- [Docker技术核心原理讲解 - 知乎](^1^)：这是一个详细介绍了Docker的架构和工作原理的文章，包含了很多图解和示例。
- [Docker原理（图解+秒懂+史上最全）_40岁资深老架构师尼 ...](^2^)：这是一个用通俗易懂的语言讲解了Docker的基本概念、特性、架构、安装、镜像和容器等方面的博客。
- [Docker 原理新手指南：一文看懂 Docker 是什么? - Red Hat](^3^)：这是一个简要介绍了Docker的简介、优势、应用场景和原理的网页。
- [Docker 基本原理 - 知乎](^4^)：这是一个简单说明了Docker容器本质上是一个进程，运行于自己独立的命名空间中，所有的数据都存储在容器中的文章。
- [docker启动容器全过程以及原理-腾讯云开发者社区-腾讯云](^5^)：这是一个介绍了Docker容器启动过程和原理的网页，包含了一些代码和命令。

源: 与必应的对话， 2023/9/26
(1) Docker技术核心原理讲解 - 知乎. https://zhuanlan.zhihu.com/p/144747363.
(2) Docker原理（图解+秒懂+史上最全）_40岁资深老架构师尼 .... https://blog.csdn.net/crazymakercircle/article/details/120747767.
(3) Docker 原理新手指南：一文看懂 Docker 是什么? - Red Hat. https://www.redhat.com/zh/topics/containers/what-is-docker.
(4) Docker 基本原理 - 知乎. https://zhuanlan.zhihu.com/p/31401897.
(5) docker启动容器全过程以及原理-腾讯云开发者社区-腾讯云. https://cloud.tencent.com/developer/article/2246223.
(6) undefined. https://docs.docker.com/develop/sdk/%29.
(7) undefined. https://bing.com/search?q=.

# 三 Docker与Kubernetes
Kubernetes和Docker是两种不同但相关的技术，它们都属于容器化的领域。容器是一种轻量级的虚拟化技术，可以在隔离的环境中运行应用程序和其依赖项，从而提高可移植性、安全性和效率。Docker是目前最流行的容器引擎，它可以创建、运行和管理容器。Kubernetes是一个开源的容器集群管理系统，它可以实现容器集群的自动化部署、自动扩缩容、维护等功能。

Kubernetes和Docker之间的关系可以说是相辅相成的。Docker为Kubernetes提供了强大的容器运行时环境，而Kubernetes则为Docker容器提供了自动化管理和编排的能力。简而言之，Docker解决了应用程序的打包和运行问题，而Kubernetes解决了应用程序的分布式管理和扩展问题。

如果你想了解更多关于Kubernetes和Docker的知识，你可以参考以下一些网页：

- [Kubernetes和Docker到底有啥关系？ - 知乎](^1^)：这是一个详细介绍了Kubernetes和Docker的架构和工作原理的文章，包含了很多图解和示例。
- [简单聊聊k8s，和docker之间的关系 - 掘金](^2^)：这是一个简单聊聊Kubernetes和Docker之间的关系的博客，用通俗易懂的语言进行了解释。
- [Kubernetes 和 Docker 的关系 - zourui4271 - 博客园](^3^)：这是一个从虚拟化角度和部署角度分析了Kubernetes和Docker之间的关系的博客，用两张图进行了对比。
- [简单聊聊k8s，和docker之间的关系 - CSDN博客](^4^)：这是一个转载了掘金博客²的网页，内容与²相同。

源: 与必应的对话， 2023/9/26
(1) Kubernetes和Docker到底有啥关系？ - 知乎. https://zhuanlan.zhihu.com/p/87186261.
(2) 简单聊聊k8s，和docker之间的关系 - 掘金. https://juejin.cn/post/7224149297951195193.
(3) Kubernetes 和 Docker 的关系 - zourui4271 - 博客园. https://www.cnblogs.com/zourui4271/p/14118708.html.
(4) 简单聊聊k8s，和docker之间的关系 - CSDN博客. https://blog.csdn.net/qq_34050399/article/details/130279127.
(5) 简单聊聊k8s，和docker之间的关系 - 掘金. https://juejin.cn/post/7224149297951195193.

