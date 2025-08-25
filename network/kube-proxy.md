几个问题：

- 某个 service 变化，需要 resync 整个 iptables 吗？

# kube-proxy

proxy 的意思就是 service 代理后端 pod。k8s 集群每台主机都会运行 kube-proxy，通过 iptables/ipvs 等方式，实现通过 service 即可访问
实际 pods.

## endpoint/endpointSlice 对象

kube-proxy 会监听系统中 service、endpointSlice 对象的变更，然后创建对应的 iptables/ipvs 规则。那么这里的 endpointSlice 对象的含义
是什么呢？又是什么时候创建的呢？跟 endpoint 的区别又是什么呢？

### endpoint

endpoint 对象跟 service 一一对应。endpoint 对象是由 endpoint controller 创建的。当 endpoint controller 监听到 service 以及 service
关联的 pod 对象变更的时候，就会创建/更新 对应的 endpoint 对象。

endpoint 对象包括 service 代理的所有 pod 的 ip+port 信息。

### endpointSlice

endpoint 的问题是，因为 endpoint 包括了 service 代理的所有 pod 信息，因此如果 kube-proxy 监听 endpoint 对象，只要任意一个 pod 的 ip 发生变化，
比如重启、添加/删除 pod，就需要将 endpoint 对象发送给 k8s 集群中的每个实例。当 service 代理的 pod 很多的时候，endpoint 对象是很大的，这会
消耗大量的带宽。

endpointSlice 对象的提出就是为了解决 endpoint 过大的问题。其思路是将一个大的 endpoint 拆分为 若干个小的 endpointSlice 对象。kube-proxy
只需要监听 endpointSlice 对象即可。这样当某个 pod 变化的时候，只会影响某几个 endpointSlice 对象，大大减少了网络带宽的消耗。

同样 endpointSlice 对象的创建也是由 k8s endpointSliceMirror controller 负责的。该 controller 会监听 endpoint 对象，然后创建/更新对应的
endpointSlice 对象。

## iptables

kube-proxy 通过监听 service、endpointSlice 对象的变更，然后创建对应的 iptables chain/rule，然后通过 iptables-restore 的方式将
iptables 规则应用到宿主机中。

接下来，我们重点看看 clusterIP、nodePort 两种类型的 service 的 iptables 实现。


### clusterIP

涉及两个 iptable: filter、nat。kube-proxy 构建 iptables chain/rule 规则如下：

- 将标准的 chain 完全重定向到自定义 chain，在自定义 chain 中描述具体的 rule
- 每一个 service 对应一个 chain，该 chain 下，每个 endpoint 对应一个 rule，而每个 rule 又 jump 到一个 endpoint 对应的 chain

我们看看源码实现：

首先是将标准的 chain 重定向到自定义 chain:

```go
var iptablesJumpChains = []iptablesJumpChain{
	{utiliptables.TableFilter, kubeExternalServicesChain, utiliptables.ChainInput, "kubernetes externally-visible service portals", []string{"-m", "conntrack", "--ctstate", "NEW"}},
	{utiliptables.TableFilter, kubeExternalServicesChain, utiliptables.ChainForward, "kubernetes externally-visible service portals", []string{"-m", "conntrack", "--ctstate", "NEW"}},
	{utiliptables.TableFilter, kubeNodePortsChain, utiliptables.ChainInput, "kubernetes health check service ports", nil},
	{utiliptables.TableFilter, kubeServicesChain, utiliptables.ChainForward, "kubernetes service portals", []string{"-m", "conntrack", "--ctstate", "NEW"}},
	{utiliptables.TableFilter, kubeServicesChain, utiliptables.ChainOutput, "kubernetes service portals", []string{"-m", "conntrack", "--ctstate", "NEW"}},
	{utiliptables.TableFilter, kubeForwardChain, utiliptables.ChainForward, "kubernetes forwarding rules", nil},
	{utiliptables.TableNAT, kubeServicesChain, utiliptables.ChainOutput, "kubernetes service portals", nil},
	{utiliptables.TableNAT, kubeServicesChain, utiliptables.ChainPrerouting, "kubernetes service portals", nil},
	{utiliptables.TableNAT, kubePostroutingChain, utiliptables.ChainPostrouting, "kubernetes postrouting rules", nil},
}
```
注意，不同的 table 可以创建同名的 chain，所以 filter 和 nat 中的 kubeServicesChain 是不同的 chain，这点要注意。

