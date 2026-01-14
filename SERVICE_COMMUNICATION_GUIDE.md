# Hướng dẫn chi tiết giao tiếp giữa các service (dành cho người mới)

> **Mục tiêu:** Tài liệu này mô tả *chi tiết nhất có thể* cách các service trong dự án giao tiếp, phân tích theo tầng (layer), kèm ví dụ code/config. Nội dung dài và đầy đủ để người mới có thể học dần từng phần.

---

## 1. Tổng quan kiến trúc giao tiếp (Bird’s‑eye view)
Hệ thống sử dụng **microservices**, mỗi service đảm trách một mảng nghiệp vụ. Dòng chảy giao tiếp gồm:

1. **Client (Web/SPA)** → **API Gateway (AgentPortalApiGateway)**
2. **API Gateway** → **Backend services** (Product/Policy/Pricing/Payment/PolicySearch/Dashboard)
3. **Service‑to‑Service** theo 2 kiểu:
   - **Synchronous REST** (cần phản hồi ngay): ví dụ PolicyService gọi PricingService để tính giá.
   - **Asynchronous events** (không cần phản hồi ngay): publish/subscribe qua RabbitMQ để cập nhật search & dashboard.
4. **Service Discovery (Eureka)**: Gateway và các service đăng ký/tìm nhau qua Eureka.
5. **Read models/Search**: Elasticsearch lưu dữ liệu tìm kiếm và phân tích.

Nội dung phía dưới sẽ phân tích theo **layer** để người mới hiểu từng tầng và cách các tầng kết nối với nhau.

---

## 2. Phân tích theo layer (tầng kiến trúc)

### 2.1. Client/UI Layer (Web/SPA)
- **Vai trò:** UI gọi API để hiển thị dữ liệu và gửi thao tác (tạo policy, tìm kiếm, dashboard, v.v.).
- **Đặc điểm:** Không gọi trực tiếp từng service mà thông qua Gateway.

**Flow cơ bản:**
```
Browser -> API Gateway -> Microservice
```

### 2.2. API Gateway Layer (AgentPortalApiGateway)
- **Vai trò:**
  - Single entry point cho client.
  - Xác thực JWT.
  - Định tuyến request tới đúng service.
  - Cache một số route GET (TTL 15s).

#### Ví dụ cấu hình route (Ocelot)
```json
{
  "DownstreamPathTemplate": "/api/Products",
  "DownstreamScheme": "http",
  "UpstreamPathTemplate": "/api/Products",
  "ServiceName": "ProductService",
  "UpstreamHttpMethod": ["Get"],
  "FileCacheOptions": { "TtlSeconds": 15 }
}
```

#### Cache ở gateway
Gateway dùng **Ocelot CacheManager** và cấu hình **in‑memory dictionary handle**:
```csharp
s.AddOcelot().AddEureka().AddCacheManager(x => x.WithDictionaryHandle());
```

> Điều này nghĩa là cache nằm trong bộ nhớ của gateway, **không phải distributed cache** (không dùng Redis trong cấu hình hiện tại).

### 2.3. Service Layer (Microservices)
Mỗi service có thể có cấu trúc lớp (layer) tương tự:
- **API layer** (Controllers hoặc handlers nhận request)
- **Application layer** (use case / business workflow)
- **Domain layer** (entities, logic domain)
- **Infrastructure layer** (DB, messaging)

Trong dự án:
- **PolicyService**: CQRS, NHibernate.
- **PricingService**: Marten (Postgres Document DB).
- **PaymentService**: Marten + Dapper + jobs.
- **PolicySearchService**: Elasticsearch.
- **DashboardService**: Elasticsearch.
- **ProductService**: EF Core (InMemory hoặc SQL Server).

### 2.4. Messaging Layer (RabbitMQ)
- **Vai trò:** truyền sự kiện giữa service, tránh coupling.
- **Ví dụ:** PolicyService publish `PolicyCreatedEvent` → PolicySearchService & DashboardService subscribe.

### 2.5. Data Layer
- **PostgreSQL**: PaymentService, PolicyService, PricingService.
- **Elasticsearch**: PolicySearchService, DashboardService.
- **InMemory/SQL Server** (ProductService).

---

## 3. Giao tiếp đồng bộ (REST) chi tiết

### 3.1. Khi nào dùng REST sync?
- Khi cần **response ngay lập tức** để tiếp tục xử lý.
- Ví dụ: PolicyService cần giá từ PricingService trước khi tạo offer/policy.

### 3.2. Flow chi tiết: PolicyService → PricingService

