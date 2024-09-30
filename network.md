本文我们来介绍下，k8s 中，pod 之间、pod 与 宿主机之间的网络是如何互通的。<br>

# 容器到容器所在的宿主机

当我们使用 docker 创建容器后，通常容器会有自己的 network namespace。那么容器与宿主机之间如何通信呢？<br>

**通过 veth pair**, veth pair 即一对虚拟的网络接口，当我们在一个接口发送数据后，数据会立即出现在另一个接口，仿佛两个接口之间通过网线连接一样。<br>
veth pair 常用来在不同的 network namespace 之间通信。<br>

当我们使用 docker 创建一个容器的时候，实际上就会创建一个 veth pair，其中一个接口对应容器的 eth0，另一个接口在宿主机的 network namespace 中<br>
并被绑定到网桥设备 docker0 上。当被绑定到网桥上之后，宿主机这一段接口就没有任何数据处理能力包括数据路由，而是全部交由 docker0 这个网桥负责。<br>

可是我们为什么要有个网桥呢？想象一下，每创建一个容器，便创建一个 veth pair，现在容器与宿主机之间可以通信了，但是容器之间如何通信呢？难道每两个<br>
容器之间也需要创建一个 veth pair 吗？<br>

这就是网桥存在的意义，**作用类似于交换机，在一台宿主机上所有的容器都在同一个网段，网桥可以转发这个网段内的消息，进而实现了容器之间的网络互通**，同时网桥扮演容器网段的网关。<br>

## 同一宿主机不同容器通信

我们现在看看一台宿主机上容器之间的数据流动，假设有两个容器：
- container1
  - ip ：192.168.1.1
  - interface: eth0
  - veth pair 在宿主机的一侧为：ethHost1
- container2
  - ip: 192.168.1.2
  - interface: eth0
  - veth pair 在宿主机的一侧为：ethHost2

假设现在 container1 要给 container2 发送消息：

- container1 根据路由表决定通过虚拟网卡 eth0 发送消息
- docker0 ethHost1 端口接收到数据
- 发现数据包的目标 ip 地址在容器网段中，根据学习到的 arp 信息，通过 ethHost2 端口把数据发送出去
- container2 的 eth0 会收到此信息，随后沿着 container2 network namespace 的协议栈把数据包传到应用进程
- container2 恢复响应的流程相反，不再赘述

以上便实现了同一宿主机不同容器之间的网络互通。


## 容器与宿主机的通信

我们还是以前面的 container1 举例：

- container1 通过 eth0 发送消息
- docker0 ethHost1 端口接收到消息，docker0 发现目标地址不在容器网段，因此交由宿主机的路由表决定下一步去向。
- 因为目的地址是宿主机地址，因此该数据包交由宿主机的协议栈处理。
- 宿主机发送响应，同样查询路由表，在宿主机的路由表上会多一条记录，凡是与容器网段匹配的地址，都从 docker0 发出，同时是一个直连网络，即 gateway 字段<br>
为 0.0.0.0，则 docker0 需要直接获取数据包目标 ip 对应的 mac 地址，这里就是 container1 eth0 对应的 mac 地址，然后拼装成一个以太网帧，然后通过 ethHost1 发送出去。
- container1 的 eth0 接收到消息，然后由协议栈处理

以上便完成了容器与宿主机之间的网络通信。<br>

但是，容器如何与其他的宿主机通信，不同宿主机上的容器之间又如何通信，**这是 docker 没有解决的问题。**

# 跨主容器通信

有不同的实现方式，核心思路是通过隧道的方式封装原始数据包(两层或者三层数据包)，利用宿主机之间的网络将数据包发送到目标宿主机，然后宿主机上解包，发送给对应的容器,<br>
这其实就是 `overlay` 的思路。<br>

我们这里以 flannel 举例，介绍下两种实现方式：vxlan 和 udp。

## vxlan

为了方便描述，假设有两个节点，其详情如下：

```json
[
  {
    "name": "node1",
    "container-sub-net": "10.244.1.0/24",
    "vtep": {
      "name": "flannel.1",
      "vtep-ip": "10.244.1.0"
    },
    "container1": {
      "ip": "10.244.1.1"
    }
  },
  {
    "name": "node2",
    "container-sub-net": "10.244.2.0/24",
    "vtep": {
      "name": "flannel.1",
      "vtep-ip": "10.244.2.0"
    },
    "container2": {
      "ip": "10.244.2.1"
    }
  }
]
```

  每个节点上都会运行一个 flannel 进程，在 k8s 中，其是通过 daemonset 的方式部署的。flannel 会创建一个 vtep(virtual tunnel endpoint) 设备，<br>
该设备工作在链路层，有 mac 地址。<br>

  每个节点上都会在路由表中加入若干个条目，每个条目与每个运行 flannel 的节点对应。比如节点 node1 其容器网络为：10.244.1.0/24，<br>
flannel 创建的 vtep 设备的 ip 为：10.244.1.0，设备名为：flannel.1。那么在 node2 上就会多这么一条 ip route:<br>

