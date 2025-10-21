
## 왜 이 글을 쓰게 되었나

![Uploaded Image](https://gamzatech-bucket.s3.ap-northeast-2.amazonaws.com/post-images/111/ac3504aa-ed2a-42e6-9cfd-675bc8597142_image.png)

감자블 프로젝트 및 졸업 프로젝트로 Next.js App Router를 도입하면서 API 통신 방법을 찾아보니, 인터넷에는 "이제 `axios` 대신 `fetch`를 쓰세요"라는 글들이 대부분이었습니다. Next.js가 `fetch`를 확장해서 강력한 캐싱을 제공한다는 이유였죠.

그런데 실제로 프로젝트에 적용하면서 많은 의문이 생겼습니다.

- "어차피 상호작용이 많아서 대부분 클라이언트 컴포넌트인데, 그럼 `fetch` 캐싱은 의미 없는 거 아닌가?"
- "클라이언트에서 `TanStack Query`를 쓸 건데, 그럼 `fetch`가 좋다는 말이랑 무슨 상관이지?"
- "왜 인터넷엔 `axios`를 쓰지 말라는 글이 이렇게 많지?"

더 혼란스러웠던 건, 인터넷을 검색하면 "Next.js에서 `axios` 쓰지 마세요"라는 글과 "클라이언트 컴포넌트에서는 `axios`나 `ky`를 쓰면 됩니다"라는 글이 섞여 있다는 점이었습니다.

이 글은 제가 겪었던 혼란을 하나씩 해결해나간 기록입니다. 
아직 완벽히 이해했다고 할 수는 없지만, 같은 고민을 하는 분들께 제가 찾은 답을 공유하고 싶어 정리했습니다.

---

## 시작: Next.js는 왜 fetch를 강조할까?

먼저 Next.js가 `fetch`를 강조하는 이유를 이해해야 했습니다.

Next.js App Router는 **서버 컴포넌트(React Server Components, RSC)**라는 새로운 패러다임을 도입했습니다. 그리고 이 서버 컴포넌트 환경에서 Next.js는 네이티브 `fetch` API를 확장(Override)해서, 강력한 데이터 캐싱 시스템을 제공합니다.

```typescript
// 서버 컴포넌트에서의 fetch
const res = await fetch('https://api.example.com/data', {
  next: { revalidate: 10 }, // 10초마다 재검증
  // 또는
  cache: 'no-store', // 매번 새로운 데이터
});
```

이런 옵션들을 통해 Next.js는 페이지 초기 로딩 속도를 극적으로 개선하고, SEO를 위한 서버 사이드 렌더링(SSR)을 완벽하게 지원합니다. 
이것이 바로 "Next.js에서는 `fetch`를 써야 한다"는 말의 핵심이었습니다.

하지만 이 설명만으로는 제 의문이 해결되지 않았습니다. 
실제 프로젝트에서는 더 복잡한 상황들이 있었으니까요.

---

## 내가 가졌던 5가지 의문점

### 의문 1: 서버 컴포넌트에서 꼭 fetch만 써야 하나? (ky는 안되나?)

> **"Next.js 서버 컴포넌트에서는 무조건 `fetch`만 써야 한다"**

이 말을 듣고 의문이 들었습니다. `ky`도 `fetch` 기반인데, 왜 안 된다는 걸까요?

#### 해답: ky는 됩니다!

결론부터 말하면, **`axios`는 안되지만 `ky`는 됩니다.**

Next.js가 `fetch`를 강력히 권장하는 이유는 **서버 캐싱** 때문입니다. 
Next.js는 서버 환경에서 네이티브 `fetch` API를 확장(Override)해서, 강력한 캐싱 시스템과 연동되도록 만들었습니다.

```typescript
// app/page.tsx (Server Component)

// 10초마다 캐시 재검증 (ISR)
const res = await fetch('https://api.example.com/data', {
  next: { revalidate: 10 },
});

// 매번 새로운 데이터 (캐시 안 함)
const res2 = await fetch('https://api.example.com/data', {
  cache: 'no-store',
});
```

**axios는?** `axios`는 `fetch` 기반이 아니라 Node.js의 `http` 모듈이나 브라우저의 `XMLHttpRequest`를 사용합니다. 따라서 Next.js의 캐싱 시스템과 전혀 연동되지 않습니다.

**ky는?** `ky`는 `fetch`의 래퍼(Wrapper)입니다. 
내부적으로 `fetch`를 사용하기 때문에, Next.js의 캐시 옵션을 그대로 전달할 수 있습니다!

```typescript
import ky from 'ky';

// ky로도 Next.js 캐싱 옵션 사용 가능!
const data = await ky.get('https://api.example.com/data', {
  next: { revalidate: 10 },
}).json();
```

#### 배운 점

`fetch`의 날것(raw) API가 불편하지만 캐싱 기능은 쓰고 싶다면, `ky`는 서버 컴포넌트에서 훌륭한 대안이 됩니다. `fetch`의 캐싱 기능 + `ky`의 편의성을 모두 챙길 수 있습니다.

**다만 주의할 점**: `ky`를 서버 컴포넌트에서 사용할 때 완전히 문제가 없는 것은 아닙니다. Next.js의 개발 환경에서 `ky`가 간헐적으로 오류를 발생시키거나, 특정 캐시 옵션이 제대로 동작하지 않는 경우가 있다고 합니다. 이는 Next.js가 `fetch`를 확장하는 방식과 `ky`의 내부 구현 사이에서 발생하는 호환성 문제입니다.

실무에서는:
- **간단한 프로젝트나 빠른 프로토타이핑**: `ky`를 시도해볼 만하다고 생각합니다
- **안정성이 중요한 프로덕션**: 순수 `fetch`를 사용하거나, 필요하다면 `fetch`를 기반으로 한 간단한 래퍼 함수를 직접 만드는 것이 더 안전할 수 있습니다

```typescript
// fetch 래퍼 함수 예시
async function apiFetch(url: string, options?: RequestInit) {
  const res = await fetch(url, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...options?.headers,
    },
  });
  
  if (!res.ok) {
    throw new Error(`API Error: ${res.status}`);
  }
  
  return res.json();
}

// 사용
const data = await apiFetch('https://api.example.com/data', {
  next: { revalidate: 10 },
});
```

---

### 의문 2: 생각보다 서버 컴포넌트 쓸 곳이 없는데?

> **"Next.js는 서버 컴포넌트(RSC)가 핵심이다"**

이 말을 듣고 프로젝트를 시작했는데, 막상 개발하다 보니 대부분의 컴포넌트에 `'use client'`를 붙이게 되었습니다. `onClick`, `onChange`, `useState` 같은 상호작용이 필요한 순간, 서버 컴포넌트를 쓸 수가 없었거든요.

"내가 Next.js를 잘못 쓰고 있는 건가?" 싶었습니다.

#### 해답: 웹 "애플리케이션"은 당연히 그렇습니다

알고 보니 이건 지극히 정상적인 현상이었습니다.

[이전 글(의문이 들어 파헤쳐본 Next.js vs Vite+React, 그리고 내 프로젝트)](https://app.gamzatech.site/posts/110)에서 설명한 바와 같이, `fetch` 캐싱이 빛을 발하는 곳은 **블로그, 상품 페이지, 뉴스 기사처럼 정적인 '읽기' 위주의 페이지**입니다. 이런 페이지들은 SEO와 빠른 초기 로딩이 중요하고, Next.js의 서버 캐싱이 극적인 성능 개선을 가져다줍니다.

하지만 우리가 만드는 대부분의 서비스는 **사용자의 상호작용(클릭, 입력, 폼 제출)이 전부**입니다. 그리고 이런 모든 컴포넌트는 `'use client'`를 붙여야 합니다.

결국 복잡한 대시보드나 SaaS를 만든다면, 페이지의 **90%** 가 거대한 클라이언트 컴포넌트 '섬'(아니, '대륙')이 되는 경우가 많습니다.

**Next.js의 이상:**
```
RSC '껍데기' + 인터랙티브한 CC '섬'
```

**우리의 현실:**
```
RSC '껍데기' + 모든 기능이 다 들어있는 거대한 CC '대륙'
```

#### 배운 점

"RSC 쓸 곳이 별로 없다"고 느꼈다면, 그건 Next.js를 잘못 쓰는 게 아니라 **'상호작용이 많은 앱'을 만들고 있다는 지극히 현실적인 증거**입니다.

**그렇다면 이런 경우 Next.js를 쓰지 말아야 할까?**

상호작용이 많은 앱을 개발하면 어쩔 수 없이 많은 클라이언트 컴포넌트가 생겨나고, 서버 컴포넌트를 많이 사용하지 못하게 됩니다. 이런 상황에서는 두 가지 선택지가 있습니다:

**1. Next.js를 계속 사용하되, 현실을 받아들이기..**

Next.js의 가치는 서버 컴포넌트만이 아닙니다:
- **초기 로딩 페이지의 이점**: 랜딩 페이지, 로그인 페이지, 공개 페이지 등 일부라도 서버 렌더링하면 SEO와 초기 로딩 속도에서 이득을 봅니다.
- **통합 프레임워크의 편의성**: 라우팅, 이미지 최적화, API Routes 등 Next.js가 제공하는 다른 기능들도 충분한 가치가 있습니다.
- **점진적 개선 가능**: 나중에 서버 컴포넌트로 리팩토링할 수 있는 여지가 있습니다.

```typescript
// Next.js에서 하이브리드 접근
// app/page.tsx - 랜딩 페이지는 RSC로 (SEO 중요)
export default async function Home() {
  const features = await fetch('https://api.example.com/features');
  return <LandingPage features={features} />;
}

// app/dashboard/page.tsx - 대시보드는 CC로 (상호작용 중심)
'use client';
export default function Dashboard() {
  // 여기서 TanStack Query + axios 사용
  return <DashboardContainer />;
}
```

**2. Vite + React로 전환 고려하기**

만약 아래 조건에 해당한다면, 순수 React(Vite)가 더 나은 선택일 수 있습니다:
- **로그인 기반 앱/대시보드**: SEO가 전혀 필요 없는 경우
- **실시간 상호작용이 대부분**: 데이터가 계속 변경되고 사용자 입력이 주를 이루는 경우
- **개발 속도 우선**: 빠른 HMR, 짧은 빌드 시간, 단순한 아키텍처가 더 중요한 경우

이는 [ComfyDeploy가 Next.js를 포기한 이유](https://www.comfydeploy.com/blog/you-dont-need-nextjs)와도 일맥상통합니다. (이 내용은 글 후반부에서 더 자세히 다룹니다)

**결론:**
- **공개 웹사이트 + 앱이 섞여있다면** → Next.js 유지 (일부라도 RSC의 이점 활용)
- **100% 로그인 기반 대시보드라면** → Vite + React 고려
- **대학생/소규모 프로젝트라면** → 제 생각에는 Next.js 하나로 시작하는 게 좋다고 봅니다 (프로젝트를 두 개로 나누면 복잡성이 오히려 더 커질 수 있습니다)

---

### 의문 3: "Next fetch 캐싱으로도 충분한데 TanStack Query가 필요할까"?

> **"Next.js의 fetch 캐싱만으로도 충분한데 굳이 TanStack Query가 필요할까?"**

그런데 다른 글들을 보면 여전히 많은 개발자들이 TanStack Query를 사용하고 있어서 혼란스러웠습니다. 또 다른 글에서는 fetch 캐싱만으로 충분하다면서 Tanstack Query 를 쓸 필요가 없다고도 합니다. 그럼에도 불구하고 여전히 많은 프로젝트에서 Tanstack Query 가 쓰이고 있는 것 같았습니다. 

왜 여전히 TanStack Query를 쓰는 걸까요?

#### 해답: 완전히 다른 역할입니다

알고 보니 이 둘은 **대체 관계가 아니라 완전히 다른 역할**을 하고 있었습니다.

- **Next.js fetch 캐시**: **서버**의 캐시입니다. 페이지가 **'처음 로드될 때'**의 속도를 결정합니다.
- **TanStack Query 캐시**: **클라이언트(브라우저)**의 캐시입니다. 사용자가 **'상호작용할 때'**의 데이터 상태를 관리합니다.

TanStack Query가 여전히 필요한 결정적인 이유는 **데이터 변경(Mutations)**과 클라이언트 상태 관리입니다.

```typescript
'use client';

import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query';
import axios from 'axios';

function TodoList() {
  const queryClient = useQueryClient();

  // 1. 데이터 조회 (useQuery)
  const { data: todos } = useQuery({
    queryKey: ['todos'],
    queryFn: async () => {
      const res = await axios.get('/api/todos');
      return res.data;
    },
  });

  // 2. 데이터 변경 (useMutation)
  const addTodo = useMutation({
    mutationFn: async (newTodo) => {
      return axios.post('/api/todos', newTodo);
    },
    // 3. 성공 후 캐시 동기화
    onSuccess: () => {
      queryClient.invalidateQueries(['todos']);
    },
  });

  return (
    <div>
      {todos?.map(todo => <div key={todo.id}>{todo.text}</div>)}
      <button onClick={() => addTodo.mutate({ text: '새 할일' })}>
        추가
      </button>
    </div>
  );
}
```

#### TanStack Query의 핵심 가치

**CRUD 작업을 예로 들어보겠습니다:**

게시판을 만든다고 가정해봅시다. 사용자는 글을 읽고(Read), 작성하고(Create), 수정하고(Update), 삭제(Delete)할 수 있어야 합니다.

**1. useQuery (Read - 조회)**: 클라이언트에서 `isLoading`, `isError` 상태를 관리하고, `stale-while-revalidate` 같은 정교한 클라이언트 캐시 전략을 제공합니다.

```typescript
// 게시글 목록 조회
const { data: posts, isLoading, isError } = useQuery({
  queryKey: ['posts'],
  queryFn: () => axios.get('/api/posts'),
});

if (isLoading) return <div>로딩중...</div>;
if (isError) return <div>에러 발생!</div>;
```

**2. useMutation (Create, Update, Delete - 생성/수정/삭제)**: 서버 데이터를 변경하는 작업을 관리합니다. `isLoading`, `isError`, 낙관적 업데이트(Optimistic Updates) 등을 완벽하게 처리합니다.

```typescript
// 게시글 작성 (Create)
const createPost = useMutation({
  mutationFn: (newPost) => axios.post('/api/posts', newPost),
  onSuccess: () => {
    queryClient.invalidateQueries(['posts']); // 목록 자동 갱신
  },
});

// 게시글 삭제 (Delete)
const deletePost = useMutation({
  mutationFn: (postId) => axios.delete(`/api/posts/${postId}`),
  onSuccess: () => {
    queryClient.invalidateQueries(['posts']); // 목록 자동 갱신
  },
});

<button onClick={() => deletePost.mutate(postId)}>
  {deletePost.isLoading ? '삭제중...' : '삭제'}
</button>
```

**3. 캐시 동기화**: '글쓰기'(Create/Update/Delete) 성공 후, '글 목록'(Query)을 자동으로 `invalidate` 시켜 화면을 갱신하는 로직은 TanStack Query의 핵심입니다.

```typescript
// 사용자가 글을 작성하면
createPost.mutate(newPost) 
  → 서버에 POST 요청
  → 성공하면 onSuccess 실행
  → queryClient.invalidateQueries(['posts']) 호출
  → 게시글 목록이 자동으로 다시 불러와짐
  → 화면에 새 글이 즉시 표시됨
```

이 모든 과정을 `fetch`만으로 구현하려면? `useState`, `useEffect`, 수동 에러 처리, 수동 리페칭... 수십 줄의 보일러플레이트 코드가 필요합니다.

#### 배운 점

Next.js의 `fetch` 캐싱은 **서버에서의 초기 페이지 생성**에 관한 것이고, TanStack Query는 **클라이언트에서의 사용자 상호작용과 데이터 관리**에 관한 것입니다. 둘은 전혀 다른 문제를 해결합니다.

**그런데 여기서 또 다른 의문이 생겼습니다**

"서버 컴포넌트에서 HTML에 데이터를 넣어서 보냈는데, 어떻게 나중에 클라이언트에서 그 데이터를 바꿀 수 있는 거지?"

이 질문의 답을 찾다가 **Hydration(수화)**이라는 개념을 알게 되었습니다. Next.js는 다음과 같이 작동합니다:

```
1. 서버: 진짜 API 호출해서 진짜 데이터 가져옴
   ↓
2. 서버: 그 데이터를 HTML에 박아서 전송 (빠른 초기 로딩!)
   ↓
3. 서버: 동시에 같은 데이터를 JSON으로도 함께 전송
   ↓
4. 브라우저: HTML 즉시 표시 (사용자는 이미 콘텐츠 볼 수 있음!)
   ↓
5. 브라우저: JavaScript 로드 (뒷단에서 조용히...)
   ↓
6. 브라우저: Hydration - 정적 HTML을 "살아있는" React 컴포넌트로 변환
   ↓
7. 완성: 이제 버튼 클릭, 상태 변경 모두 가능!
```

**사용자 관점에서 보면:**

| 시간 | 사용자가 보는 것 | 실제로 일어나는 일 |
|------|------------------|-------------------|
| 0.1초 | "오, 페이지 떴다! 글 목록 보인다!" | 서버에서 받은 HTML 표시 (정적, 클릭 안 됨) |
| 0.2초 | "흠... 글 읽어볼까?" | JavaScript 다운로드 시작 |
| 0.5초 | "이 글 재밌네" (여전히 읽는 중) | JavaScript 파싱, React 초기화 |
| 0.6초 | "버튼 눌러볼까?" (클릭!) | **이미 Hydration 완료!** 버튼 작동 |
| 10초 | "새로고침 버튼 눌렀더니 새 글 나타남!" | TanStack Query가 새 데이터 가져와서 화면 업데이트 |

```typescript
// 서버가 보내는 HTML
<html>
  <body>
    <!-- 사용자가 즉시 보는 것 (진짜 데이터!) -->
    <div id="root">
      <ul>
        <li>게시글 1</li>
        <li>게시글 2</li>
      </ul>
      <button>새로고침</button> <!-- 아직 클릭 안 됨 -->
    </div>
    
    <!-- 같은 데이터를 JSON으로도 전송 -->
    <script id="__NEXT_DATA__">
      { "initialPosts": [{"id":1,"title":"게시글 1"}, ...] }
    </script>
    
    <!-- JavaScript 로드 (뒷단에서 조용히) -->
    <script src="/app.js"></script>
  </body>
</html>
```

```typescript
// JavaScript 실행되면
'use client';
function PostList({ initialPosts }) {
  // JSON 데이터를 React 상태로 변환
  const { data: posts } = useQuery({
    queryKey: ['posts'],
    queryFn: () => axios.get('/api/posts'),
    initialData: initialPosts, // ← 서버에서 받은 초기 데이터
  });
  
  // 이제 자유롭게 수정 가능!
  const handleRefresh = () => {
    queryClient.invalidateQueries(['posts']);
  };
  
  return (
    <ul>
      {posts.map(post => <li>{post.title}</li>)}
    </ul>
  );
}
```

**핵심: 사용자는 모르게 "정적 HTML → 동적 React 컴포넌트"로 바꿔치기!**

이것이 Next.js의 영리한 점입니다:
- 사용자는 0.1초 만에 화면을 봅니다 (빠름!)
- 실제로는 0.6초에 완전히 작동합니다 (부드러움!)
- 사용자는 뭔가 바뀌었는지도 모릅니다 (매끄러움!)

만약 순수 React(Vite)였다면 빈 화면을 0.9초 동안 봐야 했을 겁니다.

**정리:**
- **초기 페이지 로드 (서버 컴포넌트)**: Next.js fetch가 데이터를 가져와 HTML 생성
- **페이지 로드 이후 (클라이언트 컴포넌트)**: TanStack Query가 모든 데이터 조작 담당
- 사용자 A를 기준으로는 초기 로딩 시에만 서버 fetch를 사용하고, 그 이후 모든 데이터 조작(새로고침, 검색, CRUD)은 TanStack Query를 통해서만 이루어집니다.

**그래서 Next.js를 사용하는 이유**

바로 이 Hydration 과정 덕분에 Next.js는 **"빠른 초기 로딩"과 "풍부한 상호작용"이라는 두 마리 토끼를 모두 잡을 수 있습니다.**

- **SEO**: 검색 엔진은 완성된 HTML을 받아가므로 완벽한 검색 노출
- **빠른 체감 속도**: 사용자는 0.1초 만에 콘텐츠를 보기 시작 (순수 React는 0.9초)
- **풍부한 인터랙션**: Hydration 이후에는 React의 모든 기능을 사용 가능

만약 순수 React(Vite)를 사용한다면? 빈 화면에서 시작해 모든 데이터를 클라이언트에서 가져와야 하므로 초기 로딩이 느립니다.

만약 전통적인 서버 렌더링만 사용한다면? 페이지는 빠르지만 버튼 하나 클릭하려면 전체 페이지를 새로고침해야 합니다.

**Next.js = 서버의 빠름 + 클라이언트의 유연함**

이것이 바로 많은 개발자들이 "SEO가 중요하거나 초기 로딩 속도가 중요한 프로젝트"에 Next.js를 선택하는 이유입니다.

---

### 의문 4: 클라이언트에서 API 통신할 때는 뭘 써야 할까?

> **"클라이언트 컴포넌트에서 API 통신할 때는 fetch, axios, ky 중 뭘 써야 할까?"**

앞에서 TanStack Query를 클라이언트에서 사용한다는 건 알았습니다. 그런데 TanStack Query는 실제 API 요청을 직접 하지 않고, 우리가 제공하는 함수(`queryFn`)를 호출할 뿐입니다.

그렇다면 실제 API 요청을 하는 그 함수 안에서는 무엇을 써야 할까요?

#### 해답: axios나 ky를 권장합니다 (특히 TanStack Query와 함께 사용할 때)

클라이언트에서 API 통신을 할 때, 특히 TanStack Query와 함께 사용한다면 `axios`나 `ky`를 사용하는 것이 순수 `fetch`보다 좋습니다.

이해하기 위해서는 TanStack Query의 역할을 명확히 알아야 합니다.

**비유: 배달 앱과 배달원**
- **axios/ky/fetch**: '배달원'입니다. 실제로 서버에 가서 데이터를 받아옵니다.
- **TanStack Query**: '배달 앱'입니다. 배달원을 관리하고, 받아온 데이터를 캐싱하고, 언제 다시 가져올지 결정합니다.

```typescript
'use client';

import { useQuery } from '@tanstack/react-query';
import axios from 'axios';

const { data } = useQuery({
  queryKey: ['todos'],
  queryFn: async () => {
    // TanStack Query가 이 함수(axios)를 호출
    const res = await axios.get('/api/todos');
    return res.data;
  },
});
```

#### 왜 클라이언트에서 axios/ky를 권장할까?

가장 결정적인 이유는 **에러 핸들링** 때문입니다.

TanStack Query를 사용한다면, `queryFn`이 Promise를 **reject** 해야 `isError: true`가 됩니다. 그런데:

- **axios / ky**: 404, 500 등 HTTP 에러 시 자동으로 Promise를 **reject** (에러를 던짐)합니다. TanStack Query가 기대하는 방식과 완벽하게 일치합니다.

- **순수 fetch**: 404, 500 에러가 나도 Promise를 **resolve** (성공)합니다. TanStack Query가 에러로 인지하지 못합니다!

```typescript
// 순수 fetch (매번 에러 처리를 직접 해야 함)
const { data } = useQuery({
  queryKey: ['todos'],
  queryFn: async () => {
    const res = await fetch('/api/todos');
    if (!res.ok) { // 이 코드가 매번 필요!
      throw new Error('에러 발생');
    }
    return res.json();
  },
});

// axios (알아서 에러를 던져줌)
const { data } = useQuery({
  queryKey: ['todos'],
  queryFn: async () => {
    const res = await axios.get('/api/todos');
    return res.data; // .json()도 필요 없음
  },
});
```

#### 배운 점

클라이언트 컴포넌트에서 API 통신을 할 때는 `axios`나 `ky`를 사용하는 것이 순수 `fetch`보다 **에러 핸들링 측면에서 월등히 편리**합니다.

이는 TanStack Query를 사용하든 안 하든 마찬가지입니다. 다만 TanStack Query와 함께 사용할 때 그 장점이 더욱 극대화됩니다.

또한 `axios`의 인터셉터(Interceptor) 기능은 모든 요청에 JWT 토큰을 자동으로 삽입하는 등의 공통 로직에 여전히 강력합니다.

```typescript
// axios 인터셉터로 모든 클라이언트 요청에 토큰 자동 삽입
axios.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// 이제 모든 axios 요청에 자동으로 토큰이 포함됨
axios.get('/api/user/profile'); // 토큰 자동 삽입!
```

---

### 의문 5: 왜 인터넷에 "axios 쓰지 마라"는 글이 많을까?

> **"Next.js axios fetch"로 검색하면 `axios` 대신 `fetch`를 쓰라는 글이 압도적으로 많습니다.**

그런데 실제로는 `axios`가 필요한 상황이 많은데, 왜 이런 현상이 생긴 걸까요?

#### 해답: 아마도 "서버 컴포넌트" 관점에만 집중했기 때문인 것 같습니다

그 이유는 아마도 **Next.js App Router의 '서버 컴포넌트' 패러다임이 워낙 강력하고, 그게 '새로운 표준'처럼 이야기되기 때문**인 것 같습니다.

"axios 쓰지 말라"는 글들이 말하는 주된 맥락을 보면 대부분 이런 것 같습니다:

**1. 'Next.js의 핵심 = 서버 캐싱'이라는 관점**
- Next.js App Router의 가장 큰 혁신은 서버 컴포넌트와 `fetch`를 결합한 강력한 데이터 캐싱입니다.
- `fetch` = 캐싱 시스템의 열쇠
- `axios` = 캐싱 시스템과 분리

**2. Vercel/Next.js 팀의 공식 권장**
- Next.js 개발팀은 플랫폼에 내장된 기능(`fetch`)을 최대한 활용하라고 강력하게 권장합니다.
- 공식 문서의 모든 예제도 `fetch`를 기반으로 합니다.
- 클라이언트 상태 관리 라이브러리로는 TanStack Query 대신 **자신들이 만든 SWR**을 권장합니다.

**3. 검색 결과의 맥락 혼재**
- 검색 결과에는 '서버 컴포넌트' 이야기와 '클라이언트 컴포넌트' 이야기가 섞여 있습니다.
- 하지만 App Router 출시 이후, 가장 화제가 되는 것은 단연 '서버 컴포넌트'입니다.

#### 배운 점

검색한 글들의 주장: **"페이지가 처음 로드될 때(RSC) 데이터를 가져오려면, Next.js 캐싱을 활용할 수 있는 `fetch`를 써야 한다."**

→ 맞습니다.

우리의 현실: **"사용자가 버튼을 클릭하거나(Mutation), 클라이언트에서 복잡한 상태 관리가 필요할 땐, `axios`/`ky`가 에러 핸들링이나 인터셉터 측면에서 훨씬 편하다."**

→ 이것도 맞습니다.

**둘은 충돌하는 것이 아니라, 함께 사용하는 관계입니다.**

---

## 내가 얻은 결론

### 현실적인 Next.js API 통신 전략

현실적인 대규모 Next.js 프로젝트는 **두 가지를 모두 사용**합니다:

#### 1. 서버 컴포넌트 (RSC): fetch 또는 ky

**사용처**: 페이지 초기 로드, SEO, 정적 데이터

**이유**: Next.js의 강력한 서버 캐싱 기능 활용

```typescript
// app/posts/[id]/page.tsx

async function PostPage({ params }: { params: { id: string } }) {
  // 10초마다 재검증
  const post = await fetch(`https://api.example.com/posts/${params.id}`, {
    next: { revalidate: 10 },
  }).then(res => res.json());

  return <article>{post.content}</article>;
}
```

#### 2. 클라이언트 컴포넌트 (CC): TanStack Query + axios/ky

**사용처**: 모든 사용자 상호작용, 데이터 변경(Mutation), 인증

**이유**: 압도적으로 편리한 클라이언트 상태 관리, 에러 핸들링, 인터셉터(인증) 활용

```typescript
'use client';

