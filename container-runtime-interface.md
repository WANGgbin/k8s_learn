描述 k8s 中 cri 相关内容。我们重点看看 docker 项目 cri 的相关实现。<br>

几个问题：

 docker 与 runc 区别？

# 整体架构

CRI 的整体链路如下：<br>

kubelet -> CRI -> containerd -> shim -> runc <br>

我们来大致了解下每个组件的定位：

- CRI 就是 k8s 与 容器实现的中间层

- containerd 定位与 docker 一致，容器运行时的一种实现。

- shim 是 containerd 与 runc 的中间层，每个容器对应一个 shim 实例，shim 是个单独的进程，接受来自 containerd 的请求，然后调用 runc 执行具体的容器操作，<br>
这一层的目的就是为了结构 containerd 与 具体的容器运行时，比如 runc。

- runc 真正的容器运行时，负责容器的创建/exec 等操作。

那么 containerd 与 runc 的区别是什么呢？<br>

containerd 是一个容器实现项目，范围更广。而 runc 负责实际的容器操作，概念更小。<br>

资源：

- [containerd](https://github.com/containerd/containerd)
- [runc](https://github.com/opencontainers/runc)

> containerd 的 cri 与 containerd 服务在一起，cri 与 containerd 其他组件部署在一个进程中，通过直接 func 的形式，调用其他组件。<br>
> shim 的实现也在 containerd 中，是一个单独的二进制。

# 容器创建

我们知道容器的三板斧：namespace、cgroup、rootfs。容器本质对应一个主进程，我们来看看这个进程是怎么创建出来的。<br>

这部分逻辑主要在 runc 中，runc 这个项目没有提供 api server，因此只能通过 exec binary 的形式调用 runc，shim 项目中就是直接通过调用 runc <br>
来调用相应的能力的。<br>

runc 创建容器对应的是 runc create --bundle 命令，这个 --bundle 参数指定的是一个目录，这个目录里面有两个重要的成员：

- config.json

    容器进程需要的所有上下文，包括 namespace 等等。

- rootfs

    容器进程对应的 rootfs，即根文件系统。

这个 bundle 目录是由 containerd 写入的。runc 内部会拉起一个 `runc init` 的子进程，该子进程负责初始化容器进程的各种上下文，包括创建各种 namespace、执行 mount、<br>
切换根目录等等。这里比较特别的是创建 namespace 的过程。<br>

因为 go 进入 main 之前已经会拉起多个 thread，所以 runc 在执行 main 之前通过 cgo 的方式执行 c 代码来初始化各种 namespace，同时还会再 clone 一次，原因是 pid namespace<br>
比较特殊，在一个进程中，及时 set pid namespace 了也不会生效，如果生效了，代码中的 getpid() 的返回就会变化，这回导致很多的代码不兼容。因此还需要 clone 一次 使得 pid namespace 生效。<br>
那么 cgo 代码是如何获取 namespace 等这些初始化信息的呢？通过 pipe，父进程将相应的数据写入 pipe，然后 runc init 就从 pipe 读取对应的数据。cgo 相关逻辑参考：`libcontainer/nsenter/nsexec.c:447`<br>


总结来说，创建容器的操作本质就是：runc 拉起一个子进程，该进程初始化容器的上下文，然后直接通过 exec 用户 binary 切换到用户应用。

# 容器 exec 原理

我们说一个容器本质就是个进程，这里的进程对应的就是容器的初始化/主进程，也就是上一步创建出来的进程，容器的声明周期跟主进程保持一致。实际上一个容器中可以包括多个进程，可以通过 exec 方式<br>
创建新的进程或者初始化进程也可以拉起子进程。我们这里看看 exec 原理。<br>

exec 其实更简单，唯一需要注意的是，其各种 namespace 直接通过容器初始化进程对应的 namespace path(比如 netns path: /proc/pid/ns/net) 指定。exec 也是通过 run init 方式拉起，不过不执行 mount，chroot 等操作。<br>
直接找到用户指定的 bin，exec 即可。<br>

runc 创建初始化进程和 exec 逻辑可以参考 runc 项目的：`libcontainer/init_linux.go:222`，具体我们这里不再展开。

# 创建 namespace

namespace 相关操作包括：

- unshare

  该系统调用会创建一个对应类型的 namespace，并将当前进程加入到这个 namespace 中。

- setns

  可以更改当前进程的 namespace.

在 pod 的概念中，pod 中的各个容器是共享 network namespace 的。那么在创建其他容器的时候，如何让其他容器进程加入到已有的 network namespace 中呢？<br>

只要我们知道 network namespace 对应的 path，然后 setns 不就可以了吗？<br>

是的，可是我们如何指定 network namespace path 呢？我们知道进程的 /proc/pid/ns/net 对应的就是其 network namespace 的 path。可是这种方式不保险，<br>
如果对应的进程结束了，这个 path 也就无效。我们来看看 containerd 如何解决此问题的。<br>

- 当前进程通过 unshare 创建一个新的 network namespace

- 然后将 /proc/pid/ns/net 挂载到另外一个路径，这样即使 network namespace 中的所有进程都结束了，该 network namespace 还在。

- 再将进程的 network namespace 该回去

- 后续 pod 中所有容器的 network namespace 都可以通过这个 mount 的 path 指定。

我们看看代码实现：

```go
func() {
		var origNS cnins.NetNS
		// 获取当前线程的 net ns path，也就是 /proc 下对应的文件
		origNS, err = cnins.GetNS(getCurrentThreadNetNSPath())
		if err != nil {
			return
		}
		defer origNS.Close()

		// unshare 系统调用用来创建一个新的 namespace 并将当前进程加入到 namespace 当中，
		// 由参数决定创建什么 namespace
		err = unix.Unshare(unix.CLONE_NEWNET)
		if err != nil {
			return
		}

		// 将当前线程的 net ns 该回去
		defer origNS.Set()

		// bind mount the netns from the current thread (from /proc) onto the
		// mount point. This causes the namespace to persist, even when there
		// are no threads in the ns.
		// 妙啊！注意这种持久化的方式。后续就可以通过 nspath 来引用这个 network namespace 了，
        // 其他进程如果想加入这个 network namespace，只需要 open + setns 即可。
		unix.Mount(getCurrentThreadNetNSPath(), nsPath, "none", unix.MS_BIND, "")
	}
```