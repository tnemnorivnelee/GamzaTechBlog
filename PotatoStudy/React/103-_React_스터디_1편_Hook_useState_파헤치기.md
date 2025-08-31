# 0. 들어가기
이제 막 프로젝트를 시작한 팀들도 있고, 동아리 신입 부원 모집도 진행 중인 시점이라 많은 분들이 프론트엔드 개발 도구로 React를 선택하실 것 같습니다.

저 또한 `NEXT.js`를 주로 사용하지만, 순수 React로 프로젝트를 깊게 경험해본 적이 없어 기본 개념부터 다시 다질 겸, 프론트엔드 개발을 시작하는 분들과 지식을 공유하고자 이 시리즈를 시작합니다.

대부분 React가 SPA(Single Page Application) 방식으로 동작한다는 것은 알고 계실 텐데요, 오늘은 React의 핵심 기능인 React Hook이 무엇이고, 그중 가장 대표적인 useState에 대해 자세히 살펴보겠습니다.


# 1. React Hook? 그게 대체 뭔가요?
React Hook 은 **클래스 컴포넌트를 작성하지 않고도 함수형 컴포넌트 상태에서 상태 관리와 생명주기 기능을  사용할 수 있게 해주는 특별한 함수**입니다.

> *과거에는 상태 관리나 생명주기 API를 사용하려면 클래스형 컴포넌트를 사용해야만 했습니다.
> 
> 하지만 리액트 16.8 버전부터 훅이 도입되면서, 더 간결하고 직관적인 함수형 컴포넌트에서도 이 모든 기능들을 "빌려와서" 사용할 수 있게 되었습니다.
> 
> 이름처럼 필요한 기능에 '갈고리(Hook)'를 걸어 가져온다고 생각하면 쉽습니다.*

라고 설명을 하는데.. 저도 최근에 프론트 개발을 시작해서 와닿지가 않습니다..

우리는 대 AI 시대에 살고 있기 때문에 쉽게 설명해달라고 부탁했더니 아래와 같이 설명해줍니다.

> React Hook은 간단한 UI 부품(컴포넌트)에 특별한 능력을 붙여주는 마법 도구라고 생각하면 쉬워요.
> 여기서 UI 부품은 웹사이트의 버튼, 입력창, 프로필 카드처럼 화면을 구성하는 하나하나의 조각을 말해요.
> 리액트에서는 이런 부품들을 보통 간단한 '함수'로 만듭니다.

> 예를 들어, 우리가 '좋아요' 버튼 UI 부품을 만든다고 상상해 볼게요.
> 1.  기억력(State)이 필요해요!
    > 이 버튼을 사용자가 몇 번 눌렀는지 기억해야 해요. 그래야 '좋아요 1개'에서 '좋아요 2개'로 숫자를 바꿔서 보여줄 수 있겠죠?
    > 하지만 일반적인 자바스크립트 함수는 무언가를 기억하는 능력이 없어요. 함수가 실행되고 나면 안에 있던 정보는 전부 사라져 버리거든요.
> 2. 어떤 일이 일어난 후에 후가 행동(Effect)을 해야 해요!
    > 만약 '좋아요' 수가 10개를 넘으면, "인기 게시물!"이라는 알림을 띄우고 싶을 수 있어요.
    > 이처럼 화면이 바뀌거나, 특정 값이 변했을 때 **연결된 추가 행동**을 해야할 때가 많습니다.

> **Hook 이 바로 이 문제를 해결해 줍니다!**
> 훅은 이런 **기억력**과 **'추가행동'** 같은 능력들을 함수로 만든 UI 부품에 손쉽게 "걸어서(Hook)" 쓸 수 있게 해줘요.

## 핵심요약
**React Hook은 평범한 UI 부품(함수)에 기억력(useState)을, 추가 행동(useEffect) 같은 기능을 달아주는 아주 편리한 도구 세트입니다.**

*(와우 역시 AI 가 아주 똑똑하군요)*

아래의 코드를 기반으로 더 자세히 알아봅시다.

