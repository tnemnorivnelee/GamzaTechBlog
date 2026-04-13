# null, undefined, undeclared, NaN

![Uploaded Image](https://gamzatech-bucket.s3.ap-northeast-2.amazonaws.com/post-images/177/189b6b88-9ea6-4199-b898-a15eba32dc69_image.png)

자바스크립트를 쓰다 보면 "값이 없다"는 상황을 표현하는 방식이 여러 가지다. `null`, `undefined`, `undeclared`는 비슷해 보이지만 **의도와 원인이 전혀 다르고**, `NaN`은 숫자처럼 보이지만 숫자가 아닌 특이한 값이다. 면접에서도 자주 묶어서 물어보는 주제라 차이를 명확히 정리해봤다.

---

## null

`null`은 **"여기에 값이 없음"을 개발자가 의도적으로 표현한 원시값**이다.

```js
let user = null; // 아직 로그인 전, 의도적으로 비워둠
```

변수 자체가 없는 게 아니라, **"빈 값"이라는 값을 직접 할당한 것**이다.

### typeof null === 'object' ?

```js
typeof null // "object"
```

직관적으로 이상하지만, 이건 자바스크립트 초기 설계의 **버그**다. 당시 값의 타입을 판별하는 방식에서 `null`이 객체 타입 비트 패턴과 같아서 생긴 문제인데, 하위 호환성 때문에 지금도 고쳐지지 않았다.

그래서 `null` 체크는 `typeof` 대신 **일치 연산자(`===`)** 를 써야 한다.

```js
// ❌ 믿을 수 없음
typeof value === 'object' // null도 통과해버림

// ✅ 정확한 null 체크
value === null
```

---

## undefined

`undefined`는 **"값이 정의되지 않았다"는 것을 JS 엔진이 자동으로 표현하는 원시값**이다. 개발자가 의도적으로 할당하는 `null`과 달리, 대부분 **JS가 알아서 세팅**한다.

```js
let a;
console.log(a); // undefined — 선언만 하고 값을 안 넣으면 JS가 자동으로 undefined 할당
```

`var`로 선언된 변수는 호이스팅될 때 **선언과 초기화가 동시에** 일어나서 `undefined`가 된다.

```js
console.log(x); // undefined (에러 아님)
var x = 5;
```

내부적으로는 이렇게 처리되는 것과 같다.

```js
var x = undefined; // 호이스팅 시점에 자동으로
// ... 코드 실행 ...
x = 5;
```

---

## undeclared

`undeclared`는 **아예 선언 자체가 없는 변수에 접근하는 것**이다. `undefined`와 달리 JS 엔진이 해당 변수의 존재 자체를 모르기 때문에 `ReferenceError`가 발생한다.

```js
console.log(y); // ReferenceError: y is not defined
```

### undeclared vs TDZ

`let`, `const`로 선언한 변수도 호이스팅은 되지만, **선언과 초기화가 분리**되어 있어서 초기화 전에 접근하면 에러가 난다. 이 구간을 **TDZ(Temporal Dead Zone)** 라고 한다.

```js
console.log(z); // ReferenceError: Cannot access 'z' before initialization
let z = 10;
```

둘 다 `ReferenceError`지만 원인이 다르다.

| | 원인 | 에러 메시지 |
|---|---|---|
| **undeclared** | 선언 자체가 없음 | `y is not defined` |
| **TDZ** | 선언은 됐지만 초기화 전 접근 | `Cannot access 'z' before initialization` |

### typeof로 undeclared 변수를 확인하면?

```js
typeof y // "undefined" (ReferenceError 안 남!)
```

`typeof`는 선언되지 않은 변수에도 에러를 던지지 않고 `"undefined"`를 반환한다. 전역 변수 존재 여부를 안전하게 체크할 때 활용할 수 있는 quirk다.

```js
// 특정 전역 변수가 있는지 안전하게 확인
if (typeof __SOME_FLAG__ !== 'undefined') {
  // 사용 가능
}
```

---

## NaN

`NaN`은 **Not a Number**의 약자로, **수치 연산을 시도했지만 결과를 숫자로 표현할 수 없을 때** 반환되는 값이다.

```js
'abc' * 2    // NaN — 문자열을 숫자로 변환할 수 없음
0 / 0        // NaN
parseInt('hello') // NaN
```

### typeof NaN === 'number' ?

```js
typeof NaN // "number"
```

"숫자가 아님"인데 타입은 `number`다. `NaN`은 IEEE 754 부동소수점 표준에서 **"표현 불가능한 수치 결과"를 나타내는 특수값**으로 정의되어 있고, 자바스크립트는 이 표준을 따르기 때문이다. 즉, 숫자 연산의 실패 결과지만, 숫자 타입의 범주 안에 있다.

### NaN === NaN이 false?

```js
NaN === NaN // false
```

`NaN`은 **자기 자신과도 같지 않다.** IEEE 754 표준에서 의도적으로 그렇게 정의한 것으로, "이 값은 정상적인 숫자가 아니므로 비교 자체가 의미 없다"는 의미다.

덕분에 `===`로는 NaN을 잡을 수 없고, **`Number.isNaN()`** 을 써야 한다.

```js
// ❌ 동작 안 함
value === NaN // 항상 false

// ❌ 구식 방법 (전역 isNaN은 타입 변환을 먼저 해서 오작동 가능)
isNaN('abc') // true — 문자열인데도 true 나옴

// ✅ 권장
Number.isNaN(NaN)   // true
Number.isNaN('abc') // false — 타입 변환 없이 NaN만 정확히 체크
```

---

## 최종 정리

`null`은 개발자가 의도적으로 "빈 값"임을 표현할 때 직접 할당하는 원시값입니다. 반면 `undefined`는 변수를 선언했지만 값을 할당하지 않았을 때 자바스크립트 엔진이 자동으로 세팅하는 값으로, 의도가 아닌 "아직 정의되지 않은 상태"를 나타냅니다. `undeclared`는 한 단계 더 나아가 선언 자체가 없는 변수에 접근하는 것으로, `ReferenceError`가 발생합니다. `let`, `const`에서 호이스팅 후 초기화 전 접근 시 발생하는 TDZ 에러와 혼동하기 쉽지만, undeclared는 변수 존재 자체를 엔진이 모르는 경우입니다. `NaN`은 숫자 연산에 실패했을 때 반환되는 특수값으로, `typeof NaN === 'number'`이고 `NaN === NaN`이 `false`인 독특한 특성이 있습니다. 때문에 NaN 여부를 체크할 때는 반드시 `Number.isNaN()`을 사용해야 합니다.
