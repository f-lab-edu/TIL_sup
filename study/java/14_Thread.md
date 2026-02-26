# 14. 자바의 쓰레드

자바에서는 쓰레드를 동작하기 위한 Runnable 인터페이스와 Thread 클래스가 있다.

자바의 쓰레드에 대해서 알아보기 전에 Thread의 개념부터 정의해보자.

## Thread란?

처음엔 보통 프로세스와 쓰레드를 혼동하기 쉽다. 하지만 명확하게 프로세스가 쓰레드 보다 상위이다. 프로세스는 프로그램을 실행하는 단위이고, 쓰레드는 그 실행 단위안에서 실행 흐름을 나눈 단위이다.

많이 드는 예시로는 프로세스는 하나의 식당을 의미하고, 쓰레드는 그 안에서 일하는 직원을 의미한다.

- 프로세스인 식당은 독립된 공간(주방/창고/홀/주문 시스템)을 가지고 있다.
- 쓰레드는 그 공간에서 잡업을 수행하는 일꾼이다.

그래서 프로세스는 프로세스마다 독립된 공간을 가지고, 쓰레드는 그 독립된 공간을 공유하며 동작한다.

## 자바 메인 스레드

```java
public static void main(String[] args) {
}
```

자바는 main 메소드가 실행되면서 하나의 스레드가 시작되는데 이것을 메인 스레드라고 한다.

- 메인 스레드는 자바 실행과 동시에 실행된다.
- 모든 자바 애플리케이션에 최소 1개 이상의 스레드를 가진다.
- 메인 스레드가 멈추면 프로그램 전체가 멈추게 된다.
- 메인 메소드가 정상 종료되거나 return을 만나게 되면 프로그램은 종료된다.
- 새로운 스레드를 생성하여 병렬 처리할 수 있다.
- 일반 스레드는 메인 스레드가 종료되어도 남아있을 수 있다. 데몬 스레드는 함께 종료된다.

자바 메인 스레드가 생성되면, JVM은 해당 스레드만을 위한 고유한 JVM스택 영역을 메모리에 생성한다.
JVM 스택에는 메서드 호출, 지역 변수, 임시 결과 등을 저장하며, 힙 영역에 생성된 객체의 참조값을 가지고 있다. 또한, 이 스레드는 메서드 영역과 힙 영역을 다른 스레드와 공유하며 사용하게 된다.

즉, 자바는 메인 스레드를 필두로 필요하면 여러 스레드를 생성하여 프로그램을 동작을 담당하게 된다.

## Runnable 인터페이스, Thread 클래스

자바에서 스레드 제어는 Runnable, Thread를 통해 할 수 있다. 스레드는 Java 1.0부터 존재했지만, 동시성 API의 본격적인 체계화는 Java 5 부터라고 보면 된다.

**초기 (Java 1.0 ~ 1.4)**

- java.lang.Thread
- java.lang.Runnable
- Object.wait/notify/notifyAll
- Thread.sleep/join/interrupt
- Thread.stop/suspend/resume

> Thread.stop/suspend/resume 등은 위험해서 사실상 사용 금지
>

### Runnable

Runnable은 인터페이스로 실행할 작업 자체를 표현하는 함수형 인터페이스이다.

- 핵심 메서드 : void run()
- 특징
    - 스레드 생성/관리 책임은 없다.
    - 작업과 실행을 분리해서 재사용/테스트/조합이 쉬움
    - ExecutorService 같은 프레임워크와 조합해여 사용하기 좋다.

즉, Runnable은 스레드는 아니다. 실행 단위이다.

> **실행 단위**
실행 단위는 CPU에 의해 스케줄링되어 실제로 수행되는 최소 작업 단위이다.
운영체제에서 실제 스케줄링 대상은 스레드다.
>

### Thread

Thread는 실행 컨텍스트를 생성하고 OS 스레드와 매핑되는 실행 주체이다.

- 새로운 호출 스택 생성
- OS 네이티브 스레드와 1:1 매핑 (HotSpot 기준)
- 스케줄링은 운영체제에 위임
- run()을 직접 호출하면 멀티스레드가 아니다.
    - run() : 스레드가 수행되는 작업
    - start() : 스레드를 시작하는 메소드

```java
public class Thread implements Runnable {
    private Runnable target;
}
```

Thread는 내부적으로 Runnable을 보유할 수 있다. (상속보다 위임)

**제공 메소드**

- sleep
    - 현재 쓰레드 멈추기
    - 자원을 놓지 않고, 제어권을 넘겨줌
    - 데드락 발생 가능
- interrupt
    - 다른 쓰레드를 깨워서 interruptedException 발생시킴
    - Interrupt가 발생한 스레드는 예외를 catch하여 다를 작업 수행 가능
- join
    - 다른 스레드의 작업이 끝날 때 까지 기다림
    - 스레드의 순서 제어 가능

### Runnable과 Thread의 차이

```java
Runnable task = () -> System.out.println("Hello");
task.run();  // 그냥 메서드 호출
```

```java
Thread t = new Thread(task);
t.start();
```

Runnable과 Thread의 차이는 run과 start이다. 위에 설명했듯이 Runnable로 run만 호출하면 일반 메소드 호출과 똑같다. 그래서 Thread 생성시 Runnable을 넘기고 Thread 내에서 start를 해야 새로운 스레드가 작업하게 된다.

- Runnable은 Task를 정의
- 실제 실행은 Thread로 가능

### 설계 관점 차이

```java
class MyThread extends Thread {
    public void run() { }
}
```

스레드는 상속받아 스레드 클래스를 만들어 사용할 수 있다. 하지만, 상속의 문제점으로 인해 보통 이렇게 사용하지는 않는다.

- 실행과 작업이 강결합
- 재사용성 낮음
- 실행 전략 변경 어려움

그리고 스레드를 직접 생성해 사용하면 다음과 같은 문제점이 있다.

- 스레드 수 상한이 없다. 즉, 계속해서 생성할 수 있다.
- 스레드가 많아지면,
    - 컨텍스트 스위칭 증가
    - CPU 캐시 효율 저하
    - OOM 위험 증가
- 실행 정책이 없어 관리가 불가하다.

  스레드 사용은 운영체제 자원을 직접적으로 소모하므로, 적절한 실행 정책이 수반되지 않으면 애플리케이션에 심각한 성능 및 안정성 문제를 초래할 수 있다.

    - 스레드 이름/우선순위/데몬 여부
    - 예외 처리 전략
    - 재시도/타임아웃/취소
    - 결과 수집
    - 거절 정책
    - 리소스 관리

그래서 Executor + Runnable 조합으로 실행 및 관리는 프레임워크에 위임하고, Runnable로 작업만 정의해서 사용한다.

### Runnable 작업 단위 사용 예시

```java
public class IndependentTask implements Runnable {

    private int localCount = 0;

    @Override
    public void run() {
        for (int i = 0; i < 1000; i++) {
            localCount++;
        }
        System.out.println(localCount);
    }
}
```

```java
Runnable task = new IndependentTask();

ExecutorService pool = Executors.newFixedThreadPool(2);
pool.submit(task);
```

Runnable을 구현한 작업 단위 클래스를 정의 및 객체 생성하여 ExcutorService에 등록하면 ExceutorService가 내부 실행 전략에 맞춰서 스레드를 실행해준다.

---

## 참조

- https://jaimemin.tistory.com/2479#google_vignette
- https://mangkyu.tistory.com/258
- 책
    - 자바의 신