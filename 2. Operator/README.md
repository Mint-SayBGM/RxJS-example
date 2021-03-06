# 연산자

RxJS의 연산자는 기본적으로 함수 형태로 제공이 된다.

map이나 filter와 같은 여러 값들을 취급할 수 있는 연산자를 제공한다.

연산자의 일부는 개발자가 작성한 함수를 인자로 사용하여 동작하는 것들도 있다.

## 연산자를 사용하기 위해서

연산자를 사용하기 위해서는 일단 옵저버블을 생성해야한다. 따라서 옵저버블을 생성하는 함수나 이미 생성된 옵저버블 인스턴스에서 pipe 함수로 값을 다룰 연산자들이 필요하다.

## 그럼 연산자를 사용해보자

예를 들어 ```Observable.create```는 옵저버블을 생성하는 일반적인 생성함수이다. 여기서 Rxjs에서 불러온 생성함수로는 1개의 값만 발행하는 옵저버블을 생성하는 of, 특정 범위의 값을 순서대로 발행하는 옵저버블을 생성하는 range등이 있다.

RxJS에서는 일정 시간마다 0부터 1씩 증가하는 값을 return 해주는 옵저버블을 만드는 ```interval``` 함수가 있다. 이를 파이퍼블 연산자인 ```filter```를 각각 ```rxjs```와 ```rxjs/operators```에서 불러와서 ```pipe``` 함수 안에서 사용한다.

```javascript
const { interval } = require('rxjs');
const { filter } = require('rxjs/operators');

let divisor = 2;

interval(1000).pipe(
  filter((value) => value % divisor == 0),
).subscribe((value) => console.log(value))
```

