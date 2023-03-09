# 스레드와 잠금장치
스레드와 잠금장치는 마치 포드 모델 T와 같다. 굴러는 가지만 원시적이고 어려운 운전이 될 것이며, 새로운 기술과 비교하면 안전성도 떨어지고 위험하다.
그럼에도 불구하고 많은 동시성 소프트웨어의 기본 선택지이다. 또한 앞으로 소개할 여러모델의 근간이 되기도 한다.

### 동작하는 가장 단순한 코드

하드웨어가 작동하는 방식을 그대로 옮긴 것과 다름이 없다. 너무 단순해서 별다른 제약도 없다. 실력이 없는 프로그래머가 쓰기엔 그만큼 위험한 방식이다. 여기서 설명하는 원시적 패턴은 실제 서비스 사례에서는 볼일이 없지만, 상위 수준의 서비스들이 어떻게 작동하는지 이해하기 위한 기반이 된다.

### 상호배제와 메모리 모델

상호배제(Mutual Exclusion) 라는 개념에 대해 들어봤을 것이다. 잠금장치에 접근하는 스레드가 한 번에 하나만 존재하도록 강제하는 방식이다.

**스레드 만들기**

```java
public class HelloWorld {
	public static vodi main() {
		Thread myThread = new Thread() {
			public void run () {
				System.out.println("Hello fron new Thread");
			}
		};
		
		myThread.start();
		Thread.yield();
		System.out.println("Hello from main Thread");
		myThread.join();
	}
}
```

위의 코드에서 join() 은 해당 스레드가 동작을 멈출 때까지 기다린다.

결과는 둘 중 하나일 것이다. 실행할 때마다 결과가 다를 수 있다.

```
Hello from main Thread
Hello from new Thread
```

```
Hello from new Thread
Hello from main Thread
```

> **Thread.yield 를 사용하는 이유는?**
yield()의 정의는 다음 과 같다.

- 현재 실행 중인 스레드가 사용 중인 프로세서를 양보할 용의가 있음을 스케줄러에 알려주는 힌트

이 줄을 포함하지 않으면 새로운 스레드를 생성하는데 따르는 오버헤드 때문에 언제나 메인 스레드의 println() 이 먼저 실행될 것이다.
> 

**첫번째 잠금 장치**

여러개의 스레드가 공유된 메모리에 접근할 때는 서로 동작이 엉킬 수 있다. 한 번에 하나의 스레드만 보유할 수 있는 잠금장치를 사용함으로써 이러한 상황을 피할 수 있다.

```java
public class Counting {
	public static vodi main() {
		class Counter {
			private int count = 0;
			public void increment() { ++count; }
			public int getCount() { return count; }
		}

		final Counter counter = new Counter();
		class CountingThread extends Thread {
			public void run() {
				for(int x = 0; x < 10000; ++i) {
					counter.increment();
				}
			}
		}
		CountingThread t1 = new CountingThread();
		CountingThread t2 = new CountingThread();
		t1.start();
		
		myThread.start();
		Thread.yield();
		System.out.println("Hello from main Thread");
		myThread.join();
	}
}
```

 위의 코드는 그냥 고장난 쓰레기 코드이다. 각 스레드가 increment()를 10,000번 호출하도록 만드려고 했지만, 매번 다른 결과를 얻게 된다. 이유는 Counter 내에 있는 count 값을 읽을 때 발생하는 경쟁 조건 때문이다. `++count` 라는 코드를 읽을 때 자바 컴파일러가 어떤 바이트 코드를 만드는지 확인하면 다음과 같다.

```
getfield #2
iconst_1
iadd
putfield #2
```

흐름은 ‘#2를 가져온다 → iadd 로 더한다 → #2에 덮어쓴다’ 이다. 단순히 생각하면, #2에 덮어쓰기 전에 두 스레드에서 iadd를 각각 호출했지만, 2가 증가된것이 아니라 1만 증가한 상태가 되어버리는 불행한 경우가 생긴다.

 이에 대한 해법으로는 count에 대한 접근을 **동기화**(syncronize) 하는 것이다. 이에는 여러 방법이 있는데 그 중 하나인 자바 객체에 포함된 내재된 잠금장치를 사용하는 것이다.

