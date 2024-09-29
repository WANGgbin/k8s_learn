# api

## api 格式

api-server 中的 api 是以 group 的方式组织的。group 中的 api path 格式如下；
`/:prefix/:group_name/:version/namespace/:namespace_name/:kind/:name/:subresource`。<br>

我们以 pod 为例，比如其对应的 namespace 为 system, name 为 example_pod，其对应的 path 的各参数如下：

```json
{
  "prefix": "/api",
  "group_name": "",
  "version": "v1",
  "namespace_name": "system",
  "kind": "pod",
  "name": "example_pod"
}
```

因此，其各个 api 对应的 path 为 `/api/v1/namespace/system/pod/example_pod`。对于该 pod 的 subresource: binding，其对应的 path
为：`/api/v1/namespace/system/pod/example_pod/binding`。

## api 注册

如果要在 api-server 中注册一个对象的 apis，应该如何做呢？<br>

api-server 自定义了一套注册系统，每个 api 对象只需要实现特定的接口，然后将 api 对象关联到某个 api-group，这样在 install api-group 的时候
就可以根据 api 对象实现的接口，注册不同的方法。我们还是以 pod 为例，看看其 api-handler 注册的流程。<br>

### api-group

定义如下：
```go
type APIGroupInfo struct {
	// 定义了该 api-group 下有多少个版本
	PrioritizedVersions []schema.GroupVersion
	
	// 版本 -> resource -> storage 对象
	// storage 对象为每个 resource 各方法的具体实现
	VersionedResourcesStorageMap map[string]map[string]rest.Storage
    
    // 其他字段
}
```

我们看看 pod 所在的 core api group 的初始化流程：
```go
func (p *legacyProvider) NewRESTStorage(apiResourceConfigSource serverstorage.APIResourceConfigSource, restOptionsGetter generic.RESTOptionsGetter) (genericapiserver.APIGroupInfo, error) {
	apiGroupInfo, err := p.GenericConfig.NewRESTStorage(apiResourceConfigSource, restOptionsGetter)
    
    // 初始化各个 resource 对应的 storage 对象。这里仅以 node 和 pods 举例
	nodeStorage, err := nodestore.NewStorage(restOptionsGetter, p.Proxy.KubeletClientConfig, p.Proxy.Transport)
	podStorage, err := podstore.NewStorage(
		restOptionsGetter,
		nodeStorage.KubeletConnectionInfo,
		p.Proxy.Transport,
		podDisruptionClient,
	)


    
    // 目前 core api group 只有一个版本：v1
	storage := apiGroupInfo.VersionedResourcesStorageMap["v1"]
	if storage == nil {
		storage = map[string]rest.Storage{}
	}
    
    // 将各个 resource 对应的 storage 注册，
	// 注册 pods
	if resource := "pods"; apiResourceConfigSource.ResourceEnabled(corev1.SchemeGroupVersion.WithResource(resource)) {
		storage[resource] = podStorage.Pod
		// 后面这些都是 pods 的 subresource
		storage[resource+"/attach"] = podStorage.Attach
		storage[resource+"/status"] = podStorage.Status
		storage[resource+"/log"] = podStorage.Log
		storage[resource+"/exec"] = podStorage.Exec
		storage[resource+"/portforward"] = podStorage.PortForward
		storage[resource+"/proxy"] = podStorage.Proxy
		storage[resource+"/binding"] = podStorage.Binding
		if podStorage.Eviction != nil {
			storage[resource+"/eviction"] = podStorage.Eviction
		}
		storage[resource+"/ephemeralcontainers"] = podStorage.EphemeralContainers
	}
    
    // 注册 nodes resource
	if resource := "nodes"; apiResourceConfigSource.ResourceEnabled(corev1.SchemeGroupVersion.WithResource(resource)) {
		storage[resource] = nodeStorage.Node
		storage[resource+"/proxy"] = nodeStorage.Proxy
		storage[resource+"/status"] = nodeStorage.Status
	}
    
    // 注册其他 resource ...
	
	// 注册到 api group 中
    apiGroupInfo.VersionedResourcesStorageMap["v1"] = storage

	return apiGroupInfo, nil
}
```