import { useMutation, useQueryClient } from '@tanstack/react-query';
import axios from 'axios';

// axios 인터셉터로 모든 요청에 토큰 자동 삽입
axios.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

function LikeButton({ postId }: { postId: string }) {
  const queryClient = useQueryClient();

  const likeMutation = useMutation({
    mutationFn: () => axios.post(`/api/posts/${postId}/like`),
    onSuccess: () => {
      // 좋아요 후 게시글 목록 자동 갱신
      queryClient.invalidateQueries(['posts']);
    },
  });

  return (
    <button onClick={() => likeMutation.mutate()}>
      {likeMutation.isLoading ? '처리중...' : '좋아요'}
    </button>
  );
}
```

### 언제 어떤 도구를 쓸까?

| 상황 | 추천 도구 | 이유 |
|------|----------|------|
| 서버 초기 로드 | fetch | Next.js 캐싱 시스템과 완벽 연동 |
| 서버 + 편의성 | ky | fetch 캐싱 + 간편한 API + 에러 처리 |
| 클라이언트 조회 | TanStack Query + axios/ky | 상태 관리 + 자동 에러 처리 |
| 클라이언트 변경 | TanStack Query useMutation | Mutation 상태 관리 + 캐시 동기화 |
| 공통 인증/로깅 | axios Interceptor | 모든 요청에 자동으로 토큰/로직 주입 |

---

## 실무에서는 어떻게?

### 대기업의 선택

제 조사에 따르면, 큰 기업에서 HTTP 클라이언트를 선택하는 기준은 다음과 같습니다:

#### 1. Axios (가장 높은 채택률)

- **사용처**: 대부분의 기존 및 신규 엔터프라이즈 프로젝트
- **이유**: 인터셉터(Interceptor) 기능이 매우 강력하고 안정성이 검증됨
- **장점**: 필요한 모든 기능(인터셉터, 타임아웃, 요청 취소, 자동 JSON 변환)이 내장
- **단점**: `fetch` 기반 라이브러리보다 번들 크기가 다소 큼

#### 2. Fetch (+ 사내 래퍼)

- **사용처**: 간단한 호출 또는 사내 래퍼 라이브러리의 기반
- **이유**: 의존성이 없음
- **특징**: 대기업에서는 `fetch`를 직접 사용하기보다, `fetch`를 기반으로 한 사내 공통 HTTP 클라이언트 모듈을 만들어 배포하는 경우가 많음

#### 3. Ky (신규 프로젝트)

- **사용처**: 번들 크기에 민감한 신규 프로젝트
- **이유**: `fetch` 기반으로 가벼우면서도 `axios`의 편의 기능 제공
- **장점**: Hooks(beforeRequest, afterResponse)로 인터셉터 유사 기능 제공, 재시도(Retry) 내장

### Next.js 프로젝트에서의 현실

**axios는 여전히 필요합니다.** 다만, Next.js의 철학에 맞춰:
- **RSC에서는** `fetch`에게 자리를 양보하고
- **CC에서는** TanStack Query와 함께 더 강력하게 사용되고 있을 뿐입니다

```
Next.js 프로젝트의 API 통신 = RSC의 fetch + CC의 TanStack Query + axios
```

---

## 번외: Next.js를 꼭 써야 할까?

이 모든 걸 알아가면서 결국 이런 질문에 다다랐습니다. "약간의 서버 컴포넌트를 위해 빌드도 오래 걸리고 복잡한 Next.js를 꼭 써야 할까?"

특히 [이전 글](https://app.gamzatech.site/posts/110)에서 고민했던 것처럼, 저희 졸업 프로젝트는 **실시간 협업 차트 생성**이 메인 기능입니다. 사용자들이 동시에 WBS를 편집하고, 실시간으로 변경사항을 주고받는 것이 핵심이죠.

이런 프로젝트 특성을 생각해보면:
- SEO가 중요하지 않음 (로그인 후 사용하는 도구)
- 대부분의 화면이 사용자 상호작용 (클릭, 드래그, 입력)
- 실시간 데이터 동기화가 핵심
- 서버 컴포넌트로 얻을 이점이 거의 없음

그렇다면 Next.js보다 순수 React(Vite)가 더 적합했을 수도 있겠다는 생각이 들었습니다.

### ComfyDeploy의 선택

실제로 [ComfyDeploy](https://www.comfydeploy.com/blog/you-dont-need-nextjs)는 Next.js를 버리고 순수 React(Vite + TanStack Router)로 돌아갔습니다. 저희 프로젝트와 비슷하게 대시보드 중심의 서비스였기 때문입니다.

**그들이 Next.js를 포기한 이유:**

1. **불필요한 기능**: 대시보드는 SEO가 전혀 필요 없었고, 대부분의 데이터가 실시간으로 갱신되어야 했습니다.
2. **느린 개발 속도**: 작은 코드 수정에도 리로드 속도가 느렸습니다.
3. **느린 빌드**: 빌드 시간이 3분 → Vite로 바꾸니 18초
4. **복잡성**: Server Actions 같은 '마법'보다 명확한 API 설계를 선호

### 현실적인 판단 기준

#### Next.js가 강력한 경우

**공개 웹사이트 (블로그, 이커머스, 마케팅 페이지)**

- SEO와 빠른 첫 페이지 로드가 비즈니스에 결정적
- Next.js의 존재 이유 그 자체
- SSR/SSG로 검색 엔진 최적화 완벽 대응

```typescript
// 블로그 게시글 페이지
export async function generateStaticParams() {
  const posts = await fetch('https://api.example.com/posts');
  return posts.map((post) => ({ slug: post.slug }));
}

