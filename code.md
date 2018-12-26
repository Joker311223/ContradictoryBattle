# 程序


## 1.多继承的实现(通过内部类)
```java
class Call {
	public void callSomebody(String phoneNum){
		System.out.println("我在打电话喔，呼叫的号码是：" + phoneNum);
	}
}

class SendMessage {
	public void sendToSomebody(String phoneNum){
		System.out.println("我在发短信喔，发送给 ：" + phoneNum);
	}
}

public class Phone {
	private class MyCall extends Call{

	}
	private class MySendMessage extends SendMessage{

	}

	private MyCall call = new MyCall();
	private MySendMessage send = new MySendMessage();

	public void phoneCall(String phoneNum){
		call.callSomebody(phoneNum);
	}

	public void phoneSend(String phoneNum){
		send.sendToSomebody(phoneNum);
	}

	public static void main(String[] args) {
		Phone phone = new Phone();
		phone.phoneCall("110");
		phone.phoneSend("119");
	}
}
```

## 2.单例模式
### 2.1 (饿汉式)
```java
public class Singleton {
    // 私有构造函数
    private Singleton() {  }

    // 单例对象
    private static Singleton instance = new Singleton();

    // 静态的工厂方法
    public static Singleton getInstance() {
        return instance;
    }
}
```

### 2.2（懒汉式  双重同步锁单例模式）
```java
public class Singleton {
    // 私有构造函数
    private Singleton() {

    }

    // 1、memory = allocate() 分配对象的内存空间
    // 2、ctorInstance() 初始化对象
    // 3、instance = memory 设置instance指向刚分配的内存

    // 单例对象 volatile + 双重检测机制 -> 禁止指令重排
    private volatile static Singleton instance = null;

    // 静态的工厂方法
    public static Singleton getInstance() {
        if (instance == null) { // 双重检测机制        // B
            synchronized (Singleton.class) { // 同步锁
                if (instance == null) {
                    instance = new Singleton(); // A - 3
                }
            }
        }
        return instance;
    }
}
```

### 2.3（枚举）
```java
public class Singleton {
    // 私有构造函数
    private Singleton() { }

    public static Singleton getInstance() {
        return Singleton.INSTANCE.getInstance();
    }

    private enum Singleton {
        INSTANCE;

        private Singleton singleton;

        // JVM保证这个方法绝对只调用一次
        Singleton() {
            singleton = new Singleton();
        }

        public Singleton getInstance() {
            return singleton;
        }
    }
}
```

## 3. 死锁
```java
public class DeadLock {
	private static String a = "aaa";
	private static String b = "bbb";

	public static void main(String[] args) throws Exception {
		Thread threadA = new Thread(() -> {
			synchronized (DeadLock.a) {
				System.out.println("已获得a锁,需要b锁");
				for (int i = 0; i < 100; i++) {
					System.out.println("threadA -----" + i);
				}
				synchronized (DeadLock.b) {
					System.out.println("已获得a,b锁");
				}
			}
		});

		Thread threadB = new Thread(() -> {
			synchronized (DeadLock.b) {
				System.out.println("已获得b锁,需要a锁");
				for (int i = 0; i < 100; i++) {

					System.out.println("threadB ----------" + i);
				}
				synchronized (DeadLock.a) {
					System.out.println("已获得a,b锁");
				}
			}
		});

		threadA.start();
		threadB.start();
	}
}
```

## 4.生产者消费者
**需注意的点：**
**1.produce()和consume()方法中的判断需要在while中，不能在if中（可能造成数据异常）**
**2.唤醒时不能只唤醒一个线程（可能造成所有线程等待，程序不能进行）**

