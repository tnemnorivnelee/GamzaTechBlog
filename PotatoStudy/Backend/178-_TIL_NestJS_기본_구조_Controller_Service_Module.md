# [TIL] NestJS 기본 구조 — Controller, Service, Module

![Uploaded Image](https://gamzatech-bucket.s3.ap-northeast-2.amazonaws.com/post-images/178/3539d1dd-93c0-408a-b5b0-07dbd8eb6589_image.png)

오랫동안 미뤄왔던 백엔드 공부를 드디어 시작했다.  
지금까지 프론트엔드 개발만 해왔기 때문에, **같은 TypeScript 생태계 안에서** 백엔드까지 익힐 수 있다는 점에서 NestJS를 선택했다. 언어 전환 없이 서버 개발을 경험해볼 수 있다는 게 가장 큰 이유다.

오늘 정리한 내용은 아래 강의의 5강을 바탕으로 한다.

> 📺 [NestJS - 5강: 프로젝트 시작과 기본 구조 이해](https://www.youtube.com/watch?v=myL2N9Kre0U&list=PL-cIzvS-5d-0t1Dd6JQ8XdNnbgM3o3vxh&index=5)

---

NestJS는 서버 프로그램을 **기능별로 역할을 나누어 구성**하는 프레임워크다.  
역할이 명확히 분리되어 있어 코드가 깔끔하고 유지보수가 쉽다는 특징이 있으며, Spring Boot와 설계 철학이 매우 유사하다.

---

## 핵심 구성 요소

NestJS의 핵심 구성 요소는 크게 세 가지다.

> `Controller` → `Service` → `Module`

---

### 1. Controller — 창구 역할

`Controller`는 클라이언트의 **HTTP 요청을 받고, 응답을 돌려주는 창구 역할**을 담당한다.  
은행 창구처럼 요청을 접수해서 처리 부서(Service)로 넘기고, 결과를 다시 클라이언트에게 전달하는 구조다.

```typescript
@Controller('users')
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Get()
  findAll() {
    return this.userService.findAll(); // 실제 로직은 Service에 위임
  }
}
```

- 비즈니스 로직을 직접 처리하지 않고, **Service에 위임**한다
- `@Get()`, `@Post()` 등의 데코레이터로 HTTP 메서드를 지정한다

---

### 2. Service — 비즈니스 로직 담당

`Service`는 **실제 비즈니스 로직을 처리하는 계층**이다.  
"사용자를 어떻게 조회할 것인가", "비밀번호를 어떻게 검증할 것인가" 같은 핵심 로직이 여기에 위치한다.

```typescript
@Injectable()
export class UserService {
  constructor(private readonly userRepository: UserRepository) {}

  findAll() {
    return this.userRepository.find(); // DB 접근은 Repository에 위임
  }
}
```

> **주의:** Service가 DB에 직접 접근하는 것처럼 보일 수 있지만, 실제로는 `Repository` 계층을 통해 DB에 접근한다.  
> Service는 **"무엇을 할 것인가"**, Repository는 **"어떻게 DB에서 가져올 것인가"** 를 담당한다.

#### Spring Boot와의 비교

Spring Boot를 배웠다면 `Controller → Service → Repository` 3계층 구조가 익숙할 것이다.  
NestJS도 동일하게 Repository를 사용할 수 있다 (TypeORM의 Repository 패턴).  
차이점은 NestJS에 **Module이라는 명시적인 조직화 단위가 추가**된다는 점이다.

| 계층 | Spring Boot | NestJS |
|------|------------|--------|
| 요청/응답 | Controller | Controller |
| 비즈니스 로직 | Service | Service |
| DB 접근 | Repository | Repository (TypeORM) |
| 조직화 단위 | 패키지/컴포넌트 스캔 | **Module (명시적)** |

---

### 3. Module — 기능 단위 묶음

`Module`은 **관련된 Controller와 Service(Provider)를 하나의 기능 단위로 묶는 역할**을 한다.  
NestJS의 모든 기능은 반드시 하나의 Module에 속해야 하며, 앱 전체는 `AppModule`(루트 모듈)을 시작점으로 동작한다.

```typescript
@Module({
  controllers: [UserController],
  providers: [UserService],
  exports: [UserService], // 다른 Module에서 UserService를 사용하려면 exports에 명시
})
export class UserModule {}
```

`@Module()` 데코레이터는 4가지 속성으로 구성된다.

| 속성 | 역할 |
|------|------|
| `controllers` | 이 모듈에 속한 Controller 목록 |
| `providers` | 이 모듈에서 사용할 Service, Repository 등 |
| `imports` | 이 모듈에서 사용할 외부 모듈 |
| `exports` | 다른 모듈에 공개할 Provider 목록 |

#### Module이 왜 필요할까?

처음에는 "React처럼 그냥 `import`해서 쓰면 되는 거 아닌가?" 싶을 수 있다.

React 컴포넌트는 **함수 정의** 자체를 가져오기 때문에 `import`만으로 충분하다.  
하지만 NestJS의 Service는 **인스턴스(객체)** 가 필요하다. 누군가가 만들어주고 관리해줘야 한다.

```ts
// ❌ new로 직접 만들면?
class AuthService {
  private userService = new UserService()
}
class PostService {
  private userService = new UserService() // UserService가 두 개 생김
}
```

DB 연결이나 설정값처럼 **앱 전체에서 하나만 존재해야 하는 것**들이 여러 개 생기면 문제가 발생한다.

더 큰 문제는 의존성이 깊어질수록 개발자가 직접 순서를 맞춰야 한다는 점이다.

```ts
// 의존성 체인이 생기면?
// ConfigService → DatabaseService → UserService → AuthService
const config = new ConfigService()
const db = new DatabaseService(config)
const userService = new UserService(db)
const authService = new AuthService(userService) // 😱 순서 틀리면 에러
```

Module(IoC 컨테이너)을 사용하면 이 과정을 NestJS가 자동으로 처리해준다.

```ts
// ✅ 선언만 해두면 NestJS가 순서 파악 후 알아서 생성
providers: [ConfigService, DatabaseService, UserService, AuthService]
```

**React의 Context와 비교하면 이해가 쉽다.**

```tsx
// Context 없이 → 컴포넌트마다 새 인스턴스
<ComponentA userService={new UserService()} />
<ComponentB userService={new UserService()} /> // 서로 다른 인스턴스!

// Context Provider → 하나만 만들어서 하위에 공유
<UserServiceProvider>
  <ComponentA /> // 같은 인스턴스 공유
  <ComponentB /> // 같은 인스턴스 공유
</UserServiceProvider>
```

NestJS의 Module이 바로 이 `Provider`와 같은 역할이다.  
**"내가 인스턴스를 만들고 관리할 테니, 너네는 그냥 쓰기만 해"** 라는 것이다.

정리하면, Module이 필요한 이유는 아래 두 가지다.

1. **DI 컨테이너의 범위 지정** — NestJS는 Module 단위로 어떤 Service가 어디서 사용 가능한지 관리한다. `exports`에 명시하지 않으면 외부 모듈에서 해당 Service를 사용할 수 없다.
2. **캡슐화** — 내부 구현을 숨기고, 필요한 것만 외부에 공개할 수 있다.

예를 들어, `AuthModule`이 `UserService`를 필요로 한다면:

```typescript
// UserModule에서 exports
@Module({
  providers: [UserService],
  exports: [UserService], // 이 줄이 없으면 AuthModule에서 쓸 수 없다
})
export class UserModule {}

// AuthModule에서 imports
@Module({
  imports: [UserModule], // UserModule을 가져와서 UserService 사용
  providers: [AuthService],
})
export class AuthModule {}
```

---

## Provider란?

`Service`는 사실 **Provider의 한 종류**다.  
`@Injectable()` 데코레이터가 붙은 클래스는 모두 Provider가 될 수 있으며, NestJS의 IoC 컨테이너가 이를 자동으로 관리한다.

```
Provider 종류: Service, Repository, Factory, Helper 등
```

DI(의존성 주입)가 어렵게 느껴진다면, 핵심만 기억하자.  
> **"내가 직접 `new UserService()`를 하지 않아도, NestJS가 알아서 필요한 곳에 넣어준다."**

---

## 모듈화 방식의 장점

NestJS의 모듈 기반 구조는 아래와 같은 장점을 가져온다.

1. **유지보수성 향상** — 기능별 코드 분리로 수정 범위가 명확해진다
2. **확장성 향상** — 새로운 기능을 독립적인 Module로 추가할 수 있다
3. **DI 자동 관리** — NestJS IoC 컨테이너가 의존성을 자동으로 주입해준다
4. **협업 용이성** — 1, 2번의 장점에서 파생되는 효과로, 팀원 간 역할 분리가 명확해진다

---

## 최종 정리

NestJS는 서버 애플리케이션을 **Controller, Service, Module**이라는 세 가지 핵심 구성 요소로 조직화합니다.

`Controller`는 HTTP 요청을 받아 응답을 반환하는 창구 역할을 담당하며, 비즈니스 로직은 `Service`에 위임합니다. `Service`는 실제 비즈니스 로직을 처리하고, DB 접근이 필요한 경우 `Repository` 계층을 통해 간접적으로 접근합니다.

`Module`은 관련된 Controller와 Provider를 하나의 기능 단위로 묶어 캡슐화하고, `imports`와 `exports`를 통해 모듈 간 의존성을 명시적으로 제어합니다. 이를 통해 NestJS의 IoC 컨테이너가 DI를 자동으로 관리할 수 있게 됩니다.

Spring Boot의 `Controller → Service → Repository` 구조와 유사하지만, NestJS는 **Module이라는 명시적인 조직화 단위**가 추가된다는 점이 가장 큰 차이입니다. 이 모듈 기반 구조 덕분에 유지보수성, 확장성, 협업 효율이 높아집니다.