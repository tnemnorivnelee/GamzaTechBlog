# TypeScript를 쓰는 이유 — 정적 타입과 컴파일 타임 에러
![Uploaded Image](https://gamzatech-bucket.s3.ap-northeast-2.amazonaws.com/post-images/174/edeb392c-fd3e-4c54-aa1f-5c02f7afaea6_image.png)

## 동적 타입 vs 정적 타입

JavaScript는 **동적 타입 언어**다. 변수에 타입을 따로 선언하지 않아도 엔진이 런타임에 값을 보고 타입을 추론한다. 덕분에 코드를 빠르게 작성할 수 있지만, 개발자의 의도와 다르게 타입이 바뀌어 예측하기 어려운 버그가 생길 수 있다.

```js
let value = 42;
value = "hello"; // 아무 문제 없이 바뀜
```

TypeScript는 JavaScript의 상위 집합(superset)으로, **정적 타입 언어**다. 변수나 함수에 타입을 명시적으로 선언하고, 한번 결정된 타입은 변경할 수 없다. 코드 작성이 다소 번거롭게 느껴질 수 있지만, 그만큼 예측 가능하고 안전한 코드를 작성할 수 있다.

```ts
let value: number = 42;
value = "hello"; // 컴파일 에러 발생
```

---

## 에러를 마주하는 시점이 다르다

두 언어의 가장 큰 차이는 **에러를 언제 만나느냐**다.

### JavaScript — 런타임 에러

JavaScript는 코드가 **실행되는 시점(런타임)**에 타입이 결정된다. 즉, 브라우저에서 직접 그 기능을 실행해보기 전까지는 에러가 있는지조차 알 수 없다. 최악의 경우, 이미 배포된 서비스에서 사용자가 특정 액션을 취하는 순간 에러가 터진다.

### TypeScript — 컴파일 타임 에러

브라우저는 TypeScript를 직접 실행할 수 없다. TypeScript 코드는 실행 전에 반드시 JavaScript로 변환(컴파일/트랜스파일)되는 과정을 거친다. TypeScript는 바로 이 **컴파일 단계에서 타입 오류를 잡아낸다.**

정확히는 두 단계가 있다.

- **IDE 단계**: `tsserver`(language server)가 코드를 작성하는 즉시 타입 오류를 빨간 줄로 표시해준다.
- **컴파일 단계**: `tsc`가 JavaScript로 변환하면서 오류가 있으면 빌드를 실패시킨다.

덕분에 버그를 배포 전에 차단할 수 있고, IDE의 자동완성 기능과도 시너지가 좋다.

---

## 타입을 항상 직접 써야 할까?

TypeScript를 처음 접하면 "타입을 일일이 다 써야 해서 번거롭지 않나요?"라는 생각이 들 수 있다. 하지만 TypeScript에도 **타입 추론(Type Inference)** 기능이 있어, 값이 명확한 경우에는 타입을 굳이 명시하지 않아도 된다.

```ts
const count = 1;       // TypeScript가 number로 추론
const name = "Alice";  // TypeScript가 string으로 추론

function add(a: number, b: number) {
  return a + b; // 반환 타입도 number로 추론
}
```

타입을 직접 명시해야 하는 경우는 주로 **함수의 파라미터**, **명확하지 않은 초기값**, **복잡한 객체 구조** 등이다. 타입 추론을 잘 활용하면 타입 안정성은 유지하면서 코드 작성 부담도 줄일 수 있다.

---

## 최종 정리

JavaScript는 동적 타입 언어로, 변수의 타입이 런타임에 결정됩니다. 덕분에 코드 작성이 유연하지만, 타입 관련 버그를 실제로 실행해보기 전까지 발견하기 어렵고 최악의 경우 배포된 서비스에서 사용자가 에러를 맞닥뜨리게 됩니다.

TypeScript는 JavaScript의 상위 집합으로, 정적 타입을 지원합니다. 코드를 JavaScript로 컴파일하는 단계에서 타입 오류를 미리 잡아낼 수 있고, IDE의 language server를 통해 코드 작성 중에도 실시간으로 오류를 확인할 수 있습니다. 또한 타입 추론 덕분에 모든 곳에 타입을 직접 작성할 필요는 없으며, 값이 명확한 경우에는 TypeScript가 타입을 자동으로 추론합니다. 이러한 특성 덕분에 TypeScript는 안정성과 개발 생산성을 동시에 높이는 데 효과적입니다.