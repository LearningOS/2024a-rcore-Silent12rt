## 简单总结你实现的功能（200字以内，不要贴代码）
我实现的功能包括对进程和线程的资源管理，主要是通过互斥量（mutex）和信号量（semaphore）来控制并发访问。具体而言，为每个线程维护了四个关键数据结构：m_allocation 和 s_allocation 用于记录已分配的互斥量和信号量的数量，m_need 和 s_need 则用于跟踪线程尚需的资源数量。在系统调用中，加入了对死锁的检测逻辑，确保在尝试获取新资源之前检查当前资源的可用性。如果发现潜在的死锁情况，则会拒绝资源请求，并返回相应的错误代码。此外，还实现了调整资源需求的方法，以便在资源分配和释放时动态更新状态。


## 简答作业
1. 在我们的多线程实现中，当主线程 (即 0 号线程) 退出时，视为整个进程退出， 此时需要结束该进程管理的所有线程并回收其资源。 - 需要回收的资源有哪些？ 其他线程的 TaskControlBlock 可能在哪些位置被引用，分别是否需要回收，为什么？
	- 内存资源、线程控制块（TaskControlBlock）、同步原语、文件描述符
	- 引用位置：进程控制块（ProcessControlBlock）、调度器、线程间通信。需要回收，所有其他线程的TCB都需要回收，因为它们不再有效，并且继续存在可能导致资源泄漏、无效访问和潜在的死锁问题。


2. 对比以下两种 Mutex.unlock 的实现，二者有什么区别？这些区别可能会导致什么问题？
```rust
	impl Mutex for Mutex1 {
	    fn unlock(&self) {
	        let mut mutex_inner = self.inner.exclusive_access();
	        assert!(mutex_inner.locked);
	        mutex_inner.locked = false;
	        if let Some(waking_task) = mutex_inner.wait_queue.pop_front() {
	            add_task(waking_task);
	         }
	    }
	}
	
	impl Mutex for Mutex2 {
	    fn unlock(&self) {
	        let mut mutex_inner = self.inner.exclusive_access();
	        assert!(mutex_inner.locked);
	        if let Some(waking_task) = mutex_inner.wait_queue.pop_front() {
	            add_task(waking_task);
	        } else {
	            mutex_inner.locked = false;
	        }
	    }
	}
```
区别：在 Mutex1 中，锁状态在唤醒任何任务之前被设置为 false。在 Mutex2 中，只有在没有等待任务的情况下才会设置锁状态为 false。
Mutex21可能导致某个任务在未完全解锁的情况下被唤醒，造成它在进入临界区时发现锁已被释放。这种情况下，唤醒的任务可能会立即获取锁，从而打破了锁的保护机制。Mutex2 由于在唤醒任务之前检查等待队列，只有在没有任务等待的情况下才会设置锁状态为 false。这可以避免不必要的竞态条件，因为它确保了在有任务等待的情况下锁不会被意外释放。

可能导致的问题：竞态条件、死锁风险。