# kubectl 创建 Pod 背后到底发生了什么



想象一下，如果我想将 nginx 部署到 Kubernetes 集群，我可能会在终端中输入类似这样的命令：

```bash
$ kubectl run --image=nginx --replicas=3
```

然后回车。几秒钟后，你就会看到三个 nginx pod 分布在所有的工作节点上。这一切就像变魔术一样，但你并不知道这一切的背后究竟发生了什么事情。

Kubernetes 的神奇之处在于：它可以通过用户友好的 API 来处理跨基础架构的 `deployments`，而背后的复杂性被隐藏在简单的抽象中。但为了充分理解它为我们提供的价值，我们需要理解它的内部原理。

本指南将引导您理解从 client 到 `Kubelet` 的请求的完整生命周期，必要时会通过源代码来说明背后发生了什么。



## Kubectl

### **验证和生成器**

当敲下回车键以后，`kubectl` 首先会执行一些客户端验证操作，以确保不合法的请求（例如，创建不支持的资源或使用格式错误的镜像名称）将会快速失败，也不会发送给 `kube-apiserver`。通过减少不必要的负载来提高系统性能。

验证通过之后， kubectl 开始将发送给 kube-apiserver 的 HTTP 请求进行封装。`kube-apiserver` 与 etcd 进行通信，所有尝试访问或更改 Kubernetes 系统状态的请求都会通过 kube-apiserver 进行，kubectl 也不例外。kubectl 使用生成器（generators）来构造 HTTP 请求。生成器是一个用来处理序列化的抽象概念。

通过 `kubectl run` 不仅可以运行 `deployment`，还可以通过指定参数 `--generator` 来部署其他多种资源类型。如果没有指定 `--generator` 参数的值，kubectl 将会自动判断资源的类型。

例如，带有参数 `--restart-policy=Always` 的资源将被部署为 Deployment，而带有参数 `--restart-policy=Never` 的资源将被部署为 Pod。同时 kubectl 也会检查是否需要触发其他操作，例如记录命令（用来进行回滚或审计）。

在 kubectl 判断出要创建一个 Deployment 后，它将使用 `DeploymentV1Beta1` 生成器从我们提供的参数中生成一个运行时对象。



### **API 版本协商与 API 组**

为了更容易地消除字段或者重新组织资源结构，Kubernetes 支持多个 API 版本，每个版本都在不同的 API 路径下，例如 `/api/v1` 或者 `/apis/extensions/v1beta1`。不同的 API 版本表明不同的稳定性和支持级别，更详细的描述可以参考Kubernetes API 概述。

API 组旨在对类似资源进行分类，以便使得 Kubernetes API 更容易扩展。API 的组名在 REST 路径或者序列化对象的 `apiVersion` 字段中指定。例如，Deployment 的 API 组名是 `apps`，最新的 API 版本是 `v1beta2`，这就是为什么你要在 Deployment manifests 顶部输入 `apiVersion: apps/v1beta2`。

kubectl 在生成运行时对象后，开始为它找到适当的 API 组和 API 版本，然后组装成一个版本化客户端，该客户端知道资源的各种 REST 语义。该阶段被称为版本协商，kubectl 会扫描 `remote API` 上的 `/apis` 路径来检索所有可能的 API 组。由于 kube-apiserver 在 `/apis` 路径上公开了 OpenAPI 格式的规范文档， 因此客户端很容易找到合适的 API。

