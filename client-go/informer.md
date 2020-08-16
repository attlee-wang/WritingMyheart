# Informer机制.md

0.参考

* [kubernetes 中 informer 的使用](https://blog.tianfeiyu.com/2019/05/17/client-go_informer/)
* [k8s-controller-custom-resource](https://github.com/resouer/k8s-controller-custom-resource)
* [极客时间深入剖析kubernetes](https://time.geekbang.org/column/article/41876)

## 一、基础

### 1.为什么需要informer?

​为了减轻对kube-apiserver的访问压力。Informer创建了本地缓存, Lister\(\)方法可以直接查找缓存在本地内存中的数据, Watch\(\)方法则可以用来更新本地缓存。客户端对kube-apiserver 数据的读取和监听操作都通过本地informer来进行，通过缓存机制，减少了对kube-apiserver的访问。

### 2.定义

​ Informer其实就是一个带有本地缓存、索引机制、可以注册EventHandler的 kube-client。

### 3.作用

* 同步数据到本地缓存，并且带有索引，便于快速查找（对kube-apiserver对象的缓存client）
* 根据对应的事件类型，触发事先注册好的ResourceEventHandler（自定义 controller 中使用）

### 4.Informer工作流程图

![Informer&#x5DE5;&#x4F5C;&#x6D41;&#x7A0B;&#x56FE;](https://raw.githubusercontent.com/attlee-wang/myimage/master/image/informer-1.png)

* Reflector 反射器: 实现对 apiserver 指定类型对象的ListAndWatch, 把监控的结果实例化成具体的对象（list的那些对象？）
* DeltaIFIFO Queue 增量队列: 将 Reflector 监控到的变化对象形成一个 FIFO 队列（为什么需要？）
* Indexer 索引: 本地缓存对象的索引
* Local Store 本地缓存: informer 的 cache, LocalStore 只会被 Lister 的 List/Get 方法访问。 并且非全部数据，有部分数据还在DeltaFIFO 中。
* WorkQueue 事件队列：DeltaIFIFO 收到数据后会先将数据存储在自己的数据结构中，然后直接操作 Store 中存储的数据，更新完 store 后 DeltaIFIFO 会将该事件 pop 到 WorkQueue 中，Controller 收到 WorkQueue 中的事件会根据对应的类型触发对应的回调函数。\(这块的先后顺序？\)

## 二、使用

### 1.作为 client 使用示例

```go
package main

import (
    "flag"
    "fmt"
    "log"
    "path/filepath"

    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/labels"
    "k8s.io/apimachinery/pkg/util/runtime"

    "k8s.io/client-go/informers"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/cache"
    "k8s.io/client-go/tools/clientcmd"
    "k8s.io/client-go/util/homedir"
)

func main() {
    var kubeconfig *string
    if home := homedir.HomeDir(); home != "" {
        kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "(optional) absolute path to the kubeconfig file")
    } else {
        kubeconfig = flag.String("kubeconfig", "", "absolute path to the kubeconfig file")
    }
    flag.Parse()

    config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
    if err != nil {
        panic(err)
    }

    // 初始化 client
    clientset, err := kubernetes.NewForConfig(config)
    if err != nil {
        log.Panic(err.Error())
    }

    //阻塞主程序
    stopper := make(chan struct{})
    defer close(stopper)

    // 初始化 informer
    factory := informers.NewSharedInformerFactory(clientset, 0) // 0 ???
    nodeInformer := factory.Core().V1().Nodes()
    informer := nodeInformer.Informer()
    defer runtime.HandleCrash()

    // 启动 informer，list & watch
    go factory.Start(stopper)

    // 从 apiserver 同步资源，即 list 
    if !cache.WaitForCacheSync(stopper, informer.HasSynced) {
        runtime.HandleError(fmt.Errorf("Timed out waiting for caches to sync"))
        return
    }

    // 使用自定义 handler
    informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc:    onAdd,
        UpdateFunc: func(interface{}, interface{}) { fmt.Println("update not implemented") }, 
        DeleteFunc: func(interface{}) { fmt.Println("delete not implemented") },
    })

    // 创建 lister
    nodeLister := nodeInformer.Lister()
    // 从 lister 中获取所有 items
    nodeList, err := nodeLister.List(labels.Everything())
    if err != nil {
        fmt.Println(err)
    }
    fmt.Println("nodelist:", nodeList)
    <-stopper
}

func onAdd(obj interface{}) {
    node := obj.(*corev1.Node)
    fmt.Println("add a node:", node.Name)
}
```

* SharedInformerFactory，Shared指的是在多个Informer/lister共享一个本地cache, 资源的变化会同时通知到cache和 listers
* lister指的就是OnAdd、OnUpdate、OnDelete 这些回调函数背后的对象。

### 2. 作为 controller 使用示例（如何编写一个CRD）

controller 的使用示例我们参考：[k8s-controller-custom-resource](https://github.com/resouer/k8s-controller-custom-resource)，[k8s-controller-custom-resource](https://github.com/resouer/k8s-controller-custom-resource) 是一个简单的网络CRD，本身并不具有实用性，但是作为CRD的实例在简单不过。下面我们对 [k8s-controller-custom-resource](https://github.com/resouer/k8s-controller-custom-resource) 做简单解析，看如何通过 Informers 编写一个完整的CRD, 也就是一个controller的完整应用。

#### 1.创建CRD目录和文件

我们看下[k8s-controller-custom-resource](https://github.com/resouer/k8s-controller-custom-resource)的核心目录结构：

```go
$ tree k8s-controller-custom-resource
.
├── controller.go
├── crd
│   └── network.yaml
├── example
│   └── example-network.yaml
├── main.go
└── pkg
    └── apis
        └── samplecrd
            ├── constants.go
            └── v1
                ├── doc.go
                ├── register.go
                └── types.go
```

我们要编写一个CRD就要创建如上的一个目录：

* crd: CRD的定义

  ```yaml
    apiVersion: apiextensions.k8s.io/v1beta1 
    kind: CustomResourceDefinition
    metadata:
      name: networks.samplecrd.k8s.io
    spec:
      group: samplecrd.k8s.io  ## CRD的group
      version: v1              ## CRD的version
      names:
        kind: Network          ## 资源类型名
        plural: networks       ## 资源类型的复数
      scope: Namespaced        ## 定义的作用范围属于 Namespaced 级别
  ```

* example: 这个CRD的具体实例

  ```yaml
   apiVersion: samplecrd.k8s.io/v1
   kind: Network
   metadata:
     name: example-network
   spec:
     cidr: "192.168.0.0/16"
     gateway: "192.168.0.1"
  ```

* pkg/apis/samplecrd: CRD 所在的group的名字
* pkg/apis/samplecrd/v1: CRD的版本
  * v1/doc.go：Golang 的文档源文件, 全局代码生成的控制文件，也被称为 Global Tags。

    ```text
    // +k8s:deepcopy-gen=package   
    // +groupName=samplecrd.k8s.io # 定义了这个包对应的 API 组的名字
    package v1
    ```

    > // +k8s:deepcopy-gen=package
    >
    > ​ +\[=value\]格式的注释：Kubernetes 进行代码生成要用的 Annotation 风格的注释，意思是为整个 v1 包里的所有类型定义自动生成 DeepCopy 方法。
    >
    > ​ 也可以在v1的type包里，明确定义给那些对象生成DeepCopy 方法，即显示的加上+k8s:deepcopy-gen=true，也可以用+k8s:deepcopy-gen=false 来阻止为某些类型生成 DeepCopy。

  * v1/types.go：定义Network 类型

    ```go
      package v1

      // 注意下面的三行, 是Network代码生成注释，意思如下
      // +genclient //请为Network这个 API 资源类型生成对应的 Client 代码
      // +genclient:noStatus //Network这个 API 资源类型定义里，没有 Status 字段, 否则，生成的 Client 就会自动带上 UpdateStatus 方法
      // +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
      // 上面这句意思是：在生成 DeepCopy 的时候，为Network实现 Kubernetes 提供的 runtime.Object 接口，固定操作，否则在某些版本的 Kubernetes 里，定义的类型定义会出现编译错误

      // Network describes a Network resource
      type Network struct {
          // TypeMeta is the metadata for the resource, like kind and apiversion
          metav1.TypeMeta `json:",inline"`
          // ObjectMeta contains the metadata for the particular object, including
          // things like...
          //  - name
          //  - namespace
          //  - self link
          //  - labels
          //  - ... etc ...
          metav1.ObjectMeta `json:"metadata,omitempty"`

          // Spec is the custom resource spec
          Spec NetworkSpec `json:"spec"`
      }

      // NetworkSpec is the spec for a Network resource
      type NetworkSpec struct { // 定义一个了Network具体的字段，自己定义的部分
          // Cidr and Gateway are example custom spec fields
          //
          // this is where you would put your custom resource data
          Cidr    string `json:"cidr"`//字段被转换成 JSON 格式之后的名字，就是 YAML 文件里的字段名字
          Gateway string `json:"gateway"` 
      }

      // +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
      // 上面这句意思是：生成 DeepCopy 的时为NetworkList实现 Kubernetes 提供的 runtime.Object 接口
      /// NetworkList没有+genclient，因为NetworkList 只是一个返回值类型，Network 才是“主类型”

      // NetworkList is a list of Network resources
      type NetworkList struct { //描述一组 Network 对象应该包括哪些字段, 供list()方法使用
          metav1.TypeMeta `json:",inline"`
          metav1.ListMeta `json:"metadata"`

          Items []Network `json:"items"`
      }
    ```

    > Kubernetes 的代码生成语法，可参考：[Kubernetes Deep Dive: Code Generation for CustomResources](https://blog.openshift.com/kubernetes-deep-dive-code-generation-customresources/)

  * v1/register.go：注册一个类型给 kube-api-server

    ```go
      package v1
      ...
      // GroupVersion is the identifier for the API which includes
      // the name of the group and the version of the API
      var SchemeGroupVersion = schema.GroupVersion{
          Group:   samplecrd.GroupName,
          Version: samplecrd.Version,
      }

      // Resource takes an unqualified resource and returns a Group qualified GroupResource
      func Resource(resource string) schema.GroupResource {
          return SchemeGroupVersion.WithResource(resource).GroupResource()
      }

      // Kind takes an unqualified kind and returns back a Group qualified GroupKind
      func Kind(kind string) schema.GroupKind {
          return SchemeGroupVersion.WithKind(kind).GroupKind()
      }

      // create a SchemeBuilder which uses functions to add types to
      // the scheme
      var (
          SchemeBuilder = runtime.NewSchemeBuilder(addKnownTypes)
          AddToScheme   = SchemeBuilder.AddToScheme
      )

      // addKnownTypes adds our types to the API scheme by registering
      // Network and NetworkList
      func addKnownTypes(scheme *runtime.Scheme) error {
          scheme.AddKnownTypes(
              SchemeGroupVersion,
              &Network{},
              &NetworkList{},
          )

          // register the type in the scheme
          metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
          return nil
      }
    ```

    > ​ 注册一个类型给 kube-api-server, 由CRD的定义告诉kube-api-server, kube-api-server自动帮我们完成了，定义这个文件主要是告诉客户端 Network 资源类型的定义。
    >
    > ​ 最主要的就是addKnownTypes\(\)方法，这个方法在Kubernetes 后面生成客户端的时候，告诉”Network“ 和 NetworkList 类型的定义。

#### 2.生成 clientset、informer 和 lister

代码生成工具自动生成client、informer和lister如下具体操作如下：

```text
# 代码生成的工作目录，也就是我们的项目路径
$ ROOT_PACKAGE="github.com/resouer/k8s-controller-custom-resource"
# API Group
$ CUSTOM_RESOURCE_NAME="samplecrd"
# API Version
$ CUSTOM_RESOURCE_VERSION="v1"

# 安装k8s.io/code-generator
$ go get -u k8s.io/code-generator/...
$ cd $GOPATH/src/k8s.io/code-generator

# 执行代码自动生成，其中pkg/client是生成目标目录，pkg/apis是类型定义目录
$ ./generate-groups.sh all "$ROOT_PACKAGE/pkg/client" "$ROOT_PACKAGE/pkg/apis" "$CUSTOM_RESOURCE_NAME:$CUSTOM_RESOURCE_VERSION"
```

执行完上面的代码生成命令后，项目就变成如下样子：

```go
$ tree
.
├── controller.go
├── crd
│   └── network.yaml
├── example
│   └── example-network.yaml
├── main.go
└── pkg
    ├── apis
    │   └── samplecrd
    │       ├── constants.go
    │       └── v1
    │           ├── doc.go
    │           ├── register.go
    │           ├── types.go
    │           └── zz_generated.deepcopy.go
    └── client
        ├── clientset
        ├── informers
        └── listers
```

* zz\_generated.deepcopy.go: 自动生成的 DeepCopy 代码文件
* client：Kubernetes 为 Network 类型生成的客户端库
  * clientset:  操作 Network 对象所需要使用的客户端
  * informers
  * listers

#### 3.编写自定义控制器编写main函数

1. 编写main函数

   ```go
   func main() {
       flag.Parse()

       // set up signals so we handle the first shutdown signal gracefully
       stopCh := signals.SetupSignalHandler()

       cfg, err := clientcmd.BuildConfigFromFlags(masterURL, kubeconfig)
       if err != nil {
           glog.Fatalf("Error building kubeconfig: %s", err.Error())
       }

     // 1. 创建一个 Kubernetes 的 client
       kubeClient, err := kubernetes.NewForConfig(cfg)
       if err != nil {
           glog.Fatalf("Error building kubernetes clientset: %s", err.Error())
       }

     // 2. 创建一个 Network 对象的 client
       networkClient, err := clientset.NewForConfig(cfg)
       if err != nil {
           glog.Fatalf("Error building example clientset: %s", err.Error())
       }

     // 3. 创建Network对象的InformerFactory的工厂, 并启动其Informer
       networkInformerFactory := informers.NewSharedInformerFactory(networkClient, time.Second*30)

     go networkInformerFactory.Start(stopCh)

     // 4. 创建Network对象的控制器，并启动控制循环
       controller := NewController(kubeClient, networkClient,
           networkInformerFactory.Samplecrd().V1().Networks())

       if err = controller.Run(2, stopCh); err != nil {
           glog.Fatalf("Error running controller: %s", err.Error())
       }
   }
   ```

2. 编写自定义控制器的定义

   ```go
   // NewController returns a new network controller
   func NewController(
       kubeclientset kubernetes.Interface,
       networkclientset clientset.Interface,
       networkInformer informers.NetworkInformer) *Controller {
       ...
       //workqueue: 负责同步Informer和控制循环之间的数据, 实际入队的并不是 API 对象本身，而是它们的 Key，即：该 API 对象的<namespace>/<name>
       controller := &Controller{
           kubeclientset:    kubeclientset,
           networkclientset: networkclientset,
           networksLister:   networkInformer.Lister(),
           networksSynced:   networkInformer.Informer().HasSynced,
           workqueue:        workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "Networks"),
           recorder:         recorder,
       }

       // Set up an event handler for when Network resources change
       networkInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
           AddFunc: controller.enqueueNetwork,
           UpdateFunc: func(old, new interface{}) {
               oldNetwork := old.(*samplecrdv1.Network)
               newNetwork := new.(*samplecrdv1.Network)
               if oldNetwork.ResourceVersion == newNetwork.ResourceVersion {
               // Periodic resync will send update events for all known Networks.
               // Two different versions of the same Network will always have different RVs.
                   return
               }
               controller.enqueueNetwork(new)
           },
           DeleteFunc: controller.enqueueNetworkForDelete,
       })

       return controller
   }
   ```

3. 编写自定义控制器的业务逻辑
4. controller.Run\(\)

   控制循环的入口在controller.Run\(\)里

   ```go
       ...
       glog.Info("Waiting for informer caches to sync")
       if ok := cache.WaitForCacheSync(stopCh, c.networksSynced); !ok {
           return fmt.Errorf("failed to wait for caches to sync")
       }// 等待 Informer 完成一次本地缓存的数据同步操作

       glog.Info("Starting workers")
       // Launch two workers to process Network resources
       for i := 0; i < threadiness; i++ { 
           go wait.Until(c.runWorker, time.Second, stopCh)// 并发多个worker,周期性的处理业务逻辑
       }
       ...
   ```

5. 自定义控制器的业务逻辑

   ```go
   func (c *Controller) runWorker() {
       for c.processNextWorkItem() {
       }
   }

   func (c *Controller) processNextWorkItem() bool {
       obj, shutdown := c.workqueue.Get() //从workqueue得到一个key
       ...
       // We wrap this block in a func so we can defer c.workqueue.Done.
       err := func(obj interface{}) error {
           ...
           if err := c.syncHandler(key); err != nil {// 执行具体的控制循环
               return fmt.Errorf("error syncing '%s': %s", key, err.Error())
           }
           c.workqueue.Forget(obj)       //删除梳理的key
           glog.Infof("Successfully synced '%s'", key)
           return nil
       }(obj)
       ...
       return true
   }

   func (c *Controller) syncHandler(key string) error {
       // Convert the namespace/name string into a distinct namespace and name
       namespace, name, err := cache.SplitMetaNamespaceKey(key)
       ...
       //通过索引从Informer缓存中得到network对象
       network, err := c.networksLister.Networks(namespace).Get(name)
       if err != nil {
           // The Network resource may no longer exist, in which case we stop
           // processing.
           if errors.IsNotFound(err) {//有可能key在但是对象已经被删除了
               glog.Warningf("Network: %s/%s does not exist in local cache, will delete it from Neutron ...",
                   namespace, name)

               glog.Infof("[Neutron] Deleting network: %s/%s ...", namespace, name)

               // FIX ME: call Neutron API to delete this network by name.
               //  Neutron API：调用真是的对象操作接口，删除真实的对象
               // neutron.Delete(namespace, name)

               return nil
           }

           runtime.HandleError(fmt.Errorf("failed to list network by: %s/%s", namespace, name))

           return err
       }

       glog.Infof("[Neutron] Try to process network: %#v ...", network)

       // 对比期望状态和实际状态不同，及采用的策略
       // FIX ME: Do diff().
       //
       // actualNetwork, exists := neutron.Get(namespace, name)
       //
       // if !exists {
       //     neutron.Create(namespace, name)
       // } else if !reflect.DeepEqual(actualNetwork, network) {
       //     neutron.Update(namespace, name)
       // }

       c.recorder.Event(network, corev1.EventTypeNormal, SuccessSynced, MessageResourceSynced)
       return nil
   }
   ```

