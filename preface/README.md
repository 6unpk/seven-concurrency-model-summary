# 서문
### 동시성 혹은 병렬성

두개는 서로 다른 개념이다. 공통적을 순차적 프로그래밍을 넘어선다. 

**동시성**

- 논리적 통제흐름(Threads of control)을 갖는다. 병렬로 실행되지 않을 수 도 있다.
- 동시성 프로그램은 여러 사건을 한꺼번에 처리해야하는 요구사항이 있다.
- 동시적 프로그램은 기본적으로 비결정적(nondeterministic)이다. 사건이 일어나는 시점에 따라 결과도 다르다.

**병렬성**

- 논리적 통제흐름이 여러개일 수 있다.
- 문제가 아닌 해법이 가진 속성으로, 각기 다른 부분을 병렬로 처리함에따라 처리속도를 빠르게 한다.
- 반대로 병렬성은 비결정성을 내포하지는 않는다. 배열의 모든 속성의 2를 곱하는 연산은 병렬로 처리하든 직렬로 처리하든 결과가 동일하다.

**롭 파이크**(Rob Pike)의 말을 빌리면 다음과 같다.

```tsx
동시성은 여러 일을 한꺼번에 다루는데 관한 것
병렬성은 여러일을 한꺼번에 실행하는 데 관한 것
```

### 병렬 아키텍처

현대의 컴퓨터는 CPU의 코어뿐만 아니라 여러 수준에서 다양한 병렬성을 지원한다.

**비트 수준의 병렬성**

32비트 보다 64비트 컴퓨터가 더 빠르다. 이는 병렬성때문이다. 64비트 수열을 덧셈하려면 32비트를 2번 처리해야한다. 하지만 64비트에서는 1번만 처리하면 된다.

**명령어 수준의 병렬성**

파이프라이닝, 비순차 실행, 추축 실행 을 참조하자. 여기에 적기는 너무 긴 개념들이다.

**데이터 병렬성**

SIMD라고도 한다. 대량의 데이러에 대해 똑같은 작업을 병렬적으로 처리한다. 이미지 처리(Image Processing)을 예로 들어보자. 각 픽셀들의 값을 조정해야 하는데, GPU가 사용이 된다. 알다시피 GPU는 매우 강력한 데이터 병렬처리기이다.

**태스크 수준의 병렬성**

 멀티프로세서에 대해 알아보자. 멀티프로세서 아키텍처에서 중요한 관심사는 바로 메모리 모델이다. 공유메모리를 사용하거나, 분산 메모리를 사용할 수 있다.

- 공유메모리는 모든 프로세서가 동일한 메모리를 사용하는 것이다.
- 분산메모리는 모든 프로게서가 개별 메모리를 사용하는 것이다.

### 동시성: 멀티코어를 넘어서

동시성은 병렬성 활용 그 이상을 의미한다. 정확하게 구현된 동시성은 소프트웨어에 대해 반응성이 높고, 장애를 허용하며 간단하게 만들어준다.

**동시적인 세계에 맞는 동시적 소프트웨어**

- 현실은 동시적이다. 스마트폰, 비행기, IDE 등 모든일을 동시에 수행한다.
- 동시성은 반응형 시스템의 핵심이다.

**분산된 세계에 맞는 분산 소프트웨어**

- 소프트웨어가 여러 대의 컴퓨텅 분산되어 있고 미리 정해진 특별한 순서에 따라 동하는 것이 아니라면 그것은 동시적이다.
- 분산 소프트웨어는 장애에 강하다. 데이터 센터 어느 한곳이 마비되어도 다른 지역의 데이터 센터가 있으면 그만이다.
- 탄력성과도 연관된다.

**예측 불가능한 세계에 맞는 탄력적 소프트웨어**

- 소프트웨어는 버그가 있으면 충돌한다.
- 동시성과 독립성은 장애감지를 통해 탄력성 혹은 장애 허용을 가능하게 한다. 독립성은 어느 한 작업에서 발생한 문제가 다른 작업에 영향을 주지 않아야하기 때문에 중요하다.
- 장애감지가 중요한 이유는 한 동작이 장애를 일으켰을때, 다른 작업에 통지가 전달되어 복구작업이 일어나도록 해야하기 때문이다.

**복잡한 세계에 맞는 단순한 소프트웨어**

- 제대로 선택한 언어와 도구를 사용하면 동시성 코드가 순차적인 코드보다 더 단순하고 명쾌하게 작성할 수 있다.(스레딩 버그는 잡기 어렵지만…?)
- 여러문제를 다루는 복잡한 스레드 하나보다는 하나믜 문제를 다루는 여러개의 스레드가 낫다.

### 일곱가지 모델

**스레드와 잠금장치**

- 문제가 많지만, 아주 기본적인 모델이자 기술이며 여전히 많은 곳에서 활용중이다.

**함수형 프로그래밍**

- 가변성이 없어 기본적으로 스레드-안전하며 동시성과 병렬성을 잘 지원한다.

**클로저 방식**

- 프로그래밍 언어 중 하나인 클로저의 명령형과 함수형의 조합을 살펴본다.

**액터**

- 광범위한 범용적 동시성 프로그래밍 모델이다. 공유 메모리와 분산 메모리 아키텍처 양측에서 활용이 가능하다.

**순차 프로세스 통신 (CSP)**

- 액터와 비슷하지만 액터와 달리 메시지 전달이 아닌 채널이라는 개념을 사용한다.

**데이터 병렬성**

- GPU는 단순 그래픽 처리를 넘어서 유한 요소 분석, 유체역학 계산, 머신러닝과 딥러닝 등 오만 엄청난 연산을 요구하는 코드에 활용가능하다.

**람다 아키텍처**

- 병렬성이 아니면 빅데이터 처리가 불가능하다. 맵리듀스와 스트리밍 프로세스의 장점을 결합해 다양한 빅데이터 문제를 해결하도록 도와준다.