接着我们看看 api group 的注册流程：

```go
// 此函数是 api-server 注册 handler 的入口函数，每个 restStorageProvider 对应一个 api-group
func (s *Server) InstallAPIs(restStorageProviders ...RESTStorageProvider) error {
	nonLegacy := []*genericapiserver.APIGroupInfo{}


	for _, restStorageBuilder := range restStorageProviders {
		groupName := restStorageBuilder.GroupName()
		apiGroupInfo, err := restStorageBuilder.NewRESTStorage(s.APIResourceConfigSource, s.RESTOptionsGetter)
        
        // 目前只有 core api group 对应的 group name 是空的。
		if len(groupName) == 0 {
			// the legacy group for core APIs is special that it is installed into /api via this special install method.
			// core api group 通过 InstallLegacyAPIGroup 注册 handler，跟 InstallAPIGroups 的唯一区别在于，
            // Legacy 对应的 path root 为：/api，其他为：/apis
			if err := s.GenericAPIServer.InstallLegacyAPIGroup(genericapiserver.DefaultLegacyAPIPrefix, &apiGroupInfo); err != nil {
				return fmt.Errorf("error in registering legacy API: %w", err)
			}
		} else {
			// everything else goes to /apis
			nonLegacy = append(nonLegacy, &apiGroupInfo)
		}
	}

	if err := s.GenericAPIServer.InstallAPIGroups(nonLegacy...); err != nil {
		return fmt.Errorf("error in registering group versions: %w", err)
	}
	return nil
}
```

最终都是通过 installAPIResources 完成注册的。我们看看该函数的实现：

```go
// installAPIResources 执行真正的 api install
func (s *GenericAPIServer) installAPIResources(apiPrefix string, apiGroupInfo *APIGroupInfo, typeConverter managedfields.TypeConverter) error {
	var resourceInfos []*storageversion.ResourceInfo
	// 遍历 api group 下的每个 version，执行注册操作
	for _, groupVersion := range apiGroupInfo.PrioritizedVersions {
        if len(apiGroupInfo.VersionedResourcesStorageMap[groupVersion.Version]) == 0 {
            klog.Warningf("Skipping API %v because it has no resources.", groupVersion)
            continue
        }

        apiGroupVersion, err := s.getAPIGroupVersion(apiGroupInfo, groupVersion, apiPrefix)

        // 执行安装
        discoveryAPIResources, r, err := apiGroupVersion.InstallREST(s.Handler.GoRestfulContainer)
    }
	
	return nil
}

func (a *APIInstaller) Install() ([]metav1.APIResource, []*storageversion.ResourceInfo, *restful.WebService, []error) {
    
    // 每个 path 就是 resource name
    paths := make([]string, len(a.group.Storage))
    var i int = 0
    for path := range a.group.Storage {
        paths[i] = path
        i++
    }
	
    for _, path := range paths {
        // 将 obj 实现的 storage 转化为 handler 并注册
        apiResource, resourceInfo, err := a.registerResourceHandlers(path, a.group.Storage[path], ws)
        if err != nil {
            errors = append(errors, fmt.Errorf("error in registering resource: %s, %v", path, err))
        }
    }
	
    return apiResources, resourceInfos, ws, errors
}

```
以上流程，便完成了一类 resource 的 api install。我们看看 registerResourceHandlers 的大概逻辑，大概思路是根据 resource 对应的
storage 实现的不同的接口，转化为不同的方法。比如实现 Creater 接口，就会注册 Create 方法。