```shell
Destination Gateway Genmask Flags Metric Ref Use Iface
10.244.1.0 10.244.1.0 255.255.255.0 UG 0 0 0 flannel.1
```

  接入现在要发送一条从 node2.container2 发往 node1.container1 的数据。路径如下：

  - node2.container2 把数据发送到对应的网桥 cni
  - cni 接收到判断不属于当前容器网段，交给宿主机的路由表判断下一步去向
  - 因为 node2 多了一条特殊的路由记录，因此，该数据包交由 flannel.1 处理
  - flannel.1 是个 vtep，工作在二层，因此需要路由表中 gateway 即node1 vtep 设备的 mac 地址(这个信息实际上也是由 flannel 维护的，猜测类似于通过 informer 机制监听 flannel config 对象的变更，然后 apply 到当前节点)
  - flannel.1 组装完毕一个完整的 ethernet frame 后，然后会再获取 10.244.1.0 网段所在的 node1 的 ip 地址(这也是由 flannel 维护的)。
  - 然后以 udp 的方式发送到 node1
  - node1 接收到数据包，沿着协议栈处理。发现数据包多了一个 vxlan header，然后交由 flannel.1 处理
  - node1 上的 flannel.1 接受数据包，判断目标 mac 地址跟自己一致，然后交由路由表决定下一步去向
  - 因为目标 ip 地址是 node1.container1，所以根据路由表，此数据包交由 node1 网桥 cni 处理，同时这是个直连网络，因此 cni 直接获取 container1 的 mac 地址，然后组装成 ethernet frame 发送给 container1

以上便实现了跨主机容器之间的通信。<br>

我们可以看到 vxlan 的关键在于 flannel 进程能够监听集群中其他节点的相关信息并应用到当前节点中，从而保证能够将数据发送到正确的 node 上。

## udp

udp 的实现方式与 vxlan 类似，不同点在于其创建的是 tun 设备，该设备**可以完成 ip 数据包在内核态和用户态之间的交互，用户态通过读取/写入文件的方式读取/写入 tun 的 ip 数据包**。<br>

因为涉及到用户态与内核态之间的数据拷贝，性能较差，这种方式已经过失了。

# CNI

如果要实现一个 k8s 的网络方案，其包括两部分内容：

- 通过 daemon pod 方式设置 k8s 集群每个节点上的网络，比如创建 vtep 设备，添加路由表等。
- CRI 在创建 pod 的时候调用 CNI(Container Network Interface)，设置 pod 网络环境。比如：创建虚拟网卡，创建 veth pair 并绑定到 cni 网桥上。

实际上，CNI 就是一系列的二进制，存放在特定目录下。那么 CRI 怎么知道要调用哪个 CNI 呢？实际上 CRI 会在特定目录 `/etc/cni/net.d/`下查找配置文件，<br>
每个网络项目都会在此目录下创建一个配置文件，描述配置 pod 网络环境的详细流程，如果存在多个文件，则 CRI 会按字母序排序取第一个文件。<br>

该配置文件的 plugin 中可以指定多个 CNI Binary，CRI 就会按需调用多个 CNI，完成对 pod 网络环境的设置。我们还是以 flannel 项目举例，其对应的 yaml 配置如下：<br>

```yaml
# 对应的配置文件，在启动 flanneld 的时候，会将此配置文件拷贝到 /etc/cni/net.d 目录下
apiVersion: v1
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel", # 底层调用 bridge 等插件，完成容器 veth pair 创建、ip 地址分配等。 
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "EnableNFTables": false,
      "Backend": {
        "Type": "vxlan"
      }
    }
kind: ConfigMap
metadata:
  labels:
    app: flannel
    k8s-app: flannel
    tier: node
  name: kube-flannel-cfg
  namespace: kube-flannel
---
apiVersion: apps/v1
# 通过 daemonset 方式部署 flannel
kind: DaemonSet
metadata:
  labels:
    app: flannel
    k8s-app: flannel
    tier: node
  name: kube-flannel-ds
  namespace: kube-flannel
spec:
  selector:
    matchLabels:
      app: flannel
      k8s-app: flannel
  template:
    metadata:
      labels:
        app: flannel
        k8s-app: flannel
        tier: node
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
      containers:
      - args:
        - --ip-masq
        - --kube-subnet-mgr
        # 拉起 flanneld 进程，完成节点网络配置。
        command:
        - /opt/bin/flanneld
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: EVENT_QUEUE_DEPTH
          value: "5000"
        image: docker.io/flannel/flannel:v0.25.6
        name: kube-flannel
        resources:
          requests:
            cpu: 100m
            memory: 50Mi
        securityContext:
          capabilities:
            add:
            - NET_ADMIN
            - NET_RAW
          privileged: false
        volumeMounts:
        - mountPath: /run/flannel
          name: run
        - mountPath: /etc/kube-flannel/
          name: flannel-cfg
        - mountPath: /run/xtables.lock
          name: xtables-lock
      hostNetwork: true
      # 会有两个 init container 完成初始化操作
      initContainers:
      - args:
        - -f
        - /flannel
        - /opt/cni/bin/flannel
        command:
        - cp
        image: docker.io/flannel/flannel-cni-plugin:v1.5.1-flannel2
        # 将 flannel 安装到 /opt/cni/bin 中
        name: install-cni-plugin
        volumeMounts:
        - mountPath: /opt/cni/bin
          name: cni-plugin
      - args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        command:
        - cp
        image: docker.io/flannel/flannel:v0.25.6
        # 将配置文件拷贝到 /etc/cni/net.d 下
        name: install-cni
        volumeMounts:
        - mountPath: /etc/cni/net.d
          name: cni
        - mountPath: /etc/kube-flannel/
          name: flannel-cfg
      priorityClassName: system-node-critical
      serviceAccountName: flannel
      # 不可调度节点上也可以运行此 pod
      tolerations:
      - effect: NoSchedule
        operator: Exists
      volumes:
      - hostPath:
          path: /run/flannel
        name: run
      - hostPath:
          path: /opt/cni/bin
        name: cni-plugin
      - hostPath:
          path: /etc/cni/net.d
        name: cni
      - configMap:
          name: kube-flannel-cfg
        name: flannel-cfg
      - hostPath:
          path: /run/xtables.lock
          type: FileOrCreate
        name: xtables-lock

```

