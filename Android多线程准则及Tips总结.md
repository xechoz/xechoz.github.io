# 一. 多线程准则

`sychronized`的原理&机制，From Orocle Java Doc- [Intrinsic Locks and Synchronization](https://docs.oracle.com/javase/tutorial/essential/concurrency/locksync.html)

> Synchronization is built around an internal entity known as the intrinsic lock or monitor lock. 
> (The API specification often refers to this entity simply as a "monitor.") Intrinsic locks play a role in both aspects of synchronization: 
> enforcing exclusive access to an object's state and > > establishing happens-before relationships that are essential to visibility.
> Every object has an intrinsic lock associated with it. By convention, a thread that needs exclusive 
> and consistent access to an object's fields has to acquire the object's intrinsic lock before accessing them, 
> and then release the intrinsic lock when it's done with them. 
> A thread is said to own the intrinsic lock between the time it has acquired the lock and released the lock. 
> As long as a thread owns an intrinsic lock, no other thread can acquire the same lock. The other thread will block when it attempts to acquire the lock.
> When a thread releases an intrinsic lock, a happens-before relationship is established between that action and any subsequent acquisition of the same lock.

1. 可能在多个线程读写的对象，需要做好同步。
	
	最简单的方法是在读写的地方使用 `synchronized `，或者定义变量的时候用 `volatile`。
	但要适度，因为会引起性能问题，虽然影响的性能对一般的应用来说很小，但对于对并发性能敏感的场景要小心。

    `volatile` 相关文章

    1. [Java Volatile Keyword - Jenkov Tutorials](http://tutorials.jenkov.com/java-concurrency/volatile.html)
    2. [InfoQ: 聊聊并发（一）——深入分析Volatile的实现原理](http://www.infoq.com/cn/articles/ftf-java-volatile#anch83442)

2. `synchronized ` 的 作用范围越小越好。

	因为 **sychronized 同步是阻塞的，会降低多线程的性能**。

3. `synchronized ` block 的执行时间一定要 **短**。

    ```
        private void demo() {
            sychronized(objectA) {
                doIt(); // 一定要快
            }

            afterWork();
        }
    ```

	理由同上，**sychronized 同步是阻塞的，会降低多线程的性能**。

4. `synchronized method` 要少用

    因为是整个 Object 的 sychronized
    
5. 多线程导致内存泄露

	对于提供了 callback， listener， register 等 注入行为的线程，要注意：

	1. 注入&引用的外部对象 最好使用非 StrongReference 的引用方式。比如用 WeakReference

	2. 在线程销毁的时候要释放引用的外部对象。

TODO： 列出依据来源

# 2. 多线程规范用法

## 2.1 什么时候使用线程

1. 频繁调用或者公共的子线程，放在 ThreadPool

## 2.2 Android 中使用线程的规范

1. Handler 及 HandlerThread 的规范用法

TODO: 示例

```Java
```

2. View.post 原理及注意事项

3. Looper, MessageQueue 相关原理简单概括

### 精简伪代码

```java
loop() {
	// loop forever until quit() is called
	for (;;) {
		Message msg = nextMsg();

		if (msg != null) {
			runMessageCallbackTask()
		} else {
			if (isQuit) {
				quitLoop();
				return;
			}

			doIdelTask();
		}
	}
}

nextMsg() {
	return system_call_epoll(); // 如果没有event，则block 线程，等待event
}
```


 ### 轮询 调用流程

```
JAVA: Looper.loop() ->
JAVA: MessageQueue.next() ->
JNI: nativePollOnce(), android_os_MessageQueue_nativePollOnce() ->
C++: NativeMessageQueue::pollOnce() ->
C++: Looper::pollOnce(): 遍历消息列表,找到则返回；否则下一步 ->
C++: Looper::pollInner() ->
C: epoll_wait(): 轮询等待event，epoll, Linux 系统调用
```

epoll_wait 说明: [Linux Programmer's Manual: EPOLL(7)](http://man7.org/linux/man-pages/man7/epoll.7.html)
> epoll_wait(2) waits for I/O events, blocking the calling thread if
> no events are currently available.


# 二. 多线程实践Tips

1. 优先使用标志对象同步，避免使用 sychronized method 或 整个顶部对象

适用场景: TODO

```Java
public class Demo <T> {
	private Object lockList = new Object();
	private List<T> publicList;

	...

	// 首选写法
	public void addItem(T item) {
		sychronized(lockList) {
			...
			publicList.addItem(item);
			... 
		}
	}

	// 尽量不要这样用。因为调用此方法的线程都会阻塞? TODO: 待验证。列出原因
	sychronized public void addItem(T item) {
		publicList.addItem(item);

		... 
	}

	// 不要这样写。sychronized(this) 导致此对象的读写阻塞? TODO： 待验证
	public void addItem(T item) {
		sychronized(this) {
			publicList.addItem(item);
		}
	}
}
```

注意 **需要同步的对象，不要暴露给外部**, 因为调用者直接对同步对象的读写会导致不同步的问题

```
	// NEVER DO THIS !!!
	public List<T> getList() {
		return publicList; 
	}
```

参考 Effective Java, Item#xx

2. Singleton 的写法

没有用到 `sychronized` 实现，性能是最好的。

```Java

public class Demo {
	private static class InstanceHolder {
		private static Demo INSTANCE = new Demo();
	}

	private Demo() {
		...
	}

	public static getInstance() {
		return InstanceHolder.INSTANCE;
	}
}
```


# 三. 必读文章

1. [Intrinsic Locks and Synchronization](https://docs.oracle.com/javase/tutorial/essential/concurrency/locksync.html)
2. [Oracle Doc: Java Concurrency](https://docs.oracle.com/javase/tutorial/essential/concurrency/index.html)
3. [Understanding Android/Java Processes and Threads Related Concepts ](http://codetheory.in/android-handlers-runnables-loopers-messagequeue-handlerthread/)

# 参考: 

1. [docs.oracle.com：Chapter 17. Threads and Locks](http://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html)
2. [docs.oracle.com: Atomic Access](https://docs.oracle.com/javase/tutorial/essential/concurrency/atomic.html)
3. [StackOverflow: Java Singleton and Synchronization](http://stackoverflow.com/questions/11165852/java-singleton-and-synchronization)
4. [Linux Programmer's Manual: EPOLL(7)](http://man7.org/linux/man-pages/man7/epoll.7.html)