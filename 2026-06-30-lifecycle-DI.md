# 2026-06-30 DI 생명주기 선택 기준

## 배경

본인인증 기능을 만들면서 `IdentityVerificationService`를 DI에 등록해야 했다.

이때 서비스를 `AddScoped`으로 등록할지, `AddTransient`로 등록할지 고민이 생겼다.

## 문제

DI 생명주기를 제대로 이해하지 못하면 서비스 객체가 의도치 않게 공유될 수 있다.

특히 본인인증 서비스처럼 사용자 요청과 관련된 처리를 하는 경우, 객체가 잘못 공유되면 사용자별 데이터나 인증 상태가 섞일 위험이 있다.

## 고민

`Singleton`은 애플리케이션 전체에서 객체를 하나만 생성해 계속 공유한다.

따라서 설정값, 캐시, 공통 리소스처럼 전역으로 공유해도 되는 객체에 적합하다.

반대로 서비스 내부에 사용자별 값, 요청별 값, 인증 결과, 토큰 등을 저장한다면 `Singleton`은 위험하다.

---

`Transient`는 서비스를 요청할 때마다 새 객체를 생성한다.

즉, 같은 HTTP 요청 안에서도 여러 번 주입되면 매번 다른 인스턴스가 만들어진다.

상태를 저장하지 않고 외부 API 호출이나 단순 처리만 수행하는 서비스라면 안전하게 사용할 수 있다.

---

`Scoped`는 HTTP 요청 하나 동안 같은 객체를 공유한다.

겉보기에는 요청 단위라서 `Transient`와 비슷해 보일 수 있지만, 중요한 차이가 있다.

* `Transient`: 같은 요청 안에서도 주입될 때마다 새로운 객체 생성
* `Scoped`: 같은 요청 안에서는 항상 동일한 객체 재사용

이 차이는 특히 다음 상황에서 중요해진다.

---

### 1. 여러 서비스가 동일한 의존성을 공유해야 할 때

예를 들어 주문 처리 흐름에서 여러 서비스가 동일한 `DbContext`를 사용해야 하는 경우를 생각해볼 수 있다.

```csharp
public class OrderService
{
    private readonly AppDbContext _db;

    public OrderService(AppDbContext db)
    {
        _db = db;
    }

    public void CreateOrder()
    {
        _db.Orders.Add(new Order());
    }
}

public class PaymentService
{
    private readonly AppDbContext _db;

    public PaymentService(AppDbContext db)
    {
        _db = db;
    }

    public void ProcessPayment()
    {
        _db.Payments.Add(new Payment());
    }
}
```

이 두 서비스가 같은 요청 안에서 동일한 `DbContext`를 공유해야 하나의 작업 흐름으로 묶을 수 있다.

`Scoped`를 사용하면 같은 요청 안에서 동일한 인스턴스를 사용하게 된다.

---

### 2. 하나의 요청 안에서 상태 일관성이 필요할 때

예를 들어 로그인 후 사용자 정보를 조회하고, 이후 권한 체크를 수행하는 경우를 생각해볼 수 있다.

```csharp
public class UserContext
{
    public int UserId { get; set; }
}

public class AuthService
{
    private readonly UserContext _context;

    public AuthService(UserContext context)
    {
        _context = context;
    }

    public void SetUser(int userId)
    {
        _context.UserId = userId;
    }
}

public class AuthorizationService
{
    private readonly UserContext _context;

    public AuthorizationService(UserContext context)
    {
        _context = context;
    }

    public bool IsAdmin()
    {
        return _context.UserId == 1;
    }
}
```

이 경우 `UserContext`를 `Scoped`로 등록하면 같은 요청 안에서 동일한 사용자 정보를 공유할 수 있다.

---

### 3. DbContext처럼 트랜잭션 단위로 동작해야 할 때

EF Core의 `DbContext`는 대표적인 `Scoped` 대상이다.

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));
```

하나의 요청 안에서 여러 작업이 수행될 때 동일한 `DbContext`를 사용해야 트랜잭션을 일관되게 유지할 수 있다.

만약 `Transient`로 등록하면 각 서비스마다 다른 `DbContext` 인스턴스를 사용하게 되어 트랜잭션 관리가 어려워진다.

---

## 현재 결론

본인인증 서비스는 `IdentityVerificationService`로 명명한다.

서비스 역할은 다음과 같다.

* 본인인증 API 요청
* 본인인증 결과 확인
* 결과 반환

서비스 내부에 사용자별 상태를 저장하지 않고, 요청마다 필요한 값을 메서드 파라미터로 받아 처리한다면 `AddTransient`로 등록한다.

```csharp
builder.Services.AddTransient<IIdentityVerificationService, IdentityVerificationService>();
```

단, 서비스 내부에서 `DbContext`를 사용하거나 요청 단위의 일관성이 중요하다면 `AddScoped`도 고려한다.

```csharp
builder.Services.AddScoped<IIdentityVerificationService, IdentityVerificationService>();
```

`AddSingleton`은 본인인증 서비스에는 기본 선택지로 두지 않는다.

---

## 의문
* 외부 API 호출용 `HttpClient`는 왜 `Singleton`처럼 재사용하는 것이 좋다고 할까?
* 본인인증 요청 이력/결과 저장까지 담당한다면 서비스 생명주기를 다시 봐야 할까?
