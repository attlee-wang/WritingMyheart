# Informer机制.md



\[top\]

1. 参考
2. [kubernetes 中 informer 的使用](https://blog.tianfeiyu.com/2019/05/17/client-go_informer/)

### 一、基础

1.为什么需要informer?

> 为了减轻对kube-apiserver的访问压力。Informer创建了本地缓存, Lister\(\)方法可以直接查找缓存在本地内存中的数据, Watch\(\)方法则可以用来更新本地缓存。客户端对kube-apiserver 数据的读取和监听操作都通过本地informer来进行，通过缓存机制，减少了对kube-apiserver的访问。

2.定义

> Informer其实就是一个带有本地缓存、索引机制、可以注册EventHandler的 kube-client。

3.作用

* 同步数据到本地缓存，并且带有索引，便于快速查找（对kube-apiserver对象的缓存client）
* 根据对应的事件类型，触发事先注册好的ResourceEventHandler（自定义 controller 中使用）

4.Informer工作流程图

![Informer&#x5DE5;&#x4F5C;&#x6D41;&#x7A0B;&#x56FE;](http://cdn.tianfeiyu.com/informer-1.png)

* Reflector 反射器: 实现对 apiserver 指定类型对象的ListAndWatch, 把监控的结果实例化成具体的对象（list的那些对象？）
* DeltaIFIFO Queue 增量队列: 将 Reflector 监控到的变化对象形成一个 FIFO 队列（为什么需要？）
* Indexer 索引: 本地缓存对象的索引
* Local Store 本地缓存: informer 的 cache, LocalStore 只会被 Lister 的 List/Get 方法访问。 并且非全部数据，有部分数据还在DeltaFIFO 中。
* WorkQueue 事件队列：DeltaIFIFO 收到数据后会先将数据存储在自己的数据结构中，然后直接操作 Store 中存储的数据，更新完 store 后 DeltaIFIFO 会将该事件 pop 到 WorkQueue 中，Controller 收到 WorkQueue 中的事件会根据对应的类型触发对应的回调函数。\(这块的先后顺序？\)

### 二、使用

1.作为 client 的使用示例

```text
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
        UpdateFunc: func(interface{}, interface{}) { fmt.Println("update not implemented") }, // 此处省略 workqueue 的使用
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
* 
