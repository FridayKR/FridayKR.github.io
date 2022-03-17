---
published: true
title:  "[Combine] Combining Operators"
date:   2022-03-16 23:54:50 +01:30
categories: [Combine]
tags: [zeri]
---
## 1. prepend(_:)
publisher 출력 앞에 특정 요소를 추가해야 할 때 사용

### prepend(Output…)
~~~swift
example(of: "prepend(Output...)") {
  let publisher = [3, 4].publisher
  
  publisher
    .prepend(1, 2)
    .prepend(0, 5)
    .sink(receiveValue: { print($0) })
    .store(in: &subscriptions)
}
~~~
~~~
——— Example of: prepend(Output...) ———
0
5
1
2
3
4
~~~

### prepend(Sequence)
Array와 Set도 사용할 수 있다.
주의할점은 Array와 다르게 Set은 순서가 보장되지 않는다.
~~~swift
example(of: "prepend(Sequence)") {
  let publisher = [5, 6, 7].publisher
  
  publisher
    .prepend([3, 4])
    .prepend(Set(1...2))
    .sink(receiveValue: { print($0) })
    .store(in: &subscriptions)
}
~~~
~~~
——— Example of: prepend(Output...) ———
2
1
3
4
5
6
7
~~~

### prepend(Publisher)
Publisher끼리도 붙칠수있는데 PassthroughSubject클래스를 사용하게되면 send를 통해 stream에 값을 주입할 수 있는데 broadcast라서 구독하고있는 모든 subscriber에게 값을 보내게된다.
send를 통해 값을 주입하고 나서 .finished를 사용해서 끝났다는걸 알려줘야 publisher1이 출력된다.

~~~swift
example(of: "prepend(Publisher) #2") {
  let publisher1 = [3, 4].publisher
  let publisher2 = PassthroughSubject<Int, Never>()
  
  publisher1
    .prepend(publisher2)
    .sink(receiveValue: { print($0) })
    .store(in: &subscriptions)

  publisher2.send(1)
  publisher2.send(completion: .finished)
  publisher2.send(2)
}
~~~
~~~
——— Example of: prepend(Output...) ———
1
3
4
~~~

## 2. append(_: )
이미 알고있는 append랑 같다!
PassthroughSubject를 사용할 경우 send를 통해 값을 주입하고 .finished를 사용해서 끝났다는걸 알려줘야 publisher에 append한 값이 출력된다. 끝을 알리지 않으면 추가하지 않는다.

### append(Output…)
~~~swift
example(of: "append(Output...) #2") {
  let publisher = PassthroughSubject<Int, Never>()

  publisher
    .append(3, 4)
    .append(5)
    .sink(receiveValue: { print($0) })
    .store(in: &subscriptions)
  
  publisher.send(1)
  publisher.send(2)
}
~~~
~~~
——— Example of: append(Output...) #2 ———
1
2
~~~

### append(Sequence)
prepend(Sequence)이랑 비슷

~~~swift
example(of: "append(Sequence)") {
  let publisher = [1, 2, 3].publisher
    
  publisher
    .append([4, 5])
    .append(Set([6, 7]))
    .append(stride(from: 8, to: 11, by: 2))
    .sink(receiveValue: { print($0) })
    .store(in: &subscriptions)
}
~~~
~~~
——— Example of: append(Sequence) ———
1
2
3
4
5
7
6
8
10
~~~

### append(Publisher)
publisher1이 완료되야 publisher2가 출력된다.
~~~swift
example(of: "append(Publisher)") {
  // 1
  let publisher1 = [1, 2].publisher
  let publisher2 = [3, 4].publisher
  
  // 2
  publisher1
    .append(publisher2)
    .sink(receiveValue: { print($0) })
    .store(in: &subscriptions)
}
~~~
~~~
——— Example of: append(Publisher) ———
1
2
3
4
~~~

## 3. switchToLatest
이 Operator는 새로운 publisher를 받으면 새로운 publisher를 바로 출력하고 이전 구독을 취소한다.
`이 기능을 사용하면 네트워크 요청 게시자를 생성하여 사용자 인터페이스 게시자를 자주 업데이트하는 것과 같이 이전 게시자가 불필요한 작업을 수행하지 못하도록 방지할 수 있습니다.`라고하는데 써봐야 알것같다. 그리고
`사용자가 네트워크 요청을 트리거하는 버튼을 탭합니다. 그 직후 사용자는 버튼을 다시 탭하여 두 번째 네트워크 요청을 트리거합니다. 그러나 보류 중인 요청을 제거하고 최신 요청만 사용하는 방법은 무엇입니까?`이때 유용하게 사용할 수 있다고 한다!

