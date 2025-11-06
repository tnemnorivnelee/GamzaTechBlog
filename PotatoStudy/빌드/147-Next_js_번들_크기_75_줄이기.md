## 배경

프로젝트가 진행되면서 빌드 시간이 점점 늘어나는 문제를 확인했습니다. 초기 10초 이내였던 빌드 시간이 현재는 약 20초로 2배 가량 증가했고, 개발 과정에서 코드를 수정하고 테스트할 때마다 상당한 시간이 소요되어 생산성 저하를 체감하게 되었습니다.

![](https://velog.velcdn.com/images/tnemnorivnelee/post/48a46fee-5ac5-4a6c-abb1-da5dddfc36f5/image.png)

빌드 결과를 분석한 결과, 다음과 같은 문제점을 발견했습니다:

- **전체 빌드 시간**: 약 18초
- **First Load JS**: 대부분의 페이지가 500KB를 초과하여 권장 크기인 200~300KB를 훨씬 상회

이러한 문제를 해결하기 위해 번들 크기 최적화 작업을 시작했습니다.


## 최적화 1: Toast UI Editor 동적 임포트

### 문제
기존 `ToastEditor` 컴포넌트에서 조건부 `require()`로 에디터를 import했지만, 빌드 시스템(Webpack)은 조건문과 무관하게 `require()`를 발견하면 무조건 번들에 포함시켰습니다.

### 해결
`DynamicComponents`에서 에디터 라이브러리를 동적으로 로드하고, `ToastEditor`에 props로 주입하는 방식으로 변경했습니다. Webpack이 이를 별도 청크로 분리하여 컴포넌트 렌더링 시에만 로드하도록 개선했습니다.

### 결과
![](https://velog.velcdn.com/images/tnemnorivnelee/post/b1aa470f-1000-4ee3-a1c5-3e347f42f42a/image.png)

| 측정 항목 | 개선 전 | 개선 후 | 개선율 |
|-----------|---------|---------|--------|
| 첫 페이지 로드 | 823KB | 544KB | **-34%** |
| Toast UI 번들 | 메인에 포함 | 별도 청크 (941KB) | 완전 분리 |

**로컬 프로덕션 빌드 실측 (Chrome DevTools):**

| 측정 항목 | develop | Toast UI 최적화 | 개선 |
|-----------|---------|-----------------|------|
| **JS transferred** | 1,636 KB | 1,355 KB | **-281 KB (-17%)** ✅ |
| **Total resources** | 3,696 KB | 2,782 KB | **-914 KB (-25%)** |
| **DOMContentLoaded** | 84 ms | 61 ms | **-23 ms (-27%)** 🚀 |
| **Load** | 248 ms | 219 ms | **-29 ms (-12%)** 🚀 |

**효과:** 홈페이지나 글 목록 페이지 방문자는 더 이상 에디터를 다운로드하지 않아 초기 로딩 속도가 개선되었습니다.

## 최적화 2: React Query Devtools 조건부 로드 및 next.config.ts 개선

### 문제
`ReactQueryDevtools`가 개발/프로덕션 환경 구분 없이 모든 빌드에 포함되었습니다. Devtools는 개발 환경에서만 필요한 디버깅 도구이므로 프로덕션 번들에는 불필요한 용량입니다.

또한 `next.config.ts`의 `optimizePackageImports` 설정을 확장하여 Radix UI 컴포넌트와 rehype/remark 플러그인의 tree shaking을 개선할 수 있었습니다.

### 해결
`Providers.tsx`에서 `dynamic()`을 사용해 Devtools를 조건부로 로드하도록 수정했습니다. `process.env.NODE_ENV === "development"` 조건을 통해 개발 환경에서만 Devtools를 import하고, 프로덕션에서는 빈 컴포넌트를 반환하여 Webpack이 번들에서 완전히 제외하도록 했습니다.

동시에 `next.config.ts`의 `optimizePackageImports` 배열에 자주 사용되는 라이브러리들을 추가하여 import 최적화를 강화했습니다.

### 결과
로컬 프로덕션 빌드 실측 (Chrome DevTools):


| 측정 항목 | Toast UI 최적화 | + Devtools 최적화 | 개선 |
|-----------|----------------|-------------------|------|
| **JS transferred** | 1,355 KB | 1,358 KB | +3 KB |
| **Total resources** | 2,782 KB | 2,791 KB | +9 KB |
| **DOMContentLoaded** | 61 ms | 50 ms | **-11 ms (-18%)** ✅ |
| **Load** | 219 ms | 209 ms | **-10 ms (-5%)** ✅ |
| **빌드 시간** | ~18s | ~16s | **-2s (-11%)** ⚡ |

JS transferred 부분에서 +3kb 늘어난 부분은 측정 환경에 따른 오차범위 내의 수치로 판단.
Load 하는 부분에서 약간의 성능 향상.

## 최적화 3: sideEffects 설정 적용

### 문제
![](https://velog.velcdn.com/images/tnemnorivnelee/post/552823be-77f6-46b4-a35b-9c022d6f2227/image.png)

이전 최적화에도 불구하고 대부분의 페이지에서 First Load JS가 여전히 500KB를 초과했습니다. 권장 수치인 200~300KB를 크게 웃도는 상태였습니다.

프로젝트에서 barrel export 패턴(`index.ts`에서 모든 컴포넌트를 재수출)을 사용하고 있었는데, 이는 Webpack이 사용하지 않는 export를 제거하지 못하게 만드는 원인이었습니다. 무거운 라이브러리들이 필요 없는 페이지에도 포함되고 있었습니다.

### 해결
`package.json`에 `sideEffects` 설정을 추가하여 Webpack에게 더 공격적인 tree shaking을 수행하도록 지시했습니다:

```json
{
  "sideEffects": [
    "*.css",
    "*.scss",
    "src/app/globals.css"
  ]
}
```

이 설정은 "CSS 파일을 제외한 모든 코드는 부작용(side effects)이 없다"는 것을 의미합니다. Webpack은 이를 기반으로 barrel export 패턴에서도 사용하지 않는 export를 안전하게 제거할 수 있게 되었습니다.

### 결과
![](https://velog.velcdn.com/images/tnemnorivnelee/post/34050374-7653-4619-8367-da86872bf0bb/image.png)


#### ✅ 부분적 성공 (5개 페이지 대폭 개선)

| 페이지 | 적용 전 | 적용 후 | 개선 | 개선율 |
|--------|---------|---------|------|--------|
| **/admin** | 539 KB | 140 KB | **-399 KB** | **-74%** 🚀 |
| **/search** | 539 KB | 164 KB | **-375 KB** | **-70%** 🚀 |
| **/posts/[id]/edit** | 539 KB | 177 KB | **-362 KB** | **-67%** 🚀 |
| **/posts/new** | 538 KB | 176 KB | **-362 KB** | **-67%** 🚀 |
| **/signup** | 541 KB | 204 KB | **-337 KB** | **-62%** 🚀 |
| **평균** | **539 KB** | **172 KB** | **-367 KB** | **-68%** |

#### ❌ 개선되지 않은 페이지 (4개)

| 페이지 | 적용 전 | 적용 후 | 변화 |
|--------|---------|---------|------|
| **/ (홈)** | 544 KB | 545 KB | +1 KB 😐 |
| **/posts/[id]** | 544 KB | 545 KB | +1 KB 😐 |
| /mypage | 544 KB | 550 KB | +6 KB ❌ |
| /profile | 544 KB | 550 KB | +6 KB ❌ |

많은 페이지에서 first load js 수치가 개선이 되었지만, 몇몇 페이지들은 아직도 500kb 가 넘어가고 있음.

## 최적화 4: Barrel Export 물리적 분리

### 문제
`sideEffects` 설정으로 많은 페이지가 개선되었지만, 핵심 페이지들(홈, 게시글 상세, 마이페이지, 프로필)은 여전히 500KB 이상을 유지하고 있었습니다.

원인을 분석한 결과, barrel export 패턴의 2단계 재수출 구조가 문제였습니다:
- `src/features/posts/index.ts` → `components/index.ts` → 실제 컴포넌트

이처럼 복잡한 의존성 체인에서는 Webpack이 완벽한 분석을 수행하지 못했고, `sideEffects` 설정만으로는 해결되지 않았습니다.

특히 `MarkdownViewer` 컴포넌트가 두 단계의 barrel export를 통해 재수출되고 있었는데, 이 컴포넌트는 내부적으로 `Highlight.js`(약 150KB, 30개 이상의 프로그래밍 언어 지원)를 포함하는 무거운 컴포넌트였습니다.

### 해결
`MarkdownViewer`는 이미 `DynamicComponents`를 통해 동적으로 사용할 수 있는 구조였으므로, barrel export에서의 재수출을 물리적으로 제거하는 방식을 선택했습니다.

`src/features/posts/index.ts`와 `src/features/posts/components/index.ts` 두 곳 모두에서 `MarkdownViewer`와 `ToastEditor`의 export를 제거했습니다. Tree shaking으로 해결되지 않는 부분을 아예 번들링 경로에서 차단한 것입니다.

### 결과
![](https://velog.velcdn.com/images/tnemnorivnelee/post/d5db5c68-4251-44de-96c3-55c7e535d12a/image.png)


| 페이지 | 물리적 분리 전 | 물리적 분리 후 | 개선 | 개선율 |
|--------|---------------|---------------|------|--------|
| **/ (홈)** | 545 KB | **267 KB** | **-278 KB** | **-51%** 🚀 |
| **/posts/[id]** | 545 KB | **267 KB** | **-278 KB** | **-51%** 🚀 |
| /mypage | 550 KB | **273 KB** | **-277 KB** | **-50%** 🚀 |
| /profile | 550 KB | **273 KB** | **-277 KB** | **-50%** 🚀 |


## 최종 성과

네 가지 최적화 전략을 단계적으로 적용한 결과, 모든 페이지에서 First Load JS를 권장 수치 이내로 낮추는 데 성공했습니다.

### 개선 전
![](https://velog.velcdn.com/images/tnemnorivnelee/post/9684d43e-5731-4001-8bf2-e46e4712db47/image.png)

대부분의 페이지가 800KB 이상의 First Load JS를 보이며, 권장 수치의 3배 이상을 초과하고 있었습니다.

### 개선 후
![](https://velog.velcdn.com/images/tnemnorivnelee/post/2c807e88-4c3f-47e2-a682-f7ddab3c8e53/image.png)

모든 페이지가 300KB 이하로 감소하여 권장 수치를 만족하게 되었습니다.

### 상세 비교

| 페이지              | develop | 최종       | 총 개선     | 개선율      |
| ------------------- | ------- | ---------- | ----------- | ----------- |
| **/ (홈)**          | 823 KB  | **267 KB** | **-556 KB** | **-68%** 🚀 |
| **/posts/[id]**     | 823 KB  | **267 KB** | **-556 KB** | **-68%** 🚀 |
| /mypage             | 823 KB  | **273 KB** | **-550 KB** | **-67%** 🚀 |
| /profile/[username] | 823 KB  | **273 KB** | **-550 KB** | **-67%** 🚀 |
| /admin              | 818 KB  | **140 KB** | **-678 KB** | **-83%** 🎉 |
| /search             | 818 KB  | **164 KB** | **-654 KB** | **-80%** 🎉 |
| /signup             | 820 KB  | **204 KB** | **-616 KB** | **-75%** 🎉 |
| /posts/new          | 817 KB  | **176 KB** | **-641 KB** | **-78%** 🎉 |
| /posts/[id]/edit    | 818 KB  | **177 KB** | **-641 KB** | **-78%** 🎉 |

### 주요 성과

- **평균 개선율**: 약 **75%**
- **모든 페이지**: 권장 수치인 200~300KB 범위 내 달성 ✅
- **최대 개선**: Admin 페이지 **83% 감소** (818KB → 140KB)
- **핵심 페이지**: 홈/게시글 상세 페이지 **68% 감소** (823KB → 267KB)

## 배포 환경 검증

develop 브랜치에 머지 후 실제 Vercel 배포 환경에서 Chrome DevTools로 성능을 측정했습니다.

### 측정 환경

- **URL**: https://preview.gamzatech.site (Vercel develop 브렌치 배포 환경)
- **측정 도구**: Chrome DevTools Network 탭
- **측정 조건**: Disable cache + Filter: JS
- **측정 횟수**: 5회 측정 후 평균값 사용


### 배포 환경 개선 효과

| 측정 항목            | Before   | After    | 개선                  |
| -------------------- | -------- | -------- | --------------------- |
| **JS transferred**   | 2,042 KB | 1,490 KB | **-552 KB (-27%)** ✅ |
| **Total resources**  | 4,999 KB | 4,281 KB | **-718 KB (-14%)** ✅ |
| **DOMContentLoaded** | 167 ms   | 137 ms   | **-30 ms (-18%)** ✅  |
| **Load**             | 395 ms   | 320 ms   | **-75 ms (-19%)** ✅  |



## 결론

이번 최적화를 통해 다음과 같은 교훈을 얻을 수 있었습니다:

1. **동적 임포트의 중요성**: Toast UI Editor처럼 특정 페이지에서만 사용되는 무거운 라이브러리는 반드시 동적 임포트를 적용해야 합니다.

2. **Tree shaking의 한계**: `sideEffects` 설정은 간단한 barrel export에는 효과적이지만, 복잡한 재수출 체인에서는 물리적 분리가 더 확실한 해결책입니다.

3. **번들 분석의 필요성**: 빌드 결과를 주기적으로 분석하고, First Load JS가 권장 수치를 초과하는 페이지를 모니터링하는 것이 중요합니다.

4. **단계적 접근**: 한 번에 모든 문제를 해결하려 하기보다는, 각 최적화 기법을 적용하고 측정하면서 점진적으로 개선하는 것이 효과적입니다.

5. **배포 환경 검증**: 로컬 빌드뿐만 아니라 실제 배포 환경에서의 성능 측정이 중요하며, 실사용자 경험을 반영할 수 있습니다.

### 아쉬운 점

초기 목표는 빌드 시간 단축이었지만, 실제로는 **18초 → 16초로 약 2초(11%)만 개선**되었습니다. 예상보다 작은 개선폭이었습니다.

하지만 이 과정에서 **사용자 경험 개선**이라는 더 중요한 성과를 얻었습니다:
- First Load JS **75% 감소** (823KB → 267KB)
- 초기 로딩 속도 **27% 개선** (JS transferred 기준)
- 모든 페이지가 권장 수치 달성

빌드 시간은 개발자 경험(DX)에 영향을 주지만, 번들 크기는 실제 사용자 경험(UX)에 직접적인 영향을 줍니다. 결과적으로 더 중요한 문제를 해결할 수 있었다고 생각합니다.

결과적으로 초기 로딩 속도가 크게 개선되었고, 사용자 경험이 향상될 것으로 기대됩니다.
