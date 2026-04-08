# 이벤트 전파 (Event Propagation)

![Uploaded Image](https://gamzatech-bucket.s3.ap-northeast-2.amazonaws.com/post-images/173/d19c1fe9-10bf-461b-90c0-835c2fc90bd7_image.png)


## 이벤트 전파란?

DOM 요소에서 이벤트가 발생하면, 그 이벤트 객체는 단순히 해당 요소에서만 머물지 않는다.
**이벤트 타겟을 중심으로 DOM 트리를 따라 전파**되는데, 이 흐름은 3단계로 나뉜다.


> document
└── html
└── body
└── ul        ← 2. 버블링 (위로 전파)
└── li   ← 1. 타겟 (이벤트 발생)
↑ 0. 캡처링 (위에서 아래로)

| 단계 | 이름 | 방향 |
|------|------|------|
| 1단계 | 캡처링(Capturing) | 상위 → 하위 |
| 2단계 | 타겟(Target) | 이벤트 발생 요소 도달 |
| 3단계 | 버블링(Bubbling) | 하위 → 상위 |

---

## 1. 캡처링 (Capturing)

이벤트가 `document`에서 시작해 **이벤트 타겟 방향으로 내려오는** 단계다.

`addEventListener`의 세 번째 인자로 `true`를 전달하면 캡처링 단계에서 이벤트를 감지할 수 있다.
```js
element.addEventListener('click', handler, true); // 캡처링 단계에서 실행
```

> 실제 개발에서는 거의 사용하지 않는다.

---

## 2. 버블링 (Bubbling)

이벤트가 타겟에서 시작해 **`document` 방향으로 올라오는** 단계다.
이름처럼 거품이 물 위로 올라오는 것에서 따왔다.
```html
<ul id="parent">
  <li id="child">클릭</li>
</ul>
```
```js
document.getElementById('parent').addEventListener('click', () => {
  console.log('부모도 실행됨!'); // li를 클릭해도 여기까지 이벤트가 전파된다
});
```

`li`를 클릭했을 때 `ul`의 이벤트 핸들러까지 실행되는 이유가 바로 버블링이다.


---

## 3. 이벤트 위임 (Event Delegation)

버블링의 원리를 활용한 패턴이다.
자식 요소마다 이벤트 리스너를 붙이는 대신, **부모 요소 하나에만 리스너를 달아** 모든 자식의 이벤트를 처리한다.

### 위임 없이 (비효율)
```js
document.querySelectorAll('li').forEach(li => {
  li.addEventListener('click', handleClick); // 100개면 리스너도 100개
});
```

### 이벤트 위임 적용 (효율적)
```js
document.getElementById('parent').addEventListener('click', (e) => {
  if (e.target.tagName === 'LI') {
    handleClick(e.target);
  }
});
```

메모리 사용이 줄고, **동적으로 추가된 자식 요소**에도 자동으로 이벤트가 적용된다는 장점이 있다.

---

## 4. event.target vs event.currentTarget

이벤트 위임을 쓸 때 반드시 구분해야 하는 두 속성이다.

| 속성 | 의미 |
|------|------|
| `event.target` | 실제로 이벤트가 **발생한** 요소 (클릭한 그 요소) |
| `event.currentTarget` | 이벤트 핸들러가 **등록된** 요소 |
```js
document.getElementById('parent').addEventListener('click', (e) => {
  console.log(e.target);        // 실제 클릭된 li
  console.log(e.currentTarget); // 핸들러가 달린 ul (항상 동일)
});
```

> 이벤트 위임 패턴에서 "어떤 자식이 클릭됐는지"를 알려면 `event.target`을 사용해야 한다.

---

## 5. event.stopPropagation()

`Event` 인터페이스의 메서드로, **현재 요소 이후로의 이벤트 전파를 차단**한다.
캡처링 단계에서 호출하면 하위 전파가, 버블링 단계에서 호출하면 상위 전파가 멈춘다.

### 실제 사용 예시 — 모달 닫기 버그
```js
// 어두운 배경(부모)을 클릭하면 모달을 닫는 구현
overlay.addEventListener('click', closeModal);

// 문제: 모달 내부(자식)를 클릭해도 버블링으로 overlay까지 전파됨
// 해결: 모달 내부 클릭 시 전파 차단
modal.addEventListener('click', (e) => {
  e.stopPropagation();
});
```

> 단, `stopPropagation()`을 남용하면 다른 핸들러가 예상치 못하게 실행되지 않는 버그가 생길 수 있다.
> 꼭 필요한 곳에서만 최소한으로 사용하는 것이 좋다.

---

## 최종 정리

이벤트가 발생하면 DOM 트리를 따라 **캡처링(위→아래) → 타겟 → 버블링(아래→위)** 순서로 전파됩니다.

캡처링은 실제 개발에서 거의 사용되지 않고, 주로 버블링 특성을 활용하게 됩니다. 버블링을 이용한 대표적인 패턴이 **이벤트 위임**인데, 부모 요소 하나에 이벤트 리스너를 등록해 자식 요소들의 이벤트를 한 번에 처리함으로써 메모리를 절약하고 동적 요소에도 유연하게 대응할 수 있습니다.

이벤트 위임을 구현할 때는 `event.target`과 `event.currentTarget`의 차이를 반드시 구분해야 합니다. `event.target`은 실제로 클릭된 자식 요소를, `event.currentTarget`은 핸들러가 등록된 부모 요소를 가리킵니다.

전파를 막아야 할 경우에는 `event.stopPropagation()`을 사용하는데, 대표적인 예로 모달 내부 클릭 시 배경까지 이벤트가 전파되어 모달이 닫히는 버그를 방지할 때 활용됩니다. 다만 남용하면 의도치 않은 동작이 생길 수 있어 꼭 필요한 곳에서만 사용하는 것이 좋습니다.