```go
func (a *APIInstaller) registerResourceHandlers(path string, storage rest.Storage, ws *restful.WebService) (*metav1.APIResource, *storageversion.ResourceInfo, error) {
	// 比如：path 为 pods/binding，那么 resource = pods, subresource = binding
	// 比如：path 为 pods，那么 resource = pods
	resource, subresource, err := splitSubresource(path)

	group, version := a.group.GroupVersion.Group, a.group.GroupVersion.Version

    namespaceScoped = scoper.NamespaceScoped()

	// what verbs are supported by the storage, used to know what verbs we support per path
	// 这里就是最核心的逻辑，判断 storage 都实现了哪些接口，进而注册不同的方法
	creater, isCreater := storage.(rest.Creater)
	namedCreater, isNamedCreater := storage.(rest.NamedCreater)
	lister, isLister := storage.(rest.Lister)
	getter, isGetter := storage.(rest.Getter)
	getterWithOptions, isGetterWithOptions := storage.(rest.GetterWithOptions)
	gracefulDeleter, isGracefulDeleter := storage.(rest.GracefulDeleter)
	collectionDeleter, isCollectionDeleter := storage.(rest.CollectionDeleter)
	updater, isUpdater := storage.(rest.Updater)
	patcher, isPatcher := storage.(rest.Patcher)
	watcher, isWatcher := storage.(rest.Watcher)


	allowWatchList := isWatcher && isLister // watching on lists is allowed only for kinds that support both watch and list.
	nameParam := ws.PathParameter("name", "name of the "+kind).DataType("string")
	pathParam := ws.PathParameter("path", "path to the resource").DataType("string")

	// Get the list of actions for the given scope.
	switch {
        default:
		namespaceParamName := "namespaces"
		// 这一步在构造 api 的 path
		namespacedPath := namespaceParamName + "/{namespace}/" + resource
		resourcePath := namespacedPath
		resourceParams := namespaceParams
		// itemPath 为最终的 path
		itemPath := namespacedPath + "/{name}"
		itemPathSuffix := ""
		if isSubresource {
			itemPathSuffix = "/" + subresource
			// 如果是 subresource，末尾还需追加 "/subresource_name"，比如 "/binding"
			itemPath = itemPath + itemPathSuffix
			resourcePath = itemPath
		}
		
		apiResource.Name = path
		apiResource.Namespaced = true
		apiResource.Kind = resourceKind
		namer := handlers.ContextBasedNaming{
			Namer:         a.group.Namer,
			ClusterScoped: false,
		}
        
        // 根据接口的实现，注册不同的 action，后续会根据 action，注册 handler。
		actions = appendIf(actions, action{"LIST", resourcePath, resourceParams, namer, false}, isLister)
		actions = appendIf(actions, action{"POST", resourcePath, resourceParams, namer, false}, isCreater)
		actions = appendIf(actions, action{"PUT", itemPath, nameParams, namer, false}, isUpdater)
		actions = appendIf(actions, action{"PATCH", itemPath, nameParams, namer, false}, isPatcher)
		actions = appendIf(actions, action{"DELETE", itemPath, nameParams, namer, false}, isGracefulDeleter)
        // ...
	}
    
    // 根据 action 构造不同的 handler
	for _, action := range actions {
		switch action.Verb {
        // 这里我们仅以 post 举例，post 对应的 create 接口
		case "POST": // Create a resource.
			var handler restful.RouteFunction
			// restfulCreate* 方法底层会调用 creator.Create 方法。
			if isNamedCreater {
				handler = restfulCreateNamedResource(namedCreater, reqScope, admit)
			} else {
				handler = restfulCreateResource(creater, reqScope, admit)
			}
			// 给 handler 加了一些中间件
			handler = metrics.InstrumentRouteFunc(action.Verb, group, version, resource, subresource, requestScope, metrics.APIServerComponent, deprecated, removedRelease, handler)
			handler = utilwarning.AddWarningsHandler(handler, warnings)
			// 注册到 web service 中。
			route := ws.POST(action.Path).To(handler).
				Doc(doc).
				Param(ws.QueryParameter("pretty", "If 'true', then the output is pretty printed. Defaults to 'false' unless the user-agent indicates a browser or command-line HTTP tool (curl and wget).")).
				Operation("create"+namespaced+kind+strings.Title(subresource)+operationSuffix).
				Produces(append(storageMeta.ProducesMIMETypes(action.Verb), mediaTypes...)...).
				Returns(http.StatusOK, "OK", producedObject).
				// TODO: in some cases, the API may return a v1.Status instead of the versioned object
				// but currently go-restful can't handle multiple different objects being returned.
				Returns(http.StatusCreated, "Created", producedObject).
				Returns(http.StatusAccepted, "Accepted", producedObject).
				Reads(defaultVersionedObject).
				Writes(producedObject)
		
			routes = append(routes, route)
	
	}

	return &apiResource, resourceInfo, nil
}
```