接着我们看看自定义 chain 的具体规则：
```go
func (proxier *Proxier) syncProxyRules() {
	// 首次需要 fullsync，后续只需要 partial sync 即可。
    // 所谓 fullsync，就是创建 iptablesJumpChains 中对应的 chain 以及从标准 chain 跳转到自定义 chain 的 rule
	tryPartialSync := !proxier.needFullSync
    
    // 将自上次 sync 以来的所有 serviceChanges、endpointsChanges 应用到 svcPortMap 和 endpointsMap中
    // proxier 就是根据 svcPortMap 和 endpointsMap 来构建 iptable rule 的
	serviceUpdateResult := proxier.svcPortMap.Update(proxier.serviceChanges)
	endpointUpdateResult := proxier.endpointsMap.Update(proxier.endpointsChanges)

	if !tryPartialSync {
		// Ensure that our jump rules (eg from PREROUTING to KUBE-SERVICES) exist.
		// We can't do this as part of the iptables-restore because we don't want
		// to specify/replace *all* of the rules in PREROUTING, etc.
		//
		// We need to create these rules when kube-proxy first starts, and we need
		// to recreate them if the utiliptables Monitor detects that iptables has
		// been flushed. In both of those cases, the code will force a full sync.
		// In all other cases, it ought to be safe to assume that the rules
		// already exist, so we'll skip this step when doing a partial sync, to
		// save us from having to invoke /sbin/iptables 20 times on each sync
		// (which will be very slow on hosts with lots of iptables rules).
		// 上面注释说的很清楚了，每次都调用，就会调用 iptables 20 次，因为 len(iptablesJumpChain) = 10，
        // 每一个 item 都涉及 ensureChain 和 ensureRule 两次调用。
        // 因此只有 fullsync 的时候，才会调用。
		for _, jump := range append(iptablesJumpChains, iptablesKubeletJumpChains...) {
			// 创建 chain
			if _, err := proxier.iptables.EnsureChain(jump.table, jump.dstChain); err != nil {
				return
			}
			args := jump.extraArgs
			args = append(args, "-j", string(jump.dstChain))
			// 创建对应的 rule
			if _, err := proxier.iptables.EnsureRule(utiliptables.Prepend, jump.table, jump.srcChain, args...); err != nil {
				return
			}
		}
	}

    // 后面逻辑都是以 iptables-restore 可以识别的语法来保存 iptables chain/rule
    // 最后通过 iptables-restore 使这些规则生效。
    // 文件规则如下：
    /*
       *filter
	   :INPUT DROP [0:0]
	   :FORWARD DROP [0:0]
	   :OUTPUT ACCEPT [0:0]
	   -A INPUT -s 192.168.1.0/24 -j ACCEPT
	   -A INPUT -p tcp --dport 22 -j ACCEPT
	   COMMIT
     */
	// Write chain lines for all the "top-level" chains we'll be filling in
	for _, chainName := range []utiliptables.Chain{kubeServicesChain, kubeExternalServicesChain, kubeForwardChain, kubeNodePortsChain, kubeProxyFirewallChain} {
		proxier.filterChains.Write(utiliptables.MakeChainLine(chainName))
	}
	for _, chainName := range []utiliptables.Chain{kubeServicesChain, kubeNodePortsChain, kubePostroutingChain, kubeMarkMasqChain} {
		proxier.natChains.Write(utiliptables.MakeChainLine(chainName))
	}

    // 这一部分主要定义 PostroutingChain 对应的 rule	
	// ! --mark 表示没有对应标记的直接返回
	proxier.natRules.Write(
		"-A", string(kubePostroutingChain),
		"-m", "mark", "!", "--mark", fmt.Sprintf("%s/%s", proxier.masqueradeMark, proxier.masqueradeMark),
		"-j", "RETURN",
	)
	// Clear the mark to avoid re-masquerading if the packet re-traverses the network stack.
	// 清空标记
	proxier.natRules.Write(
		"-A", string(kubePostroutingChain),
		"-j", "MARK", "--xor-mark", proxier.masqueradeMark,
	)
    
    // 进行 snat 转化
	// MASQUERADE 规则的含义是：将数据包的源 ip 地址自动转化为发送数据包的网络接口的 ip 地址
	// 而当发送数据包网卡地址变化的时候，SNAT 需要变更配置。
	masqRule := []string{
		"-A", string(kubePostroutingChain),
		"-m", "comment", "--comment", `"kubernetes service traffic requiring SNAT"`,
		"-j", "MASQUERADE", // 读音：ˌmæskəˈreɪd 含义：伪装
	}
	proxier.natRules.Write(masqRule)

	// Install the kubernetes-specific masquerade mark rule. We use a whole chain for
	// this so that it is easier to flush and change, for example if the mark
	// value should ever change.
	proxier.natRules.Write(
		"-A", string(kubeMarkMasqChain),
		"-j", "MARK", "--or-mark", proxier.masqueradeMark,
	)

	// Build rules for each service-port.
	for svcName, svc := range proxier.svcPortMap {
		svcInfo, ok := svc.(*servicePortInfo)
		if !ok {
			proxier.logger.Error(nil, "Failed to cast serviceInfo", "serviceName", svcName)
			continue
		}
		protocol := strings.ToLower(string(svcInfo.Protocol()))
		svcPortNameString := svcInfo.nameString

		// Figure out the endpoints for Cluster and Local traffic policy.
		// allLocallyReachableEndpoints is the set of all endpoints that can be routed to
		// from this node, given the service's traffic policies. hasEndpoints is true
		// if the service has any usable endpoints on any node, not just this one.
		allEndpoints := proxier.endpointsMap[svcName]
		clusterEndpoints, localEndpoints, allLocallyReachableEndpoints, hasEndpoints := proxy.CategorizeEndpoints(allEndpoints, svcInfo, proxier.nodeLabels)

		// 每个 service 对应一个 chain
		clusterPolicyChain := svcInfo.clusterPolicyChainName
		usesClusterPolicyChain := len(clusterEndpoints) > 0 && svcInfo.UsesClusterEndpoints()
		internalPolicyChain := clusterPolicyChain
		internalTrafficChain := internalPolicyChain

		var internalTrafficFilterTarget, internalTrafficFilterComment string
		if !hasEndpoints {
			// The service has no endpoints at all; hasInternalEndpoints and
			// hasExternalEndpoints will also be false, and we will not
			// generate any chains in the "nat" table for the service; only
			// rules in the "filter" table rejecting incoming packets for
			// the service's IPs.
			// 上面注释说的很清楚，如果 service 没有对应的 endpoint，nat 不会有对应的 rule 
            // filter 会有对应的 rule，rule 的行为为 reject 
			internalTrafficFilterTarget = "REJECT"
			internalTrafficFilterComment = fmt.Sprintf(`"%s has no endpoints"`, svcPortNameString)
		}

		filterRules := proxier.filterRules
		natChains := proxier.natChains
		natRules := proxier.natRules

		// Capture the clusterIP.
		if hasInternalEndpoints {
			// 如果 service 有 endpoint，则在 nat 的 kubeServicesChain 创建一个跟 service 对应的 chain
			natRules.Write(
				"-A", string(kubeServicesChain),
				"-m", "comment", "--comment", fmt.Sprintf(`"%s cluster IP"`, svcPortNameString),
				"-m", protocol, "-p", protocol,
				"-d", svcInfo.ClusterIP().String(),
				"--dport", strconv.Itoa(svcInfo.Port()),
				"-j", string(internalTrafficChain))
		} else {
			// 没有 endpoint，filter 的 kubeServicesChain 加入一条 reject rule
			filterRules.Write(
				"-A", string(kubeServicesChain),
				"-m", "comment", "--comment", internalTrafficFilterComment,
				"-m", protocol, "-p", protocol,
				"-d", svcInfo.ClusterIP().String(),
				"--dport", strconv.Itoa(svcInfo.Port()),
				"-j", internalTrafficFilterTarget,
			)
		}

		// Set up internal traffic handling.
		if hasInternalEndpoints {
			args = append(args[:0],
				"-m", "comment", "--comment", fmt.Sprintf(`"%s cluster IP"`, svcPortNameString),
				"-m", protocol, "-p", protocol,
				"-d", svcInfo.ClusterIP().String(),
				"--dport", strconv.Itoa(svcInfo.Port()),
			)
			if proxier.masqueradeAll {
				// 每个 service chain 加入一条 rule，负责给数据包打标
				natRules.Write(
					"-A", string(internalTrafficChain),
					args,
					"-j", string(kubeMarkMasqChain))
			}
		}

		// If Cluster policy is in use, create the chain and create rules jumping
		// from clusterPolicyChain to the clusterEndpoints
		if usesClusterPolicyChain {
			// 创建每个 service 对应的 chain
			natChains.Write(utiliptables.MakeChainLine(clusterPolicyChain))
			// 这一步很关键，创建跳转到 endpoint 的 rule
			// 后面可以看到每一个 endpoint 对应一个 chain
			proxier.writeServiceToEndpointRules(natRules, svcPortNameString, svcInfo, clusterPolicyChain, clusterEndpoints, args)
		}

		// Generate the per-endpoint chains.
		// 给每一个 endpoint 生成 chain/rule
		for _, ep := range allLocallyReachableEndpoints {
			epInfo, ok := ep.(*endpointInfo)
			endpointChain := epInfo.ChainName

			// Create the endpoint chain
			natChains.Write(utiliptables.MakeChainLine(endpointChain))

			// 每个 endpointChain 两条 rule
			// 1. 打标
			// 2. dnat
			args = append(args[:0], "-A", string(endpointChain))
			args = proxier.appendServiceCommentLocked(args, svcPortNameString)
			// Handle traffic that loops back to the originator with SNAT.
			natRules.Write(
				args,
				"-s", epInfo.IP(),
				"-j", string(kubeMarkMasqChain))

			// DNAT to final destination.
			args = append(args, "-m", protocol, "-p", protocol, "-j", "DNAT", "--to-destination", epInfo.String())
			natRules.Write(args)
		}
	}

	// Delete chains no longer in use. Since "iptables-save" can take several seconds
	// to run on hosts with lots of iptables rules, we don't bother to do this on
	// every sync in large clusters. (Stale chains will not be referenced by any
	// active rules, so they're harmless other than taking up memory.)
	// 删除 chain，思路是通过 iptables-save 获取当前的 rule，然后与本轮计算得到的 rule 取 diff
	deletedChains := 0
	if !proxier.largeClusterMode || time.Since(proxier.lastIPTablesCleanup) > proxier.syncPeriod {
		proxier.iptablesData.Reset()
		if err := proxier.iptables.SaveInto(utiliptables.TableNAT, proxier.iptablesData); err == nil {
			existingNATChains := utiliptables.GetChainsFromTable(proxier.iptablesData.Bytes())
			for chain := range existingNATChains.Difference(activeNATChains) {
				chainString := string(chain)
				if !isServiceChainName(chainString) {
					// Ignore chains that aren't ours.
					continue
				}
				// We must (as per iptables) write a chain-line
				// for it, which has the nice effect of flushing
				// the chain. Then we can remove the chain.
				proxier.natChains.Write(utiliptables.MakeChainLine(chain))
				// -X 删除 chain
				proxier.natRules.Write("-X", chainString)
			}
		}
	}
	
	// Sync rules.
	// 以 iptables-save/iptables-store 的格式将 rule 序列化到 iptablesData 中
	// 可以看到就涉及两个 table: filter、nat
	proxier.iptablesData.Reset()
	proxier.iptablesData.WriteString("*filter\n")
	proxier.iptablesData.Write(proxier.filterChains.Bytes())
	proxier.iptablesData.Write(proxier.filterRules.Bytes())
	proxier.iptablesData.WriteString("COMMIT\n")
	proxier.iptablesData.WriteString("*nat\n")
	proxier.iptablesData.Write(proxier.natChains.Bytes())
	proxier.iptablesData.Write(proxier.natRules.Bytes())
	proxier.iptablesData.WriteString("COMMIT\n")

	// 通过 iptables-restore 将 iptablesData 加载到内核中
	err := proxier.iptables.RestoreAll(proxier.iptablesData.Bytes(), utiliptables.NoFlushTables, utiliptables.RestoreCounters)

```

