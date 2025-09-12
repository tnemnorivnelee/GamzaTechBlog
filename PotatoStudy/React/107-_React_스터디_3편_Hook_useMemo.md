# 0. 들어가기

지난 글에서는 useState를 통해 컴포넌트에 '기억력'을, useEffect를 통해 그 기억의 변화에 따른 '행동'을 부여하는 방법을 알아보았습니다. useState와 useEffect가 컴포넌트의 기본적인 동작을 책임진다면, 오늘은 이 동작을 더욱 똑똑하고 효율적으로 만들어주는 친구, `useMemo`에 대해 알아보겠습니다.

컴포넌트가 렌더링될 때마다 화면을 새로 그리는 것은 물론, 내부의 함수들도 다시 호출됩니다. 만약 그중에 수천 개의 목록을 정렬하거나 복잡한 계산을 하는 함수가 있다면 어떨까요? **사용자가 입력창에 글자 하나만 입력해도 이 무거운 작업이 반복적으로 실행되어 앱 전체가 느려지는 원인이 될 수 있습니다.** 바로 이럴 때, "이 계산, 꼭 지금 다시 해야 해?"라고 물으며 불필요한 낭비를 막아주는 `useMemo`가 등장합니다.

# 1. useMemo 의 역할이 뭔데?

![Uploaded Image](https://gamzatech-bucket.s3.ap-northeast-2.amazonaws.com/post-images/107/ed689c76-ef0f-493c-8a6f-d1fd68da60c5_image.png)

`useMemo`는 계산량이 많은 함수의 **결과값을 '기억(Memoization)'**해두었다가, 필요할 때만 다시 계산하고 그렇지 않으면 기억해 둔 값을 재사용하게 해주는 성능 최적화 Hook입니다.

useState가 값 자체를 기억했다면, `useMemo`는 함수의 '계산 결과'를 기억한다는 점에서 차이가 있습니다. 마치 어려운 수학 문제를 풀고 나서, 다음에 똑같은 문제가 나오면 다시 풀지 않고 답안지에 적어둔 답을 바로 사용하는 것과 같습니다.

예를 들어, 우리가 학생들의 평균 점수를 계산하는 기능을 만든다고 상상해 봅시다.

- **기억력 (useState)**: 학생들의 점수 목록을 기억합니다.

- **계산 (함수)**: 이 점수 목록을 받아 평균을 계산합니다. (계산이 매우 복잡하다고 가정)

- **최적화 (useMemo)**: useMemo는 '학생 점수 목록'을 계속 지켜봅니다. 만약 점수 목록에 변화가 없다면, 이전에 계산했던 평균값을 그대로 다시 사용합니다. 새로운 학생이 추가되는 등 점수 목록이 실제로 바뀌었을 때만 평균을 다시 계산합니다.



아래의 코드를 기반으로 더 자세히 알아봅시다.

# 2. 코드로 알아보기

```javascript
import { useMemo, useState } from "react";

// 평균값을 구하는 함수
const getAverage = (numbers: number[]) => {
  console.log("평균값 계산 중..."); // 이 콘솔이 언제 찍히는지 주목하세요!

  if (numbers.length === 0) return 0;

  const sum = numbers.reduce((acc, cur) => acc + cur);
  return sum / numbers.length;
};


function App() {
  const [list, setList] = useState<number[]>([]);
  const [number, setNumber] = useState<string>("");

  const onInsert = () => {
    // 입력된 문자열을 숫자로 변환하여 list에 추가
    const newList = list.concat(parseInt(number));
    setList(newList);
    setNumber("");
  };

  // list가 변할 때만 getAverage 함수가 호출됩니다.
  const average = useMemo(() => getAverage(list), [list]);

  return (
    <div>
      <input type="text" value={number} onChange={(e) => setNumber(e.target.value)} />
      <button onClick={onInsert}>등록</button>

      <ul>
        {list.map((item: number, index: number) => {
          return <li key={index}>{item}</li>;
        })}
      </ul>

      <div>
        <b>평균 값: {average}</b>
      </div>
    </div>
  );
}

export default App;
```

## 2-1. 동작원리

useEffect와 마찬가지로 `useMemo`도 두 개의 인자를 받습니다.

1. 첫 번째 인자: 값을 계산해서 반환하는 '생성 함수'.

2. 두 번째 인자: 함수가 의존하는 값들이 담긴 '의존성 배열'.

```javascript
const average = useMemo(() => getAverage(list), [list]);
```

이 코드는 이렇게 동작합니다.

useMemo는 두 번째 인자인 **[list]**를 주시합니다.

만약 list의 내용이 바뀌면, useMemo는 첫 번째 인자인 () => getAverage(list) 함수를 호출하여 평균값을 다시 계산하고, 그 결과를 average 변수에 담습니다.

만약 list의 내용이 바뀌지 않았다면, 함수를 호출하지 않고 이전에 계산해서 '기억해 두었던' 평균값을 그대로 average 변수에 넣어줍니다.


#### 직접 확인하기!

코드를 실행하고 개발자 도구의 콘솔을 열어 확인해 보세요.

숫자 입력: input 창에 숫자를 입력해 보세요. number 상태가 계속 바뀌면서 컴포넌트는 리렌더링되지만, 콘솔에는 "평균값 계산 중..." 메시지가 나타나지 않습니다. list가 변하지 않았기 때문에 `useMemo`가 getAverage 함수를 호출하지 않고 이전에 기억한 값을 재사용한 것입니다.

등록 버튼 클릭: '등록' 버튼을 눌러 list에 숫자를 추가해 보세요. 이때 list 상태가 변했기 때문에 `useMemo`는 getAverage 함수를 다시 호출하고, 콘솔에 "평균값 계산 중..." 메시지가 나타나는 것을 볼 수 있습니다.

이처럼 `useMemo`를 사용하면 상태가 변경되어 리렌더링이 발생하더라도, 꼭 필요한 시점에만 무거운 계산을 하도록 제어하여 애플리케이션의 성능을 지킬 수 있습니다.


***
오늘 우리는 useState와 useEffect의 친구인 useMemo에 대해 알아보았습니다. 

`useMemo`는 매번 렌더링될 때마다 실행될 필요가 없는 무거운 계산의 결과값을 기억해두고 재사용함으로써 성능을 최적화하는 강력한 도구입니다.

**useState**는 컴포넌트에 **'기억'**을,

**useEffect**는 그 기억의 변화에 따른 **'행동'**을,

**useMemo**는 그 기억을 바탕으로 한 **'효율적인 계산'**을 가능하게 합니다.

복잡하고 인터랙티브한 애플리케이션을 만들다 보면 성능 문제는 반드시 마주하게 되는 과제입니다. useMemo의 작동 원리를 잘 이해하고 적재적소에 활용한다면, 사용자에게 훨씬 더 쾌적한 경험을 제공하는 애플리케이션을 만들 수 있을 것입니다.

다음 글에서는 useMemo와 아주 비슷한 친구이자, 함수 자체를 재사용하여 최적화를 돕는 useCallback에 대해 알아보겠습니다!

