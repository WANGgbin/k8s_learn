# pod

pod 就是若干共享资源(network namespace, volume)的容器。这是一种设计模式，pod 也是一个逻辑概念，可以有多种实现。通过若干个共享资源的容器，可以解决单个容器无法解决的问题。