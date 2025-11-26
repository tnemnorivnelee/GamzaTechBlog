# Interface, Type 사용법
타입스크립트의 type과 interface 사용에 있어서 큰 차이가 없이 팀 컨벤션에 따라 정해서 사용하면 된다는 내용을 어디선가 본 기억이 있다.

그래도 어느정도 어떤 차이가 있는지 알아보기 위해 공식 문서를 살펴보기로 했다.

![Uploaded Image](https://gamzatech-bucket.s3.ap-northeast-2.amazonaws.com/post-images/151/6f4ecd23-2ae5-4264-acb5-e28f4628f1ef_image.png)

공식 문서의 Object Types 섹션을 보면, 자바스크립트의 객체 데이터 묶음을 타입스크립트로 표현하는 방법으로 다음 3가지를 소개한다.

1. 익명 방식 (Anonymous)
2. 인터페이스 방식 (Interface)
3. 타입 별칭 방식 (Type Alias)

공식 문서에도 예제와 같이 일반적인 객체 타입을 사용하는 경우에는 차이가 없으니 편한 것이나 팀 컨벤션에 따라 사용하면 된다는 이야기가 나온 것 같다.

---

# 그래도 차이점은 있다.

## 1. interface 는 선언 병합이 가능하다.

`interface`는 같은 이름으로 여러 번 선언하면 자동으로 합쳐지지만, `type`은 중복 선언 에러가 발생한다.

```typescript
// [Interface] - 가능 (자동으로 합쳐짐)
interface User {
  name: string;
}
interface User {
  age: number;
}

// 결과: User는 name과 age를 모두 가진 타입이 됨
interface User {
  name: string;
  age: number;
}

// [Type] - 불가능 (에러 발생)
type User = {
  name: string;
}
type User = { // Error: Duplicate identifier 'User'.
  age: number;
}
```

처음엔 "그냥 처음에 age까지 다 적으면 되지, 왜 굳이 나눠서 적을까?"라는 의문이 들었다.
이 기능은 내가 만든 코드가 아니라, 외부 라이브러리를 사용할 때 빛을 발한다. 예를 들어 남이 만든 라이브러리의 타입에 내가 필요한 속성을 추가해서 확장해야 하는 경우(예: window 객체 확장 등), interface의 선언 병합 기능을 활용하면 원본 코드를 건드리지 않고도 타입을 확장할 수 있다.

## 2. 확장 방식이 다르다.

타입을 확장하거나 상속받을 때 사용하는 문법이 다르다.

```typescript
// Interface -> 객체 지향의 상속(extends) 느낌
interface Animal { name: string }
interface Bear extends Animal { honey: boolean }

// Type -> 집합의 교집합(&) 느낌type Animal = { name: string }
type Bear = Animal & { honey: boolean }
```

- Interface: extends 키워드를 사용한다. "이 규격을 상속받아 확장한다"는 명확한 위계가 느껴진다.
- Type: & (Intersection) 연산자를 사용한다. 수학의 교집합처럼 "Animal의 속성과 honey 속성을 모두 만족하는 타입을 만든다"는 의미다.


## 3. 표현 범위가 다르다.

- Interface: 오직 **객체(Object)**의 형태만 정의 가능하다.
- Type: 객체뿐만 아니라 단순 원시값, 유니온(Union), 튜플 등 모든 타입을 정의할 수 있다.

```typescript
// Union Type (Interface로는 불가능)
type ID = string | number;
```

---

# 왜 과거에는 Interface를 선호했을까?

위와 같은 차이점들이 존재한다는 것을 알고 아무거나 사용하면 될 것 같다.

과거에는 interface와 type의 성능 차이가 존재해서 성능상 interface 를 사용했다고 하지만, 요즘은 type도 성능 개선이 많이 이루어져서 어떤 것을 사용해도 괜찮다고 한다.

그럼 과거에는 왜 interface가 성능이 좋았던 걸까?

추측하건데, 프론트엔드 개발에서 함수형 프로그래밍 자체가 도입이 된 것이 그렇게 오래되지 않은 것으로 알고 있다. 이전에는 백엔드 개발처럼 객체 지향 프로그래밍 방식으로 진행한 것으로 알고 있는데 이에 대해 더 자세히 알고 싶어 찾아보았다. 이유를 찾아보니 프론트엔드 개발 패러다임의 변화와 관련이 깊었다.

## 과거, 객체 지향 프로그래밍의 시대
2010년대 중반까지만 하더라도 프론트엔드 개발에서도 객체 지향 프로그래밍을 사용했다고 한다. 또, React가 유행하기 전에 구글에서 만든 Angular 라는 프레임워크는 아예 백엔드처럼 MVC(Model-View-Controller) 패턴을 따르도록 설계했기 때문에 자연스래 백엔드의 객체 지향 프로그래밍 방식이 프론트 개발에도 녹아든 것 같다.
> 실제로 지금 Angular코드는 스프링 부트와 매우 유사하다고 한다.

왜 Angular에 대한 이야기가 나왔나면,, 초기 typescript의 가장 큰 후원자가 구글이었고, 구글이 만든 프레임워크인 Angular는 객체 지향 프로그래밍 방식을 권장하고 있었기도 하며, React조차 2019년 이전까지는 클래스형 컴포넌트 방식을 이용하고 있었기 때문이다.

당시 시장 분위기가 객체 지향 프로그래밍이 기본이었기 때문에 interface가 핵심이었다. 클래스는 인터페이스를 구현하고 확장하는 것이 기본이기에 typescript 팀은 자연스럽게 가장 많이 쓰이는 interface의 처리를 최적화하는 데 집중할 수밖에 없었다.
> 초기 typescript에는 type alias가 아예 없었기도 하고, 다른 객체 지향 언어들도 interface 개념을 사용하고 있었기 때문에 그대로 가져왔다고도 한다.

그리고 또 기술적인 이유도 따로 있다고 하는데,, 이런 역사가 있었다는 정도만 알면 될 것 같다.

그 이후 2019년, React 16.8 버전으로 업데이트 되면서 Hook이라는 개념이 도입되면서 함수형 프로그래밍 방식으로 점차 넘어오게 되었다.

Hook 도입 이전까지는 상태를 관리하려면 무조건 클래스라는 큰 틀을 만들어야 했고, this라는 개념 때문에 헷갈리기도 하고 간단한 기능 하나 추가하려고 해도 복잡한 구조를 전부 이해해야 했었다.

Hooks는 필요한 기능들을 작은 조각으로 나눠서 조립할 수 있게 마치 레고 블록처럼 만들어 사용할 수 있게 되었다.

```typescript
// 예전: 클래스로 복잡하게
class Counter extends React.Component {
  constructor(props) {
    super(props);
    this.state = { count: 0 };
  }
  // ... 많은 코드
}

// Hooks: 간단하게
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

1. 훨씬 코드 작성 방식이 간결해졌다. 
	- 10줄 짜리 코드를 3줄로 구현
2. 재사용이 쉬워졌다. 
	- 로그인 체크하는 로직을 useAuth()처럼 만들어서 어디서든 가져다 쓸 수 있음

결국 Hooks는 같은 일을 더 쉽고 깔끔하게 할 수 있는 방법을 제공했고, 개발자들이 자연스럽게 함수형 프로그래밍 방식으로 넘어가게 되었다. 마치 스마트폰이 나왔을 때 다들 피처폰을 버린 것처럼..

---

# 그래서 뭘 쓰지?

외부 라이브러리나 전역 객체(window 등)의 타입을 확장해야 하는 특수한 경우를 제외하면, 서비스 비즈니스 로직을 구현할 때 선언 병합이 필요한 경우는 대부분 없다.

또한, 최근 typescript는 interface와 type의 성능 차이가 거의 없는 수준이다.

오히려 의도치 않은 병합을 막고, 유니온 타입 등 다양한 기능을 쓸 수 있는 **type**을 사용하는 것이 더 안전하고 편리할 수 있다.

일단은 위와 같은 차이점들이 있다는 것을 알아만 두고 type 방식으로 일관성을 유지해야겠다.