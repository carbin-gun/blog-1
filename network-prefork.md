在深入浅出网络模型里面，我大概介绍了几种个人理解的网络编程模型，从涛哥和葱头的反馈来看，讲的东西还不怎么详细，所以后续会不断的补充完善。

# prefork模型

今天主要探讨的是prefork模型，我在前文说过，因为unix的fork很迅速，所以可以采用multi process模式，即每次accept到一个socket之后，就fork一个子进程，然后盖子进程进行处理。这样就有一个问题，如果连接数过多，会导致fork过多的子进程，造成系统压力。

所以就有了prefork模型，也就是预先分配一些子进程进行后续网络处理。显而易见的好处就在于不用每次都fork，减少了很多系统开销。

## main accept + prefork handle

prefork我认为有两种模型，第一种就是主进程bind，listen之后，就一直accept，当接收到新的连接的时候，将这个socket发给其中一个子进程处理。

这里就涉及到一个问题，父进程如何将收到的socket发送给子进程。因为此时子进程在accept之前已经创建，所以不可能共享后续父进程新创建的socket。所以我们需要一套机制来进行进程之间的socket传递，而这个是早已经有的实现。

- \*nix平台下面，使用socketpair + sendmsg + recvmsg
- windows平台下面，使用WSADuplicateSocket

因为网上这方面的资料很多的，这里就不在详细说明，只是觉得对于这种方法，还需要写跨平台的代码，并且进程传递socket也需要系统开销，所以这种模型我没有实现过。flup的prefork模型采用的是这种做法，后续可以看看。

## prefork accept

当主进程bind，listen之后，各个子进程自己去accept，这样当有一个连接请求到来的时候，只要有一个子进程能够accept，就能进行后续处理了。这个模型相对来说，比较简单，较易实现，tornado的多进程模型就采用的是这样方案。

不过这种方案有可能出现惊群效应，也就是说，当所有子进程都在accept的时候，如果来了一个请求，会将所有子进程唤醒，但是只有一个进程能够处理这个请求，而其他 进程继续休眠等待。

解决这个问题，通常的做法就是在accept的时候进行锁保护，保证只能有一个进行进行accept，但是我觉得，惊群效应出现的情况在于休眠的进程很多，但请求很少的情况，如果有大量的连接请求，所有子进程都在忙碌，这个问题自然就没有了。所以也不需要怎么特别关注。