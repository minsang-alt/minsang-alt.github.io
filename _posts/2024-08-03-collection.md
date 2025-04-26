---
title: Collections Framework
date: 2024-08-03 10:19:10 +/-0800
categories: [Tech Notes, Java]
tags: java     # TAG names should always be lowercase
---

## 컬렉션 프레임워크란 

컬렉션 프레임워크는 객체 그룹을 저장하고 관리하기 위한 통합 아키텍처입니다. 
이 프레임워크는 `List, Set, Queue, Map`등의 인터페이스와 그 구현체들 그리고 이들을 다루는 다양한 메소드들로 구성되어 있습니다.

## ArrayList에서 indexof 메소드는 값 비교를 어떻게 할까

```java
int indexOfRange(Object o, int start, int end) {
    Object[] es = elementData;
    if (o == null) {
        for (int i = start; i < end; i++) {
            if (es[i] == null) {
                return i;
            }
        }
    } else {
        for (int i = start; i < end; i++) {
            if (o.equals(es[i])) {
                return i;
            }
        }
    }
    return -1;
}
```

**배열의 처음부터 끝까지 순회하며 equals 메소드로 객체를 비교합니다.** 



## 컬렉션의 요소들을 반복하는 방법 

1. **향상된 for 문 이용** : 컬렉션과 배열 뿐 아니라 iterable 인터페이스를 구현한 모든 객체는 순회가 가능합니다.

```java
List<Integer> elements = new ArrayList<>();

for(element : elements){
    System.out.println(element);    
}
```
2. **iterator() 메소드 사용** : `Collection` 인터페이스는 `Iterable` 인터페이스를 구현하고 있기 때문에 iterator() 메소드를 사용할 수 있습니다. iterator() 메소드를 구현한 클래스는, 반환된 `Iterator` 객체를 통해 순회합니다.

```java

List<Integer> elements = new ArrayList<>();
Iterator<Integer> iterator = elements.iterator();

while (iterator.hasNext()){
    System.out.println("iterator.next() = " + iterator.next());
}

```

3. **기본 for loop 사용**
4. **forEach 메소드 사용** : iterable 인터페이스에는 `forEach()`가 구현되어 있습니다. 이를 이용해 내부적으로 반복을 구현할 수 있습니다. 

```java
elements.forEach((Integer element) -> {
    System.out.println("element = " + element);
});

elements.forEach(element-> System.out.println("element = " + element));
```

## 기본 for loop vs for-each loop 차이점

`for-each loop`는 컴파일러에 의해 해당 반복문을 iterator로 구성합니다. 그리고 이는 순회 중 컬렉션에 변경이 일어나거나 삭제가 일어나면 예외가 발생하기 때문에
`removeIf`을 사용해 요소를 제거해야 합니다. 또한 인덱스가 없다는 게 특징입니다.

성능은 컬렉션의 구현체마다 다릅니다.
`LinkedList`로 구현한 객체는 get() 호출 시 시간복잡도 O(N)만큼 걸려서 요소를 가져오기 때문에 **기본 반복문에서 O(N^2)이 걸립니다.**
하지만 `for-each loop`는 **iterator에 의해 순회하기 때문에** 항상 조회한 다음 바로 다음 요소 위치로 이동하여 대기합니다. 따라서 **최종적으로 O(N)**이기 때문에 조회성능이 빠릅니다.

`ArrayList`로 구현한 객체는 요소를 방문하는 시간복잡도가 항상 O(1)이기 때문에 **기본 반복문은 최종적으로 O(N)이며**, `for-each loop`도 마찬가지지만 Iterator라는 객체를 생성해야하는 시간도 포함되기 때문에 약간의 성능차이가 존재합니다.

## ArrayList와 LinkedList 차이점

**ArrayList**
1. List 인터페이스를 구현한 동적으로 크기 조정 가능한 배열입니다.
2. 요소에 접근하는 데 즉, 읽기에 걸리는 시간은 시간복잡도 O(1)이 걸리며, 만약 특정 위치에 요소의 추가나 제거가 일어날 때는 그 뒤 요소들이 뒤로 밀려나가기 때문에 최악의 경우 O(n) 시간이 걸립니다.
3. 요소가 계속 삽입되어 로드팩터보다 커지면 크기를 조정하고 배열을 복사해야 하기 때문에 시간이 오래 걸립니다.
4. 메모리 위치는 연속적이며, 캐시친화적 입니다.

**LinkedList**
1. List 인터페이스와 Deque 인터페이스를 모두 구현한 이중 연결 리스트 입니다.
2. 삽입,삭제 작업이 O(1)으로 ArrayList에 비해 빠르며, 탐색 작업은 최악의 경우 O(N)의 시간이 걸립니다. 
3. 앞과 뒤를 기억하는 포인터도 저장해야 하기 때문에 메모리를 더 많이 사용합니다.