**Bước 1:** Client tạo offer (POST `/api/Offers`) qua Gateway.

**Bước 2:** Gateway route sang PolicyService.

**Bước 3:** PolicyService gọi PricingService để lấy giá.

#### Pseudo‑code minh hoạ (PolicyService)
```csharp
// 1. Nhận request tạo offer
public async Task<OfferDto> CreateOffer(CreateOfferRequest request)
{
    // 2. Gọi PricingService
    var price = await pricingClient.CalculatePrice(request);

    // 3. Dùng giá để tạo offer
    var offer = Offer.Create(request, price);

    // 4. Lưu vào DB
    await offerRepository.SaveAsync(offer);

    return OfferDto.From(offer);
}
```

### 3.3. Ưu/nhược điểm REST sync
- ✅ **Ưu:** đơn giản, dễ hiểu, phù hợp flow cần kết quả ngay.
- ❌ **Nhược:** phụ thuộc chặt (nếu PricingService down thì PolicyService fail).

---

## 4. Giao tiếp bất đồng bộ (RabbitMQ) chi tiết

### 4.1. Khi nào dùng async events?
- Khi chỉ cần **thông báo** cho service khác và không yêu cầu phản hồi ngay.
- Ví dụ: PolicyService tạo policy xong thì gửi event để các service khác cập nhật read model.

### 4.2. Publish event từ PolicyService

**Pseudo‑code publish:**
```csharp
// Sau khi tạo policy thành công
await bus.PublishAsync(new PolicyCreatedEvent
{
    PolicyNumber = policy.Number,
    CustomerId = policy.CustomerId,
    ProductCode = policy.ProductCode,
    CreatedAt = DateTime.UtcNow
});
```

### 4.3. Subscribe event ở PolicySearchService

**Pseudo‑code subscribe:**
```csharp
bus.SubscribeAsync<PolicyCreatedEvent>("policy-search", async message =>
{
    // Chuyển thành read model
    var document = new PolicySearchDocument
    {
        PolicyNumber = message.PolicyNumber,
        CustomerId = message.CustomerId,
        ProductCode = message.ProductCode,
        CreatedAt = message.CreatedAt
    };

    // Index vào Elasticsearch
    await elasticClient.IndexAsync(document);
});
```

### 4.4. Subscribe event ở DashboardService

**Pseudo‑code subscribe:**
```csharp
bus.SubscribeAsync<PolicyCreatedEvent>("dashboard", async message =>
{
    var salesRecord = new SalesRecord
    {
        PolicyNumber = message.PolicyNumber,
        ProductCode = message.ProductCode,
        CreatedAt = message.CreatedAt
    };

    await elasticClient.IndexAsync(salesRecord);
});
```

### 4.5. Ưu/nhược điểm async events
- ✅ **Ưu:** loose coupling, mở rộng tốt, service có thể xử lý độc lập.
- ❌ **Nhược:** eventual consistency (không nhất quán ngay lập tức).

---

## 5. API Gateway chi tiết

### 5.1. Route mapping
Gateway định tuyến dựa trên `ocelot.json`. Ví dụ:

| Upstream (Client) | Downstream (Service) | ServiceName |
|---|---|---|
| `/api/Products` | `/api/Products` | ProductService |
| `/api/Policies` | `/api/Policy` | PolicyService |
| `/api/Dashboard` | `/api/Dashboard` | DashboardService |

### 5.2. Cache ở Gateway
- Cache GET routes với TTL 15s.
- Dùng Ocelot CacheManager in‑memory.

**Khi nào cache hiệu quả?**
- Dữ liệu ít thay đổi (danh sách products, lookup).

---

## 6. Service Discovery (Eureka) chi tiết

### 6.1. Lý do dùng Eureka
- Trong môi trường container, IP/port có thể thay đổi.
- Eureka giúp Gateway và service khác tìm đúng địa chỉ.

### 6.2. Ví dụ config Eureka (docker)
```json
{
  "Eureka": {
    "Client": {
      "ServiceUrl": "http://eureka-server:8761/eureka/"
    },
    "Instance": {
      "HostName": "dotnet-policy-service"
    }
  }
}
```

### 6.3. Quy trình
1. Service khởi động → đăng ký Eureka.
2. Gateway query Eureka → biết endpoint service.
3. Request được route đến đúng service.

---

## 7. Phân tích từng service & vai trò giao tiếp

### 7.1. ProductService
- **Nhận request từ Gateway:** `GET /api/Products`.
- **DB:** EF Core (InMemory/SQL Server).
- **Không publish event**.

