
---
title: Kubernetes 源码剖析 -- Client-go
date: 2020-08-10 15:00:00
tags: 术
---

```properties
Kubernetes Version: release-1.14
Reference: Kubernetes 源码剖析
LastUpdate: 2020.08.16
```

## 源码结构

| 目录名     | 用途                                                         |
| ---------- | ------------------------------------------------------------ |
| discovery  | discovery client，对 rest 客户端的进一步封装，用于发现 apiserver 所支持的能力和信息 |
| dynamic    | dynamic client ，对 rest 客户端的进一步封装，动态客户端，面向处理 CRD |
| examples   | 例子，比如对 deployment 创建、修改，如何选主，workqueue 如何使用等等 |
| informers  | 这就是 client-go 中非常有名的 informer 机制的核心代码        |
| kubernetes | clientset 的代码，对 rest 客户端的封装，提供复杂的内置资源访问和管理能力 |
| listers    | 为每个 k8s 资源提供 lister 功能，提供了只读缓存功能          |
| plugin     | 提供云服务商授权插件                                         |
| pkg        | 主要是一些功能函数，比如版本函数                             |
| rest       | 这是最基础的 client，其它的 client 都是基于此派生的          |
| scale      | scale client 的代码，对 rest 客户端的进一步封装，用于扩容和缩容 |
| tools      | 工具函数库，主要是和 k8s 相关的工具函数；提供 Client 查询和缓存 |
| util       | 通用的一些工具函数，比如 WorkQueue 工作队列，Certificate 证书管理 |
| transport  | 提供安全 tcp 链接                                            |

## Client 客户端对象

首先 Rest 是最基础的客户端，RESTCLient 对 HTTP Request 进行了封装，实现 RESTful 的API 风格。ClientSet，DynamicClient 以及 DiscoveryClient 客户端都是基于 RESTClient 实现的。

* ClientSet 在 RESTClient 基础上封装了 Resource 和 Version 的管理方法，一个 Resource 可以理解为一个客户端，ClientSet 是多个客户端的集合，ClientSet 只能处理 K8s 内置资源；
* DynamicClient 可以处理 K8s 所有资源对象，包括内置资源与 CRD 自定义资源；
* DiscoveryClient 用于发现 kube-apiserver 所支持的资源组，资源版本和资源信息（Group，Version 以及 Resources）。

要使用以上四种客户端，**需要先通过 kubeconfig 连接到指定的 Kubernetes API Server**（确定权限粒度）。

看下面一个例子：

```go
// Code from： https://jeremy-boo.github.io/post/client-go/
package main
import (
    "flag"
    "fmt"
    "os"
    "path/filepath"
    "time"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/clientcmd"
)
func main() {
  var kubeconfig *string
  if home := homeDir(); home != "" {
        kubeconfig = flag.String("kubeconfig", filepath.Join(home, ".kube", "config"), "(optional) absolute path to the kubeconfig file")
    } else {
        kubeconfig = flag.String("kubeconfig", "", "absolute path to the kubeconfig file")
    }
  flag.Parse()
  // uses the current context in kubeconfig
  config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
  if err != nil {
      panic(err.Error())
  }
  // creates the clientset
  clientset, err := kubernetes.NewForConfig(config)
  if err != nil {
      panic(err.Error())
  }
  // get pods list
  for {
      pods, err := clientset.CoreV1().Pods("").List(metav1.ListOptions{})
      if err != nil {
          panic(err.Error())
      }
      fmt.Printf("There are %d pods in the cluster\n", len(pods.Items))
      time.Sleep(10 * time.Second)
  }
}
func homeDir() string {
  if h := os.Getenv("HOME"); h != "" {
    return h
  }
  return os.Getenv("USERPROFILE") // windows
}
```

### Kubeconfig

以上面这个例子为例，读取 Kubeconfig 配置文件的关键代码在于 `clientcmd.BuildConfigFromFlags`，它会读取配置信息并且实例化一个 res.Config 对象，由于 kubeconfig 会有多个 kube-apiserver 集群的配置信息，因此它需要能够合并多个配置信息，该过程由 Load 函数完成。分为**加载 kubeconfig 文件**和**合并 kubeconfig 配置**信息两步：

1. 加载 kubeconfig 文件；

`vendor/k8s.io/client-go/tools/clientcmd/loader.go`

```go
func (rules *ClientConfigLoadingRules) Load() (*clientcmdapi.Config, error) {
	// ...

	kubeConfigFiles := []string{}

	// Make sure a file we were explicitly told to use exists
  // ExplicitPath 为显式地址，用于合并
	if len(rules.ExplicitPath) > 0 {
		if _, err := os.Stat(rules.ExplicitPath); os.IsNotExist(err) {
			return nil, err
		}
		kubeConfigFiles = append(kubeConfigFiles, rules.ExplicitPath)

	} else {
    // Precedence 是个 []string
		kubeConfigFiles = append(kubeConfigFiles, rules.Precedence...)
	}

	kubeconfigs := []*clientcmdapi.Config{}
	// read and cache the config files so that we only look at them once
	for _, filename := range kubeConfigFiles {
		// ...

    // LoadFromFile takes a filename and deserializes the contents into Config object
    // 把变量从内存中取出变成可存储的传输对象成为序列化，反之从可存储的传输对象变成内存对象为反序列化；
    // 这里就是读取文件，转换成一个 config 对象
		config, err := LoadFromFile(filename)
		// ...

		kubeconfigs = append(kubeconfigs, config)
	}

	// ... merge 部分代码
	return config, utilerrors.NewAggregate(errlist)
}
```

