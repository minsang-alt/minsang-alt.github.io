---
title: 생산자 소비자 문제
date: 2024-09-12 10:19:10 +/-0800
categories: [Tech Notes, Java]
tags: thread    # TAG names should always be lowercase
---

## 생산자 소비자 문제란

생산자 소비자 문제는 멀티스레드 프로그래밍에서 자주 등장하는 동시성 문제 중 하나로, 여러 스레드가 동시에 데이터를 생산하고 소비하는 과정에서 발생합니다.

**생산자란,** 데이터를 생성하는 역할을 하며, 파일에서 데이터를 읽어오거나 네트워크에서 데이터를 받아와 버퍼에 저장하는 역할을 하는 스레드입니다.

**소비자란,** 생성된 데이터를 사용하는 역할을 합니다. 버퍼에 있는 데이터를 처리하는 역할을 하는 스레드입니다.

**버퍼란,** 생산자가 생성한 데이터를 일시적으로 저장하는 공간입니다. 이 버퍼는 한정된 크기를 가지며, 생산자와 소비자가 이 버퍼를 통해 데이터를 주고 받습니다.

여기서 문제 상황이란, **생산자 스레드가 너무 빨리 생성하여 버퍼가 가득차면 이후 생성한 데이터를 버리는 문제가 발생합니다.
또는 소비자 스레드가 너무 빨라 버퍼가 비었는데, 이로인해 더이상 소비할 데이터가 없어 소비자 스레드는 null 값을 꺼내는 문제가 발생합니다.**

## 해결 방법

가장 이상적인 해결 방법은 생산자 스레드가 생성한 데이터가 버려지지 않고 소비자 스레드가 소비할 수 있도록 대기하며, 반대로
소비자스레드는 가져갈 데이터가 없다면 생산자 스레드가 데이터를 추가할 때까지 기다리는 것 입니다.

### While 문 (해결 불가능)

생산자 스레드가 먼저 `synchronized` 블록에서 모니터락을 가져가고 버퍼에 가득찬 것을 보고 다음과 같이 대기를 하면 문제가 발생합니다.

```java
synchronized(this){
    while(queue.isEmpty()){
        sleep(10);
    }
}
...
```

이 코드의 목적은 버퍼가 비어있다면 CPU 자원을 아끼기 위해 일단 대기 상태에 있다가 다시 깨어나서 확인하는 등의 작업입니다.
하지만 이것은 락을 가지고 대기를 하는 것이라 소비자 스레드가 진입도 못하고 `BLOCKED`상태로 바뀝니다.

즉, While문과 대기상태 조합으로는 생성자-소비자 문제를 해결할 수 없습니다.


### Object - wait, notify

자바의 `Object.wait()` , `Object.notify()` 를 사용하면 락을 가진 스레드가 대기하려고 할때 다른 스레드에게 락을 양보하여 해결할 수 있습니다.

- `Object.wait()` :  현재 스레드가 가진 락을 반납하고 `WAITING` 합니다. 이 메소드는 현재 스레드가 `synchronized` 블록이나 메소드에서 락을 소유하고 있을때만 호출할 수 있습니다.
- `Object.notify()` : 대기 중인 스레드 중 하나를 깨웁니다. 이 메소드는 `synchronized` 메소드나 블록 내에서 호출되야 하며, 대기하다가 깨워진 스레드는 락을 다시 획득할 기회를 얻습니다.
- `Object.notifyAll()` : 대기 중인 모든 스레드를 깨웁니다.


**활용 예제**

