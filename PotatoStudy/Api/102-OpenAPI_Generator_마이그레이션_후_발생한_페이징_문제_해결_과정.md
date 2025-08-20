# OpenAPI Generator 마이그레이션 진행
API Generator를 도입하여 API 인터페이스의 일관성을 맞추는 작업을 진행했습니다. 

최근에는 API 호출 함수까지 일관성 있게 적용하기 위해, 
기존에 스웨거(Swagger)를 보고 직접 작성했던 API 호출 코드를 
OpenAPI Generator가 자동으로 생성해준 함수로 마이그레이션했습니다.

그런데.. 두둥...

**메인 페이지에서 페이지가 넘어가지 않는 심각한 버그가 발생했습니다!**

# 추측 1. 스웨거에서 데이터를 잘못 넘겨주고 있다.

![Uploaded Image](https://gamzatech-bucket.s3.ap-northeast-2.amazonaws.com/post-images/102/f573544c-d440-4ea1-9c96-5dd8d3fd07e9_image.png)


현재 프로젝트는 페이지 값을 받으려면 쿼리 파라미터 형식으로 URL을 구성해야 합니다. 예를 들어, ```app.gamzatech.site/?page=2```와 같은 형식이죠. 프론트엔드에서는 이 파라미터를 읽어서 백엔드 API를 호출할 때 다시 쿼리 파라미터로 넘겨주어야 합니다.

스웨거 UI에서 직접 ```page``` 값을 ```1```로 넘겨서 테스트해보니, 
**페이지 1에 해당하는 데이터가 정상적으로 반환되는 것을 확인했습니다.**

이걸 보고 '아, 문제는 프론트엔드구나!'라고 판단했습니다.

# 추측 2. 서버/클라이언트 컴포넌트 구조 변경 문제?
최근 성능 최적화를 위해 메인 페이지의 구조를 변경한 것이 떠올랐습니다.

**서버 컴포넌트(Server Component)** 는 페이지가 처음 로드될 때 서버에서 데이터를 미리 가져와 렌더링하는 방식입니다. 초기 로딩 속도를 개선하고 검색 엔진 최적화(SEO)에 유리하다는 장점이 있죠.

이러한 이유로, 기존의 거대한 단일 클라이언트 컴포넌트(Client Component)로 구성되어 있던 메인 페이지를 서버 컴포넌트로 전환하고, 해당 컴포넌트에서 초기 데이터를 직접 호출하도록 리팩토링을 진행했습니다.


```javascript
export default function Home({
  searchParams,
}: {
  searchParams: Promise<{ tag?: string; page?: string }>;
}) {
  return (
    <>
      <DynamicWelcomeModal />

      <div className="mx-auto flex flex-col gap-12">
        {/* 로고 섹션 - 즉시 렌더링 */}
        <LogoSection />

        {/* 메인 콘텐츠 - 스트리밍 */}
        <div className="flex pb-10">
          {/* 게시글 목록 섹션 - 독립적 스트리밍 */}
          <Suspense fallback={<PostListSkeleton count={3} />}>
            <PostListSection searchParams={searchParams} />
          </Suspense>

          {/* 사이드바 섹션 - 독립적 스트리밍 */}
          <Suspense fallback={<SidebarSkeleton />}>
            <SidebarSection />
          </Suspense>
        </div>
      </div>
    </>
  );
}

```

```javascript
export default async function PostListSection({ searchParams }: PostListSectionProps) {
  const { tag, page } = await searchParams;
  const currentPage = Number(page) || 1;
  const pageSize = 10;

  // 서버에서 초기 게시글 데이터 페칭
  const initialData = tag
    ? await postService.getPostsByTag(
        tag,
        {
          page: currentPage - 1,
          size: pageSize,
          sort: ["createdAt,desc"],
        },
        { cache: "no-store" }
      )
    : await postService.getPosts(
        {
          // page: 1,
          page: currentPage - 1,
          size: pageSize,
          sort: ["createdAt,desc"],
        },
        { cache: "no-store" }
      );

  // const response = await postService.getPosts1({ page: 1, size: 10 });

  // console.log("posts response", response);

  return (
    <main className="flex-3">
      <div className="mb-6 flex items-center justify-between">
        <h2 className="text-2xl font-semibold">{tag ? `#${tag} 태그 게시글` : "Posts"}</h2>
      </div>

      {/* 인터랙티브 게시글 목록 */}
      <InteractivePostList initialData={initialData} initialTag={tag} initialPage={1} />

      {/* 페이지네이션 */}
      {initialData.totalPages && initialData.totalPages > 1 && (
        <div className="mt-12 flex justify-center">
          <InteractivePagination
            currentPage={currentPage}
            totalPages={initialData.totalPages}
            tag={tag}
          />
        </div>
      )}
    </main>
  );
}