2. 合并 kubeconfig 配置

这部分的合并有一个优先级的关系，上面 Load 函数的 merge 部分是：

```go
  // first merge all of our maps
	mapConfig := clientcmdapi.NewConfig()

	for _, kubeconfig := range kubeconfigs {
		mergo.MergeWithOverwrite(mapConfig, kubeconfig)
	}

	// merge all of the struct values in the reverse order so that priority is given correctly
	// errors are not added to the list the second time
	nonMapConfig := clientcmdapi.NewConfig()
	for i := len(kubeconfigs) - 1; i >= 0; i-- {
		kubeconfig := kubeconfigs[i]
		mergo.MergeWithOverwrite(nonMapConfig, kubeconfig)
	}

	// since values are overwritten, but maps values are not, we can merge the non-map config on top of the map config and
	// get the values we expect.
	config := clientcmdapi.NewConfig()
	mergo.MergeWithOverwrite(config, mapConfig)
	mergo.MergeWithOverwrite(config, nonMapConfig)
```

[MergeWithOverWrite](https://pkg.go.dev/github.com/imdario/mergo?tab=doc#MergeWithOverwrite) 是来自 [mergo](https://pkg.go.dev/github.com/imdario/mergo?tab=doc) 包的一个函数，第一个参数为 src，第二个参数为 dst，merge 后的结构例子如下：

```
src:      T{X: "two", Z:{A: "three", B: 4}}
dst:      T{X: "one", Y: 5, Z:{A: "four", B: 6}}
result:   T{X: "two", Y: 5, Z:{A: "three", B: 4}}
```

### RESTClient

一个使用 RESTClient 的例子见书本 134页。使用的关键代码为：

```go
// result := &corev1.PodList{}
restClient.Get().Namespaces("default").Resources("pods").VersionParams(&metav1.ListOptions{Limit: 500}, scheme.ParamterCodec).Do().Into(result)
```

其中，restClient 为通过 kubeconfig 文件生成的 Client 对象，Get 方法封装了 HTTP get 请求，VersionParams 将一些查询选项添加到请求参数中，通过 Do 函数执行请求，并将 kube-apiserver 返回的结果（结果是一个 Result 对象）解析到 corev1.PodList 对象中。

看下面这个 Do 方法：

`vendor/k8s.io/client-go/rest/request.go`

```go
// Do formats and executes the request. Returns a Result object for easy response
// processing.
//
// Error type:
//  * If the request can't be constructed, or an error happened earlier while building its
//    arguments: *RequestConstructionError
//  * If the server responds with a status: *errors.StatusError or *errors.UnexpectedObjectError
//  * http.Client.Do errors are returned directly.
func (r *Request) Do() Result {
	r.tryThrottle()

	var result Result
	err := r.request(func(req *http.Request, resp *http.Response) {
    // transformResponse 把一个 API response 转化为一个 structured API object
		result = r.transformResponse(resp, req)
	})
	if err != nil {
		return Result{err: err}
	}
	return result
}
```

其中，request 方法的实现可以参见 [学习 client-go 中 request 函数的实现](https://jinrunsen.com/learn-clien-go-request/)，写得非常详细了。

### ClientSet

对于 RESTClient 而言，必须指定 Resource 和 Version，ClientSet 在 RESTClient 的基础上封装了 Resource 和 Version，通过函数的方法直接调用，比如：

```go
clientset, err := kubernetes.NewForConfig(config)
// err handling
podClient := clientset.CoreV1().Pods(apiv1.NamespaceDefault)
list, err := podClient.List(metav1.ListOptions{limit: 500})
```

相比于 RESTClient 的多次调用，`CoreV1().Pods(apiv1.NamespaceDefault)`  表示对 core 核心资源组的 v1 资源版本下的 Pod 资源对象。而关于 Pod 的操作也是一层封装，比如 List 操作：

`vendor/k8s.io/client-go/kubernetes/typed/core/v1/pod.go`

```go
func (c *pods) List(opts metav1.ListOptions) (result *v1.PodList, err error) {
	var timeout time.Duration
	if opts.TimeoutSeconds != nil {
		timeout = time.Duration(*opts.TimeoutSeconds) * time.Second
	}
	result = &v1.PodList{}
  // 这一步就是调用原来的 RestClient
	err = c.client.Get().
		Namespace(c.ns).
		Resource("pods").
		VersionedParams(&opts, scheme.ParameterCodec).
		Timeout(timeout).
		Do().
		Into(result)
	return
}
```

### DynamicClient

ClientSet 只能访问 k8s 自带的资源，而 DynamicClient 可以访问 CRD 自定义资源，这因为 ClientSet 预先实现每种 Resource 和 Version，内部的数据都是结构化的。而 DynamicClient 内部实现了 Unstructured，用于处理非结构化数据。

假设获取 List Pod 的操作（PodList），DynamicClient 的处理过程将所有 Resource 转化为 Unstructured 结构类型，进行相应的处理，处理完成后再将 Unstructured 的结构转换成 PodList，整个过程类似于 Go 语言的 interface{} 断言转化过程。而 Unstructured 结构类型是通过 map[string]interface{} 转换的。

⚠️  DynamicClient 不是类型安全，访问 CRD 自定义资源时，操作指针不当可能导致程序崩溃。

如书本第 140 页的例子，关键流程为：

```go
dynamicClient, err := dynamic.NewForConfig(config)
// 资源组，资源版本，资源名称
gvr := schema.GroupVersionResource{Version: "v1", Resource: "pods"}
// 得到的 Pod 列表是一个 unstructured.UnstructuredList 的指针类型
unstructObj, err := dynamicClient.Resource(gvr).NameSpace(apiV1.NamespaceDefault).List(metav1.ListOptions{limit: 500})
podList := &corev1.PodList{}
// 转换成 PodList 类型
err = runtime.DefaultUnstructuredConverter.FromUnstructured(unstructObj.UnstructuredContent(), podList)
// err handling
```

### DiscoveryClient

DiscoveryClient 用于查看 Kubernetes API Server 所支持的资源组，资源版本和资源信息。

它同样是在 ClientSet 上的一层封装，实际上，kubectl 的api-versions 和 api-resources 的命令就是通过 DiscoveryClient 实现的。

DiscoveryClient 除了可以发现所支持的资源组，版本和信息，还可以将这些内容缓存到本地，从而缓解 Kubenertes API Server 访问的压力，默认存储在 `~/.kube/cache` 和  `~/.kube/http-cache` 下。

如书本第 142 页的例子，关键流程为：

```go
discoveryClient, err := discovery.NewDiscoveryForConfig(config)
_, APIResourceList, err := discoveryClient.ServerGroupsAndResources()
for _, list := range APIResourceList {
  gv, err := schema.ParseGroupVersion(list.GroupVersion)
  for _, resource := range list.APIResources {
    fmt.Printf("name: %v, group: %v, version: %v\n", resource.Name, gv.Group, gv.Version)
  }
}
```

1. 获取 Kubernetes API Server 支持的资源组，资源版本，资源信息。

APIServer 对外暴露的是 /api 和 /apis 两个接口；DiscoveryClient 通过访问这两个接口还获取信息，核心实现位于 ServerGroupsAndResources 中的 ServerGroups 方法中：

`vendor/k8s.io/client-go/discovery/discovery_client.go`

```go
func (d *DiscoveryClient) ServerGroups() (apiGroupList *metav1.APIGroupList, err error) {
	// Get the groupVersions exposed at /api
	v := &metav1.APIVersions{}
	err = d.restClient.Get().AbsPath(d.LegacyPrefix).Do().Into(v)
	// error handling

	// Get the groupVersions exposed at /apis
	apiGroupList = &metav1.APIGroupList{}
	err = d.restClient.Get().AbsPath("/apis").Do().Into(apiGroupList)
	if err != nil && !errors.IsNotFound(err) && !errors.IsForbidden(err) {
		return nil, err
	}
	// error handling

	// prepend the group retrieved from /api to the list if not empty
	if len(v.Versions) != 0 {
		apiGroupList.Groups = append([]metav1.APIGroup{apiGroup}, apiGroupList.Groups...)
	}
	return apiGroupList, nil
}
```

通过 RESTClient 请求 /api 接口，结果存放到 APIVersions 结构体，再次通过 RESTClient 请求 /apis 接口，请求结果存放到 APIGroupList 中。最后通过 /api 检索到的资源组信息合并到 apiGroupList 中返回。

**其实整个这些 API 的设计可以看出来，kubernetes 都是先声明一个对象，再将请求得到的数据通过 Into 方法装载进去，从而保证数据的“干净”性，这种设计理念贯穿了 RESTClient**。

2. DiscoveryClient 的本地缓存

缓存机制本身很简单，首次获取时，先查询本地缓存，不存在（没有命中）则请求 Kubernetes API Server 接口（回源）；Cache 把得到的响应数据存储本地一份并返回给 DiscoveryClient。下一次再次获取的时候，直接从本地缓存返回（命中）给 Client。默认10分钟与 APIServer 同步一次缓存，因为资源组，资源版本，资源信息几乎不变。

## Informer 机制

Kubernets 中使用 http 进行通信，**如何不依赖中间件的情况下保证消息的实时性，可靠性和顺序性等呢**？答案就是利用了 Informer 机制。

### Informer 机制架构设计

![](001.jpg)

图片源自 [Client-go under the hood](https://github.com/kubernetes/sample-controller/blob/master/docs/controller-client-go.md)

⚠️ 这张图分为两部分,黄色图标是开发者需要自行开发的部分，而其它的部分是client-go已经提供的，直接使用即可。

1. **Reflector**：用于 Watch 指定的 Kubernetes 资源，当 watch 的资源发生变化时，触发变更的事件，比如 Added，Updated 和 Deleted 事件，并将资源对象存放到本地缓存 DeltaFIFO；
2. **DeltaFIFO**：拆开理解，FIFO 就是一个队列，拥有队列基本方法（ADD，UPDATE，DELETE，LIST，POP，CLOSE 等），Delta 是一个资源对象存储，保存存储对象的消费类型，比如 Added，Updated，Deleted，Sync 等；
3. **Indexer**：Client-go 用来存储资源对象并自带索引功能的本地存储，Reflector 从 DeltaFIFO 中将消费出来的资源对象存储到 Indexer，Indexer 与 Etcd 集群中的数据完全保持一致（❓ **需要关注这一步是如何达成的，以及性能上如何优化**）。从而 client-go 可以本地读取，减少 Kubernetes API 和 Etcd 集群的压力。

看书本 p146 的一个例子，关键流程如下：

```go
clientset, err := kubernetes.NewForConfig(config)
stopCh := make(chan struct{})
defer close(stopch)
sharedInformers := informers.NewSharedInformerFactory(clientset, time.Minute)
informer := sharedInformer.Core().V1().Pods().Informer()
informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
  AddFunc: func(obj interface{} {
    // ...
  },
  UpdateFunc: func(obj interface{} {
    // ...
  },
  DeleteFunc  : func(obj interface{} {
    // ...
  })
  informer.Run(stopCh)
})
```

* Informer 需要通过高 ClientSet 与 Kubernetes API Server 交互；
* 创建 stopCh 是用于在程序进程退出前通知 Informer 提前退出，Informer 是一个持久运行的 goroutine；
* NewSharedInformerFactory 实例化了一个 SharedInformer 对象，用于进行本地资源存储；
* sharedInformer.Core().V1().Pods().Informer() 得到了具体 Pod 资源的 informer 对象；
* AddEventHandler 即图中的第6步，这是一个资源事件回调方法，上例中即为当创建/更新/删除 Pod 时触发事件回调方法；
* 一般而言，其他组件使用 Informer 机制触发资源回调方法会将资源对象推送到 WorkQueue 或其他队列中，具体推送的位置要去回调方法里自行实现。

上面这个示例，当触发了 Add，Update 或者 Delete 事件，就通知 Client-go，告知 Kubernetes 资源事件发生变更并且需要进行相应的处理。

1. 资源 Informer

每一个 k8s Resource 都实现了 Informer 机制，均有 Informer 和 Lister 方法，以 PodInformer 为例：

```go
type PodInformer interface {
	Informer() cache.SharedIndexInformer
	Lister() v1.PodLister
}
```

2. Shared Informer 共享机制

Informer 又称为 Shared Informer，表明是可以共享使用的，在使用 client-go 写代码时，若同一资源的 Informer 被实例化太多次，每个 Informer 使用一个 Reflector，会运行过多的相同 ListAndWatch（即图中的第一步），太多重复的序列化和反序列化会导致 k8s API Server 负载过重。

而 Shared Informer 通过对同一类资源 Informer 共享一个 Reflector 可以节约很多资源，这通过 map 数据结构即可实现这样一个共享 Informer 机制。

`vendor/k8s.io/client-go/informers/factory.go`

```go
type sharedInformerFactory struct {
  // ...
	informers map[reflect.Type]cache.SharedIndexInformer
	// ...
}
// ...
// InternalInformerFor returns the SharedIndexInformer for obj using an internal client.
// 当示例中调用 xxx.Informer() 时，内部调用了该方法
func (f *sharedInformerFactory) InformerFor(obj runtime.Object, newFunc internalinterfaces.NewInformerFunc) cache.SharedIndexInformer {
	f.lock.Lock()
	defer f.lock.Unlock()

	informerType := reflect.TypeOf(obj)
	informer, exists := f.informers[informerType]
	if exists {
		return informer
	}

	resyncPeriod, exists := f.customResync[informerType]
	if !exists {
		resyncPeriod = f.defaultResync
	}

	informer = newFunc(f.client, resyncPeriod)
	f.informers[informerType] = informer

	return informer
}
```

### Reflector

Reflector 用于 Watch 指定的 Kubernetes 资源，当 watch 的资源发生变化时，触发变更的事件，并将资源对象存放到本地缓存 DeltaFIFO。

通过 NewReflector 实例化 Reflector 对象，实例化过程必须传入 ListerWatcher 数据接口对象，它拥有 List 和 Watch 方法。Reflector 对象通过 Run 行数启动监控并处理监控事件，在实现中，其核心为 ListAndWatch 函数。

1. **获取资源数据列表**

以 Example 的代码为例，我们获取了所有 Pod 的资源数据，List 流程如下：

```
1. r.ListWatcher.List 获取资源数据
2. listMetaInterface.GetResourceVersion 获取资源版本号
3. meta.ExtractList 将资源数据转换为资源对象列表
4. r.SyncWith 将资源对象列表的资源对象和资源版本号存储到 DeltaFIFO 中
5. r.setLastSyncResourceVersion 设置最新的资源版本号
```

具体的，

* r.ListWatcher.List 根据 ResourceVersion 获取资源下的所有对象数据，List 具有类似于 “断点传输“ 的功能，当传输工程中遇到网络故障中断，下次连接时，会根据版本号继续传输未完成部分，是本地缓存与 etcd 集群中保持一致。以 Example 为例，该方法调用的就是 Pod Informer 下的 ListFunc 函数，通过 ClientSet 客户端与 Kubernetes API Server 交互并获取 Pod 资源列表数据；
* listMetaInterface.GetResourceVersion 获取 ResourceVersion，即资源版本号，注意这里的资源版本号并不是指前面各个客户端的不同 kind 的不同 Version，所有资源都拥有 ResourceVersion，标识当前资源对象的版本号。每次修改 etcd 集群中存储的对象时，Kubernetes API Server 都会更改 ResourceVersion，使得 client-go 执行 watch 时可以根据 ResourceVersion 判断当前资源对象是否发生变化；
* meta.ExtractList 将 runtime.Object 对象转换为 []runtime.Object 对象。因为 r.ListWatcher.List 获取的是资源下所有对象的数据，因此应当是一个列表；
* r.SyncWith 将结果同步到 DeltaFIFO 中；
* r.setLastSyncResourceVersion 设置最新的资源版本号

2. **监控资源对象**

Watch 通过 HTTP 协议与 Kubernets API Server 建立长连接，接收 Kubernets API Server 发来的资源变更事件。Watch 操作的实现机制使用 HTTP 协议的分块传输编码（Chunked Transfer Encoding）（❓ **如何实现**？）。——当 client-go 调用 Kubernets API Server 时，Kubernets API Server 在 Response 的 HTTP Header 中设置 Transfer-Encoding 的值为 chunked，表示采用分块传输编码，客户端收到消息后，与服务端进行连接，并等待下一个数据块。

在源码中关键为 watch 和 watchHandler 函数：

`staging/src/k8s.io/client-go/tools/cache/reflector.go`

```go
func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
	// list part
  
  // watch part
	for {
		// give the stopCh a chance to stop the loop, even in case of continue statements further down on errors
		select {
		case <-stopCh:
			return nil
		default:
		}

		// ...
		options = metav1.ListOptions{
			ResourceVersion: resourceVersion,
			// We want to avoid situations of hanging watchers. Stop any wachers that do not
			// receive any events within the timeout window.
			TimeoutSeconds: &timeoutSeconds,
		}

		w, err := r.listerWatcher.Watch(options)
		// error handling
    
		if err := r.watchHandler(w, &resourceVersion, resyncerrc, stopCh); err != nil {
			if err != errorStopRequested {
				klog.Warningf("%s: watch of %v ended with: %v", r.name, r.expectedType, err)
			}
			return nil
		}
	}
}
```

以之前的 Example 为例子，r.listerWatcher.Watch 实际调用了 Pod Informer 下的 Watch 函数，通过 ClientSet 客户端与 Kubernetes API Server 建立长链接，监控指定资源的变更事件，如下：

`staging/src/k8s.io/client-go/informers/core/v1/pod.go`

```go
func NewFilteredPodInformer(...) cache.SharedIndexInformer {
	return cache.NewSharedIndexInformer(
		  // ...
			WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
				if tweakListOptions != nil {
					tweakListOptions(&options)
				}
				return client.CoreV1().Pods(namespace).Watch(options)
			},
		},
		// ...
	)
}
```

r.watchHandler 处理资源的变更事件，将对应资源更新到本地缓存 DeltaFIFO 并更新 ResourceVersion 资源版本号。

```go
// watchHandler watches w and keeps *resourceVersion up to date.
func (r *Reflector) watchHandler(w watch.Interface, resourceVersion *string, errc chan error, stopCh <-chan struct{}) error {
	start := r.clock.Now()
	eventCount := 0
	defer w.Stop()

// 循环嵌套循环时，可以在break 后指定标签。用标签决定哪个循环被终止
loop:
	for {
		select {
		case <-stopCh:
			return errorStopRequested
		case err := <-errc:
			return err
		case event, ok := <-w.ResultChan():
      if !ok {
        // 直接跳出 for 循环
				break loop
			}
			// ...
			newResourceVersion := meta.GetResourceVersion()
			switch event.Type {
			case watch.Added:
				err := r.store.Add(event.Object)
				// ...
			case watch.Modified:
				err := r.store.Update(event.Object)
				// ...
			case watch.Deleted:
				err := r.store.Delete(event.Object)
				// ...
			default:
				// ...
			}
			*resourceVersion = newResourceVersion
			r.setLastSyncResourceVersion(newResourceVersion)
			eventCount++
		}
	}
// ...
}
```

### DeltaFIFO

DeltaFIFO 拆开理解，FIFO 就是一个队列，拥有队列基本方法（ADD，UPDATE，DELETE，LIST，POP，CLOSE 等），Delta 是一个资源对象存储，保存存储对象的消费类型，比如 Added，Updated，Deleted，Sync 等。

看 DeltaFIFO 的数据结构：

`vendor/k8s.io/client-go/tools/cache/delta_fifo.go`

```go
type DeltaFIFO struct {
	// lock/cond protects access to 'items' and 'queue'.
	lock sync.RWMutex
	cond sync.Cond

	// We depend on the property that items in the set are in
	// the queue and vice versa, and that all Deltas in this
	// map have at least one Delta.
	items map[string]Deltas
	queue []string

	// ...
}
```

其中，Deltas 部分的数据结构如下：

`staging/src/k8s.io/client-go/tools/cache/delta_fifo.go`

```go
// DeltaType is the type of a change (addition, deletion, etc)
type DeltaType string

// Delta is the type stored by a DeltaFIFO. It tells you what change
// happened, and the object's state after* that change.
//
// [*] Unless the change is a deletion, and then you'll get the final
//     state of the object before it was deleted.
type Delta struct {
	Type   DeltaType
	Object interface{}
}

// Deltas is a list of one or more 'Delta's to an individual object.
// The oldest delta is at index 0, the newest delta is the last one.
type Deltas []Delta
```

DeltaFIFO 与其他队列最大的不同之处在于：它会保留所有关于资源对象（obj）的操作类型，队列中会存在拥有不同操作类型的同一资源对象，**使得消费者在处理该资源对象时能够了解资源对象所发生的事情**。queue 字段存储资源对象的 key，这个 key 通过 KeyOf 函数计算得到，items 字段通过 map 数据结构的方式存储，value 存储的是对象 Deltas 数组，结构图如下：

![](004.png)

作为一个 FIFO 的队列，有数据的生产者和消费者，其中生产者是 Reflector 调用的 Add 方法，消费者是 Controller 调用的 Pop 方法。三个核心方法为生产者方法，消费者方法和 Resync 机制。

#### 生产者方法

DeltaFIFO 队列中的资源对象在调用 Added，Updated，Deleted 等事件时都调用了 queueActionLocked 函数：

![](002.png)

它是 DeltaFIFO 实现的关键：

`vendor/k8s.io/client-go/tools/cache/delta_fifo.go`

```go
// queueActionLocked appends to the delta list for the object.
// Caller must lock first.
func (f *DeltaFIFO) queueActionLocked(actionType DeltaType, obj interface{}) error {
	id, err := f.KeyOf(obj)
	if err != nil {
		return KeyError{obj, err}
	}

	newDeltas := append(f.items[id], Delta{actionType, obj})
	newDeltas = dedupDeltas(newDeltas)

	if len(newDeltas) > 0 {
		if _, exists := f.items[id]; !exists {
			f.queue = append(f.queue, id)
		}
		f.items[id] = newDeltas
		f.cond.Broadcast()
	} else {
		// We need to remove this from our map (extra items in the queue are
		// ignored if they are not in the map).
		delete(f.items, id)
	}
	return nil
}
```

1. 通过 KeyOf 函数计算出对象的 key；
2. 将 actionType 以及对应的 id 添加到 items 中，并通过 dedupDeltas 对数组中最新的两次添加进行去重；
3. 更新构造后的 Deleta 并通过 cond.Broadcast() 广播所有消费者解除阻塞。

#### 消费者方法

Pop 函数作为消费者使用方法，从 DeltaFIFO 的头部取出最早进入队列中的资源对象数据。Pop 方法必须传入 process 函数，用于接收并处理对象的回调方法，如下：

`vendor/k8s.io/client-go/tools/cache/delta_fifo.go`

```go
func (f *DeltaFIFO) Pop(process PopProcessFunc) (interface{}, error) {
	f.lock.Lock()
	defer f.lock.Unlock()
	for {
		for len(f.queue) == 0 {
			// When the queue is empty, invocation of Pop() is blocked until new item is enqueued.
			// When Close() is called, the f.closed is set and the condition is broadcasted.
			// Which causes this loop to continue and return from the Pop().
			if f.IsClosed() {
				return nil, FIFOClosedError
			}

			f.cond.Wait()
		}
		id := f.queue[0]
		f.queue = f.queue[1:]
		if f.initialPopulationCount > 0 {
			f.initialPopulationCount--
		}
		item, ok := f.items[id]
		if !ok {
			// Item may have been deleted subsequently.
			continue
		}
    // ⚠️ 这里在执行之前会把该 obj 旧的 delta 数据清空
		delete(f.items, id)
		err := process(item)
		if e, ok := err.(ErrRequeue); ok {
			f.addIfNotPresent(id, item)
			err = e.Err
		}
		// Don't need to copyDeltas here, because we're transferring
		// ownership to the caller.
		return item, err
	}
}
```

首先，使用 f.lock.Lock() 确保了数据的同步，当队列不为空时，取出 f.queue 的头部数据，将对象传入 process 回调函数，由上层消费者进行处理，如果 process 回调方法处理出错，将该对象重新存入队列。

Controller 的 processLoop 方法负责从 DeltaFIFO 队列中取出数据传递给 process 回调函数，process 函数的类型如下：

```go
type PopProcessFunc func(interface{}) error
```

一个 process 回调函数代码示例如下：

`vendor/k8s.io/client-go/tools/cache/shared_informer.go`

```go
func (s *sharedIndexInformer) HandleDeltas(obj interface{}) error {
	s.blockDeltas.Lock()
	defer s.blockDeltas.Unlock()

	// from oldest to newest
	for _, d := range obj.(Deltas) {
		switch d.Type {
		case Sync, Added, Updated:
			isSync := d.Type == Sync
			s.cacheMutationDetector.AddObject(d.Object)
			if old, exists, err := s.indexer.Get(d.Object); err == nil && exists {
				if err := s.indexer.Update(d.Object); err != nil {
					return err
				}
				s.processor.distribute(updateNotification{oldObj: old, newObj: d.Object}, isSync)
			} else {
				if err := s.indexer.Add(d.Object); err != nil {
					return err
				}
				s.processor.distribute(addNotification{newObj: d.Object}, isSync)
			}
		case Deleted:
			if err := s.indexer.Delete(d.Object); err != nil {
				return err
			}
			s.processor.distribute(deleteNotification{oldObj: d.Object}, false)
		}
	}
	return nil
}
```

在这个例子中，HandleDeltas 作为 process 的一个回调函数，当资源对象操作类型为 Added，Updated 和 Delted 时，该资源对象存储至 Indexer（它是并发安全的），并通过 distribute 函数将资源对象分发到 SharedInformer，在之前 [Informer 机制架构设计](#informer-机制架构设计)的示例代码中，通过 `informer.AddEventHandler` 函数添加了对资源事件进行处理的函数，distribute 函数将资源对象分发到该事件处理函数。

>  一个当时学到这里遇到的问题：[【提问】DeltaFIFO 消费者方法中 process 回调函数的实现](https://github.com/cloudnativeto/sig-k8s-source-code/issues/29)

#### Resync 机制

> 本节内容基本搬运自 [【提问】Informer 中为什么需要引入 Resync 机制？](https://github.com/cloudnativeto/sig-k8s-source-code/issues/11)

Resync 机制会将 Indexer 本地存储中的资源对象同步到 DeltaFIFO 中，并将这些资源对象设置为 Sync 的操作类型，

```go
// k8s.io/client-go/tools/cache/delta_fifo.go
// 重新同步一次 Indexer 缓存数据到 Delta FIFO 队列中
func (f *DeltaFIFO) Resync() error {
	f.lock.Lock()
	defer f.lock.Unlock()

	if f.knownObjects == nil {
		return nil
	}
	// 遍历 indexer 中的 key，传入 syncKeyLocked 中处理
	keys := f.knownObjects.ListKeys()
	for _, k := range keys {
		if err := f.syncKeyLocked(k); err != nil {
			return err
		}
	}
	return nil
}

func (f *DeltaFIFO) syncKeyLocked(key string) error {
	obj, exists, err := f.knownObjects.GetByKey(key)
	if err != nil {
		klog.Errorf("Unexpected error %v during lookup of key %v, unable to queue object for sync", err, key)
		return nil
	} else if !exists {
		klog.Infof("Key %v does not exist in known objects store, unable to queue object for sync", key)
		return nil
	}
	// 如果发现 FIFO 队列中已经有相同 key 的 event 进来了，说明该资源对象有了新的 event，
	// 在 Indexer 中旧的缓存应该失效，因此不做 Resync 处理直接返回 nil
	id, err := f.KeyOf(obj)
	if err != nil {
		return KeyError{obj, err}
	}
	if len(f.items[id]) > 0 {
		return nil
	}
    // 重新放入 FIFO 队列中
	if err := f.queueActionLocked(Sync, obj); err != nil {
		return fmt.Errorf("couldn't queue object: %v", err)
	}
	return nil
}
```

为什么需要 Resync 机制呢？因为在处理 SharedInformer 事件回调时，可能存在处理失败的情况，定时的 Resync 让这些处理失败的事件有了重新 onUpdate 处理的机会。

那么经过 Resync 重新放入 Delta FIFO 队列的事件，和直接从 apiserver 中 watch 得到的事件处理起来有什么不一样呢？还是看 HandleDeltas：

```go
// k8s.io/client-go/tools/cache/shared_informer.go
func (s *sharedIndexInformer) HandleDeltas(obj interface{}) error {
	s.blockDeltas.Lock()
	defer s.blockDeltas.Unlock()

	// from oldest to newest
	for _, d := range obj.(Deltas) {
		// 判断事件类型，看事件是通过新增、更新、替换、删除还是 Resync 重新同步产生的
		switch d.Type {
		case Sync, Replaced, Added, Updated:
			s.cacheMutationDetector.AddObject(d.Object)
			if old, exists, err := s.indexer.Get(d.Object); err == nil && exists {
				if err := s.indexer.Update(d.Object); err != nil {
					return err
				}
				
				isSync := false
				switch {
				case d.Type == Sync:
					// 如果是通过 Resync 重新同步得到的事件则做个标记
					isSync = true
				case d.Type == Replaced:
					...
				}
				// 如果是通过 Resync 重新同步得到的事件，则触发 onUpdate 回调
				s.processor.distribute(updateNotification{oldObj: old, newObj: d.Object}, isSync)
			} else {
				if err := s.indexer.Add(d.Object); err != nil {
					return err
				}
				s.processor.distribute(addNotification{newObj: d.Object}, false)
			}
		case Deleted:
			if err := s.indexer.Delete(d.Object); err != nil {
				return err
			}
			s.processor.distribute(deleteNotification{oldObj: d.Object}, false)
		}
	}
	return nil
}
```

从上面对 Delta FIFO 的队列处理源码可看出，如果是从 Resync 重新同步到 Delta FIFO 队列的事件，会分发到 updateNotification 中触发 onUpdate 的回调。Resync 机制的引入，定时将 Indexer 缓存事件重新同步到 Delta FIFO 队列中，在处理 SharedInformer 事件回调时，让处理失败的事件得到重新处理。并且通过入队前判断 FIFO 队列中是否已经有了更新版本的 event，来决定是否丢弃 Indexer 缓存不进行 Resync 入队。在处理 Delta FIFO 队列中的 Resync 的事件数据时，触发 onUpdate 回调来让事件重新处理。

关于为什么使用 Resync，这个问题在《Programming Kubernetes》的第一章详细解释了一遍，我搬运一下。主要的目的是为了不丢数据，处理 `resync` 机制还有边缘触发与水平获取的设计，一起来保证不丢事件、数据同步并能及时响应事件。

> 在分布式系统中，有许多操作在并行执行，事件可能会以任意顺序异步到达。 如果我们的 controller 逻辑有问题，状态机出现一些错误或外部服务故障时，就很容易丢失事件，导致我们没有处理所有事件。 因此，我们必须更深入地研究如何应对这些问题。
>
> 在图中，您可以看到可用的不同方案：
>
> 1. 仅使用边缘驱动逻辑的示例，其中可能错过第二个的状态更改事件。
> 2. 边缘触发逻辑的示例，在处理事件时始终会获取最新状态（即水平）。换句话说，逻辑是边缘触发的（edge-triggered），但是水平驱动的（level-driven）。
> 3. 该示例的逻辑是边缘触发，水平驱动的，但同时还附加了定时同步的能力。
>
> ![](003.png)
>
> 考虑到仅使用单一的边缘驱动触发会产生的问题，Kubernetes controller 通常采用第 3 种方案。

### Indexer

Client-go 用来存储资源对象并自带索引功能的本地存储，Reflector 从 DeltaFIFO 中将消费出来的资源对象存储到 Indexer，Indexer 与 Etcd 集群中的数据完全保持一致。从而 client-go 可以本地读取，减少 Kubernetes API 和 Etcd 集群的压力。

了解 Indexer 之前，先了解 ThreadSafeMap，ThreadSafeMap 是实现并发安全存储，就像 Go 1.9 后推出 `sync.Map` 一样。Kubernetes 开始编写的时候还没有 `sync.Map`。Indexer 在 ThreadSafeMap 的基础上进行了封装，继承了 ThreadSafeMap 的存储相关的增删改查相关操作方法，实现了 Indexer Func 等功能，例如 Index，IndexKeys，GetIndexers 等方法，这些方法为 ThreadSafeMap 提供了索引功能。如下图：

![](005.png)

#### ThreadSafeStore

ThreadSafeStore 是一个内存中存储，数据不会写入本地磁盘，增删改查都会加锁，保证数据一致性。结构如下：

`vendor/k8s.io/client-go/tools/cache/store.go`

```go
// threadSafeMap implements ThreadSafeStore
type threadSafeMap struct {
	lock  sync.RWMutex
	items map[string]interface{}

	// indexers maps a name to an IndexFunc
	indexers Indexers
	// indices maps a name to an Index
	indices Indices
}
```

items 字段存储资源对象数据，其中 items 的 key 通过 keyFunc 函数计算得到，计算默认使用 MetaNamespaceKeyFunc 函数，该函数根据资源对象计算出 `<namespace>/<name>` 格式的 key，value 用于存储资源对象。

而后面两个字段的定义类型如下：

`vendor/k8s.io/client-go/tools/cache/index.go`

```go
// Index maps the indexed value to a set of keys in the store that match on that value
type Index map[string]sets.String

// Indexers maps a name to a IndexFunc
type Indexers map[string]IndexFunc

// Indices maps a name to an Index
type Indices map[string]Index
```

#### Indexer 索引器

每次增删改 ThreadSafeStore 的数据时，都会通过 updateIndices 或 deleteFormIndices 函数变更 Indexer。Indexer 被设计为可以自定义索引函数，他有重要的四个数据结构，**Indexers**，**IndexFunc**，**Indices** 和 **Index**。

看书本 159 页这个例子。

首先定义了一个索引器函数（IndexFunc），UsersIndexFunc。该函数定义查询所有 Pod 资源下 Annotations 字段的 key 为 users 的 Pod：

```go
func UsersIndexFunc(obj interfaces{}) ([]string, error) {
  pod := obj.(*v1.Pod)
  usersString := pod.Annotations["users"]
  return strings.Split(userString, ","), nil
}
```

Main 函数中 `cache.NewIndexer` 实例化了一个 Indexer 对象：

```go
index := cache.NewIndexer(cache.MetaNamespaceKeyFunc, cache.Indexers{"byUser": UsersIndexFunc})
```

第一个参数计算资源对象的 key，默认就是 MetaNamespaceKeyFunc，第二个参数是一个 Indexers 对象，如上一节展示的定义那样，key 为索引器（IndexFunc）的名称，value 为索引器函数。

通过 index.Add 添加了三个 Pod，再通过 index.ByIndex 函数查询使用 byUser 索引器下匹配 ernie 字段的 Pod 列表：

```go
erniePods, err := index.ByIndex("byUser", "ernie")
```

回看这四个类型：

```go
// Indexers maps a name to a IndexFunc
type Indexers map[string]IndexFunc

// IndexFunc knows how to provide an indexed value for an object.
type IndexFunc func(obj interface{}) ([]string, error)

// Indices maps a name to an Index
type Indices map[string]Index

// Index maps the indexed value to a set of keys in the store that match on that value
type Index map[string]sets.String
```

* Indexers：存储索引器，key 为 索引器名称，value 为索引器实现函数；
* IndexFunc：索引器函数，定义为接收一个资源对象，返回检索结果列表；
* Indices：存储缓存器，key 为缓存器名称，value 为缓存数据；（❓书中表示这个缓存器命名和索引器命名相对应未看懂）
* Index：存储缓存数据，结构为 K/V。

#### Indexer 索引器核心实现

`vendor/k8s.io/client-go/tools/cache/thread_safe_store.go`

```go
// ByIndex returns a list of items that match an exact value on the index function
func (c *threadSafeMap) ByIndex(indexName, indexKey string) ([]interface{}, error) {
	c.lock.RLock()
	defer c.lock.RUnlock()

	indexFunc := c.indexers[indexName]
	if indexFunc == nil {
		return nil, fmt.Errorf("Index with name %s does not exist", indexName)
	}

	index := c.indices[indexName]

	set := index[indexKey]
	list := make([]interface{}, 0, set.Len())
	for _, key := range set.List() {
		list = append(list, c.items[key])
	}

	return list, nil
}
```

ByIndex 接收两个参数：indexName（索引器名字）以及 indexKey（需要检索的 key），首先从 c.indexers 查找制定的索引器函数，然后从 c.indices 查找返回的缓存器函数，最后根据需要索引的 indexKey 从缓存数据中查到并返回。

⚠️ K8s 将 map 结构类型的 key 作为 Set 数据结构，实现 Set 去重特性。