```java
class Counter {
	private int count = 0;
	public synchronized void increment() { ++count; }
	public int getCount() { return count; } 
}
```

물론 그렇다고 다 해결된 건 아니다. 수정된 코드에서 내포하는 문제를 다음에서 설명한다.

**메모리의 미스터리**
```java
public class Puzzle {
	static boolean answerReady = false;
	static int answer = 0;
	static Thread t1 = new Thread() {
		public void run() {
			answer = 42;
			answerReady = true;
		}
	};
	static Thread t2 = new Thread() {
		public void run() {
			if (answerReady)
				System.out.println("The meaning of life is:" + answer);
			else
				System.out.println("I don't know the answer");
		}
	};

	public static void main(String[] args) throws InterruptedException {
		t1.start(); t2.start();
		t1.join();t2.join();
	}
}
```

위의 코드를 실행시 다음과 같은 결과를 볼 수도 있다.

```
The meaning of life is: 0
```

이유가 뭘까?

**메모리 가시성**

 핵심은 읽는 스레드와 쓰는 스레드가 동기화 되지 않으면 가시성이 보장되지 않는다는 것이다.  앞에서 increment()만 동기화하는 것이 아니라 getCount() 도 동기화를 해야한다. 

**여러 개의 잠금장치**

 그러나 “메서드를 다 동기화하면 되는가?” 라는 해결책은 멍청한 방법이다. 효율성도 떨어지고 스레드 대부분은 블로킹 된 상태를 유지할 것이다. 또한 둘 이상의 스레드에서 공유 메모리를 사용하다 보면 필연적으로 데드락이 발생할 가능성이 높아진다. 

“식사하는 철학자” 모델을 생각해보자. 각 철학자는 양쪽의 젓가락을 들어올려서 식사를 한다. 철학자는 생각을 하거나 배고픔을 느낀다. 배고프면 양쪽의 젓가락을 집어 올리고 식사를 한다. 식사가 끝이나면 젓가락을 내려놓는다.

### 내재된 잠금 장치를 넘어서
내재된 잠금장치는 편리하지만 다음과 같은 한계를 가지고 있다.

- 블로킹에 빠진 스레드를 원상복귀할 방법이 없다.
- 타임아웃기능이 없다.
- synchronized 블록 또는 키워드 말고는 방법이 없다
- 그리고 일반적으로 synchronized 100배정도 느리다..

이번에는 ReentrantLock 을 사용하는 예시를 살펴보자

```java
Lock lock - new ReentrantLock();
lock.lock();
try {
	// <<공유되는 자원 사용>>
} finally {
	lock.unlock();
}
```

 내재된 잠금장치를 얻으려다 블로킹된 스레드는 중간에 중단할 수 밖에 없기 떄문에, 데드락이 발생하면 회복할 방법이 없다. 일부러 데드락을 만들고 해당 스레드의 블로킹 상태를 중단하는 코드를 통해서 이를 증명해보자

```java
public class Uninterruptible {
	public static void main(String[] args) throws InterruptedException {
		final Object o1 = new Object(); final Object o2 = new Object();

		Thread t1 = new Thread() {
			public void run() {
				try {
					synchronized(o1) {
						Thread.sleep(1000);
						synchronized(o2) {}
					}
				} catch (InterruptedException e) { System.out.println("t1 interrupted"); }
			}
		};

		Thread t2 = new Thread() {
			public void run() {
				try {
					synchronized(o2) {
						Thread.sleep(o1);
					} synchronized(o1) {}
				} catch (InterruptedException e) { System.out.println("t2 interrupted"); }
			}
		}
	}
}
```

 이 프로그램은 데드락을 영원히 유지한다. 상태를 벗어나는 유일한 방법은 JVM 전체를 중단하는 것이다.

**타임아웃**

ReentrantLock은 타임아웃을 지원한다. 가령 “철학자 문제”에서 제한된 시간안에 젓가락 두개를 획득하지 못하면 타임아웃을 발생시킨다.

