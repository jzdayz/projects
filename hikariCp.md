# HikariCP(3.4.5)
- 主要思路
1. 实现DataSource，Connection等jdbc的接口
2. 将connection的close方法，代理为，归还连接到连接池
3. 管理数据库连接池

- 一些额外话
像如果在操作statement的时候，执行了一些sql，但是又不是自动提交，并且代码也没有操作提交，在close连接的时候，会rollback这些操作

## 类与接口

- 其实对于数据库连接池最主要的就是管理数据库连接，所以hikaricp的核心类也就是ConcurrentBag

### ConcurrentBag

- 核心思想

**主要是做到尽量无锁化，获取连接尝试从之前用过的连接，和交换队列进行获取，最后再从所有的连接容器获取连接**

1. 使用了两个容器去存储连接，一个使用threadLocal(姑且叫T容器)，一个使用线程安全的CopyOnWriteArrayList(ALL容器)，然后还有一个交换的容器(SynchronousQueue，不同线程交换数据)
2. T容器存储所有的连接，ALL容器存储每个线程使用过的连接
- 新增连接
1. 每次进行新增连接时，先加入到ALL容器中
2. 如果发现当前有线程在等待连接时，尝试将连接扔到交换队列，如果失败，尝试让出cpu时间片，一直重试
- 获取连接
1. 获取连接时，尝试从之前用过的容器(T容器)中获取未在使用中的连接，并且删除已经尝试过的连接(T容器)
2. 如果从T容器中没获取到连接，则从ALL容器中尝试获取连接，并将本线程加入到`等待者队列`
3. 如果从ALL容器中没有获取连接，则尝试建立一个连接(如果未达到连接池允许的连接数上限)
4. 最后，如果还是没有获取到连接，则尝试从交换队列中获取连接，如果达到超时时间，则抛出异常
- 归还连接
1. 首先设置连接状态为`未使用`
2. 如果发现有等待者，则尝试将连接塞入到交换队列
3. 最后加入连接到T容器(使用过的连接，会有上限，不可以超过50个连接)


