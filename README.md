# k8s-controller-example

CRD+Controller是K8s中用来扩展API的一种方式，[Kubernetes模式](https://www.redhat.com/en/engage/kubernetes-containers-architecture-s-201910240918)一书中有介绍Controller模式和Operator模式，应该说Operator模式是Controller模式的高级版本，
它在Controler模式基础上引入CRD，使得Controller不仅能够监视K8s原生的资源类型，也能利用自定义资源类型来做更丰富的应用管理和自动化。
Controller 主动监控和维护一组Kubernetes 资源,使其处于所需的状态，本示例主要讨论如何创建一个Controller，而如何创建CRD可以参考[官方文档](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/)或者[从零开始写CRD](https://www.servicemesher.com/blog/kubernetes-crd-quick-start/)，不在这里讨论。

目前基本上有两种方式来构建一个Controller，一个是仿照官方提供的示例代码，一是利用kuberbuilder工具来构建。个人感觉如果还没深入了解Controller的必要组成，最好还是先仿照示例代码来写，kubebuilder虽然提供了一些方便，但有一些门槛，如果在各个概念还不清楚的情况下使用，一开始容易晕头转向，并不是一个对初学者友好的方式。

## sample-controller

如果不借助任何工具来创建一个Controller，建议参考官方提供的[sample-controller](https://github.com/kubernetes/sample-controller),这里简单说下里面的结构和一些必要元素。

首先看下目录结构主体：
```
├── CONTRIBUTING.md
├── LICENSE
├── OWNERS
├── README.md
├── SECURITY_CONTACTS
├── artifacts
│   └── examples
│       ├── crd-status-subresource.yaml
│       ├── crd.yaml
│       └── example-foo.yaml
├── code-of-conduct.md
├── controller.go
├── controller_test.go
├── docs
│   ├── controller-client-go.md
│   └── images
│       └── client-go-controller-interaction.jpeg
├── go.mod
├── go.sum
├── hack
│   ├── boilerplate.go.txt
│   ├── custom-boilerplate.go.txt
│   ├── tools.go
│   ├── update-codegen.sh
│   └── verify-codegen.sh
├── main.go
└── pkg
    ├── apis
    │   └── samplecontroller
    │       ├── register.go
    │       └── v1alpha1
    ├── generated
    │   ├── clientset
    │   ├── informers
    │   └── listers
    └── signals
        ├── signal.go
        ├── signal_posix.go
        └── signal_windows.go
```

`pkg/apis/samplecontroller/v1apha1`下是用户自定义资源类型的源代码，CRD的定义在`artifacts/examples/crd.yaml`，如果CRD数据结构比较简单，那么仿照示例里的yaml文件写即可，scheme需要和源代码里定义的一致。如果比较复杂，可以借助kubebuilder工具来做，后面会介绍。
pkg/generated文件夹里的文件和`pkg/apis/samplecontroller/v1alpha1/zz_generated.deepcopy.go`是通过[code-generator](https://github.com/kubernetes/code-generator)自动生成的。如果pkg/apis/samplecontroller/v1apha1路径下的类型定义有任何变化，需要手动运行`./hack/update-codegen.sh`脚本来重新生成所有自动代码。

`main.go`是程序的入口，处理CRD的Controller代码逻辑在`controller.go`。
`main.go`中主要负责初始化Controller,下面是初始化最核心的部分，可以看到Controller被初始化后调用了`controller.Run(2, stopCh)`方法，该进程通过stopCh来接收系统的关闭信号，当收到关闭信号Controller才会退出。

```golang
kubeInformerFactory := kubeinformers.NewSharedInformerFactory(kubeClient, time.Second*30)
exampleInformerFactory := informers.NewSharedInformerFactory(exampleClient, time.Second*30)

controller := NewController(kubeClient, exampleClient,
	kubeInformerFactory.Apps().V1().Deployments(),
	exampleInformerFactory.Samplecontroller().V1alpha1().Foos())

// notice that there is no need to run Start methods in a separate goroutine. (i.e. go kubeInformerFactory.Start(stopCh)
// Start method is non-blocking and runs all registered informers in a dedicated goroutine.
kubeInformerFactory.Start(stopCh)
exampleInformerFactory.Start(stopCh)

if err = controller.Run(2, stopCh); err != nil {
	klog.Fatalf("Error running controller: %s", err.Error())
}
```

`controller.go`中`Run(threadiness int, stopCh <-chan struct{})`方法里有如下代码,可以看到`main.go`中传入的2表示启用两个线程来跑`runWorker`
```golang
for i := 0; i < threadiness; i++ {
	go wait.Until(c.runWorker, time.Second, stopCh)
}
```

`runWorker`方法如下，内容比较简单，就是不停的调用`processNextWorkItem()`。
```golang
func (c *Controller) runWorker() {
	for c.processNextWorkItem() {
	}
}
```
`processNextWorkItem()`方法主要处理CRD对象队列，而其中调用的`syncHandler(key string)`方法才是处理CRD的核心逻辑。

整个Controller代码核心跟K8s提供的`clieng-go`代码库有很紧密的关系，示例代码里的文档也给了[介绍](https://github.com/kubernetes/sample-controller/blob/master/docs/controller-client-go.md)，另外可以参看[Informer源码分析](https://jimmysong.io/kubernetes-handbook/develop/client-go-informer-sourcecode-analyse.html)以及[client-go学习](https://qiankunli.github.io/2020/07/20/client_go.html)
以上是构建Controller最核心的部分，参照示例代码再根据自己的需求构建CRD和`syncHandler(key string)`就可以自定义一个自己的Controller。

## kubebuilder

关于kubebuilder，官方提供了很详细的[使用文档](https://book.kubebuilder.io/introduction.html),也有云原生社区翻译的[中文版](https://cloudnative.to/kubebuilder/introduction.html)，我在这里仅记录在试用过程中生成本示例代码库的一些重点。

### GKV & GKR

在介绍本项目的构建具体内容之前，先来了解下K8s中[GKVR](https://cloudnative.to/kubebuilder/cronjob-tutorial/gvks.html)的概念，GVK = Group Version Kind, GVR = Group Version Resources.
Kubernetes 中的 API 有不同的Group，而API Group简单来说就是相关功能的集合。每个Group都有一个或多个版本，顾名思义，它允许我们随着时间的推移改变 API 的职责。
每个 API Group-Version 包含一个或多个 API 类型，我们称之为 Kinds。resources（资源） 只是 API 中的一个 Kind 的使用方式。通常情况下，Kind 和 resources 之间有一个一对一的映射，但也有例外， 比如`Scale` Kind。

### 构建指南

在我本地使用的kubebuilder版本是`v3.0.0-alpha.0-184-g84f357b5`，关于如何安装kubebuilder和快速使用可以看[这里](https://cloudnative.to/kubebuilder/quick-start.html)。

1. 初始化项目: `kubebuilder init --domain demo.io`,其中`demo.io`是域名，作用是？。。。
2. 启用multigroup: `kubebuilder edit --multigroup=true`,如果对API有分组的要求，需要启用multigroup，这一步让KubeBuilder在 PROJECT 中新增一行`multigroup: true`，标记该项目是一个 Multi-group 项目，主要改变的是后续命令执行时生成的文件路径结构。具体差异可以看[这里](https://cloudnative.to/kubebuilder/migration/multi-group.html)的介绍。
3. 创建API: `kubebuilder create api --group batch --version v1alpha1 --kind ServiceExport --controller --resource`，如果不带参数`--controller --resource`,创建API时会弹出提示让用户选择是否创建controller和resource。额外的参数可以通过`kubebuilder create api -h`查看。

经过前面三步后，来看下目前的项目结构, 我在重要文件或文件夹后面加了注释。
```
.
├── Dockerfile                                   -----> 构建Controller镜像所用的Dockerfile
├── Makefile
├── PROJECT                                      -----> Kubebuilder用到的项目元数据
├── README.md
├── apis
│   └── batch
│       └── v1alpha1                             -----> 自定义CRD所在目录
│           ├── groupversion_info.go             -----> GroupVersion的元数据，用于CRD生成以及Scheme创建方法
│           ├── serviceexport_types.go           -----> 自定义CRD类型文件，若想创建多个Kind，可以重复第三步，替换ServiceExport为新的Kind名称。
│           └── zz_generated.deepcopy.go         -----> kubebuilder调用controller-gen自动生成的runtime.Object接口的实现。
├── bin
│   └── controller-gen
├── config
│   ├── crd                                      -----> 包含部署CRD的YAML模版的文件夹
│   │   ├── kustomization.yaml
│   │   ├── kustomizeconfig.yaml
│   │   └── patches
│   ├── default
│   │   ├── kustomization.yaml
│   │   ├── manager_auth_proxy_patch.yaml
│   │   └── manager_config_patch.yaml
│   ├── manager                                 -----> 包含部署Controller的YAML模版的文件夹
│   │   ├── controller_manager_config.yaml
│   │   ├── kustomization.yaml
│   │   └── manager.yaml
│   ├── prometheus
│   │   ├── kustomization.yaml
│   │   └── monitor.yaml
│   ├── rbac                                    -----> 包含所有Controller运行所需的RBAC的文件夹
│   │   ├── auth_proxy_client_clusterrole.yaml
│   │   ├── auth_proxy_role.yaml
│   │   ├── auth_proxy_role_binding.yaml
│   │   ├── auth_proxy_service.yaml
│   │   ├── kustomization.yaml
│   │   ├── leader_election_role.yaml
│   │   ├── leader_election_role_binding.yaml
│   │   ├── role_binding.yaml
│   │   ├── service_account.yaml
│   │   ├── serviceexport_editor_role.yaml
│   │   └── serviceexport_viewer_role.yaml
│   └── samples
│       └── batch_v1alpha1_serviceexport.yaml
├── controllers
│   └── batch
│       ├── serviceexport_controller.go         -----> 自定义Controller核心逻辑的文件
│       └── suite_test.go
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
└── main.go                                     -----> Controller的进程入口
```

4. 在不修改任何代码的情况下，我们运行`make manifests`后会发现新增两个文件如下：
```
├── config
│   ├── crd
│   │   ├── bases
│   │   │   └── batch.demo.io_serviceexports.yaml-----> 可以用来部署ServiceExport CRD的完整YAML文件
│   ├── rbac
│   │   ├── role.yaml                            -----> 运行Controller所需的完整RBAC文件
```

5. 增加一个新的API Kind:`kubebuilder create api --group batch --version v1alpha1 --kind ServiceImport --controller --resource`，再来看看项目差异：

a. 文件改动：
* PROJECT
* apis/batch/v1alpha1/zz_generated.deepcopy.go
* config/crd/kustomization.yaml
* controllers/batch/suite_test.go
* main.go: 每添加一个新Kind，main.go中就会添加新类型的Reconciler初始化步骤。
```go
if err = (&batchcontrollers.ServiceImportReconciler{
        Client: mgr.GetClient(),
        Log:    ctrl.Log.WithName("controllers").WithName("batch").WithName("ServiceImport"),
        Scheme: mgr.GetScheme(),
}).SetupWithManager(mgr); err != nil {
        setupLog.Error(err, "unable to create controller", "controller", "ServiceImport")
        os.Exit(1)
}
```
b. 新文件：
* apis/batch/v1alpha1/serviceimport_types.go
* config/crd/patches/cainjection_in_serviceimports.yaml
* config/crd/patches/webhook_in_serviceimports.yaml
* config/rbac/serviceimport_editor_role.yaml
* config/rbac/serviceimport_viewer_role.yaml
* config/samples/batch_v1alpha1_serviceimport.yaml
* controllers/batch/serviceimport_controller.go

关于添加新类型后所有改动的详细信息，可以查看相关的[commit](https://github.com/shadowlan/k8s-controller-example/commit/b732fb0fef60598f96795e8c5cb222ea6048b0d5)。

6. 再次运行`make manifests`后，可以看到添加了`config/rbac/role.yaml`文件中针对ServiceExport的权限,并添加了创建ServerExport类型的CRD文件`config/crd/bases/batch.demo.io_serviceimports.yaml`.

经过以上六个步骤后，整体的项目框架已经搭建完毕，可以正式进入Controller具体逻辑代码的编写。在这里主要通过ServiceExport以及相关Controller来介绍一些值得关注的Kubebuilder特性。

More Coming...

## 参考

[kubebuilder 及controller-runtime学习](https://qiankunli.github.io/2020/08/10/controller_runtime.html)
[让编写 CRD 变得更简单](https://mp.weixin.qq.com/s/Gzpq71nCfSBc1uJw3dR7xA)