## ArrayList와 Vector 차이점

Vector 와 ArrayList 모두 List 인터페이스를 상속 받으므로 사용방법이 비슷합니다. 하지만 Vector는 **thread-safe**하며 동기화되어 있습니다. 반면, ArrayList는 **thread-safe 하지 않고 동기화되있지 않습니다.** 

따라서 조회상황에서도 동기화 작업이 일어나기 떄문에 ArrayList보다 성능이 떨어집니다.

## HashMap 과 HashTable 차이점

`HashMap`은 key와 value에 null을 허용하고 동기화를 보장하지 않아 thread-safe 하지 않습니다. 따라서 `싱글 쓰레드 환경`에서 사용하는 게 좋습니다.

`HashTable`은 key와 value에 null을 허용하지 않으며, 동기화를 보장해 thread-safe합니다. `멀티 쓰레드 환경`에서 사용할 수 있습니다.

`HashTable`이 아닌 `ConcurrentHashMap`을 멀티쓰레드환경에 써야하는이유는 Entry에 대해서만 락을 걸기때문에  데이터를 다루는 속도가 빠릅니다.


## HashSet 과 HashMap 차이점

`HashSet`과 `HashMap` 모두 해싱을 사용하여 요소를 빠르게 저장하고 검색하는 자료구조 입니다. 두 컬렉션 모두 **해시 테이블을 사용**하여 요소를 저장하지만 다음과 같은 차이점이 있습니다.

`HashSet`은 Set 인터페이스를 구현했으며, 고유한 요소를 저장하는 컬렉션입니다. 즉, 중복된 값을 허용하지 않습니다. 따라서 이미 있는 요소를 추가하려고 하면 해당 요소가 추가되지 않습니다.
내부적으로 `HashMap`을 사용하여 **요소의 해시코드를 키로 사용하여 해시 테이블에 저장**됩니다. 이를 통해 요소를 추가, 제거 및 검색 속도가 빠릅니다.

`HashMap`은 `Map`인터페이스를 구현했으며 해시테이블에 키-값 쌍을 저장하는 자료구조입니다. 이를 통해 키를 기반으로 값에 액세스할 수 있으며 상수시간을 보장합니다. `HashMap`에서 키는 고유하지만 값은 중복될 수 있습니다.
그리고 중복된 키를 추가하려고 하면 해당 키와 연결된 값이 새 값으로 덮어쓰여집니다. 

또한 `HashSet`과 `HashMap`은 서로 다른 키가 동일한 인덱스로 해시가 될 때 `Seperate Chaining`방식으로 동일한 인덱스, 즉 한 버킷에 담겨진 데이터가 6개일때는 연결리스트, 8개일때는 레드-블랙 트리로 저장함과 보조 해시함수를 통해 충돌을 해결합니다. 

`HashMap과 HashSet`에서는 요소가 특정 순서로 저장되지 않습니다. 이는 요소를 반복할 때 해당 요소를 저장한 순서가 보장되지 않음을 의미합니다.

### HashMap 시간복잡도

HashMap의 시간 복잡도는 **기본적으로 O(1)**입니다. 이는 해시 함수를 통해 키의 저장 위치를 바로 찾아갈 수 있기 때문입니다.  하지만 실제로는 **해시 충돌이 발생할 수 있습니다.**

이런 충돌을 처리하기 위해 체이닝 방식을 사용합니다. 즉, 같은 버킷에 여러 데이터가 들어가면 **링크드 리스트 형태로 연결**합니다.
이 경우, 최악의 상황에서는 모든 데이터가 하나의 버킷에 들어가 **O(n)의 시간 복잡도**를 가질 수 있습니다.

그런데, 자바 8부터는 **하나의 버킷에 데이터가 8개 이상 쌓이면, 그 구조를 링크드 리스트에서 레드-블랙 트리로 바꿉니다**. 
레드-블랙 트리는 균형 이진 탐색 트리의 일종으로, 이를 통해 최악의 경우에도 **O(log n)의 시간 복잡도를 보장**할 수 있게 됩니다.


## hashcode() 메소드란?

- Java의 hashCode 메소드는 Object 클래스의 내장 함수입니다. 
- Java의 모든 클래스는 이를 사용하거나 재정의할 수 있습니다. 
- Object 클래스의 native hashCode() 메소드는 객체의 메모리 주소를 기반으로 정수값을 생성합니다. 
- 객체마다 고유한 해시값을 생성하는 것이 목적입니다.
- 이 정수는 무조건 고유하지 않지만 해시 충돌을 최소화하는 데 도움이 되는 방식으로 생성됩니다.

