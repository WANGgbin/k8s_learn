描述 k8s 中 cri 相关内容。我们重点看看 docker 项目 cri 的相关实现。<br>

几个问题：

 docker 与 runc 区别？

# 整体架构

CRI 的整体链路如下：<br>

kubelet -> CRI -> containerd -> shim -> runc

我们来大致了解下每个组件的定位：

- CRI 

  就是 k8s 与 容器实现的中间层

- containerd

  定位与 docker 一致，容器运行时的一种实现。

- runc

  负责容器进程的创建操作，一次性工具，创建完容器进程后退出。

- shim

  是 containerd 与 runc 的中间层，每个容器对应一个 shim 实例，shim 是个单独的进程，接受来自 containerd 的请求，然后调用 runc 执行具体的容器操作.
  因为 runc 是个一次性工具，创建完容器进程后就退出了。

  管理容器进程的工作(监控状态等)为什么不直接交给 containerd 呢？如果交给 containerd，如果 containerd 异常退出/升级，那么所有的容器进程的父进程就会
  成为1号进程，后续 containerd 就无法监控容器进程(通过 waitpid 监控容器的状态)的状态了。而 通过每一个容器一个 shim，即使 shim 进程异常退出了，也只影响
  一个容器中的进程，而不是所有容器的进程。

资源：

- [containerd](https://github.com/containerd/containerd)
- [runc](https://github.com/opencontainers/runc)

> containerd 的 cri 与 containerd 服务在一起，cri 与 containerd 其他组件部署在一个进程中，通过直接 func 的形式，调用其他组件。<br>
> shim 的实现也在 containerd 中，是一个单独的二进制。

# 容器创建

我们知道容器的三板斧：namespace、cgroup、rootfs。容器是一个抽象的概念，可以理解为是进程的运行时环境。容器的生命周期与容器中的首进程生命周期一样，
当容器主进程(pid = 1 的进程)退出时，容器退出。我们来看看容器进程是怎么创建出来的。<br>

这部分逻辑主要在 runc 中，runc 这个项目没有提供 api server，因此只能通过 exec binary 的形式调用 runc，shim 项目中就是直接通过调用 runc
来调用相应的能力的。<br>

runc 创建容器对应的是 runc create --bundle 命令，这个 --bundle 参数指定的是一个目录，这个目录里面有两个重要的成员：

- config.json

    容器进程需要的所有上下文，包括 namespace 等等。

- rootfs

    容器进程对应的 rootfs，即根文件系统。

这个 bundle 目录是由 containerd 写入的。runc 内部会拉起一个 `runc init` 的子进程，该子进程负责初始化容器进程的各种上下文，包括创建各种 namespace、执行 mount、<br>
切换根目录等等。这里比较特别的是创建 namespace 的过程。<br>

因为 go 进入 main 之前已经会拉起多个 thread，所以 runc 在执行 main 之前通过 cgo 的方式执行 c 代码来初始化各种 namespace，同时还会再 clone 一次，原因是 pid namespace<br>
比较特殊，在一个进程中，即使 set pid namespace 了也不会生效，如果生效了，代码中的 getpid() 的返回就会变化，这回导致很多的代码不兼容。因此还需要 clone 一次 使得 pid namespace 生效。<br>
那么 cgo 代码是如何获取 namespace 等这些初始化信息的呢？通过 pipe，父进程将相应的数据写入 pipe，然后 runc init 就从 pipe 读取对应的数据。cgo 相关逻辑参考：`libcontainer/nsenter/nsexec.c:447`<br>


总结来说，创建容器的操作本质就是：runc 拉起一个子进程，该进程初始化容器的上下文，然后直接通过 exec 用户 binary 切换到用户应用。

# 容器 exec 原理

我们说一个容器本质就是个进程，这里的进程对应的就是容器的初始化/主进程，也就是上一步创建出来的进程，容器的生命周期跟主进程保持一致。实际上一个容器中可以包括多个进程，可以通过 exec 方式<br>
创建新的进程或者初始化进程也可以拉起子进程。我们这里看看 exec 原理。

exec 其实更简单，唯一需要注意的是，其各种 namespace 直接通过容器初始化进程对应的 namespace path(比如 netns path: /proc/pid/ns/net) 指定。exec 也是通过 run init 方式拉起。
直接找到用户指定的 bin，exec 即可。

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
  这里我们需要注意 namespace 的生命周期，namespace 结束的条件为：

  - namespace 中的所有进程退出。
  - 所有打开 namespace 的句柄关闭，而通过 mount 操作，可以增加底层 ns 对应文件的 inode 计数，这样即使原来的 /proc 文件都不存在，我们还是可以
    通过挂载点访问 ns。

- 再将进程的 network namespace 改回去

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

		// 将当前线程的 net ns 改回去
		defer origNS.Set()

		// bind mount the netns from the current thread (from /proc) onto the
		// mount point. This causes the namespace to persist, even when there
		// are no threads in the ns.
		// 妙啊！注意这种持久化的方式。后续就可以通过 nspath 来引用这个 network namespace 了，
        // 其他进程如果想加入这个 network namespace，只需要 open + setns 即可。
		unix.Mount(getCurrentThreadNetNSPath(), nsPath, "none", unix.MS_BIND, "")
	}
```