以上便是 api-server 注册 api 的流程。

# 高可用及负载均衡

我们来看看 controller-plane 是如何实现高可用/负载均衡的。我们知道可以通过 deployment 的方式部署业务应用，这样即使某个 pod 挂了，controller 也会拉起<br>
一个新的 pod，从而实现高可用。至于负载均衡，可以通过 service 方式实现，service 创建 iptables 规则，从而当 client 以 vip 方式访问业务应用的时候，<br>
可以将流量均摊到后端 pod 上。<br>

但是对于 controller-plane master 节点来说，k8s 并没有实现其高可用以及负载均衡，我们需要借助 `keepalived + haproxy` 或者其他工具实现高可用/负载均衡功能。<br>
关于如何使用 `keepalived + haproxy` 配置 controller-plane 高可用，可以参考：https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md#bootstrap-the-cluster<br>
我们这里重点描述下 `keepalived + haproxy` 的实现原理。<br>

## haproxy

要实现负载均衡，我们需要有一个负载均衡器。haproxy 是一个四/七层负载均衡器。同时，haproxy 支持健康检查后端服务，当后端服务宕机的时候，能够及时修改<br>
ipvs 信息，从而将后续的客户端流量发送到正常运行的后端服务上。<br>

有了 haproxy 是不是就足够了，客户端流量打到 haproxy，haproxy 将流量分发到真正的后端服务，有问题的后端服务也能被及时监测到并摘除。看着好像没什么问题。<br>
可是**haproxy 所在的站点挂了，是不是整个控制面就不用了？**

## keepalived

keepalived 就是用来实现 haproxy 的高可用的。我们可以部署多个节点，每个节点都部署 keepalived + haproxy。这几个节点对外暴露一个同一的 vip,<br>
client 通过这个 vip 访问 keepalived 集群。keepalived 集群是有状态的，只有一个 master 节点，其他节点都是 backup 节点。任意时刻，只有 master<br>
节点对外提供服务，当 master 节点挂了之后，backup 节点会选举一个新的 master 节点，该 master 节点会接管 vip，客户端的流量会达到这个新的 master 节点，<br>
从而实现了高可用。<br>

这里就有问题了，多个 keepalived 节点对外暴露一个同一个的 vip，其他节点还可以接管这个 vip，这怎么理解如何实现的呢？<br>

这就是 `vrrp(virtual route redundant protocol)`即虚拟路由冗余协议。这个协议要解决的问题是路由器的高可用问题。想象一下，如果一个局域网就只有<br>
一个路由器，如果这个路由器挂了，那岂不是上不了忘了？vrrp 解决该问题的思路是将若干个路由器组成一个路由器组，在客户端看来，这个路由器组就是一个虚拟的路由器<br>
这个路由器组对外暴露一个 virtual ip 和 virtual mac 地址，任何时候只有一个路由器接管这两个地址。**这里的接管其实就是将虚拟ip 绑定到某个网卡上，接受<br>
所有目标地址为虚拟 ip 的 arp 请求，接受所有目标地址为 virtual mac 的数据包。**<br>

vrrp 中节点分两个角色：master 和 backup。只有 master 对外提供服务，master 会定时给所有的 backup 组播消息，注意在 vrrp 中，只有一种消息，就是<br>
这个组播消息，当 backup 节点收到 master 的组播消息后，就知道 master 节点是正常工作的，当超过一定时间没有收到数据包后，就会更改自动状态为 master，<br>
然后给其他 backup 节点发送组播消息，当接收到其他 backup 节点的响应后，就可以根据优先级回退到 backup 状态或者保持 master 状态。新 master 会将<br>
vip 绑定到自己的网卡上，后续接管所有目的地是 vip 的流量。<br>