```

```javascript
export default function InteractivePostList({
  initialData,
  initialTag,
  initialPage,
}: InteractivePostListProps) {
  const searchParams = useSearchParams();
  const currentTag = searchParams.get("tag") || undefined;
  const currentPage = Number(searchParams.get("page")) || 1;

  // 초기 로딩인지 확인 (URL 파라미터가 초기값과 같은지)
  const isInitialLoad = currentTag === initialTag && currentPage === initialPage;

  // 쿼리 파라미터
  const queryParams = useMemo(
    () => ({
      page: currentPage - 1, // 원래 로직으로 복원 (0-based indexing)
      size: 10,
      sort: ["createdAt,desc"],
    }),
    [currentPage]
  );

  const postsQuery = usePosts(queryParams, {
    enabled: !currentTag && !isInitialLoad,
    initialData: !currentTag && isInitialLoad ? initialData : undefined,
  });

  const postsByTagQuery = usePostsByTag(currentTag!, queryParams, {
    enabled: !!currentTag && !isInitialLoad,
    initialData: currentTag === initialTag && isInitialLoad ? initialData : undefined,
  });

  // 현재 사용할 데이터 결정
  const activeQuery = currentTag ? postsByTagQuery : postsQuery;
  const data = isInitialLoad ? initialData : activeQuery.data;
  const isLoading = !isInitialLoad && activeQuery.isLoading;
  const isFetching = activeQuery.isFetching;
  const error = activeQuery.error;

  const posts = data?.content || [];
  
  ...
}
```

하지만 이곳 역시 문제가 아니었습니다.

```InteractivePostList ``` 컴포넌트 내부에서 아예 `page `값을 `1`로 하드코딩해서 함수를 호출해봐도, 계속해서 페이지 0에 해당하는 데이터만 호출되었습니다.

# 추측 3. OpenAPI Generator 마이그레이션 자체의 문제?

이제 시선은 마이그레이션된 함수로 향했습니다.

> **마이그레이션 작업 전 함수**
```javascript
 async getPosts(params?: Pageable): Promise<PagedResponsePostListResponse> {
    const endpoint = API_PATHS.posts.base;
    const url = new URL(API_CONFIG.BASE_URL + endpoint);

    if (params?.page !== undefined) url.searchParams.append("page", String(params.page));
    if (params?.size !== undefined) url.searchParams.append("size", String(params.size));
    if (params?.sort) params.sort.forEach((sort: string) => url.searchParams.append("sort", sort));

    try {
      const response = await fetch(url.toString(), {
        method: "GET",
        headers: { "Content-Type": "application/json" },
        cache: "no-cache",
      });

      if (!response.ok) {
        const errorData = await response.json().catch(() => ({}));
        throw new PostServiceError(
          response.status,
          errorData.message || "Failed to fetch posts",
          endpoint
        );
      }

      const apiResponse: ResponseDtoPagedResponsePostListResponse = await response.json();
      return apiResponse.data as PagedResponsePostListResponse;
    } catch (error) {
      if (error instanceof PostServiceError) throw error;
      throw new PostServiceError(
        500,
        (error as Error).message || "An unexpected error occurred",
        endpoint
      );
    }
  },
```


> **마이그레이션 작업 후 수정된 함수**
```javascript
  async getPosts(params: Pageable, options?: RequestInit): Promise<PagedResponsePostListResponse> {
    const response = await apiClient.getPosts({ pageable: params }, options);
    return response.data as PagedResponsePostListResponse;
  },