- 下面是我加了一些注释的代码
```java

/**
 * This is a specialized concurrent bag that achieves superior performance
 * to LinkedBlockingQueue and LinkedTransferQueue for the purposes of a
 * connection pool.  It uses ThreadLocal storage when possible to avoid
 * locks, but resorts to scanning a common collection if there are no
 * available items in the ThreadLocal list.  Not-in-use items in the
 * ThreadLocal lists can be "stolen" when the borrowing thread has none
 * of its own.  It is a "lock-less" implementation using a specialized
 * AbstractQueuedLongSynchronizer to manage cross-thread signaling.
 *
 * Note that items that are "borrowed" from the bag are not actually
 * removed from any collection, so garbage collection will not occur
 * even if the reference is abandoned.  Thus care must be taken to
 * "requite" borrowed objects otherwise a memory leak will result.  Only
 * the "remove" method can completely remove an object from the bag.
 *
 *
 *
 *   这个类是hikariCp的主要容器，利用了 threadLocal 减少了锁的争抢，同时利用
 *   jdk SynchronousQueue ，直接在归还或者借用中，使用交换的方式( A借用的同时，如果
 *   B归还，那么A可以直接获取到这个连接)
 *
 *   上述方式比直接在池中使用锁会好很多
 *
 *   在这个类中，主要是发挥无锁化，尽量做到无锁化，获取最大性能
 *
 * note:
 *   这个类是一个容器，我在注释中，将容器的条目都视为"connection"，这个类也可以装其他
 *   IConcurrentBagEntry的子类
 *
 * @author Brett Wooldridge
 *
 * @param <T> the templated type to store in the bag
 */
public class ConcurrentBag<T extends IConcurrentBagEntry> implements AutoCloseable
{
   private static final Logger LOGGER = LoggerFactory.getLogger(ConcurrentBag.class);

   private final CopyOnWriteArrayList<T> sharedList;
   private final boolean weakThreadLocals;

   private final ThreadLocal<List<Object>> threadList;
   private final IBagStateListener listener;
   private final AtomicInteger waiters;
   private volatile boolean closed;

   private final SynchronousQueue<T> handoffQueue;

   public interface IConcurrentBagEntry
   {
      int STATE_NOT_IN_USE = 0;
      int STATE_IN_USE = 1;
      int STATE_REMOVED = -1;
      int STATE_RESERVED = -2;

      boolean compareAndSet(int expectState, int newState);
      void setState(int newState);
      int getState();
   }

   public interface IBagStateListener
   {
      void addBagItem(int waiting);
   }

   /**
    * Construct a ConcurrentBag with the specified listener.
    *
    * @param listener the IBagStateListener to attach to this bag
    */
   public ConcurrentBag(final IBagStateListener listener)
   {
      this.listener = listener;
      // 使用弱引用
      this.weakThreadLocals = useWeakThreadLocals();

      // 这个是juc的一个api，在hikariCp中主要使用了#poll() 和 #offer()
      // 起到的作用，主要是不同线程的连接置换，比如 在com.zaxxer.hikari.util.ConcurrentBag#requite(归还)方法中
      // 如果有线程在等待获取，那么首先应该尝试将队列扔在交换队列中
      this.handoffQueue = new SynchronousQueue<>(true);
      this.waiters = new AtomicInteger();
      this.sharedList = new CopyOnWriteArrayList<>();
      if (weakThreadLocals) {
         this.threadList = ThreadLocal.withInitial(() -> new ArrayList<>(16));
      }
      else {
         this.threadList = ThreadLocal.withInitial(() -> new FastList<>(IConcurrentBagEntry.class, 16));
      }
   }

   /**
    * The method will borrow a BagEntry from the bag, blocking for the
    * specified timeout if none are available.
    *
    * @param timeout how long to wait before giving up, in units of unit
    * @param timeUnit a <code>TimeUnit</code> determining how to interpret the timeout parameter
    * @return a borrowed instance from the bag or null if a timeout occurs
    * @throws InterruptedException if interrupted while waiting
    */
   public T borrow(long timeout, final TimeUnit timeUnit) throws InterruptedException
   {
      // Try the thread-local list first
      // 首先从当前线程使用过的connection中获取，connection
      final List<Object> list = threadList.get();
      // 从尾部开始
      // 原因有2：
      // 1 : 从最近的开始循环
      // 2 : 移除连接不需要移动list中的数组
      for (int i = list.size() - 1; i >= 0; i--) {
         final Object entry = list.remove(i);
         @SuppressWarnings("unchecked")
         final T bagEntry = weakThreadLocals ? ((WeakReference<T>) entry).get() : (T) entry;
         if (bagEntry != null && bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
            // 如果使用过的connection中，可以获取到，并且可以cas修改掉状态，那么，直接返回。
            return bagEntry;
         }
      }

      // Otherwise, scan the shared list ... then poll the handoff queue
      // 视自己为等待者
      final int waiting = waiters.incrementAndGet();
      try {
         // 循环所有线程的共用list，增加也会直接加在这个list中
         for (T bagEntry : sharedList) {
            // 可以使用
            if (bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
               // If we may have stolen another waiter's connection, request another bag add.
               if (waiting > 1) {
                  // 发起一个添加连接的请求(会根据条件判断是否成功)
                  listener.addBagItem(waiting - 1);
               }
               return bagEntry;
            }
         }
         // 发起一个添加连接的请求(会根据条件判断是否成功)
         listener.addBagItem(waiting);
         // 转化纳秒
         timeout = timeUnit.toNanos(timeout);
         do {
            final long start = currentTime();
            // 队列中等待指定的时间，如果有别的线程，在队列中添加，那么可以获取到
            final T bagEntry = handoffQueue.poll(timeout, NANOSECONDS);
            if (bagEntry == null || bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_IN_USE)) {
               // 返回空或者可以使用的连接
               return bagEntry;
            }
            // timeout = now - 逝去的时间
            timeout -= elapsedNanos(start);
            // 大于10000纳秒，继续循环
         } while (timeout > 10_000);
         // 返回空
         return null;
      }
      finally {
         // 减少等待者
         waiters.decrementAndGet();
      }
   }

   /**
    * This method will return a borrowed object to the bag.  Objects
    * that are borrowed from the bag but never "requited" will result
    * in a memory leak.
    *
    *  归还连接
    *
    * @param bagEntry the value to return to the bag
    * @throws NullPointerException if value is null
    * @throws IllegalStateException if the bagEntry was not borrowed from the bag
    */
   public void requite(final T bagEntry)
   {
      // 未使用状态
      bagEntry.setState(STATE_NOT_IN_USE);

      // 循环等待者的数量的次数
      for (int i = 0; waiters.get() > 0; i++) {
         // 归还的对象，被设置为正在使用了，也就是说，别的线程获取到了，并且"拿"走了 || 连接可以传递给另外的等待者
         if (bagEntry.getState() != STATE_NOT_IN_USE || handoffQueue.offer(bagEntry)) {
            return;
         }
         // 255 或者 255的倍数
         else if ((i & 0xff) == 0xff) {
            // 阻塞10微秒
            parkNanos(MICROSECONDS.toNanos(10));
         }
         else {
            // 尽量让出cpu时间片
            Thread.yield();
         }
      }

      final List<Object> threadLocalList = threadList.get();
      // 当前线程集合小于50
      if (threadLocalList.size() < 50) {
         threadLocalList.add(weakThreadLocals ? new WeakReference<>(bagEntry) : bagEntry);
      }
   }

   /**
    * Add a new object to the bag for others to borrow.
    *
    * @param bagEntry an object to add to the bag
    */
   public void add(final T bagEntry)
   {
      if (closed) {
         LOGGER.info("ConcurrentBag has been closed, ignoring add()");
         throw new IllegalStateException("ConcurrentBag has been closed, ignoring add()");
      }

      // 加入集合
      sharedList.add(bagEntry);

      // spin until a thread takes it or none are waiting
      // 有线程等待 && 添加的连接是未使用状态 && 队列交换失败
      while (waiters.get() > 0 && bagEntry.getState() == STATE_NOT_IN_USE && !handoffQueue.offer(bagEntry)) {
         Thread.yield();
      }
   }

   /**
    * Remove a value from the bag.  This method should only be called
    * with objects obtained by <code>borrow(long, TimeUnit)</code> or <code>reserve(T)</code>
    *
    * @param bagEntry the value to remove
    * @return true if the entry was removed, false otherwise
    * @throws IllegalStateException if an attempt is made to remove an object
    *         from the bag that was not borrowed or reserved first
    */
   public boolean remove(final T bagEntry)
   {
      // 如下三个操作都失败了，那么就返回false
      // status: in_user -> removed
      // status: reserved -> removed
      // 容器已经关闭
      if (!bagEntry.compareAndSet(STATE_IN_USE, STATE_REMOVED) && !bagEntry.compareAndSet(STATE_RESERVED, STATE_REMOVED) && !closed) {
         LOGGER.warn("Attempt to remove an object from the bag that was not borrowed or reserved: {}", bagEntry);
         return false;
      }

      // 容器中移除
      final boolean removed = sharedList.remove(bagEntry);
      if (!removed && !closed) {
         LOGGER.warn("Attempt to remove an object from the bag that does not exist: {}", bagEntry);
      }

      threadList.get().remove(bagEntry);

      return removed;
   }

   /**
    * Close the bag to further adds.
    */
   @Override
   public void close()
   {
      closed = true;
   }

   /**
    * This method provides a "snapshot" in time of the BagEntry
    * items in the bag in the specified state.  It does not "lock"
    * or reserve items in any way.  Call <code>reserve(T)</code>
    * on items in list before performing any action on them.
    *
    * @param state one of the {@link IConcurrentBagEntry} states
    * @return a possibly empty list of objects having the state specified
    */
   public List<T> values(final int state)
   {
      final List<T> list = sharedList.stream().filter(e -> e.getState() == state).collect(Collectors.toList());
      Collections.reverse(list);
      return list;
   }

   /**
    * This method provides a "snapshot" in time of the bag items.  It
    * does not "lock" or reserve items in any way.  Call <code>reserve(T)</code>
    * on items in the list, or understand the concurrency implications of
    * modifying items, before performing any action on them.
    *
    * @return a possibly empty list of (all) bag items
    */
   @SuppressWarnings("unchecked")
   public List<T> values()
   {
      return (List<T>) sharedList.clone();
   }

   /**
    * The method is used to make an item in the bag "unavailable" for
    * borrowing.  It is primarily used when wanting to operate on items
    * returned by the <code>values(int)</code> method.  Items that are
    * reserved can be removed from the bag via <code>remove(T)</code>
    * without the need to unreserve them.  Items that are not removed
    * from the bag can be make available for borrowing again by calling
    * the <code>unreserve(T)</code> method.
    *
    * @param bagEntry the item to reserve
    * @return true if the item was able to be reserved, false otherwise
    *
    *  切换状态至保留(不可用)
    */
   public boolean reserve(final T bagEntry)
   {
      return bagEntry.compareAndSet(STATE_NOT_IN_USE, STATE_RESERVED);
   }

   /**
    * This method is used to make an item reserved via <code>reserve(T)</code>
    * available again for borrowing.
    *
    * @param bagEntry the item to unreserve
    *
    */
   @SuppressWarnings("SpellCheckingInspection")
   public void unreserve(final T bagEntry)
   {
      // 置换状态
      if (bagEntry.compareAndSet(STATE_RESERVED, STATE_NOT_IN_USE)) {
         // spin until a thread takes it or none are waiting
         // 有等待线程 && 交换失败
         while (waiters.get() > 0 && !handoffQueue.offer(bagEntry)) {
            // 尽量让出cpu时间片
            Thread.yield();
         }
      }
      else {
         LOGGER.warn("Attempt to relinquish an object to the bag that was not reserved: {}", bagEntry);
      }
   }

   /**
    * Get the number of threads pending (waiting) for an item from the
    * bag to become available.
    *
    * @return the number of threads waiting for items from the bag
    */
   public int getWaitingThreadCount()
   {
      return waiters.get();
   }

   /**
    * Get a count of the number of items in the specified state at the time of this call.
    *
    * @param state the state of the items to count
    * @return a count of how many items in the bag are in the specified state
    */
   public int getCount(final int state)
   {
      int count = 0;
      for (IConcurrentBagEntry e : sharedList) {
         if (e.getState() == state) {
            count++;
         }
      }
      return count;
   }

   public int[] getStateCounts()
   {
      final int[] states = new int[6];
      for (IConcurrentBagEntry e : sharedList) {
         ++states[e.getState()];
      }
      states[4] = sharedList.size();
      states[5] = waiters.get();

      return states;
   }

   /**
    * Get the total number of items in the bag.
    *
    * @return the number of items in the bag
    */
   public int size()
   {
      return sharedList.size();
   }

   public void dumpState()
   {
      sharedList.forEach(entry -> LOGGER.info(entry.toString()));
   }

   /**
    * Determine whether to use WeakReferences based on whether there is a
    * custom ClassLoader implementation sitting between this class and the
    * System ClassLoader.
    *
    * @return true if we should use WeakReferences in our ThreadLocals, false otherwise
    *  弱引用，容器中的对象
    */
   private boolean useWeakThreadLocals()
   {
      try {
         if (System.getProperty("com.zaxxer.hikari.useWeakReferences") != null) {   // undocumented manual override of WeakReference behavior
            return Boolean.getBoolean("com.zaxxer.hikari.useWeakReferences");
         }

         return getClass().getClassLoader() != ClassLoader.getSystemClassLoader();
      }
      catch (SecurityException se) {
         return true;
      }
   }
}

```