为了提高性能，kubectl [将 OpenAPI 规范缓存](https://mp.weixin.qq.com/s/ctdvbasKE-vpLRxDJjwVMw?tdsourcetag=s_pctim_aiomsg)到了 `~/.kube/cache` 目录。如果你想了解 API 发现的过程，请尝试删除该目录并在运行 kubectl 命令时将 `-v` 参数的值设为最大值，然后你将会看到所有试图找到这些 API 版本的HTTP 请求。参考 kubectl 备忘单。

最后一步才是真正地发送 HTTP 请求。一旦请求发送之后获得成功的响应，kubectl 将会根据所需的输出格式打印 success message。



### **客户端身份认证**

在发送 HTTP 请求之前还要进行客户端认证，这是之前没有提到的，现在可以来看一下。

为了能够成功发送请求，kubectl 需要先进行身份认证。用户凭证保存在 `kubeconfig` 文件中，kubectl 通过以下顺序来找到 kubeconfig 文件：

- 如果提供了 `--kubeconfig` 参数， kubectl 就使用 --kubeconfig 参数提供的 kubeconfig 文件。
- 如果没有提供 --kubeconfig 参数，但设置了环境变量 `$KUBECONFIG`，则使用该环境变量提供的 kubeconfig 文件。
- 如果 --kubeconfig 参数和环境变量 `$KUBECONFIG` 都没有提供，kubectl 就使用默认的 kubeconfig 文件 `$HOME/.kube/config`。

解析完 kubeconfig 文件后，kubectl 会确定当前要使用的上下文、当前指向的群集以及与当前用户关联的任何认证信息。如果用户提供了额外的参数（例如 --username），则优先使用这些参数覆盖 kubeconfig 中指定的值。一旦拿到这些信息之后， kubectl 就会把这些信息填充到将要发送的 HTTP 请求头中：

- x509 证书使用 tls.TLSConfig 发送（包括 CA 证书）。
- `bearer tokens` 在 HTTP 请求头 `Authorization` 中发送。
- 用户名和密码通过 HTTP 基本认证发送。
- `OpenID` 认证过程是由用户事先手动处理的，产生一个像 bearer token 一样被发送的 token。



## kube-apiserver

### 认证

现在我们的请求已经发送成功了，接下来将会发生什么？这时候就该 `kube-apiserver` 闪亮登场了！kube-apiserver 是客户端和系统组件用来保存和检索集群状态的主要接口。为了执行相应的功能，kube-apiserver 需要能够验证请求者是合法的，这个过程被称为认证。

那么 apiserver 如何对请求进行认证呢？当 kube-apiserver 第一次启动时，它会查看用户提供的所有 [CLI 参数](https://mp.weixin.qq.com/s/ctdvbasKE-vpLRxDJjwVMw?tdsourcetag=s_pctim_aiomsg)，并组合成一个合适的令牌列表。

**举个例子：**如果提供了 `--client-ca-file` 参数，则会将 x509 客户端证书认证添加到令牌列表中；如果提供了 `--token-auth-file` 参数，则会将 breaer token 添加到令牌列表中。

每次收到请求时，apiserver 都会[通过令牌链进行认证，直到某一个认证成功为止](https://mp.weixin.qq.com/s/ctdvbasKE-vpLRxDJjwVMw?tdsourcetag=s_pctim_aiomsg)：

- [x509 处理程序](https://mp.weixin.qq.com/s/ctdvbasKE-vpLRxDJjwVMw?tdsourcetag=s_pctim_aiomsg)将验证 HTTP 请求是否是由 CA 根证书签名的 TLS 密钥进行编码的。
- [bearer token 处理程序](https://mp.weixin.qq.com/s/ctdvbasKE-vpLRxDJjwVMw?tdsourcetag=s_pctim_aiomsg)将验证 `--token-auth-file` 参数提供的 token 文件是否存在。
- [基本认证处理程序](https://mp.weixin.qq.com/s/ctdvbasKE-vpLRxDJjwVMw?tdsourcetag=s_pctim_aiomsg)确保 HTTP 请求的基本认证凭证与本地的状态匹配。

如果[认证失败](https://mp.weixin.qq.com/s/ctdvbasKE-vpLRxDJjwVMw?tdsourcetag=s_pctim_aiomsg)，则请求失败并返回相应的错误信息；如果验证成功，则将请求中的 `Authorization` 请求头删除，并[将用户信息添加到](https://mp.weixin.qq.com/s/ctdvbasKE-vpLRxDJjwVMw?tdsourcetag=s_pctim_aiomsg)其上下文中。这给后续的授权和准入控制器提供了访问之前建立的用户身份的能力。



### **授权**

OK，现在请求已经发送，并且 kube-apiserver 已经成功验证我们是谁，终于解脱了！

然而事情并没有结束，虽然我们已经证明了**我们是合法的**，但我们有权执行此操作吗？毕竟身份和权限不是一回事。为了进行后续的操作，kube-apiserver 还要对用户进行授权。

kube-apiserver 处理授权的方式与处理身份验证的方式相似：通过 kube-apiserver 的启动参数 `--authorization_mode`参数设置。它将组合一系列授权者，这些授权者将针对每个传入的请求进行授权。如果所有授权者都拒绝该请求，则该请求会被禁止响应并且[不会再继续响应](https://mp.weixin.qq.com/s/ctdvbasKE-vpLRxDJjwVMw?tdsourcetag=s_pctim_aiomsg)。如果某个授权者批准了该请求，则请求继续。

kube-apiserver 目前支持以下几种授权方法：

- [webhook](https://mp.weixin.qq.com/s/ctdvbasKE-vpLRxDJjwVMw?tdsourcetag=s_pctim_aiomsg): 它与集群外的 HTTP(S) 服务交互。
- [ABAC](https://mp.weixin.qq.com/s/ctdvbasKE-vpLRxDJjwVMw?tdsourcetag=s_pctim_aiomsg): 它执行静态文件中定义的策略。
- [RBAC](https://mp.weixin.qq.com/s/ctdvbasKE-vpLRxDJjwVMw?tdsourcetag=s_pctim_aiomsg): 它使用 `rbac.authorization.k8s.io` API Group实现授权决策，允许管理员通过 Kubernetes API 动态配置策略。
- [Node](https://mp.weixin.qq.com/s/ctdvbasKE-vpLRxDJjwVMw?tdsourcetag=s_pctim_aiomsg): 它确保 kubelet 只能访问自己节点上的资源。



### **准入控制**

突破了之前所说的认证和授权两道关口之后，客户端的调用请求就能够得到 API Server 的真正响应了吗？答案是：不能！

从 kube-apiserver 的角度来看，它已经验证了我们的身份并且赋予了相应的权限允许我们继续，但对于 Kubernetes 而言，其他组件对于应不应该允许发生的事情还是很有意见的。所以这个请求还需要通过 `Admission Controller` 所控制的一个 `准入控制链` 的层层考验，官方标准的 “关卡” 有近十个之多，而且还能自定义扩展！

虽然授权的重点是回答用户是否有权限，但准入控制器会拦截请求以确保它符合集群的更广泛的期望和规则。它们是资源对象保存到 `etcd` 之前的最后一个堡垒，封装了一系列额外的检查以确保操作不会产生意外或负面结果。不同于授权和认证只关心请求的用户和操作，准入控制还处理请求的内容，并且仅对创建、更新、删除或连接（如代理）等有效，而对读操作无效。

> 准入控制器的工作方式与授权者和验证者的工作方式类似，但有一点区别：与验证链和授权链不同，如果某个准入控制器检查不通过，则整个链会中断，整个请求将立即被拒绝并且返回一个错误给终端用户。

准入控制器设计的重点在于提高可扩展性，某个控制器都作为一个插件存储在 `plugin/pkg/admission` 目录中，并且与某一个接口相匹配，最后被编译到 kube-apiserver 二进制文件中。

大部分准入控制器都比较容易理解，接下来着重介绍 `SecurityContextDeny`、`ResourceQuota` 及 `LimitRanger`这三个准入控制器。

- **SecurityContextDeny** 该插件将禁止创建设置了 Security Context 的 Pod。
- **ResourceQuota** 不仅能限制某个 Namespace 中创建资源的数量，而且能限制某个 Namespace 中被 Pod 所请求的资源总量。该准入控制器和资源对象 `ResourceQuota` 一起实现了资源配额管理。
- **LimitRanger** 作用类似于上面的 ResourceQuota 控制器，针对 Namespace 资源的每个个体（Pod 与 Container 等）的资源配额。该插件和资源对象 `LimitRange` 一起实现资源配额管理。



## etcd