async function BlogPost({ params }: { params: { slug: string } }) {
  const post = await fetch(`https://api.example.com/posts/${params.slug}`, {
    next: { revalidate: 3600 }, // 1시간마다 재검증
  });
  
  return <article>{post.content}</article>;
}
```

#### Vite + React가 나을 수 있는 경우

**로그인 기반 SaaS/대시보드**

- SEO 불필요 (검색 엔진에 노출될 필요 없음)
- 실시간 상호작용이 대부분
- 빠른 개발 속도와 단순한 아키텍처가 중요

```typescript
// Vite + React + TanStack Query
function Dashboard() {
  const { data: stats } = useQuery({
    queryKey: ['dashboard-stats'],
    queryFn: () => axios.get('/api/stats'),
    refetchInterval: 5000, // 5초마다 폴링
  });

  return <div>{/* 실시간 대시보드 */}</div>;
}
```

### 하이브리드 전략 (이상적)

많은 SaaS 기업이 사용하는 전략:

- **flowplan.com (마케팅 사이트)**: Next.js
  - 회사 소개, 기능 소개, 블로그, 랜딩 페이지
  - SEO, SSG/SSR, 빠른 초기 로딩 최적화

- **app.flowplan.com (실제 앱)**: Vite + React
  - 로그인 후 사용하는 실제 서비스
  - 빠른 개발 속도, 단순한 아키텍처, 빠른 빌드

**하지만 대학생/소규모 프로젝트는?**

→ 제 생각에는 **Next.js 하나로 시작하는 게 나을 것 같습니다.**

프로젝트를 두 개로 쪼개는 순간, 관리할 레포지토리, 빌드/배포 설정, 인증(쿠키/토큰) 공유 등 신경 쓸 것이 2배 이상으로 늘어납니다. 핵심 기능 개발에 집중하는 것이 우선이니까요.

```typescript
// Next.js 하나로 충분
// app/(marketing)/page.tsx - RSC로 랜딩 페이지
// app/(app)/dashboard/page.tsx - 'use client'로 대시보드
```

---

## 마치며

"Next.js면 fetch만", "axios는 구식" 같은 이분법적 조언보다는, **각 도구의 역할을 이해하고 상황에 맞게 선택하는 것**이 중요합니다.

### 핵심 요약

1. **서버 컴포넌트에서는** `fetch` (또는 `ky`)를 사용해 Next.js 캐싱을 활용하세요
2. **클라이언트 컴포넌트에서는** TanStack Query + axios/ky로 상태 관리와 Mutation을 처리하세요
3. **axios는 죽지 않았습니다** - 역할만 명확해졌을 뿐입니다
4. **RSC를 쓸 곳이 없다고 느껴도** 괜찮습니다 - 상호작용이 많은 앱은 원래 그렇습니다
5. **프로젝트 특성에 맞게** Next.js와 Vite+React 중 선택하세요

저처럼 혼란스러웠던 분들께 이 글이 명확한 기준을 제시할 수 있기를 바랍니다.

---

## 참고 자료

- [Next.js 공식 문서 - Data Fetching](https://nextjs.org/docs/app/building-your-application/data-fetching)
- [TanStack Query 공식 문서](https://tanstack.com/query/latest)
- [ComfyDeploy - You Don't Need Next.js](https://www.comfydeploy.com/blog/you-dont-need-nextjs)
- [ky GitHub Repository](https://github.com/sindresorhus/ky)
- [axios 공식 문서](https://axios-http.com/)