```

수정된 코드는 한눈에 봐도 확연하게 줄어들었습니다. (여러분... OpenAPI Generator 꼭 써보세요... 생산성이 달라집니다! 👍)

물론, 에러 처리 같은 부분에서 단점이 있긴 합니다,,

아무튼, 그게 문제가 아니니..

기존 로직은 쿼리 파라미터를 하나씩 수동으로 매핑하는 구조였지만, Generator로 생성된 함수와 인터페이스를 활용하여 Pageable이라는 인터페이스 객체에 담아 넘기도록 수정되었습니다.

로직 자체는 문제가 없어 보였습니다. 실제로 디버깅을 해봐도...

![Uploaded Image](https://gamzatech-bucket.s3.ap-northeast-2.amazonaws.com/post-images/102/b99562cd-b857-4834-a94f-5c3f11f0c89a_image.png)

getPosts 인자로 넘어온 params.page 값이 1 로 잘 넘어오고 있습니다.

![Uploaded Image](https://gamzatech-bucket.s3.ap-northeast-2.amazonaws.com/post-images/102/bd3a1d17-d23e-41d8-b180-baf0f1fb131a_image.png)
하지만 API의 반환 값은 여전히 페이지 0에 해당하는 데이터였습니다.

이 시점에서 **"Generator로 생성된 함수 자체가 문제다!"** 라고 확신하게 되었습니다.

# 추측 4. API 명세의 문제!
OpenAPI Generator로 만들어진 코드들은 전적으로 API 명세를 기반으로 생성됩니다. 현재 백엔드는 스웨거를 사용 중이니, 백엔드 소스 코드를 기반으로 API 문서가 자동으로 생성되었을 겁니다.

여기서 결정적인 단서를 찾았습니다.

백엔드 서버는 `/?page=숫자&size=`숫자 형식으로 파라미터를 받길 원하지만, API 명세에는 `Pageable`이라는 객체를 받도록 정의되어 있었던 것입니다. 그래서 Generator가 `Pageable `객체를 인자로 받는 함수를 생성했던 것이죠.

그래서 위의 getPosts 함수를 보면 Pageable 객체를 넘기도록 함수가 생성된 것이었습니다..!

**드디어 찾았습니다..!!!!! 끼얏호!!!!!!!**

API 명세에 잘못된 부분이 존재한다는 것까지 파악할 수 있었습니다..!

그래서 어느 부분이...?

실제 서버는 
`GET /api/v1/posts?page=1&size=10&sort=createdAt,DESC` 
와 같은 형식을 받길 원하지만 

OpenAPI 명세 기준으로는 
`GET /api/v1/posts?pageable[page]=1&pageable[size]=10&pageable[sort]=createdAt,DESC` 
처럼 받길 원하는 상태였습니다..!

이를 해결할 수 있는 방법을 Gemini 에게 물어보았습니다..

![Uploaded Image](https://gamzatech-bucket.s3.ap-northeast-2.amazonaws.com/post-images/102/2dd0e324-b442-48f4-8e58-c1760b7f69d4_image.png)


그래서 이를 어찌하면 조으냐고 물으니..!

![Uploaded Image](https://gamzatech-bucket.s3.ap-northeast-2.amazonaws.com/post-images/102/964b2302-e056-46ff-8f79-2d013798d95a_image.png)

**@ParameterObject 어노테이션 하나면 한방에 해결이라는 것!!!!**

자 지금까지 엄청난 삽질이였습니다..

이렇게 찾고 보니까 더 쉬운 방법이 있었다는 사실....

생각해보니 메인 페이지의 페이징만 불가능했고, 다른 기능들의 페이징은 모두 정상적으로 동작하고 있었습니다.

그렇다면 처음부터...
1. 메인 페이지 API만 호출이 안 되는 상황을 인지하고,
2. 정상 동작하는 다른 API와 명세를 비교해보고,
3. 명세는 스웨거가 자동으로 작성하니, 백엔드 소스 코드의 차이점을 확인했다면...

네.. 

메인 페이지에서 호출하는 API의 컨트롤러 코드에만 @ParameterObject 어노테이션이 누락되어 있었습니다... ㅜㅜ

조금만 더 넓게 생각하고 접근했다면 몇 시간의 삽질을 크게 줄일 수 있었을 텐데... 그래도 스스로 원인을 찾아냈다는 사실에 기쁩니다..!!


# 결론은?!
일단 여기까지는 저의 모든 추측일 뿐...!

현재 시각은 새벽이므로, 백엔드 팀에 @ParameterObject 어노테이션 추가를 요청하는 메시지를 남겨두었습니다. (새벽에 연락 남겨서 미안합니다..)

부디 이것으로 모든 문제가 해결되었으면 좋겠습니다...!!!!

오랜 삽질의 시간으로 인해 배가 고프니 빵 한조각 먹고 저는 취침을 하러 마무리 해보겠습니다...

그럼 20000....