```java
class Phillosopher extends Thread {
	private Reentrantlock leftChopstick, rightChopstick;
	private Random random;

	public Philosopher(ReentrantLock leftChopstick, ReentrantLock rightChopstick) {
		this.leftChopstick = leftChopstick;
		this.rightChopstick = rightChopstick;
		random = new Random();
	}

	public void run() {
		try {
			while(true) {
				Thread.sleep(random.nextInt(1000));
				leftChopstick.lock();
				if (rightChopstick.tryLock(1000, TimeUnit.MILLISECONDS) {
					try {
						Thread.sleep(random.nextInt(1000)); // 잠시 먹는다.
					} finally { rightChopstick.unlock(); }
				} else {
					// 오른쪽 젓가락의 락을 획득하지 못해 생각하는 상태로 돌아간다. 
				}
			} finally { leftChopstick.unlock(); }
		} catch(InterruptedException e) {}
	}
}
```

> **라이브락**
 tryLock 을 사용하면 데드락을 피할 수 있지만(데드락을 회피하는 게 아니라 탈출하는 방법을 제공한다), 라이브락 문제가 발생한다. 모든 스레드가 동시에 타임아웃을 발생하면 곧바로 데드락에 빠지는 것도 가능하다. 그렇게되면 데드락은 빠져나오겠지만, 실질적으로 스레드가 아무런 일도 하지 않는 상태와 같아진다.
 이러한 문제는 스레드가 서로 다른 타임아웃을 갖게 하는 방법으로 해결 할 수 있다. 그러나 근본적으로 타임아웃을 이용하는게 그닥 좋은 해법은 아닐 것이다.
> 

**협동 잠그기**

 여러 스레드가 연결리스트(Linked List)에 노드를 삽입하려는 경우를 생각해보자. 가장 쉬운 방법은 하나의 잠금장치가 리스트 전체를 보호하도록 만드는 것이다. 하지만 그렇게 하면 다른 스레드들은 접근할 수가 없다. 협동 잠그기는 리스트의 일부분만 잠그도록 한다. 

노드를 삽입하려면 원하는 지점 양쪽에 위치한 노드를 잠글 필요가 있다. 

1. 처음 두 개의 노드를 잠근다.
2. 여기가 아니라면 첫번째를 풀고 세번째를 잠근다.
3. 원하는 장소를 찾을 때까지 1, 2를 순서대로 계속 반복한다.
4. 장소를 찾으면 새로운 노드를 삽입하고 양쪽에 있는 노드 잠금을 푼다.

```java
class ConcurrentSortedList {
	private class Node {
		int value;
		Node prev;
		Node next;
		ReentrantLock lock = new ReentrantLock();
		
		Node() {}
		Node(int value, Node prev, Node next) {
			this.value = value; this.prev = prev; this.next = next;
		}
	}

	private final Node head;
	private final Node tail;

	public ConcurrentSortedList() {
		head = new Node(); tail = new Node();
		head.next = tail; tail.prev = head;
	}

	public void insert(int value) {
		Node current = head;
		current.lock.lock();
		Node next = current.next;
		try {
			while (true) {
				next.lock.lock();
				try {
					if (next == tail || next.value < value) {
						Node node = new Node(value, current, next);
						next.prev = node;
						current.next = node;
						return;
					}
				} finally { current.lock.unlock(); }
				current = next;
				next = current.next;
			}
		} finally { next.lock.unlock(); }
	}
}
```

 insert() 메서드는 새로운 값보다 작은 값을 담고 있는 노드를 찾을 때까지 검색을 수행하여 리스트가 항상 정렬되어 있음을 보장한다. 그런 노드를 찾으면 그 노드의 바로 앞에 새 노드를 삽입한다.

**조건변수**

 동시성 프로그래밍은 어떤 일이 벌어지는 것을 기다리는 상황도 포함한다. 예컨대 어떤 큐에 담긴 값을 제거하려는 경우에는 큐가 적어도 하나의 값을 가질때까지 기다리는 경우도 있다. 조건 변수는 이런 상황을 위해 고안된 기능이다. 다음의 패턴을 확인해보자.