RxJs의 연산자는 순수함수<sup id="a1">[1](#footnote1)</sup> 형태이다. 다만 인자로 사용하는 함수가 순수 함수가 아니라면 이 연산자의 동작은 순수 함수가 아니다.

## 파이퍼블 연산자
파이퍼블 연산자<sup>Pipeable Operator</sup>는 생성 함수로 만들어진 옵저버블 인스턴스를 pipe 함수 안에서 다룰 수 있는 연산자다.

```<옵저버블 인스턴스>.pipe(연산자1(), 연산자2())```처럼 나열해서 사용이 가능하며 혹은 ```<옵저버블 인스턴스>.pipe(연산자1()).pipe(연산자2())```처럼 ```pipe```함수 뒤에 또 ```pipe```함수를 호출하여 사용해도 동일하게 작동한다.

기본적으로 파이어블 연산자는 ```rxjs/operators```에서 불러올 수 있고 ```range```같은 생성함수는 rxjs 아래 있다. 

다만 예외가 있다면 ```Obseravble.create```는 rxjs에서 ```Obseravble```클래스를 불러온 후 정적 메서드 형태로 호출하는 연산자다. 대표적으로 ```map```과 ```filter```가 있다.

그럼 생성 함수와 파이퍼블 연산자를 연결하는 것은 앞에서 설명했듯이 하면 될 것이다.

```javascript
const { range } = require('rxjs');
const { filter, map } = require('rxjs/operators');

range(0, 10)
  .pipe(
    filter((x) => x % 2 === 0),
    map((x) => x),
  )
```

먼저 range 함수가 옵저버블을 생성하고 pipe 함수로 filter라는 연산자 뒤에 map이라는 연산자를 연결 후 리턴한다. 

filter 연산자가 새로운 옵저버블 인스턴스를 만든 후, map이라는 파이퍼블 연산자를 연결하여 호출 할 수 있도록 하기 때문이다.

### 그럼 왜 파이퍼블 연산자를 연결하면 새로운 옵저버블 인스턴스를 생성할 수 있을까?
이는 파이퍼블 연산자의 각각의 구현을 보면 알 수 있는데 대표적으로 filter 연산자의 구현을 보게되면 Observable.js에 있는 lift 함수를 사용한다.

이 lift 함수은 다음과 같이 구현되어 있다.
```javascript
lift(operator) {
  const observable = new Observable();
  observable.source = this;
  observable.operator = operator;
  return observable;
}
```

보다시피 lift 함수는 기존 옵저버블에 영향을 주지 않는다. 기존 옵저버블은 this로 새로운 옵저버블의 source로 설정하고 source를 감싸는 연산자를 지정하여 나중에 구독할 때 어떤 연산자를 실행할지 알 수 있게끔 했다.

이는 새로운 옵저버블은 연산자를 호출할 때마다 lift 함수를 거친 옵저버블을 리던하게 된다. 즉 기존 옵저버블을 감싸서 새로운 옵저버블을 한 단계 끌어 올려주는 역할을 한다.

작성자의 생각으로는 HOC와 비슷한 기능을 하는 것이라 생각된다. 혹은 데코레이터...

### Subscribe은 어떻게 구현 되어있을까?
```subscribe```도 Observable.js에 구현 되어 있다.

```subscribe```는 연산자가 있는지 판단한 후 연산자가 있으면 연산자에 해당하는 동작을 실행한다.

#### subscribe 함수의 연산자 동작 실행 부분
```javascript
const sink = toSubscriber(observerOrNext, error, complete);

  if (operator) {                             // 연산자가 있는지 확인
    operator.call(sinkz this.source);         // 있으면 연산자 call
  }
  else {
    sink.add(this._trySubscribe(sink));
  }
```

남은 연산자가 더 없으면 ```_trySubscribe```에서 ```subscribe```함수에 위치해 있는 최종 옵저버인 ```this._subscribe(sink)```에 결과를 전달한다.

#### _trySubscribe 함수의 구현 부분
```javascript
_trySubscribe(sink) {
  try {
    return this._subscribe(sink);
  }
  catch(err) {
    sink.syncErrorThrown = true;
    sink.syncErrorValue = err;
    sink.error(err);
  }
}
```

### 이제 연산자를 연결할 때 연산자를 호출하는 옵저버블은 소스 옵저버블<sup>source observable</sup>이라 할 것이다.
왜 그렇게 부르느냐고?
```javascript
lift(operator) {
  const observable = new Observable();
  observable.source = this;
  observable.operator = operator;
  return observable;
}
```
여기 보면 기존 옵저버블인 this를 observable.source에 넣기 때문이다.

## 배열과 비교해 본 옵저버블 연산자
옵저버블 객체는 여러 값을 다룰 수 있는 함수형 연산자를 제공한다. Array#extras나 로대시에서 여러 값을 처리하는 데 제공하는 함수형 프로그래밍의 연산자와 비슷하다

아래 코드는 옵저버블 생성 예제에 ```map```이라는 변환 연산자를 적용한 것이다.

```javascript
const { Obseravable } = require('rxjs');
const { map } = require('rxjs/operators');

const observableCreated$ = Obseravble.create((observer) => {
  observer.next(1);
  observer.next(2);
  observer.complete();
})

observableCreated$.pipe(
  map((value) => value * 2)
).subscribe(
  function next(item) {
    console.log(item);
  }
)
```

이 코드를 실행하게 되면
```
2
4
```
와 같은 결과를 얻게 된다. 그럼 배열로 구현하게 되면 어떻게 될까?

```javascript
console.log([1 ,2].map((value) => value * 2));
```
```javascript
[2, 4]
```

배열과 옵저버블에서 map 연산자를 사용하는 데에 큰 차이가 없어 보이지만 옵저버블은 실제 각각의 값 처리를 구독하는 시점에 하지만 배열은 연산자를 호출할 때마다 새로운 배열을 만들기 때문에 옵저버블이 성능상으로 장점이 있다.

그럼 극단적인 사례로 성능을 비교해 보자

배열을 이용한 사례
```javascript
console.log(
  [1 ,2]
  .map((value) => value * 2)
  .map((value) => value + 1)
  .map((value) => value * 3)
  );
```
```javascript
[9, 15]
```

map 연산자를 한 번 호출할 때마다 같은 크기의 새로운 배열을 생성한다. 즉, map 연산자 수에 비례하여 배열을 생성하기 때문에 메모리 공간을 차지해 가비지 컬렉터에서 제거해야 하는 문제가 있다. 또한 배열을 생성하는 시간 비용도 발생하기 때문에 문제가 심각하다

그럼 옵저버블에서 map 연산자를 여러번 호출하는 예제를 보자

```javascript
const { Observable } = require('rxjs');
const { map, toArray } = require('rxjs/operators');

const observableCreated$ = Observable.create((observer) => {
  const arr = [1, 2];
  for(i in arr) {
    observer.next(arr[i]);
  }
  observer.complete();
})

observableCreated$.pipe(
  map((value) => value * 2),
  map((value) => value + 1),
  map((value) => value * 3),
  toArray()
).subscribe(
  function next(item) {
    console.log(item);
  }
)
```

이 코드를 실행하게 되면 ```map```함수를 사용하였을 때와 똑같은 값이 나온다.

파이퍼블 연산자인 ```map```으로 값을 감쌀 때 마다 새로운 옵저버블 객체만 생성되고 구독 전까지 실행하지 않으므로 배열이 생성도리 때처럼 실제 연산자가 동작하지 않는다.

아래 코드를 보면 어떻게 동작하는지 이해가 갈 수 있다.
```javascript
// subscribe 호출 시 toArray에서 필요한 array 생성
const array = [];

// observableCreated 안 for문에서 observer.next(arr[i])를 호출할 때
const aInput = arr[i];

// observableCreated 안 함수에서 사용하는 시작 옵저버
observerA.next(aInput);
const bInput = aInput*2;

observerB.next(aInput);
const cInput = bInput+1;

observerC.next(aInput);
const dInput = cInput*3;

observerD.next(aInput);
array.put(dInput);

// observableCreated 안에서 observer.complete를 호출할 때, toArray()에서 실행되는 동작
// subscribe 안에 있는 마지막 읍저버이기도 하다.
observerE.next(array);
```
 
 
 이런 옵저버블으니 동작 방식은 **지연 실행**이 가능하다는 장점이 있다. 배열은 map 연산자를 호출하는 순간 새로운 배열이 나오도록 동작이 바로 실행된다. 옵저버블은 구독하는 시점까지 실행을 미룰 수 있어서 지연 실행할 수 있다.

 마치 함수 호출처럼 선언은 해두었지만 실제로 호출 하기 전까지는 실행하지 않는 것처럼 말이다. 다만 함수는 return 값이 하나지만 옵저버블은 error나 complete 함수를 호출 할 때가지 next 함수로 여러 개 값을 보낼 수 있다는 차이가 있다.


RxJS 공식문서에서는 RxJS를 이벤트를 위한 로대시로 생각해보라고 설명한다.

로대시에서도 함수를 미리 합성한 후 마지막에 value라는 함수를 호출하면 연산자를 여러 번 감싸도 배열은 한 번만 생성하도록 사용할 수 있다. RxJS는 배열뿐만 아니라 이벤트나 비동기 연산에도 적용할 수 있다는 특징이 있는게 로대시와의 차별점이다.

---
[<b id="footnote1">1</b>](#a1) 부수효과가 없는 함수 즉, 어떤 함수에 동일한 인자를 주었을 때 항상 같은 값을 리턴하는 함수이자 외부의 상태를 변경하지 않는 함수