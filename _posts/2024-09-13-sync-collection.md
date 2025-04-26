---
title: 동시성 컬렉션
date: 2024-09-13 10:19:10 +/-0800
categories: [Tech Notes, Java]
tags: thread    # TAG names should always be lowercase
---

컬렉션 프레임워크는 원자적인 연산이 아니고 여러 연산이 들어가 있기 때문에 멀티 스레드 상황에서 문제가 생깁니다. 따라서 동시성 컬렉션을 만들기 위해
여러가지 도입을 했습니다.

## 프록시 도입

`ArrayList`, `LinkedList`, `HashSet`, `HashMap` 등의 코드도 모두 복사해서 `synchronized` 기능을 추가한다는 것은 매우 비효율적입니다.

`synchronized` 기능만 살짝 추가하면 되는 상황에 프록시 도입은 매우 효율적입니다.

프록시란, 대신 처리해주는 자 입니다. 예를들어 다음과 같이 구현할 수 있습니다.

```java
public class SyncProxyList<T> implements List<T>{
    
    private List<T> target;
    
    public SyncProxyList(List<T> target) {
        this.target = target;
    }
    
    @Override
    public synchronized void add(T e) {
        target.add(e);
    }
    
    @Override
    public synchronized T get(int index) {
        return target.get(index);
    }
    
    @Override
    public synchronized int size() {
        return target.size();
    }
}
```

이제 껍데기만 씌우면 동시성을 제공하는 프레임워크를 사용할 수 있습니다.

```java
public static void main(String[] args) {
    test(new SyncProxyList(new LinkedList<String> hello));
}

private void test(List<?> hello){
    
}
```

test 메소드는 인터페이스인 List에만 의존하며 List 구현체인 LinkedList를 사용하는 지 SyncProxyList 구현체를 사용하는 지 신경안쓰고 변경없이 사용할 수 있습니다.

단지 List 인터페이스에서 정의한 add, get, size 등을 호출할 뿐인데, SyncProxyList 구현체가 들어오면 `synchronized`블록내에서 실제 구현체의 메소드를 대신 호출합니다.

**즉, 프록시 패턴이란, 객체지향 디자인 패턴 중 하나로, 어떤 객체에 대한 접근을 제어하기 위해 그 객체의 대리인 또는 인터페이스 역할을 하는 객체를 제공하는 패턴입니다.** 

프록시 객체는 실제 객체에 대한 참조를 유지하면서, 그 객체에 접근하거나 행동을 수행하기 전에 추가적인 처리를 할 수 있도록 합니다.

`Collections.synchronizedList(target)`은 프록시 패턴으로 `synchronized` 동기화 메서드를 지원합니다. 그외에도 다양한 메소드로 프레임워크가 동기화 프록시를 만들어 낼 수 있게 합니다.

- `synchronizedList()`
- `synchronizedCollection()` 
- `synchronizedMap()` 
- `synchronizedSet()` 
- `synchronizedNavigableMap()` 
- `synchronizedNavigableSet()` 
- `synchronizedSortedMap()` 
- `synchronizedSortedSet()`

## synchronized 프록시 방식의 단점

동기화 비용은 매우 비싸서 **성능저하가 발생**합니다. 또한, **모든 메소드에 대해 동기화를 적용하기 때문에 여러 스레드들이 대기해야 하는 상황이 빈번**해집니다.
그리고 정교한 동기화가 불가능합니다. **프록시 특성상 특정 블록만 동기화를 적용하는 것은 어렵습니다. 따라서 불필요한 부분까지 동기화가 이루어지면서 매우 비효율적**입니다.

## 동시성 컬렉션

자바 1.5부터 concurrent 패키지에 동시성을 위한 컬렉션을 제공합니다.  예를 들어, `ConcurrentHashMap` , `CopyOnWriteArrayList` , `BlockingQueue` 등은 다양한 최적화 기법을 적용했습니다.

`synchronized` , `Lock` ( `ReentrantLock` ), `CAS` , 분할 잠금 기술(segment lock)등 다양한 방법을 섞어서 매우 정교한 동기화를 구현하면서 동시에 성능도 최적화했습니다.

## Reference

[김영한의 실전 자바 - 고급 1편, 멀티스레드와 동시성](https://inf.run/DqqWF)