### 7.2. PolicyService
- **Nhận request:** tạo offer/policy.
- **Gọi PricingService** (REST sync).
- **Publish event** khi tạo policy xong.

### 7.3. PricingService
- **Nhận request từ PolicyService** để tính giá.
- **Không publish event**.

### 7.4. PaymentService
- **Nhận request:** quản lý policy account & payment.
- **Có background jobs (Hangfire)**.

### 7.5. PolicySearchService
- **Nhận event từ PolicyService**.
- **Index vào Elasticsearch**.
- **Nhận request GET search từ Gateway**.

### 7.6. DashboardService
- **Nhận event từ PolicyService**.
- **Index dữ liệu** để tính thống kê.
- **Nhận request dashboard từ Gateway**.

---

## 8. Luồng nghiệp vụ chi tiết (End‑to‑End)

### 8.1. Mua bảo hiểm (Offer → Policy)
1. Client gọi `POST /api/Offers` qua Gateway.
2. Gateway route → PolicyService.
3. PolicyService gọi PricingService (REST) lấy giá.
4. PolicyService lưu offer/policy.
5. PolicyService publish `PolicyCreatedEvent` lên RabbitMQ.
6. PolicySearchService nhận event → index Elasticsearch.
7. DashboardService nhận event → cập nhật thống kê.

### 8.2. Tìm kiếm policy
1. Client gọi `GET /api/PolicySearch` qua Gateway.
2. Gateway route → PolicySearchService.
3. PolicySearchService query Elasticsearch và trả kết quả.

---

## 9. Các ví dụ code đầy đủ (minh hoạ cấu trúc)

> Đây là ví dụ cấu trúc code theo style của dự án để người mới dễ hình dung.

### 9.1. Ví dụ Controller (API Layer)
```csharp
[ApiController]
[Route("api/[controller]")]
public class PoliciesController : ControllerBase
{
    private readonly IPolicyApplicationService service;

    public PoliciesController(IPolicyApplicationService service)
    {
        this.service = service;
    }

    [HttpPost]
    public async Task<IActionResult> CreatePolicy(CreatePolicyRequest request)
    {
        var result = await service.CreatePolicyAsync(request);
        return Ok(result);
    }
}
```

### 9.2. Ví dụ Application Service
```csharp
public class PolicyApplicationService : IPolicyApplicationService
{
    private readonly IPricingClient pricingClient;
    private readonly IPolicyRepository policyRepository;
    private readonly IBus bus;

    public async Task<PolicyDto> CreatePolicyAsync(CreatePolicyRequest request)
    {
        var price = await pricingClient.CalculatePrice(request);
        var policy = Policy.Create(request, price);

        await policyRepository.SaveAsync(policy);

        await bus.PublishAsync(new PolicyCreatedEvent { PolicyNumber = policy.Number });

        return PolicyDto.From(policy);
    }
}
```

### 9.3. Ví dụ Repository (Data Layer)
```csharp
public class ProductRepository : IProductRepository
{
    private readonly ProductDbContext db;

    public ProductRepository(ProductDbContext db)
    {
        this.db = db;
    }

    public async Task AddAsync(Product product)
    {
        await db.Products.AddAsync(product);
        await db.SaveChangesAsync();
    }
}
```

### 9.4. Ví dụ RabbitMQ Subscriber
```csharp
public class PolicyCreatedSubscriber
{
    private readonly IBus bus;
    private readonly ElasticsearchClient elastic;

    public Task StartAsync()
    {
        return bus.SubscribeAsync<PolicyCreatedEvent>("policy-search", async message =>
        {
            await elastic.IndexAsync(message);
        });
    }
}
```

---

## 10. Lưu ý cho người mới
- **Bắt đầu với REST sync** trước, sau đó mới tìm hiểu RabbitMQ.
- **Đọc cấu hình Gateway** để hiểu route.
- **Elasticsearch** chỉ dùng cho search & analytics, không phải DB chính.

---

## 11. Tóm tắt nhanh (cheat sheet)
- **Gateway**: route + auth + cache.
- **PolicyService**: core workflow, gọi PricingService, publish events.
- **PolicySearchService**: lắng nghe events, index Elasticsearch.
- **DashboardService**: lắng nghe events, tính thống kê.
- **Eureka**: service discovery.

---

> Nếu bạn cần thêm ví dụ cụ thể từ code thật trong project (class, method, hoặc file cụ thể), hãy yêu cầu và mình sẽ trích xuất trực tiếp từ repo.