以上便是 iptables 生成 iptables 规则的主逻辑。

### nodePort

nodePort 模式的 service 就是通过 nodeIp + srv.port 来代理后端 pods，其 iptables 规则如下：

- 首先，在 kubeServicesChain 中为每个 nodeIP 添加一条跳转到 kubeNodePortChain 的 rule

```go
nodeIPs, err := proxier.nodePortAddresses.GetNodeIPs(proxier.networkInterfacer)
		if err != nil {
			proxier.logger.Error(err, "Failed to get node ip address matching nodeport cidrs, services with nodeport may not work as intended", "CIDRs", proxier.nodePortAddresses)
		}
		for _, ip := range nodeIPs {
			// create nodeport rules for each IP one by one
			proxier.natRules.Write(
				"-A", string(kubeServicesChain),
				"-m", "comment", "--comment", `"kubernetes service nodeports; NOTE: this must be the last rule in this chain"`,
				"-d", ip.String(),
				"-j", string(kubeNodePortsChain))
		}
```

- 接着，kubeNodePortsChain 中为每个 service 创建一个 rule

```go
	if svcInfo.NodePort() != 0 {
			if hasEndpoints {
				// 根据 srv.nodePort，为每个 srv 创建一条 rule，这条 rule 跳转到每个 service 对应的 chain
				natRules.Write(
					"-A", string(kubeNodePortsChain),
					"-m", "comment", "--comment", svcPortNameString,
					"-m", protocol, "-p", protocol,
					"--dport", strconv.Itoa(svcInfo.NodePort()),
					"-j", string(externalTrafficChain))
			}
			
		}
```