### 4.1（wait-notify）
```java
class Resource {
	private int i;
	private int size;
	private LinkedList<Integer> list;

	public Resource(int size) {
		this.list = new LinkedList<Integer>();
		this.size = size;
		this.i = 0;
	}

	public synchronized void produce() {
		try {
			while (list.size() == size) {
				System.out.println("资源达到最大值，生产者：	" + Thread.currentThread().getName() + "开始wait");
				wait();
				System.out.println("生产者：	" + Thread.currentThread().getName() + "退出wait");
			}
			list.addLast(new Integer(++i));
			System.out.println("生产者" + Thread.currentThread().getName() + " 生产数据" + i);
			notifyAll();
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}

	public synchronized void consume() {
		try {
			while (list.isEmpty()) {
				System.out.println("资源为空，消费者：	" + Thread.currentThread().getName() + "开始wait");
				wait();
				System.out.println("消费者：	 " + Thread.currentThread().getName() + "退出wait");
			}
			Integer num = list.removeFirst();
			System.out.println("消费者" + Thread.currentThread().getName() + " 消费数据" + num);
			notifyAll();
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}

class Consumer implements Runnable {
	Resource resource;

	Consumer(Resource resource) {
		this.resource = resource;
	}

	public void run() {
		while (true) {
			resource.consume();
		}
	}
}

class Producer implements Runnable {
	Resource resource;

	Producer(Resource resource) {
		this.resource = resource;
	}

	public void run() {
		while (true) {
			resource.produce();
		}
	}
}

public class ConProWaitNotify {
	public static void main(String[] args) throws Exception {
		int size = 5;
		Resource resource = new Resource(size);

		Producer producer = new Producer(resource);
		Consumer consumer = new Consumer(resource);

		new Thread(producer, "producerA").start();
		new Thread(producer, "producerB").start();

		new Thread(consumer, "consumerA").start();
		new Thread(consumer, "consumerB").start();
		new Thread(consumer, "consumerC").start();
```
### 4.2（Lock-Condition）
```java
import java.util.LinkedList;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

class Resources {
	private int i;
	private int size;
	private LinkedList<Integer> list;

	private ReentrantLock lock = new ReentrantLock(true); // 设置为公平锁
	Condition conCondition = lock.newCondition();
	Condition proCondition = lock.newCondition();

	public Resources(int size) {
		this.list = new LinkedList<Integer>();
		this.size = size;
		this.i = 0;
	}

	public void produce() {
		lock.lock();
		try {
			while (list.size() == size) {
				System.out.println("资源达到最大值，生产者：	" + Thread.currentThread().getName() + "开始wait");
				proCondition.await();
				System.out.println("生产者：	" + Thread.currentThread().getName() + "退出wait");
			}
			list.addLast(new Integer(++i));
			System.out.println("生产者" + Thread.currentThread().getName() + " 生产数据" + i);
			conCondition.signal();
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			lock.unlock();
		}
	}

	public void consume() {
		lock.lock();
		try {
			while (list.isEmpty()) {
				System.out.println("资源为空，消费者：	" + Thread.currentThread().getName() + "开始wait");
				conCondition.await();
				System.out.println("消费者：	 " + Thread.currentThread().getName() + "退出wait");
			}
			Integer num = list.removeFirst();
			System.out.println("消费者" + Thread.currentThread().getName() + " 消费数据" + num);
			proCondition.signal();
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			lock.unlock();
		}
	}
}

class ConsumerTwo implements Runnable {
	Resources resources;

	ConsumerTwo(Resources resources) {
		this.resources = resources;
	}

	public void run() {
		while (true) {
			resources.consume();
		}
	}
}

class ProducerTwo implements Runnable {
	Resources resources;

	ProducerTwo(Resources resources) {
		this.resources = resources;
	}

	public void run() {
		while (true) {
			resources.produce();
		}
	}
}

public class ConProLockCondition {
	public static void main(String[] args) throws Exception {
		int size = 5;
		Resources resources = new Resources(size);

		ProducerTwo poducer = new ProducerTwo(resources);
		ConsumerTwo consumer = new ConsumerTwo(resources);

		new Thread(poducer, "producerA").start();
		new Thread(poducer, "producerB").start();

		new Thread(consumer, "consumerA").start();
		new Thread(consumer, "consumerB").start();
		new Thread(consumer, "consumerC").start();

	}
}
```

### 4.3（BlockingQueue）
```java
import java.util.concurrent.ArrayBlockingQueue;

class BlockingQueueResources {
	private int i; // 用来计数
	private ArrayBlockingQueue<Integer> queue;

	public BlockingQueueResources(int size) {
		this.i = 0;
		this.queue = new ArrayBlockingQueue<Integer>(size);
	}

	public void produce() {
		try {
			queue.put(new Integer(++i));
			System.out.println("生产者" + Thread.currentThread().getName() + " 生产数据" + i);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}

	public void consume() {
		try {
			Integer num = queue.take();
			System.out.println("消费者" + Thread.currentThread().getName() + " 消费数据" + num);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}

class ConsumerThree implements Runnable {
	BlockingQueueResources resources;

	ConsumerThree(BlockingQueueResources resources) {
		this.resources = resources;
	}

	public void run() {
		while (true) {
			resources.consume();
		}
	}
}

class ProducerThree implements Runnable {
	BlockingQueueResources resources;

	ProducerThree(BlockingQueueResources resources) {
		this.resources = resources;
	}

	public void run() {
		while (true) {
			resources.produce();
		}
	}
}

public class ConProLockBlockingQueue {
	public static void main(String[] args) throws Exception {
		int size = 5;
		BlockingQueueResources resources = new BlockingQueueResources(size);

		ProducerThree poducer = new ProducerThree(resources);
		ConsumerThree consumer = new ConsumerThree(resources);

		new Thread(poducer, "producerA").start();
		new Thread(poducer, "producerB").start();

		new Thread(consumer, "consumerA").start();
		new Thread(consumer, "consumerB").start();
		new Thread(consumer, "consumerC").start();

	}
}
```





