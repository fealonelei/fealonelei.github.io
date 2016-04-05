# The heap vs the stack [swift]

当创建像 class 一样的引用类型时，系统将真实值存储在内存里叫做 `heap` 的区域，而值类型的像 struct 等存储在 `stack` 区域。
heap 和 stack 在任何可执行的程序中都扮演着必须的角色。对 heap 和 stack 是什么和他们怎么工作有一个基本的理解，有助于你理清 class 和 structure 之间的功能性区别。

* 系统用 `stack` 存储任何正在执行的线程所需的任何东西(东西 ？？？， anything 怎么翻译嘛)，它完全由 CPU 来管理和优化。当函数创建了一个变量，`stack` 将存储这个变量并在函数退出后销毁这个变量。因为 stack 十分有序，并且是高效的，因此 stack 运行很快。
* 系统用 `heap` 来存储被其他对象（objects）引用的数据。Heap 通常是内存里一块大区域，系统可以在这里动态的分配和请求内存区块。和 stack 不同，heap 并不会自动销毁它持有的对象，所以，alloc 和 delloc 都是外部的责任。因此，和 stack 相比，创建和移除 heap 上的数据比较慢。

通过下图能更清楚的看清楚 heap 和 stack 的区别：
![heapStack.png](http://7xi6q9.com1.z0.glb.clouddn.com/%E5%A0%86%E6%A0%88.png)


* 当创建一个 class 的实例，代码在 heap 上请求一块内存，并且存储对象本身；这个实例的 first name 和 last name 在图表的右侧。命名的变量在内存的地址存储在 stack 上，即图表上显示的`引用`存储在左侧。
* 当创建一个 struct 的时候，值本身存储在 stack 上，heap 不参与其中。

以上就是 heap 和 stack 的简短介绍。