正式因为有了 vrrp，实现了 keepalived 集群的高可用，即使某个节点挂了，也可以将流量切到其他节点上，而这一步**对用户来说完全透明的。**<br>

当然 keepalived 也可以实现负载均衡，不过其**只能实现四层均衡，通过 ipvs 实现**，要实现一些高级的负载均衡策略，我们还是要借助 haproxy。因此，<br>
`keepalived + haproxy`是一对在负载均衡场景下，经常使用到的组合。<br>

同样 keepalived 也有监控检查的功能，在我们的例子中，可以用来监控 haproxy 是否正常运行，如果 haproxy 异常退出了，keepalived 也就停止给其他节点<br>
发送 vrrp 组播消息，这样其他节点就可以接管 vip。


## 应用

有了 haproxy + keepalived 的基础知识后，我们看看如何配置 k8s 控制面的高可用以及负载均衡，我们来分析下官网给的例子：

### keepalived 配置

两个文件，一个配置文件，一个健康检查脚本。

- 配置文件
```text
! /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}

! 健康检查脚本
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance VI_1 {
    ! 节点的状态，可以是 master or backup
    state ${STATE}
    ! vip 要绑定的 网卡
    interface ${INTERFACE}
    ! 相同 router_id 的一组节点为一个虚拟节点
    virtual_router_id ${ROUTER_ID}
    ! 当前节点的优先级，优先级高的，在选举的时候，会成为新的 master
    priority ${PRIORITY}
    authentication {
        auth_type PASS
        auth_pass ${AUTH_PASS}
    }
    ! 对外暴露的 vip
    virtual_ipaddress {
        ${APISERVER_VIP}
    }
    ! 健康检查脚本
    track_script {
        check_apiserver
    }
}
```

- 健康检查脚本

```shell
#!/bin/sh

errorExit() {
    echo "*** $*" 1>&2
    exit 1
}

# APISERVER_DEST_PORT 为 haproxy 监听的端口，这里其实就是监控 haproxy process 的运行状态。
curl -sfk --max-time 2 https://localhost:${APISERVER_DEST_PORT}/healthz -o /dev/null || errorExit "Error GET https://localhost:${APISERVER_DEST_PORT}/healthz"
```

### haproxy 配置

```text
# /etc/haproxy/haproxy.cfg
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    log stdout format raw local0
    # 以 daemon 方式运行
    daemon 

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    # haproxy 可以工作四/七层，由 mode 指定。
    # 七层：http
    # 四层：tcp
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 1
    timeout http-request    10s
    timeout queue           20s
    timeout connect         5s
    timeout client          35s
    timeout server          35s
    timeout http-keep-alive 10s
    timeout check           10s

#---------------------------------------------------------------------
# apiserver frontend which proxys to the control plane nodes
#---------------------------------------------------------------------
frontend apiserver
    # haproxy 监听的端口
    bind *:${APISERVER_DEST_PORT}
    mode tcp
    option tcplog
    default_backend apiserverbackend

#---------------------------------------------------------------------
# round robin balancing for apiserver
#---------------------------------------------------------------------
backend apiserverbackend
    option httpchk

    http-check connect ssl
    # 通过 GET /healthz 健康检查后端实例
    http-check send meth GET uri /healthz
    http-check expect status 200

    mode tcp
    # 负载均衡策略
    balance     roundrobin
    # 实例的后端 instance 列表
    # HOST1_ADDRESS host1 的地址，APISERVER_SRC_PORT api-server 实例监听的端口 
    server ${HOST1_ID} ${HOST1_ADDRESS}:${APISERVER_SRC_PORT} check verify none
    # [...]
```

# master 节点中其他组件如何访问 api-server

master 节点中的组件比如：controller-manager、scheduler 直接通过 localhost 访问本节点上的 api-server。

# api-server 如何访问 etcd

api-server 也是通过 localhost 方式直接访问本节点上的 etcd 实例。因为到 api-server 的请求已经做了均衡，所以 api-server 直接访问本节点<br>
上的 etcd 实例，流量也是均衡的。<br>