~~~swift
example(of: "switchToLatest") {
  let publisher1 = PassthroughSubject<Int, Never>()
  let publisher2 = PassthroughSubject<Int, Never>()
  let publisher3 = PassthroughSubject<Int, Never>()

  let publishers = PassthroughSubject<PassthroughSubject<Int, Never>, Never>()

  publishers
    .switchToLatest()
    .sink(
      receiveCompletion: { _ in print("Completed!") },
      receiveValue: { print($0) }
    )
    .store(in: &subscriptions)

  publishers.send(publisher1)
  publisher1.send(1)
  publisher1.send(2)

  publishers.send(publisher2)
  publisher1.send(3)
  publisher2.send(4)
  publisher2.send(5)

  publishers.send(publisher3)
  publisher2.send(6)
  publisher3.send(7)
  publisher3.send(8)
  publisher3.send(9)

  publisher3.send(completion: .finished)
  publishers.send(completion: .finished)
}
~~~
~~~
——— Example of: switchToLatest ———
1
2
4
5
7
8
9
Completed!
~~~

## 4. merge(with:)
이 Operator는 publisher들의 출력을 끼워넣는다?
~~~swift
example(of: "merge(with:)") {
  let publisher1 = PassthroughSubject<Int, Never>()
  let publisher2 = PassthroughSubject<Int, Never>()


  publisher1
    .merge(with: publisher2)
    .sink(
      receiveCompletion: { _ in print("Completed") },
      receiveValue: { print($0) }
    )
    .store(in: &subscriptions)

  publisher1.send(1)
  publisher1.send(2)

  publisher2.send(3)

  publisher1.send(4)

  publisher2.send(5)

  publisher1.send(completion: .finished)
  publisher2.send(completion: .finished)
}
~~~
~~~
——— Example of: merge(with:) ———
1
2
3
4
5
Completed
~~~

## 5. combineLatest
다른 publisher와 값을 결합하는데 튜플로 출력한다.
예제에서는 publisher2가 하나 이상의 값을 방출해야 publisher1이랑 결합해서 튜플로 값이 출력될 수 있다.
그래서 1은 출력이 안됨!

~~~swift
example(of: "combineLatest") {
  let publisher1 = PassthroughSubject<Int, Never>()
  let publisher2 = PassthroughSubject<String, Never>()

  publisher1
    .combineLatest(publisher2)
    .sink(
      receiveCompletion: { _ in print("Completed") },
      receiveValue: { print("P1: \($0), P2: \($1)") }
    )
    .store(in: &subscriptions)

  publisher1.send(1)
  publisher1.send(2)
  
  publisher2.send("a")
  publisher2.send("b")
  
  publisher1.send(3)
  
  publisher2.send("c")

  publisher1.send(completion: .finished)
  publisher2.send(completion: .finished)
}
~~~
~~~
——— Example of: combineLatest ———
P1: 2, P2: a
P1: 2, P2: b
P1: 3, P2: b
P1: 3, P2: c
Completed
~~~

## 6. zip
다른 두 게시자의 요소를 결합하고 요소 그룹을 튜플로 전달한다.
combineLatest랑 비슷한데 다른 점은 방출된 값들이 페어링되면 튜플로 방출된다는 점이 다르다.
즉, 먼저 방출된 값이 다른 값이랑 페어링되서 튜플로 값이 방출된다!

~~~swift
example(of: "zip") {
  let publisher1 = PassthroughSubject<Int, Never>()
  let publisher2 = PassthroughSubject<String, Never>()

  publisher1
      .zip(publisher2)
      .sink(
        receiveCompletion: { _ in print("Completed") },
        receiveValue: { print("P1: \($0), P2: \($1)") }
      )
      .store(in: &subscriptions)

  publisher1.send(1)
  publisher1.send(2)
  publisher2.send("a")
  publisher2.send("b")
  publisher1.send(3)
  publisher2.send("c")
  publisher2.send("d")

  publisher1.send(completion: .finished)
  publisher2.send(completion: .finished)  
}
~~~
~~~

~~~