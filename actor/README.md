# 액터
액터는 렌터카와 비슷하다. 원할 때 쉽게 얻을 수 있고, 사용하던 것이 고장나도 굳이 고치지 않아도 된다. 렌트카 회사에 전화하면 새로운 액터가 도달한다. 액터 모델은 상당히 범용적인 동시성 프로그래밍 모델이다. 

객체보다 더욱 객체 지향적인

함수형 프로그래밍은 가변 상태를 사용하지 않음으로써, 공유된 가변 상태가 야기하는 문제를 원천적으로 피한다. 이와 달리 액터 모델은 가변상태를 사용하기는 하지만 공유되는 부분을 제거한다. 액터는 객체 지향 프로그램에서 사용하는 객체와 비슷하다. 상태를 캡슐화하고 메시지를 전달하는 방식으로 다른 액터와 의사소통한다. 차이가 있다면 액터는 동시에 수행 가능하다는 점이다. 액터는 어느 프로그래밍 언어에서든 적용 가능하지만 실제로는 얼랭과 가장 많은 연관을 가지고 있다. 여기서는 얼랭이 아닌 엘릭서를 이용해 액터를 설명한다.

**메시지와 메일 박스**

> **액터 혹은 프로세스?**
> 
> 엘릭서에서는 액터가 ‘프로세스’로 불린다. 여기서 말하는 프로세스는 OS 수준의 프로세스가 아닌 그보다 훨씬 가벼운 모델이다. 심지어는 스레드보다도 가볍다.
> 

**첫 번째 액터** 

간단한 액터를 만들고 그에게 메시지를 전달하는 예를 살펴보자. 다음은 액터의 코드이다.

```elixir
defmodule Talker do
	def loop do
		receive do
			{:greet, name} -> IO.puts("Hello #{name}")
			{:praise, name} -> IO.puts("#{name}, you're amazing")
			{:celebrate, name} -> IO.puts("Here's to another #{age} years, #{name}")
		end
		loop
	end
end
```

위 코드는 세 종류의 메시지를 받아, 각각에 대해서 적절한 문자열을 출력하는 액터이다. 다음은 액터에 메시지를 전달하는 코드이다.

```elixir
pid = spawn(&Talker.loop/0)
send(pid, {:greet, "Huey"})
send(pid, {:praise, "Dewey"})
sedn(pid, {:celebrate, "Louie", 16})
sleep(1000)
```

우선 액터의 인스턴스를 생성하고, 프로세스 식별자를 받아서 pid라는 변수에 바인딩한다. 프로세스는 단순히 하나의 함수, 이 경우에는 Talker 모듈 내부에 존재하고 아무 인수 도 받지 않는 loop 함수를 실행한다. 코드를 실행하면 다음과 같은 결과를 볼 수 있다.

```
Hello Huey
Dewey, you're amazing
Here's to another 16 years, Louie
```

**메일박스는 큐다**

액터 프로그래밍의 특징 중 가장 중요한 것은 메시지들이 비동기적인 방식으로 전달되는 것이다. 이것들은 액터에 직접 전달되지 않고, 메일박스에 전달된다. 

**메시지 전달받기**

액터는 보통 receive로 메시지를 받아 처리하는 작업을 무한히 반복한다. 

```elixir
defmodule Talker do
	def loop do
		receive do
			{:greet, name} -> IO.puts("Hello #{name}")
			{:praise, name} -> IO.puts("#{name}, you're amazing")
			{:celebrate, name} -> IO.puts("Here's to another #{age} years, #{name}")
		end
		loop
	end
end
```

이 함수는 자신을 재귀적으로 호출하는 무한루프 방식이다. receive 블록은 메시지가 전달되기를 기다리다가 메시지가 전달되면 패턴 매치로 판단한다.

앞에서 보았던 코드는 프로그램을 종료하기 전에 메시지가 처리되는 시간을 허락하기 위해 1초 동안 sleep 을 호출한다. 딱히 바람직한 방법은 아니므로 해결해보자.

> **무한한 재귀가 스택을 넘치게 하지 않을까?**
> 
> 꼬리 재귀 최적화를 수행하기 때문에 그런일은 일어나지 않는다. 꼬리재귀 최적화에서는 재귀호출을 점프(JUMP) 명령어로 바꾸는 일을 수행한다.
> 

**프로세스 연결하기**

프로그램을 깔끔히 종료하려면 두 가지 방법이 있다. 첫째, 큐에 있는 모든 메시지를 처리했으면 동작을 멈추라고 액터에 알려줘야 한다. 둘째, 액터가 실제로 종료했는지 확인하는 방법이 필요하다. 첫번째 방법은 다음과 같다.

```elixir
defmodule Talker do
	def loop do
		receive do
			{:greet, name} -> IO.puts("Hello #{name}")
			{:praise, name} -> IO.puts("#{name}, you're amazing")
			{:celebrate, name} -> IO.puts("Here's to another #{age} years, #{name}")
			{:shutdown} -> exit(:normal)
		end
		loop
	end
end
```

둘째 방법은 액터가 완전히 종료되었음을 확인하는 것이다. :trap_exit 를 true로 설정한 다음, 그것을 spawn() 대신 spawn_link()를 이용해서 연결하면 된다. 

```elixir
Process.flag(:trap_exit, true)
pid = spawn_link(&Talker.loop/0)
```

위 코드는 생성된 프로세스가 작업을 종료하면 통지가 된다. 전달되는 트리플은 다음과 같다.

```elixir
{:EXIT, pid, reason}
```

이제 남은 일은 액터에 종료 메시지를 보내고 시스템 통보를 기다리는 것이다. 

**상태 액터**