```java
ReentrantLock lock = new ReentrantLock();
Condition condition = lock.newCondition();

lock.lock();
try {
	while (! // 조건이 참) {
		condition.await();
		// 공유되는 자원 사용
	}
} finally { lock.lock(); }
```

 조건이 참이 아니라면 await()을 호출한다. 그렇게 하면 가지고 있던 잠금아치를 반납하고 해당 조건이 참이 될 때까지 블로킹되는 작업을 원자적(atomically)으로 수행한다. 어떤 동작이 원자적으로 수행한다는 것은 다른 스레드 관점에서 보았을 때, 해당 동작이 완전히 수행되거나 아니면 전혀 수행되지 않는 것을 의미한다. 

 조건이 참이 되었는지에 대한 신호를 보내기 위해 다른 스레드가 signal()이나 signalAll()을 호출하면, await()은 블로킹을 멈추고 자동으로 원래 잠금장치를 다시 획득한다. 

“식사하는 철학자” 문제에 대한 또 다른 해법을 제공한다.

```java
class Philosopher extends Thread {
	private boolean eating;
	private Philosopher left;
	private Philosopher right;
	private ReentrantLock table;
	private Condition condition;
	private Random random;
	public Philosopher(ReentrantLock table) {
		eating = false;
		this.table = table;
		condition = table.newCondition();
		random = new Random();
	}

	public void setLeft(Philosopher left) { this.left = left; }
	public void setRight(Philosopher right) { this.right = right; }
	
	public void run() {
		try {
			while (true) {
				think();
				eat();	
			}
		}
	}

	public void think() throws InterruptedException {
		table.lock();
		try {
			eating = false;
			left.condition.signal();
			right.condition.signal();
		} finally { table.unlock(); }
		Thread.sleep(1000);
	}

	public void eat() throws InterruptedException {
		table.lock();
		try {
			while (left.eating || right.eating) {
				condition.await();
			}
			eating = true;
		} finally { table.unlock(); }
		Thread.sleep(1000);
	}
}
```

 앞에서 보였던 코드와 다르게 다소 복잡해지기는 했지만 이 해법이 훨씬 낫다. 앞에서는 한 번에 한 철학자만 식사를 할 수 있었고, 나머지는 젓가락을 한 개만 들고 나머지 한 개를 사용할 수 있을 때까지 기다려야만 했다. 이 해법에서는 조건이 충족되면 언제든지 식사를 시작할 수 있다.

**원자 변수**

 이전에는 increment() 메서드를 동기화하는 방식으로 해결했다. atomic 패키지는 더 나은 방법을 제공한다.

```java
public class Counting {
	public static void main(String[] args) throws InterruptedException {
		final AtomicInteger counter = new AtomicInteger();
		
		class CountingThread extends Thread {
			public void run() {
				for (int x = 0; x < 10000; ++x) {
					counter.incrementAndGet();
				}
			}
		}
		
		CountingThread t1 = new CountingThread();
		CountingThread t2 = new CountingThread();

		t1.start(); t2.start();
		t1.join(); t2.join();	
		System.out.println(counter.get());
	}
}
```

 AtomicInteger의 incrementAndGet() 메서드는 기능적으로 ++count 와 동일하다. 다른 점은 ++count 와 달리 원자적이다. 원자변수를 사용하면 잠금잠치의 획득과 해제를 신경쓸 필요가 없다. 그리고 잠금장치가 개입되지 않아서 원자변수에 대해 데드락이 걸리는 일도 불가능하다.
 
 ### 거인의 어깨위에서

 스레드는 비싸다. 이말은 즉 스레드의 수가 늘어나면 늘어날 수록 잡아먹는 리소스가 늘어날 것이고, 비효율을 발생시킨다. 그래서 고안된 것이 바로 **“스레드 풀”** 이다. 웹 서버를 생각해보면, 외부로 들어오는 요청이 하나 늘어날 때 마다 똑같이 스레드를 계속해서 만들다 보면, 과도한 요청이 들어오거나 디도스 공격 같은일이 발생하면 서버가 금방 죽어버릴 정도로 매우 취약하다는 것이다.

 다음과 같이 스레드 풀을 쓰는 예제를 보자.