# flannel

[flannel](https://github.com/flannel-io/flannel) 是一个典型的 CNI 实现，我们看看其源码实现。

flannel pod 启动后，会创建 vtep 虚拟设备，并将自己的所在节点的 ip、vtep mac 地址等信息写入到 node 对象的 annotations 中。同时会通过 node informer<br>
监控 k8s 集群中 node 对象的信息，然后从 annotations 中获取对应的信息，然后 apply 到当前节点，包括创建 route、更新 fdb(forward database) 等。<br>

我们看看 flannel 监控集群 node 信息并更新路由、fdb 等信息的代码实现：
```go
func (nw *network) handleSubnetEvents(batch []lease.Event) {
	// event 对应一个 node 的信息
	for _, event := range batch {
		sn := event.Lease.Subnet
		v6Sn := event.Lease.IPv6Subnet
		attrs := event.Lease.Attrs

		var (
			vxlanAttrs            vxlanLeaseAttrs
			vxlanRoute            netlink.Route
		)

		if event.Lease.EnableIPv4 && nw.dev != nil {
			// 额外信息都存储在 attrs.BackendData 中
			if err := json.Unmarshal(attrs.BackendData, &vxlanAttrs); err != nil {
				log.Error("error decoding subnet lease JSON: ", err)
				continue
			}

			// 生成对应的 route 信息
			vxlanRoute = netlink.Route{
				LinkIndex: nw.dev.link.Attrs().Index,
				Scope:     netlink.SCOPE_UNIVERSE,
				Dst:       sn.ToIPNet(),
				Gw:        sn.IP.ToIP(),
			}
			vxlanRoute.SetFlag(syscall.RTNH_F_ONLINK)

		switch event.Type {
		// 如果是 add 类别，则在当前节点添加 route
		case lease.EventAdded:
			if event.Lease.EnableIPv4 {
					log.V(2).Infof("adding subnet: %s PublicIP: %s VtepMAC: %s", sn, attrs.PublicIP, net.HardwareAddr(vxlanAttrs.VtepMAC))
					if err := retry.Do(func() error {
						// 添加 node vtep ip 到 mac 的映射
						return nw.dev.AddARP(neighbor{IP: sn.IP, MAC: net.HardwareAddr(vxlanAttrs.VtepMAC)})
					})

					if err := retry.Do(func() error {
						// fdb 添加信息，这样能够根据 vtepmac 找到对应的 node 的 publicIP
						return nw.dev.AddFDB(neighbor{IP: attrs.PublicIP, MAC: net.HardwareAddr(vxlanAttrs.VtepMAC)})
					})

					if err := retry.Do(func() error {
						// 添加路由信息
						return netlink.RouteReplace(&vxlanRoute)
					})
			}
        // 如果是 remove 类别，则删除对应信息
		case lease.EventRemoved:
			if event.Lease.EnableIPv4 {
					log.V(2).Infof("removing subnet: %s PublicIP: %s VtepMAC: %s", sn, attrs.PublicIP, net.HardwareAddr(vxlanAttrs.VtepMAC))

					// Try to remove all entries - don't bail out if one of them fails.
					if err := retry.Do(func() error {
						return nw.dev.DelARP(neighbor{IP: sn.IP, MAC: net.HardwareAddr(vxlanAttrs.VtepMAC)})

					if err := retry.Do(func() error {
						return nw.dev.DelFDB(neighbor{IP: attrs.PublicIP, MAC: net.HardwareAddr(vxlanAttrs.VtepMAC)})
					})

					if err := retry.Do(func() error {
						return netlink.RouteDel(&vxlanRoute)
					})
			}
		}
	}
}
```

同样，flannel 与 内核交互的方式都是通过 `netlink socket` 进行的。