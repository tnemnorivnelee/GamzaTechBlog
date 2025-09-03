# 0. 들어가기

지난 글([\[React 스터디 1편\] Hook?? - useState 파헤치기](https://app.gamzatech.site/posts/104))에서는 React의 핵심 Hook인 `useState`를 통해 함수형 컴포넌트에 '기억력(State)'을 부여하는 방법을 알아보았습니다. `useState`가 상태를 기억하고 변경하는 역할을 한다면, 오늘은 그 상태가 '변경되었을 때' 어떤 추가적인 행동을 할 수 있게 해주는 단짝 친구, useEffect에 대해 알아보겠습니다.

상태가 바뀌면 화면이 업데이트되는 것만으로는 부족할 때가 많습니다. "이름이 바뀌었을 때 서버에 자동으로 저장하고 싶다"거나, "컴포넌트가 처음 나타났을 때 외부에서 데이터를 불러오고 싶다"와 같은 추가 작업이 필요하죠. 바로 이럴 때 `useEffect`가 등장합니다.

# 1. useEffect? '그래서' 뭘 하는 친구인가요?

![Uploaded Image](https://gamzatech-bucket.s3.ap-northeast-2.amazonaws.com/post-images/106/09af061d-c888-470d-8a3a-fd484fec8a28_image.png)

`useEffect`는 리액트 컴포넌트가 렌더링될 때마다 특정 '추가 행동(Effect)'을 수행하도록 설정할 수 있는 Hook입니다.

여기서 '추가 행동' 혹은 'Side Effect(부수 효과)'란, 컴포넌트의 주된 임무인 '화면을 그리는 것(렌더링)' 외의 모든 작업을 의미합니다.

`useState`가 컴포넌트에 '기억력'을 주었다면, `useEffect`는 그 기억이 바뀌었을 때 연결된 추가 행동을 실행하는 '약속 장치'라고 생각하면 쉽습니다.

예를 들어, 우리가 만든 '좋아요' 버튼을 다시 떠올려 봅시다.

기억력 (useState): '좋아요'를 몇 번 눌렀는지 숫자를 기억합니다.

추가 행동 (useEffect): 만약 '좋아요' 숫자가 10개를 넘으면, "인기 게시물!"이라는 알림을 띄우기로 약속합니다. useEffect는 '좋아요' 숫자를 계속 지켜보다가, 10개를 넘는 순간 약속된 행동(알림 띄우기)을 실행합니다.

## 핵심요약
`useEffect`는 컴포넌트의 '상태(State)'가 변하는 것과 같은 특정 시점에 맞춰, 우리가 약속한 '추가 행동(Effect)'을 실행해 주는 아주 유용한 도구입니다.

아래의 코드를 기반으로 더 자세히 알아봅시다.

# 2. 코드로 알아보기
이제 실제 코드를 보면서 `useEffect`가 어떻게 다양한 '추가 행동'을 약속하고 실행하는지 알아봅시다.


```javascript
import { useEffect, useState } from "react";

function App() {
  const [name, setName] = useState<string>("");
  const [nickname, setNickname] = useState<string>("");

  // 1. 렌더링 될 때마다 실행
  // useEffect(() => {
  //   console.log("컴포넌트가 렌더링 될 때마다 특정 작업을 수행합니다.");
  // });

  // 2. 마운트 될 때만 실행
  useEffect(() => {
    console.log("마운트가 될 때만 수행합니다. - 최초 1회 실시");
  }, []);

  // 3. 특정 값이 업데이트 될 때만 실행
  useEffect(() => {
    console.log("name 이라는 상태 값이 변할 경우에만 수행합니다.");
    console.log("name: ", name);
  }, [name]);
  
  // 4. 뒷정리(cleanup) 기능
  useEffect(() => {
    console.log("effect 실행");
    console.log("updated name: ", name);

    return () => {
      // 다음 effect가 실행되기 직전, 혹은 언마운트 될 때 실행
      console.log("cleanup");
      console.log("previous name: ", name);
    };
  }, [name]);

  const onChangeName = (e: React.ChangeEvent<HTMLInputElement>) => {
    setName(e.target.value);
  };

  const onChangeNickname = (e: React.ChangeEvent<HTMLInputElement>) => {
    setNickname(e.target.value);
  };

  return (
    <div>
      <input type="text" value={name} onChange={onChangeName} />
      <input type="text" value={nickname} onChange={onChangeNickname} />

      <div>
        <b>이름: {name}</b>
      </div>
      <div>
        <b>닉네임: {nickname}</b>
      </div>
    </div>
  );
}

export default App;
```

## 2-1. useEffect 의 3가지 실행 조건
`useEffect`는 두 번째 인자로 배열(Dependency Array, 의존성 배열)을 받는데, 이 배열에 무엇을 넣느냐에 따라 실행되는 시점이 달라집니다.

1. 렌더링 될 때마다 실행 (의존성 배열 생략)
 ```javascript
 useEffect(() => {
  console.log("컴포넌트가 렌더링 될 때마다 특정 작업을 수행합니다.");
});
```
두 번째 인자를 아예 전달하지 않으면, 컴포넌트가 처음 렌더링될 때와 `name`이나 `nickname` 등 어떤 상태라도 변경되어 리렌더링될 때마다 `useEffect` 내부의 코드가 실행됩니다.

2. 마운트 될 때만 실행 (빈 배열 [])
```javascript
useEffect(() => {
  console.log("마운트가 될 때만 수행합니다. - 최초 1회 실시");
}, []);
```
두 번째 인자로 빈 배열 []을 전달하면, `useEffect`는 오직 컴포넌트가 화면에 맨 처음 나타났을 때(마운트) 딱 한 번만 실행됩니다. 마치 가게를 열 때 딱 한 번 '오픈' 간판을 거는 것과 같습니다. 주로 처음 한 번만 필요한 작업, 예를 들어 외부 API를 통해 데이터를 불러오는 경우에 사용됩니다.

3. 특정 값이 업데이트될 때만 실행 (배열 안에 [값] 전달)
```javascript
useEffect(() => {
  console.log("name 이라는 상태 값이 변할 경우에만 수행합니다.");
}, [name]);
```
의존성 배열 안에 `name`과 같은 특정 상태나 props를 넣어주면, `useEffect`는 오직 그 값이 변경될 때만 실행됩니다. 위 코드에서는 사용자가 '이름' 입력창에 무언가를 입력해 `name` 상태가 바뀔 때만 콘솔이 찍히고, '닉네임' 입력창을 수정해도 이 `useEffect`는 반응하지 않습니다. React가 `name`이라는 값을 계속 지켜보다가 변화가 감지될 때만 약속을 실행하는 것이죠.

## 2-2. 뒷정리는 필수! 클린업(Cleanup) 함수
`useEffect`에는 숨겨진 강력한 기능이 하나 더 있습니다. 바로 클린업(Cleanup) 입니다.

```javascript
useEffect(() => {
  console.log("effect 실행"); // (B)

  return () => {
    console.log("cleanup"); // (A)
  };
}, [name]);
```
`useEffect` 안에서 함수를 return하면, 이 함수는 다음 두 가지 시점에 실행됩니다.

1. 컴포넌트가 화면에서 사라질 때 (언마운트)
2. **다음 Efect 가 실행되기 직전에**

이는 마치 방에 들어갈 때 불을 켜고(Effect 실행), 방에서 나가기 전에 불을 끄는(Cleanup) 것과 같습니다. 불필요한 작업이 계속 남아서 문제를 일으키는 것을 방지하는 '뒷정리' 역할이죠.

위 코드를 예로 들어, `name` 입력창에 '리액트'라고 입력하는 과정을 살펴봅시다.

1. **최초 렌더링**: `name`은 ""(빈 문자열)입니다.
    - `effect 실행` 콘솔이 찍힙니다.

2. **'리' 입력**: `name`이 `'리'`로 바뀝니다. 컴포넌트가 리렌더링됩니다.
    - `(A) cleanup` 함수가 먼저 실행됩니다. 이때 `cleanup` 함수는 이전 상태인 `name=""`을 기억하고 있습니다.
    - 그다음 `(B) effect 실행` 함수가 실행됩니다.

3. **'액' 입력**: `name`이 `'리액'`으로 바뀝니다. 컴포넌트가 리렌더링됩니다.
    - `(A) cleanup 함수`가 먼저 실행됩니다. 이번에는 이전 상태인 `name='리'`를 기억하고 있습니다.
    - 그다음 `(B) effect 실행` 함수가 실행됩니다.

이처럼 클린업 함수는 업데이트되기 직전의 값을 참조하여 '뒷정리'를 먼저 수행한 후, 새로운 Effect를 실행합니다. 이는 메모리 누수를 방지하고 애플리케이션을 안정적으로 만드는 데 매우 중요한 역할을 합니다.


# 3. 마무리

오늘 우리는 `useState`의 단짝 친구인 `useEffect`에 대해 알아보았습니다. `useEffect`는 컴포넌트의 렌더링에 맞춰 서버와 통신하거나, 타이머를 설정하는 등 다양한 '추가 행동'을 수행하게 해주는 강력한 도구입니다.

- **의존성 배열**을 통해 Effect의 실행 시점을 자유자재로 제어할 수 있고,
- **클린업 함수**를 통해 불필요한 작업이 남지 않도록 깔끔하게 뒷정리할 수 있습니다.

`useState`가 컴포넌트에 '기억'을, `useEffect`가 그 기억의 변화에 따른 '행동'을 부여합니다. 이 두 가지 Hook의 관계와 작동 방식을 이해하는 것은 인터랙티브한 React 애플리케이션을 만드는 데 가장 중요한 열쇠입니다.

다음 글에서는 불필요한 연산을 줄여 앱의 성능을 최적화해 주는 `useMemo`에 대해 자세히 알아보겠습니다!