```java
int threadPoolSize = Runtime.getRuntime()
```

스레드 풀을 사용하면 문제가 해결된다.

```java
int threadPoolSize = Runtime.getRuntime().availableProcessors() * 2;
ExecutorService executor = Executors.newFixedThreadPool(threadPoolSize);
while (true) {
	Socket socket = server.accept();
	executor.execute(new ConnectionHandler(socket));
}
```

**카피온라이트**

이전에 우리는 동시성 프로그램 내부에서 청취자를 안전한게 호출하는 방법을 살펴보았다. updateProgress()를 수정해서 방어적인 카피를 만들도록 했다. 자바 표준 라이브러리는 이보다 더 깔끔하고 미리 준비된 해법인 CopyOnWriteArrayList를 제공한다.

> **스레드 풀의 크기는 어느 정도로 만들어야 하나요?**
스레드가 I/O 바운드 한가 아니면 CPU 바운드한가와 같은 여러 요소에 영향을 받는다. 전자의 경우 코어의 수와 동일한 스레드를 풀에 담는 것이 일반적이고, 후자의 경우 코어보다 많은 수를 사용하는 편이 낫다.
> 

**완전한 프로그램**

실질적인 문제를 하나 풀어보자. 위키피디아에서 가장 흔히 사용되는 단어는 무엇일까? XML을 일단 내려받고 하나씩 다 읽어서 워드 카운팅을 한다. 문제는 데이터 크기가 40기가 바이트 정도여서 오래걸린다. 병렬화를 통해 속도를 높여보자. 일단은 순차적으로 구현한다.

```java
public class WordCount {
	private static final HashMap<String, Integer> counts = new HashMap<String, Integer>();
	
	public static void main(String[] args) throws Exception {
		Iterable<Page> pages = new Pages(10000, "enwiki.xml");
		for (Page page: pages) {
			Iterable<String> words = new Words(page.getText());
			for (String word: words)
				countWord(word);
		}
	}

	private static void countWord(String word) {
		Integer currentCount = counts.get(word);
		if (currentCount == null)
			counts.put(word, 1);
		else
			counts.put(word, currentCount + 1);
	}
}
```

병렬화된 버전에서 생각해보자. 메인 루프에서는 두 가지 업무를 수행한다. 우선 XML을 해석해 Page를 하나 만들고, 그 다음 그 안에 담긴 단어를 센다. 

이를 해결할 고전적인 패턴이 있는데 바로, ‘생산자-소비자’ 패턴이다. 하나의 스레드는 값을 생성하고, 또 다른 스레드는 값을 소비하는 작업을 한다. 다음은 생산자 스레드이다.

```java
class Parser implements Runnable {
	private BlockingQueue<Page> queue;

	public Parser(BlockingQueue<Page> queue) {
		this.queue = queue;
	}

	public void run() {
		try {
			
		} catch (Exception e) { e.printStackTrace(); }
	}
}
```

다음은 소비자 스레드이다.

```java
class Counter implements Runnable {
	private BlockingQueue<Page> queue;
	private Map<String, Integer> counts;
	public Counter(BlockingQueue<Page> queue, Map<String, Integer> counts) {
		this.queue = queue;
		this.counts = counts;
	}

	public void run() {
		
	}
}
```

 큐를 이용해서 가져오는 방식이다. 두 개 스레드를 생성하는 메인 루프는 다음과 같다.

```java
ArrayBlockingQueue<Page> queue = new ArrayBlockingQueue<Page>(100);
HashMap<String, Integer> counts = new HashMap<String, Integer>();

Thread counter = new Thread(new Counter(queue, counts));
Thread parser = new Thread(new Parser(queue));

counter.start();
parser.start();
parser.join();
queue.put(new PosionPill());
counter.join();
```

ArrayBlockingQueue 는 일반적으로 이러한 생산자-소비자 패턴을 구현할 때 쓰이는 자료구조이다. 