### equals()를 재정의 했을때 hashcode()도 반드시 재정의 해야하는 이유

equals()를 재정의 하는 이유는 동일성과 동등성을 보장받아 같은 객체임을 알려주기 위함입니다. 만약 equals만 재정의를 했다면 hashcode()의 값은 서로 다를 것입니다.
이 말은 즉, `HashMap`이나 `HashSet`에서 해당 객체들을 저장한다면 서로 다른 객체로 판단하여 본인 해시테이블에 두 객체 모두를 저장합니다.

이는 우리가 원하는 의도가 아니기 때문에 반드시 `HashCode`도 재정의하여 동일성을 같는 객체는 hashcode도 값이 같도록 해야합니다. 

반대로, equals 메소드에 따라 동일하지 않을 경우에는 반드시 hashcode도 달라야할 필요는 없습니다. 다만, 고유한 정수를 반환하게 한다면 성능은 더 좋을 수 있습니다. 


## Set과 Map의 타입이 Wrapper Class가 아닌 Object를 받을 때 중복 검사는 어떻게 할건가요

hashCode() 메소드를 오버라이딩하여 리턴된 해시코드 값이 같은지를 보고 해시코드 값이 다르다면 다른 객체로 판단하고,해시코드 값이 같으면 equals() 메소드로 다시 비교합니다. 
이 두 개에서 같은 객체로 판단되면 중복 객체입니다.

```java
if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
```

## Map에 저장할 객체가 Comparable 인터페이스를 구현하면 성능이 좋아지는 이유

`HashMap`은 버킷에 담겨진 데이터의 개수가 많아질 때(6개 -> 8개) LinkedList 구현에서 Red-Black Tree로 구성합니다. 이때 각 노드들이 ordering이 되야하는데,
즉 정렬을 위해서는 객체를 서로 비교해야한다는 것입니다. 그때 해당 객체가 `comparable`인터페이스를 구현하지 않았을 경우, 내부에서 자체 비교함수를 호출합니다.
근데 이 비교함수에는 연산이 더 많기 때문에 성능이 떨어집니다. 따라서 `comparable`인터페이스를 구현한 클래스로 만들면 좋습니다.


## Comparable 과 Comparator 차이

객체를 비교하기 위해 Comparable 인터페이스나 Comparator 인터페이스를 구현해야 합니다.

`Comparable` 인터페이스는 compareTo(T o) 메소드를 구현해야 하며, 본인 인스턴스와 매개변수 객체를 비교하기 위한 것이며,
`Comparator` 인터페이스는 compare(T o1, T o2) 메소드를 구현해야 하며, 두 매개변수 객체를 비교하는 것입니다.

`compareTo`를 구현할 때, 특정 값을 비교해서 정수를 반환해야 합니다. 즉, 자기자신을 기준을 삼아 대소관계를 파악해야 합니다. 
자기자신이 매개변수의 값보다 크면 양수, 같으면 정수, 작으면 음수를 반환합니다. 

```java
	@Override
	public int compareTo(Student o) {
	//	return this.age - o.age; // 이렇게 하면, 값의 범위를 넘어서 1 - (-2,147,483,648) 양수가 아닌 음수가 나옵니다.
    
        if(this.age > o.age) {
            return 1;
        }
        else if(this.age == o.age) {
            return 0;
        }
        else {
            return -1;
        }
    }
```

`compare(T o1, T o2)`를 구현할 때는 A 객체 내부에서 T o1, T o2인 T 타입의 객체끼리의 비교이기 때문에 A객체는 아무런 상관이 없습니다. 그리고 선행원소 후행원소를 비교해
양수,0,음수를 반환하면 됩니다. 

이 compare를 활용하는 방법은 익명 객체를 사용하면 됩니다. 

```java
Comparator<Student> comp1 = new Comparator<Student>() {
    @Override
    public int compare(Student o1, Student o2) {
        return o1.classNumber - o2.classNumber;
    }
};
```

**정렬의 관계를 살펴보면** 자바에서 정렬은 오름차순을 디폴트로 둡니다. 오름차순이라는 것은 선행원소가 후행원소보다 작다는 것이고 
이 둘을 compareTo나 compare로 두개의 원소를 비교했을 때 음수를 반환하면 정렬를 구현한 클래스 입장에서는 오름차순으로 판단하여 두 원소의 위치를 바꿀 필요가 없다는 것을 알 수있습니다.

만약, compareTo나 compare로 두개의 원소를 비교했을 때 양수가 나오면 선행원소가 후행원소보다 크다는 뜻이고 오름차순으로 설정된 sort 구현부에서 두 개의 위치를 바꿀 것 입니다. 