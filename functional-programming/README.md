# 함수형 프로그래밍

함수형 프로그래밍은 진보적이고 미래적이다. 아직은 대중적으로 사용되지는 않지만, 향후 20년을 책임질 수소 연료 자동차를 운전하는 것과 비슷하다. 

 명령형 프로그래밍과 달리 함수형 프로그래밍은 평가하는 방식으로 계산을 수행하며, 부작용(Side Effect)이 없는 순수 수학적 함수에서 만들어진다. 부작용이 없기 때문에 스레드 안전하고, 훨씬 간단하게 동시성 프로그래밍을 가능하도록 만들어준다. 

## 문제가 있으면 멈추는 것이 상책이다.

  2장(스레드와 잠금장치)에서 언급한 잠그기(Lock)과 관련해 논의한 규칙은, 가변이면서 공유되는 상태에만 적용된다. 그러나 불변 값은 잠금장치(Lock 따위)를 필요로 하지 않는다. 애초에 변하지 않기 때문에 메모리의 가시성과 같은 문제는 발생하지 않는것이다.

### 가변상태 없이 프로그래밍 하기

**가변상태의 위험**

 여기서는 병렬성에 초점을 맞춘다. 간단한 함수 프로그램을 작성한 다음, 그것이 함수형 패러다임을 쓰기 때문에 병렬성 구현이 쉽다는 것을 보여줄 것이다.

**숨겨진 가변 상태**

 다음은 가변 상태를 갖지 않아 스레드 안전한 클래스 처럼 보인다.

```java
class DateParser {
	private final DateFormat format = new SimpleDateFormat("yyyy-MM-dd");

	public Date parse(String s) throws ParserException {
		return format.parse(s);
	}
}
```

 하지만 에러메시지는 실행할 때마다 다르게 뜬다.  분명히 final을 선언하고 변경이 안될 것이라고 생각하지만, 사실 SimpleDateFormat 내부 깊숙한 곳에 가변상태를 가지고 있기 때문이다. 자바는 이런식으로 가변 상태를 만들기가 너무 쉽다는 문제점을 가지고 있다. 그렇기 때문에 SimpleDateFormat의 API를 보는 것 만으로는 스레드-안전(Thread Safe)한지 알 수가 없다. 

**가변 상태는 탈출왕**

  어떤 토너먼트를 주관하기 위한 웹 서버를 만든다고 생각해보자. 선수 명단 관리를 위해 코드를 다음과 같이 작성한다.

```java
public class Tournament {
	private List<Player> players = new LinkedList<Player>();
	
	public synchronized void addPlayer(Player p) {
		players.add(p);
	}

	public synchronized void getPlayerIterator(Player p) {
		return players.iterator();
	}
}
```

 언뜻 보기에 스레드 안전해 보이지만, 이 코드는 그렇지 않다. 이유는 getPlayerIterator에 의해 리턴된 순한자가 player 내부에 저장된 가변상태를 참조할 수도 있기 때문이다. 순환자가 사용되는 동안 다른 스레드가 addPlayer를 호출하면 어떤 예외를 만나게 될지 모른다. 

**첫 번째 함수 프로그램**