- externalTrafficChain 与 service 一一对应，内部存储到每个 endpoint 的 rule

- 同样，nodePort 的 postRouteingChain 也需要加 masquerade 实现 snat

这里有一点，需要注意的是，为什么在 postRouteingChain 中要实现 snat?

假设调用路径如下：client -> node1 -> node2。node2 是后端 pod 所在的节点。如果没有实现 snat，从 node1 -> node2 的请求的源地址还是
client 的 ip 地址，当 node2 发送响应的时候，响应的目的 ip 地址就是 client，这样 node2 就直接把响应发送给 client。可是 client 不一定有
与 node2 对应的套接字，大概率会报错。

而有了 snat，node1 发送请求给 node2 的时候，就会把请求的源地址改成网卡的 ip 地址，这样 node2 发送响应也是发送到 node1，然后 node1 会根据
snat 表，将目的地址转化为 client 的 ip，然后再通过路由将响应发送给 client.

## ipvs

ipvs 模式与 iptables 模式类似，也是监听 service、endpointSlice 对象的变更。其流程如下：

- 创建一个虚拟网卡(dummy interface)，如果是 clusterIP 模式，将所有 service 对应的 vip 都绑定到这个虚拟网卡上。当然，如果是 nodePort 模式，
不会绑定 nodeIp 到 dummy interface 上，nodeIp 只能对应一个 interface。
- 对于每一个 service + port， 都会创建一个虚拟 server，然后通过 netlink socket 添加到内核中。
- 对于每一个 endpoint + port，创建一个 real server 添加到内核中。
- 在 ipvs 模式下，仍然需要通过 iptables 完成 snat 的功能。

我们常说 ipvs 比 iptabls 性能好，为什么。目前能知道的是 ipvs 中维护后端 ips 的数据结构是 hash table，相比于 iptables 的 O(n) 时间复杂度，
ipvs 查找后端 ip 的复杂度只有 O(1)。其他原因待后续补充。

### netlink socket

一种与操作系统内核进行通信的方式。k8s 并没有使用 ipvsadm 来设置相关 ipvs 规则，而是直接通过 netlink socket 来构建各种规则。
有空可以研究下对应的开源库。