# 2. useState 코드로 깊게 파헤치기
이제 실제 코드를 보면서 `useState`가 어떻게 "기억력"을 부여하는지 알아봅시다.
```javascript

import { useState } from "react";

function App() {
  const [value, setValue] = useState<number>(0);
  const [name, setName] = useState<string>("빈 문자열로 할당하지 않은 name 상태값 입니다.");
  const [nickname, setNickname] = useState<string>("빈 문자열로 할당하지 않은 nickname 상태값 입니다.");

  const increment = () => setValue(value + 1);
  const decrement = () => setValue(value - 1);

  const onChangeName = (event: React.ChangeEvent<HTMLInputElement>) => setName(event.target.value);
  const onChangeNickname = (event: React.ChangeEvent<HTMLInputElement>) => setNickname(event.target.value);

  return (
    <div>
      <p>
        현재 카운터 값은: <b>{value}</b>
      </p>

      <button onClick={increment}>1 증가</button>
      <button onClick={decrement}>1 감소</button>

      <div>
        <input
          type="text"
          value={name}
          onChange={onChangeName}
        />
        <input
          type="text"
          value={nickname}
          onChange={onChangeNickname}
        />
      </div>

      <div>
        <b>이름: {name}</b>
        <b>별명: {nickname}</b>
      </div>
    </div>
  );
}

export default App;


```

## 2-1.  React의 렌더링 구조
위 코드는 `App.tsx` 파일의 내용입니다.
App 컴포넌트를 작성하여 외부에서 사용 가능하도록 export 되어 있습니다.

먼저 리액트가 이를 어떻게 실행하는지 간단하게 알아보겠습니다.

```html

index.html

<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/vite.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Vite + React + TS</title>
  </head>
  <body>
    <div id="root">
      <!-- 페이지 컴포넌트가 렌더링 될 곳입니다. -->
    </div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

리액트 앱을 생성하게 되면 최상위 폴더에 `index.html` 파일이 존재하게 됩니다.

script 태그로 스크립트 파일 `main.tsx` 를 연결하고 있습니다.

그럼 main.tsx 파일을 확인해보겠습니다.


```javascript

main.tsx

import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import './index.css'
import App from './App.tsx'

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <App />
  </StrictMode>,
)
```

`main.tsx` 코드를 보시면 드디어 App 컴포넌트가 등장합니다.

자세히는 모르겠지만, `createRoot` 함수를 이용하여 태그의 id 가 root 인 element 를 찾고 `render` 라는 함수를 이용하여 해당 태그에 렌더링 하는 것 같이 보여집니다.

그럼 다시 `index.html` 를 확인해보면,

```javascript

<body>
    <div id="root">
      <!-- 페이지 컴포넌트가 렌더링 될 곳입니다. -->
    </div>
    <script type="module" src="/src/main.tsx"></script>
</body>
```

id 가 root 인 div 태그가 있는 걸 확인할 수 있습니다.

바로 저곳에 렌더링이 되는 것 같습니다.

## 2-2. useState의 작동 원리: 리렌더링(Re-rendering)
```javascript

import { useState } from "react";

function App() {
  const [value, setValue] = useState<number>(0);
  const [name, setName] = useState<string>("");
  const [nickname, setNickname] = useState<string>("");

  const increment = () => setValue(value + 1);
  const decrement = () => setValue(value - 1);

  const onChangeName = (event:                React.ChangeEvent<HTMLInputElement>) => setName(event.target.value);
  const onChangeNickname = (event: React.ChangeEvent<HTMLInputElement>) => setNickname(event.target.value);

  return (
    <div>
      <p>
        현재 카운터 값은: <b>{value}</b>
      </p>

      <button onClick={increment}>1 증가</button>
      <button onClick={decrement}>1 감소</button>

      <div>
        <input
          type="text"
          value={name}
          onChange={onChangeName}
        />
        <input
          type="text"
          value={nickname}
          onChange={onChangeNickname}
        />
      </div>

      <div>
        <b>이름: {name}</b>
        <b>별명: {nickname}</b>
      </div>
    </div>
  );
}

