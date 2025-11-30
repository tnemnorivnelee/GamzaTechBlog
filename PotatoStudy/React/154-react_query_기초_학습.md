## 새 프로젝트를 시작하며

새로운 프로젝트를 준비하면서 문득 이런 생각이 들었다. "나는 React Query를 '제대로' 이해하고 쓰고 있는 걸까?" 단순히 API를 호출하는 용도로만 쓰기엔, 이 라이브러리가 해결하려는 본질적인 문제가 무엇인지 제대로 모르는 것 같았다.

특히 `staleTime`이나 `gcTime` 같은 옵션들은 대충 기본값으로 두고 넘어갔었는데, 이번 기회에 **"왜 이런 옵션이 존재하는지"**, **"useEffect로 짜는 것과 근본적으로 뭐가 다른지"**를 확실히 정리하고 싶었다. 그래서 공식 문서를 다시 읽고, 여러 아티클을 찾아보며 정리한 내용을 글로 남긴다.

---

## 1. React Query, 일단 써보자
![Uploaded Image](https://gamzatech-bucket.s3.ap-northeast-2.amazonaws.com/post-images/154/dedc627c-726a-4887-9b22-2900da64a3ae_image.png)

React Query를 한마디로 정의하자면 **서버 상태 관리**를 개발자 대신 해주는 라이브러리다. 

### 왜 서버 상태와 클라이언트 상태를 분리해야 할까?

과거에는 Redux나 MobX 같은 전역 상태 관리 라이브러리에 모든 상태를 몰아넣어 관리했다. 하지만 **'서버 상태(Server State)'** 와 **'클라이언트 상태(Client State)'** 는 본질적으로 다른 특성을 가지고 있다.

**클라이언트 상태란?**
- 우리 앱이 완전히 제어할 수 있는 상태
- 예: 모달 열림/닫힘, 폼 입력값, 탭 선택 상태, 다크모드 on/off
- 특징: 초기값을 우리가 정하고, 언제 어떻게 바뀔지 완전히 예측 가능

**서버 상태란?**
- 서버에 저장되어 있고, 우리는 그걸 '빌려오는' 상태
- 예: 게시글 목록, 유저 프로필 정보, 댓글 데이터, 좋아요 수
- 특징: 
  - **우리가 소유하지 않음** - 다른 사용자가 언제든 수정할 수 있음
  - **비동기적** - 네트워크를 통해 가져와야 하므로 시간이 걸림
  - **항상 최신 상태가 아닐 수 있음** - 다른 곳에서 데이터가 바뀌었을 수도 있음

**왜 섞어서 관리하면 문제일까?**

Redux로 게시글 목록을 관리한다고 가정해보자:
```tsx
// Redux로 서버 상태 관리 시
const postsSlice = createSlice({
  name: 'posts',
  initialState: {
    data: [],
    loading: false,
    error: null,
    lastFetched: null, // 마지막 조회 시간
    isStale: false,    // 데이터가 오래되었는지
  },
  reducers: {
    fetchPostsStart: (state) => { state.loading = true; },
    fetchPostsSuccess: (state, action) => { 
      state.data = action.payload;
      state.loading = false;
      state.lastFetched = Date.now();
    },
    fetchPostsFailure: (state, action) => { 
      state.error = action.payload;
      state.loading = false;
    },
    markAsStale: (state) => { state.isStale = true; },
  },
});

// 실제 사용
dispatch(fetchPostsStart());
try {
  const posts = await fetchPosts();
  dispatch(fetchPostsSuccess(posts));
} catch (error) {
  dispatch(fetchPostsFailure(error));
}

// 5분마다 데이터 상했다고 표시해야 함
setInterval(() => dispatch(markAsStale()), 5 * 60 * 1000);

// 다른 곳에서 게시글을 작성했으면 목록도 다시 불러와야 함
dispatch(fetchPostsStart()); // 또 호출...
```

문제가 보이나요?

1. **보일러플레이트 폭발**: loading, error, data, lastFetched... 상태 관리 코드가 너무 많음
2. **캐싱 로직을 직접 구현**: 언제 다시 받아올지, 얼마나 보관할지 전부 직접 관리
3. **중복 요청 방지를 직접 처리**: 같은 데이터를 여러 컴포넌트에서 동시에 요청하면?
4. **데이터 동기화 문제**: A 컴포넌트에서 수정했는데 B 컴포넌트는 어떻게 업데이트?

반면 클라이언트 상태는 훨씬 간단하다:
```tsx
// 클라이언트 상태 - 간단명료
const [isModalOpen, setIsModalOpen] = useState(false);
const [selectedTab, setSelectedTab] = useState('home');
```

**React Query를 사용하면?**
```tsx
// 서버 상태는 React Query에게
const { data: posts, isLoading, error } = useQuery({
  queryKey: ['posts'],
  queryFn: fetchPosts,
  staleTime: 30 * 1000, // 30초는 신선하다고 판단
  gcTime: 5 * 60 * 1000, // 5분간 캐시 유지
});

// 클라이언트 상태는 그대로 useState나 Zustand 등으로
const [isModalOpen, setIsModalOpen] = useState(false);
```

- **캐싱, 리페칭, 중복 제거** 등은 React Query가 알아서 처리
- **서버 상태 관리 로직** 없이 깔끔한 코드
- 개발자는 **UI와 클라이언트 상태**에만 집중

결국 React Query는 "서버에서 빌려온 데이터"의 특수성을 이해하고, 그에 맞는 최적화된 관리 방법을 제공하는 도구다.

### 1-1. 기본 세팅

가장 먼저 앱의 최상단을 `QueryClientProvider`로 감싸주어야 한다.

```tsx
// App.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient();

export default function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <MainComponent />
    </QueryClientProvider>
  );
}
```

### 1-2. 데이터 조회: useQuery

데이터를 가져올 때(GET)는 `useQuery` 훅을 사용한다. 과거에는 `loading`, `error`, `data` 상태를 각각 `useState`로 만들어 관리했지만, React Query는 이 모든 것을 기본으로 제공해준다.

```tsx
import { useQuery } from '@tanstack/react-query';
import axios from 'axios';

function TodoList() {
  const { data, isLoading, isError } = useQuery({
    queryKey: ['todos'], // 데이터의 고유 식별자
    queryFn: () => axios.get('/todos').then(res => res.data),
  });

  if (isLoading) return <div>로딩 중...</div>;
  if (isError) return <div>에러 발생</div>;

  return (
    <ul>
      {data.map(todo => <li key={todo.id}>{todo.title}</li>)}
    </ul>
  );
}
```

### 1-3. 데이터 수정: useMutation

데이터를 생성, 수정, 삭제(POST, PUT, DELETE)할 때는 `useMutation`을 사용한다. 

여기서 핵심은 **수정이 성공한 직후**다. 데이터가 바뀌었으니, 기존에 화면에 보여주던 데이터는 '낡은 데이터'가 된다. 따라서 `invalidateQueries`를 통해 캐시를 무효화하여 자동으로 최신 데이터를 다시 받아오게 해야 한다.

```tsx
const queryClient = useQueryClient();

const mutation = useMutation({
  mutationFn: (newTodo) => axios.post('/todos', newTodo),
  onSuccess: () => {
    // 'todos' 키를 가진 쿼리를 무효화 -> 즉시 데이터를 다시 받아옴
    queryClient.invalidateQueries({ queryKey: ['todos'] });
  },
});
```

---

## 2. "그냥 useEffect 쓰면 안 되나?"

기본 사용법은 익혔지만, 문득 근본적인 의문이 들었다. "그냥 `useEffect` 안에서 fetch하고 `useState`에 담으면 되는데, 왜 굳이 라이브러리를 써야 하지?"

단순히 코드가 짧아져서가 아니다. **리액트의 렌더링 사이클과 사용자 경험(UX) 관점에서 결정적인 차이**가 있기 때문이다.

### useEffect 방식의 문제점

예를 들어 `productId`가 바뀔 때마다 상품 정보를 가져오는 페이지를 생각해보자.

```tsx
// useEffect 사용 예시
useEffect(() => {
  setIsLoading(true);
  fetch(`/products/${productId}`).then(data => {
    setData(data);
    setIsLoading(false);
  });
}, [productId]);
```

이 코드는 다음과 같은 순서로 실행된다:

> 1. **렌더링**: `productId`가 바뀌어 컴포넌트가 다시 그려진다. 이때 데이터가 없으므로 화면이 깜빡이거나 빈 화면이 노출된다.
> 2. **Effect 실행**: 화면이 다 그려진 **후에야** `useEffect`가 실행되어 API 요청을 보낸다.
> 3. **데이터 도착 & State 변경**: 데이터를 받아와 `setState`를 실행한다.
> 4. **또 렌더링**: 상태가 바뀌었으니 화면을 다시 그린다.

즉, 데이터를 가져오는 시점이 **'화면 렌더링 이후'** 로 밀리게 된다. 매번 빈 화면이나 로딩 스피너를 먼저 보여주게 되어, 사용자에게 미세한 깜빡임을 주어 UX를 해친다.

### React Query의 해결책

반면 React Query는 컴포넌트가 데이터를 **'구독(Subscription)'** 하는 개념으로 동작한다.

> 1. `productId`가 바뀌면 React Query는 먼저 내부 **캐시**를 조회한다.
> 2. 캐시에 데이터가 있다면? **로딩 없이 즉시** 그 데이터를 보여준다.
> 3. 그리고 뒤에서는 조용히 서버에 요청을 보내 최신 데이터를 받아오고, 데이터가 다르면 그때 살짝 바꿔치기한다.

결과적으로 개발자가 캐싱 로직이나 중복 요청 방지, 로딩 상태 관리에 머리를 싸매지 않아도, 사용자에게 깜빡임 없는 자연스러운 경험을 제공할 수 있게 된다.

---

## 3. 알아야 할 핵심 옵션

이제 "왜 써야 하는지"는 이해했다. 그렇다면 실제로 어떻게 효과적으로 사용할 수 있을까? 실무에서 알면 좋을 것 같은 옵션들을 정리했다.

### 3-1. QueryKey: 의존성 배열이자 식별자

`queryKey`는 `useEffect`의 **의존성 배열**과 역할이 똑같다. 키에 포함된 값이 바뀌면 쿼리가 자동으로 다시 실행된다.

```tsx
// filter나 sort가 바뀔 때마다 React Query가 감지하고 다시 fetch함
useQuery({
  queryKey: ['todos', filter, sort], 
  queryFn: () => fetchTodos(filter, sort),
});
```

하지만 `queryKey`는 단순히 의존성 배열 이상의 역할을 한다. React Query의 **캐시 시스템 전체를 관리하는 핵심 식별자**이기 때문이다.

#### QueryKey의 계층적 설계

`queryKey`는 배열 형태로 작성하는데, 이를 **계층적으로 설계**하면 나중에 캐시 무효화나 관리가 훨씬 편해진다.

```tsx
// ❌ 나쁜 예: 평면적인 키 설계
['todos']
['todoDetail']
['userTodos']

// ✅ 좋은 예: 계층적인 키 설계
['todos']                    // 모든 할일 목록
['todos', 'list']            // 할일 목록
['todos', 'detail', 1]       // 1번 할일 상세
['todos', 'user', 'john']    // john의 할일 목록
```

왜 이렇게 해야 할까? `invalidateQueries`와 함께 사용할 때 진가가 드러난다.

#### QueryKey와 invalidateQueries의 유기적 동작

새 할일을 추가했다고 가정해보자. 이때 어떤 캐시를 무효화해야 할까?

```tsx
const mutation = useMutation({
  mutationFn: (newTodo) => axios.post('/todos', newTodo),
  onSuccess: () => {
    // 방법 1: 모든 todos 관련 쿼리를 무효화
    queryClient.invalidateQueries({ queryKey: ['todos'] });
    
    // 방법 2: 특정 쿼리만 무효화
    queryClient.invalidateQueries({ queryKey: ['todos', 'list'] });
  },
});
```

**핵심**: `invalidateQueries`는 **접두사(prefix) 매칭**으로 동작한다.

- `queryKey: ['todos']`로 무효화하면 → `['todos']`, `['todos', 'list']`, `['todos', 'detail', 1]` 등 **todos로 시작하는 모든 쿼리**가 무효화된다.
- `queryKey: ['todos', 'list']`로 무효화하면 → `['todos', 'list']`만 무효화되고, `['todos', 'detail', 1]`은 그대로 유지된다.

#### 실전 예시: 게시글 CRUD

일반적인 블로그 시스템에서 사용할 수 있는 `queryKey` 설계 패턴이다.

```tsx
// 1. 게시글 목록 조회 (필터링, 페이지네이션 포함)
useQuery({
  queryKey: ['posts', 'list', { category, page }],
  queryFn: () => fetchPosts({ category, page }),
});

// 2. 게시글 상세 조회
useQuery({
  queryKey: ['posts', 'detail', postId],
  queryFn: () => fetchPost(postId),
});

// 3. 내가 쓴 게시글 목록
useQuery({
  queryKey: ['posts', 'my'],
  queryFn: fetchMyPosts,
});
```

이제 각 상황에서 어떻게 캐시를 무효화하는지 보자.

**케이스 1: 새 게시글 작성**

```tsx
const createMutation = useMutation({
  mutationFn: createPost,
  onSuccess: () => {
    // 모든 게시글 목록 캐시를 무효화 (전체 목록, 내 글 목록 등)
    queryClient.invalidateQueries({ queryKey: ['posts', 'list'] });
    queryClient.invalidateQueries({ queryKey: ['posts', 'my'] });
    // 상세 페이지는 그대로 유지
  },
});
```

**케이스 2: 특정 게시글 수정**

```tsx
const updateMutation = useMutation({
  mutationFn: ({ postId, data }) => updatePost(postId, data),
  onSuccess: (_, variables) => {
    // 해당 게시글 상세 캐시만 무효화
    queryClient.invalidateQueries({ 
      queryKey: ['posts', 'detail', variables.postId] 
    });
    // 목록도 무효화 (제목이 바뀌었을 수 있으니)
    queryClient.invalidateQueries({ queryKey: ['posts', 'list'] });
  },
});
```

**케이스 3: 게시글 삭제**

```tsx
const deleteMutation = useMutation({
  mutationFn: (postId) => deletePost(postId),
  onSuccess: (_, postId) => {
    // 삭제된 게시글의 상세 캐시 제거
    queryClient.removeQueries({ queryKey: ['posts', 'detail', postId] });
    // 모든 목록 캐시 무효화
    queryClient.invalidateQueries({ queryKey: ['posts', 'list'] });
    queryClient.invalidateQueries({ queryKey: ['posts', 'my'] });
  },
});
```

#### QueryKey 설계 원칙

실무에서 `queryKey`를 설계할 때 지키면 좋은 원칙들이다.

**1. 일관된 구조 유지하기**

```tsx
// ✅ 좋은 예: 명확한 계층 구조
['resource', 'action', ...params]
['posts', 'list', { filter }]
['posts', 'detail', id]

// ❌ 나쁜 예: 들쑥날쑥한 구조
['postList', filter]
['post', id, 'detail']
```

**2. 동적 파라미터는 객체로 감싸기**

```tsx
// ✅ 좋은 예: 확장성 좋음
['posts', 'list', { category: 'tech', page: 1, sort: 'latest' }]

// ❌ 나쁜 예: 순서가 중요해져서 헷갈림
['posts', 'list', 'tech', 1, 'latest']
```

**3. 무효화 범위를 고려한 설계**

```tsx
// 예: 댓글을 추가했을 때, 게시글의 댓글 목록만 무효화하고 싶다면
['posts', 'detail', postId]              // 게시글 상세
['posts', 'detail', postId, 'comments']  // 해당 게시글의 댓글들

// 댓글 추가 시
queryClient.invalidateQueries({ 
  queryKey: ['posts', 'detail', postId, 'comments'] 
});
// 이렇게 하면 게시글 상세 내용은 그대로 유지되고, 댓글만 새로 받아옴
```

---

### 3-2. staleTime vs gcTime 

React Query를 처음 쓸 때 가장 많이 헷갈렸던 두 가지 개념이다. 쉽게 비유하자면 음식물의 **'유통기한'** 과 **'냉장고 보관 기간'** 관계와 같다.

#### staleTime (유통기한 / 기본값: 0)

- 데이터가 **"신선하다(fresh)"** 고 인정해 주는 시간이다.
- 기본값이 0이므로, 데이터를 받아오자마자 "상했다(stale)"고 판단한다. 
- 그래서 창을 다시 포커스하거나 컴포넌트가 다시 마운트되면 무조건 서버에 다시 요청한다.
- 서버 부하를 줄이려면 이 값을 적절히 늘려주면 된다. (예: `60 * 1000` = 1분 동안은 재요청 안 함)

#### gcTime (냉장고 보관 기간 / 기본값: 5분)

- **"메모리에 저장하는 시간"** 이다. (Garbage Collection Time)
- 컴포넌트가 언마운트(화면에서 사라짐) 되어도, 이 시간 동안은 데이터가 메모리에 남아있다.
- 사용자가 5분 안에 다시 해당 페이지로 돌아오면? 로딩 없이 캐시된 데이터를 먼저 보여준다.

#### 두 옵션이 함께 작동하는 방식

이 두 옵션의 관계를 제대로 이해하려면 React Query가 데이터를 다루는 전체 흐름을 봐야 한다.

**시나리오: 게시글 목록 페이지 → 상세 페이지 → 다시 목록 페이지**

```tsx
// staleTime: 30초, gcTime: 5분으로 설정
useQuery({
  queryKey: ['posts'],
  queryFn: fetchPosts,
  staleTime: 30 * 1000, // 신선한 시간, 유통기한
  gcTime: 5 * 60 * 1000, // 보관 기간
});
```

| 시점 | 경과 시간 | 상태 | React Query의 동작 |
|------|----------|------|-------------------|
| 최초 진입 | 0초 | Fresh | 서버에서 데이터를 받아온다. 30초 동안 신선 상태 유지 |
| 상세 페이지 이동 | 10초 | Fresh | 목록 컴포넌트 언마운트, 데이터는 메모리에 유지 |
| 목록으로 복귀 | 15초 | Fresh | 캐시 데이터를 즉시 표시, 서버 요청 없음 ✨ |
| 다시 상세 이동 후 복귀 | 40초 | Stale | 캐시 데이터를 먼저 표시 → 백그라운드에서 서버 요청 |
| 한참 뒤 복귀 | 6분 | 캐시 없음 | 캐시가 메모리에서 삭제됨, 로딩 후 새로 받아옴 |

**핵심 포인트**:
- `staleTime`은 "언제 서버에 재요청할지"를 결정한다.
- `gcTime`은 "언제까지 캐시를 메모리에 보관할지"를 결정한다.
- `staleTime < gcTime`이어야 의미가 있다. (gcTime이 더 짧으면 캐시가 먼저 사라져버린다)

---

### 3-3. enabled: 조건부 실행

`useEffect` 안에 if문을 넣는 것과 같다. 특정 조건이 만족될 때만 쿼리를 실행하고 싶을 때 사용한다.

```tsx
// 1. 유저 정보를 먼저 가져온다
const { data: user } = useQuery({ queryKey: ['user'], ... });

// 2. 유저 ID가 있을 때만 게시글을 가져온다
const { data: posts } = useQuery({
  queryKey: ['posts', user?.id],
  queryFn: fetchPosts,
  enabled: !!user?.id, // user.id가 존재할 때만 실행
});
```

이 패턴은 **순차적으로 데이터를 받아와야 할 때** 특히 유용하다. 첫 번째 요청의 결과가 있어야 두 번째 요청을 보낼 수 있는 경우 말이다.

---

## 4. 마무리

정리하자면, React Query는 단순히 fetch 코드를 줄여주는 유틸리티가 아니다.

- **서버 상태와 클라이언트 상태를 완벽히 분리**해주고,
- **캐싱 전략을 통해 불필요한 네트워크 요청을 줄이고** UX를 개선하며,
- **복잡한 비동기 상태 관리를 추상화**해주는 강력한 도구다.

이번에 React Query를 제대로 공부하면서 가장 크게 깨달은 점은, 단순히 API 호출 도구로만 쓰기엔 너무 아까운 라이브러리라는 것이었다. 캐싱, 리페칭, 에러 처리 같은 반복적인 작업에서 해방되니 정작 중요한 UI/UX 개선에 집중할 수 있다.

단순히 `useQuery`, `useMutation` API만 사용하는 것을 넘어, 위에서 다룬 '캐싱 라이프사이클'과 'queryKey 설계 전략'을 제대로 이해하고 나니 이제 새 프로젝트에서 React Query를 제대로 활용해볼 수 있을 것 같다.