```java
import java.util.ArrayDeque;

public class BoundedQueue {
    private final Queue<String> queue = new ArrayDeque<>();
    private final int max;
    
    public BoundedQueue(int max){
        this.max = max;
    }
    
    public synchronized void put(String data){
        while(queue.size() == max){
            System.out.println("큐가 가득참. 생산자 대기");
            try {
                wait(); // RUNNABLE -> WAITING + 락 반납 후 스레드 대기 집합으로 이동
                System.out.println("생성자 깨어남");
            }catch (InterruptedException e){
                throw new RuntimeException(e);
            }
        }
        queue.offer(data);
        System.out.println("생성자 데이터를 저장했으므로 notify() 호출하여 소비자 스레드 깨어나게 함");
        notify(); // 대기 집합에 있던 대기 스레드는 WAIT -> BLOCKED : 왜냐하면 아직 락이 반납되지 않았기 때문 반납되어 이 대기했던 스레드가 가져가면 BLOCKED -> RUNNABLE
        // notifyAll();
    }
    
    public synchronized String take() {
        while (queue.isEmpty()){
            System.out.println("큐에 데이터가 없기 때문에 소비자 스레드는 대기");
            try {
                wait();
                System.out.println("소비자 스레드 깨어남");
            }catch (InterruptedException e){
                throw new RuntimeException(e);
            }
        }
        String data = queue.poll();
        notify(); // 대기하고 있던 생성자 스레드, WAIT -> BLOCKED 
        //notifyAll();
        return data;
    }
}
```

**- 생성자1 스레드가 락 획득 후, data1을 큐에 넣는 상황**

![](/assets/img/producer-consumer/img.png)

**- 생성자2 스레드는 큐가 꽉 차서 대기상태로 바꾼 후, 락 반납**
![](/assets/img/producer-consumer/img_2.png)

**- 소비자1 스레드는 락을 획득 후, 큐에서 데이터를 얻은 다음, notify를 호출하여 대기 집합에 있던 생성자스레드를 깨웁니다.** 
![](/assets/img/producer-consumer/img_3.png)

소비자 스레드가 먼저 진행되는 경우에도 같은 원리로 **생성자-소비자 문제**를 해결할 수 있습니다.

### notify() 한계

예를들어, 스레드 대기집합에 **소비자 스레드 s2,s3** **생성자 스레드 p1,p2**가 있고 **큐(버퍼)에 데이터가 하나 있을 때** 소비자 s1 스레드가 데이터를 소비할 경우를 가정해봅니다.

s1 스레드는 데이터 소비후, notify() 호출을 통해 스레드 대기집합에 있는 스레드 중 하나를 깨울텐데, s2를 깨우면 비효율적 문제가 생깁니다.

왜냐하면, s2는 BLOCKED -> RUNNABLE 락 획득 후, 큐를 확인했는데 아무 데이터가 없기 때문에 다시 락을 반납하고 대기집합으로 이동합니다. 만약 p1,p2 둘 중 하나가 깨면 이런 비효율 문제가 발생하지 않을 것 입니다.

**즉, notify() 메소드는 같은 종류의 스레드를 깨울 때 비효율이 발생합니다. 또한, 어떤 스레드가 깨어날 지 알 수 없기 때문에 스레드 기아 문제가 발생할 수 있습니다.**

notifyAll()은 스레드 기아 문제는 해결할 수 있겠지만, 원치 않는 스레드들도 모두 깨기 때문에 역시나 비효율적입니다.

## 같은 종류의 스레드를 깨우는 비효율 문제를 해결하는 방법

위 문제를 해결하는 방법은 소비자스레드 대기집합과 생성자스레드 대기집합을 나누면 해결할 수 있습니다.
즉, **소비자스레드가 데이터를 소비하면 생성자스레드 대기집합에 알려주고, 생성자스레드가 데이터를 생산하면 소비자스레드 대기집합에 알려주면 됩니다.**

### ReentrantLock 과 Condition

```java
public class BoundedQueue{
    private final Lock lock = new ReentrantLock();
    private final Condition producerCondition = lock.newCondition();
    private final Condition consumerCondition = lock.newCondition();
    
    private final Queue<String> queue = new ArrayDeque<>();
    private final int max;

    public BoundedQueue(int max){
        this.max = max;
    }

    public void put(String data){
        lock.lock();
        try {
            while(queue.size() == max){
                System.out.println("큐가 가득참. 생산자 대기");
                producerCondition.await();
                System.out.println("생성자 깨어남");

            }
            queue.offer(data);
            System.out.println("생성자 데이터를 저장했으므로 notify() 호출하여 소비자 스레드 깨어나게 함");
            consumerCondition.signal();
        }catch (InterruptedException e){
            throw new RuntimeException(e);
        }finally {
            lock.unlock();
        }
    }
}
```