export default App;
```

위 코드는 useState 를 사용한 간단한 예제 코드입니다.

useState Hook 은 어떤 역할을 할까요?
 - 함수 컴포넌트에서 상태를 관리할 수 있게 해주는 가장 기본적인 Hook.
 - useState 함수를 호출하면 상태 값과 이 값을 변경하는 함수를 배열 형태로 반환.
 - useState 함수의 매개변수에는 상태의 초기값을 설정


예시 : `const [상태값, 값 변경 함수] = useState<타입>(상태의 초기값);`

```javascript
const [value, setValue] = useState<number>(0);

const increment = () => setValue(value + 1);
const decrement = () => setValue(value - 1);

<button onClick={increment}>1 증가</button>
<button onClick={decrement}>1 감소</button>
```

useState의 가장 중요한 핵심은 **상태가 바뀌면, 컴포넌트가 다시 렌더링된다**는 것입니다.

1. 최초 렌더링: App 컴포넌트가 처음 실행될 때, useState(0)는 value에 0을 할당합니다. 화면에는 "현재 카운터 값은: 0"이 표시됩니다.

2. 이벤트 발생: 사용자가 1 증가 버튼을 클릭하면 increment 함수가 호출됩니다.

3. 상태 업데이트 요청: increment 함수 내부의 setValue(value + 1) 즉, setValue(1)이 실행됩니다.
    - 주의! 이 함수가 호출된다고 해서 value 변수가 그 즉시 1로 바뀌는 것은 아닙니다.
    - setValue는 React에게 "value 상태를 1로 바꿔달라고 요청"하고, "이 컴포넌트를 다시 렌더링 해줘!"라고 알리는 역할을 합니다.

4. 리렌더링: React는 요청을 받아 App 컴포넌트 함수를 처음부터 다시 실행합니다.
    - 이때 const [value, setValue] = useState<number>(0); 라인이 다시 실행되지만, React는 "아, 이 컴포넌트는 이미 한번 렌더링됐었지. 아까 업데이트 요청받은 최신 값 1을 줘야겠다."라고 판단합니다.
    - 따라서 이번에는 value 변수에 0이 아닌 1이 할당됩니다.

5. 화면 업데이트: 새로운 value 값 1을 포함한 JSX가 반환되고, 변경된 부분(카운터 값)이 실제 화면에 업데이트됩니다. 이제 사용자는 "현재 카운터 값은: 1"을 보게 됩니다.

이러한 '상태 변경 → 리렌더링 → 화면 업데이트' 사이클이 React의 핵심 동작 원리입니다.

# 2-3. 제어 컴포넌트 (Controlled Component)
`name`과 `nickname`을 다루는 `<input>` 태그 예제는 `useState`의 또 다른 중요한 활용법을 보여줍니다. 이를 **제어 컴포넌트**라고 부릅니다.

- `<input value={name} ... />` 처럼, React의 상태(`name`)를 HTML `input` 태그의 `value`와 직접 연결했습니다.
- 사용자가 키보드를 입력할 때마다 `onChange` 이벤트가 발생하고, `setName`함수를 통해 `name` 상태가 즉시 업데이트됩니다.
- 결과적으로, `input`에 표시되는 내용은 항상 React의 `name` 상태 값과 일치하게 됩니다.

이렇게 하면, 자바스크립트 코드(React State)가 UI(input)를 완전히 통제(제어)하게 되어 데이터의 흐름을 예측하기 쉽고 관리하기 편해집니다. 사용자의 입력값을 검증하거나, 특정 형식으로 변환하는 등의 로직을 쉽게 추가할 수 있습니다.


# 3. 마무리
오늘 우리는 React의 가장 기본적이면서도 핵심적인 Hook인 `useState`에 대해 알아보았습니다. `useState`는 단순한 함수에 불과했던 컴포넌트에 '기억력'을 부여하고, 그 기억(상태)이 바뀔 때마다 화면을 알아서 다시 그려주는(리렌더링) 역할을 합니다.

'**상태 변경 → 리렌더링 → 화면 업데이트**' 사이클이야말로 React 애플리케이션을 동적이고 인터랙티브하게 만드는 핵심 원리입니다. 이 개념을 확실히 이해하는 것이 React 개발에 익숙해지기 위한 가장 중요한 첫걸음입니다.

다음 글에서는 useState의 단짝 친구인 useEffect에 대해 자세히 알아보겠습니다!