> **왜 블로킹 큐를 이용하는가?**
 블로킹되지 않는 ConcurrentLinkedQueue 도 있다. 여기서 사용하지 않은 이유는, 생산자와 소비자의 속도를 맞추기 위해서다. 생산자가 소비자보다 빠르게 동작하면, 큐가 금방 차는데, 그렇게 되면 큐의 메모리 용량이 너무 커지는 문제가 있다.
> 

소비자를 조금더 병렬화 해서 해석속도를 높이는 방법이 있다. 이에 대한 해법으로 counts 맵에 대한 접근을 동기화하는 방법이 필요하다. 어떻게 동기화 할 수 있을까? 첫번째로는 synchronizedMap() 메서드를 호출하면 얻을 수 있는 동기화된 맵을 사용하는 방법이 있다. 그러나  수정하기와 쓰기는 원자적으로 이뤄지지 않기 때문에 별 도움이 안 된다. 따라서 접근 동기화를 직접 구현해야 한다.

```java
private void countWord(String word) {
	lock.lock();
	try {
		Integer currentCount = counts.get(word);
		if (currentCount == null)
			counts.put(word, 1);
		else
			counts.put(word, currentCount + 1);
	}  finally { lock.lock(); }
}
```

그러나 막상 여러개의 스레드를 만들어서 실행을 해보면 다음과 같은 결과가 나온다.

| 소비자 | 시간(초) | 성능 향상 |
| --- | --- | --- |
| 1 | 101 | 1.04 |
| 2 | 212 | 0.49 |

 오히려 시간이 더 늘었다. 이유는 지나친 경합에 있다. 소비자 스레드가 counts 맵에 접근하기 위해서 너무 많은 시간을 기다려야 해서, 실제로 유용한 작업을 하는 시간보다 기다리는 시간이 더 많은 것이다. 

 따라서 대신 ConcurrentHashMap 을 사용해보자. 

```java
private void countWord(String word) {
	while (true) {
		Integer currentCount = counts.get(word);
		if (currentCount == null) {
			if (counts.putIfAbsent(word, 1) == null)
				break;
		}
		else if (counts.replace(word, currentCount, currentCount + 1)) {
			break;
		}	
	}
}
```

이 경우 다음과 같이 성능이 향상되었다.

| 소비자 | 시간 (초) | 성능 향상 |
| --- | --- | --- |
| 1 | 120 | 0.87 |
| 2 | 83 | 1.26 |
| 3 | 65 | 1.61 |
| 4 | 63 | 1.67 |
| 5 | 70 | 1.50 |
| 6 | 79 | 1.33 |

어느정도 성공했다. 근데 소비자 스레드가 4를 넘어선 이후 다시 느려진다. 이유는 마찬가지로 직전의 해법이 counts 맵에 대해 필요 이상의 경합을 만들어내고 있어서 그렇다. 따라서 하나로 공유하는 counts를 쓰는게 아니라 각 소비자가 일괄적으로 counts를 가지고 사용하는 것이 효과적이다. 마지막에 병합하면 된다.

| 소비자 | 시간(초) | 성능 향상 |
| --- | --- | --- |
| 1 | 95 | 1.10 |
| 2 | 57 | 1.83 |
| 3 | 40 | 2.62 |
| 4 | 39 | 2.69 |
| 5 | 35 | 2.96 |
| 6 | 33 | 3.14 |
| 7 | 41 | 2.55 |

덕분에 4배나 빨라졌다. 지금까지의 병렬 프로그램 작업의 성능 향상 과정을 보면 알겠지만, 초기에는 선형적으로 증가하다가 어느 시점부터는 둔화된다. 성능이 최고점에 도달하고 이후 스레드를 더 추가해도 오히려 속도가 느려진다.

참고로 위와 같은 현상은 (암달의 법칙)[https://ko.wikipedia.org/wiki/%EC%95%94%EB%8B%AC%EC%9D%98_%EB%B2%95%EC%B9%99] 으로도 설명이 가능하다. 제 아무리 병렬화를 시킨다 하더라도 그에 따라 성능 향상이 선형적으로 증가하지는 않는 이유를 설명하는 법칙이기도 하다.
