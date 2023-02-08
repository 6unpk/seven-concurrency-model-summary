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

```

 위의 코드를 실행시 다음과 같은 결과를 볼 수도 있다.

이유가 뭘까? 

**메모리 가시성**

 핵심은 읽는 스레드와 쓰는 스레드가 동기화 되지 않으면 가시성이 보장되지 않는다는 것이다.  앞에서 increment()만 동기화하는 것이 아니라 getCount() 도 동기화를 해야한다. 

**여러 개의 잠금장치**

 그러나 “메서드를 다 동기화하면 되는가?” 라는 해결책은 멍청한 방법이다. 효율성도 떨어지고 스레드 대부분은 블로킹 된 상태를 유지할 것이다. 또한 둘 이상의 스레드에서 공유 메모리를 사용하다 보면 필연적으로 데드락이 발생할 가능성이 높아진다. 

“식사하는 철학자” 모델을 생각해보자. 각 철학자는 양쪽의 젓가락을 들어올려서 식사를 한다. 

### 내재된 잠금 장치를 넘어서