`consumerCond`는 소비자를 위한 스레드 대기 공간이며, `producerCond`는 생산자를 위한 대기 공간입니다.

**put 메서드** 

**큐가 가득 찬 경우,** `producerCond.await()` 를 호출해서 생산자 스레드를 생산자 전용 스레드 대기 공간에 보관합니다.

**데이터를 저장한 경우,** 생산자가 데이터를 생산하면 큐에 데이터가 추가됩니다. 따라서 소비자를 깨우는 것이 좋습니다.
`consumerCond.signal()` 를 호출해서 소비자 전용 스레드 대기 공간에 신호를 보냅니다. 이렇게 하면 대기중 인 소비자 스레드가 하나 깨어나서 락을 얻을 때까지 BLOCKED 상태에 있다가 
생산자가 락을 반납하여 소비자가 해당 락을 가져오면 데이터를 소비할 수 있습니다.

![](/assets/img/producer-consumer/img_4.png)

## synchronized 와 ReentrantLock 대기 차이

먼저 `synchronized`는 두 가지 대기 방법이 있습니다. 

첫번째로, 모니터 락을 얻지 못한 스레드는 `BLOCKED` 상태로 락 획득을 대기합니다.
`synchronized`를 빠져나가 락을 반환하면 다른 스레드가 락을 획득하기를 시도하고 락을 획득하면 락 대기 집합을 벗어납니다.

두번째로, wait()를 호출했을 때 락을 반납하고 스레드 대기 집합으로 이동합니다. 이때 `WATING`상태로 대기합니다. 락을 얻은 다른 스레드가
`notify()`를 호출 했을 때 스레드 대기 집합을 빠져나갑니다. 빠져나간 스레드는 락 대기집합으로 이동해 락을 얻기 위해 다시 경쟁하고, 모니터 락을 얻지 못하면 락 대기집합에 들어가서
`BLOCKED` 상태로 락을 기다립니다. 

`ReentrantLock`은 `lock.lock()`을 호출했을 때 락이 없다면 `WAITING` 상태로 락 대기 큐에 대기하고 있습니다. 다른 스레드가 `lock.unlock()`을 호출 했을 때
대기가 풀리며 락 획득 시도를 하고 락을 획득하면 대기 큐를 빠져나갑니다.

또한 `condition.await()`를 호출 했을 때, `condition` 객체의 스레드 대기 공간에서 WAITING 상태로 대기합니다. 다른 스레드가 `condition.signal()`를
호출 했을 때, `condition` 객체의 스레드 대기 공간에서 빠져나갑니다.

깨어난 스레드는 `WAITING` 상태로 락 대기 큐에 진입하고, 락을 획득하면 `RUNNABLE` 상태가 되면서 코드를 실행할 수 있습니다.

**공통점은 둘 다 생산자-소비자 문제를 해결하기 위해 2번의 대기소를 거쳐야 한다는 점 입니다.**

## BlockingQueue

자바의 생산자 소비자 문제를 해결하기 위해 `BlockingQueue` 인터페이스와 `ArrayBlockingQueue` 구현체, `LinkedBlockingQueue` 구현체를 제공합니다.

앞에서 구현한 것처럼 이미 자바에서는 concurrent 패키지에 해당 클래스들을 제공하고 있습니다.

- `ArrayBlockingQueue` : 배열 기반으로 구현되어 있고, 버퍼의 크기가 고정되어 있습니다.
- `LinkedBlockingQueue` : 링크 기반으로 구현되어 있고, 버퍼의 크기를 고정할 수도, 또는 무한하게 사용할 수도 있습니다.
- `put(e)` 과 `take()`는 앞서 구현한 내용과 매우 비슷하게 구현되어 있습니다.
- 더 많은 메소드 내용은 [공식문서](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/BlockingQueue.html)에서 확인할 수 있습니다.

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

BlockingQueue<String> queue = new ArrayBlockingQueue<>(10);

public static void main(String[] args) {
    queue.put("data0");
    String take = queue.take();
}
```

## Reference

[김영한의 실전 자바 - 고급 1편, 멀티스레드와 동시성](https://inf.run/DqqWF)
