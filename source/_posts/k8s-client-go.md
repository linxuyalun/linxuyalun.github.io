
---
title: Kubernetes 源码剖析 -- Client-go
date: 2020-08-10 15:00:00
tags: 术
---

```properties
Kubernetes Version: release-1.14
Reference: Kubernetes 源码剖析
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