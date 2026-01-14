# Hướng dẫn chi tiết giao tiếp giữa các service

> **Mục đích:** Tài liệu này dành cho người mới để hiểu cách các service trong dự án giao tiếp với nhau, gồm **giao tiếp đồng bộ (REST)** và **bất đồng bộ (RabbitMQ)**, cũng như vai trò của **API Gateway** và **Service Discovery (Eureka)**.

## 1. Tổng quan kiến trúc giao tiếp
Hệ thống sử dụng kiến trúc microservices. Giao tiếp được chia làm hai loại chính:

1. **Đồng bộ (synchronous)**: gọi REST giữa các service.
2. **Bất đồng bộ (asynchronous)**: publish/subscribe qua RabbitMQ.

Ngoài ra, hệ thống có:
- **API Gateway (AgentPortalApiGateway)**: điểm vào duy nhất từ frontend.
- **Service Discovery (Eureka)**: quản lý đăng ký và tìm kiếm service.
- **Search/Analytics**: ElasticSearch cho tìm kiếm và thống kê.

## 2. Giao tiếp từ client (Web) tới hệ thống
### 2.1 Client gọi API Gateway
- Frontend **Web (VueJS)** không gọi trực tiếp từng service mà gọi qua **AgentPortalApiGateway**.
- Gateway chịu trách nhiệm:
  - Xác thực (JWT)
  - Định tuyến request tới service phù hợp
  - Cache một số route GET (TTL 15s)

**Ví dụ:**
- `/api/Products` → Gateway định tuyến tới **ProductService**.
- `/api/Policies` → Gateway định tuyến tới **PolicyService**.

### 2.2 Cache tại API Gateway
- Gateway dùng **Ocelot CacheManager** với in-memory dictionary handle.
- Một số route GET có `FileCacheOptions` TTL 15s trong `ocelot.json`.

## 3. Giao tiếp đồng bộ giữa các service (REST)
### 3.1 PolicyService → PricingService
- **Use case:** Khi tạo offer/policy, PolicyService cần tính giá.
- **Cách hoạt động:** PolicyService gọi HTTP REST sang PricingService.
- **Ý nghĩa:** Đây là giao tiếp đồng bộ, PolicyService chờ kết quả giá trả về.

### 3.2 Các service nhận request từ Gateway
- **ProductService**: cung cấp danh sách sản phẩm.
- **PolicyService**: tạo offer, policy, truy vấn policy.
- **PolicySearchService**: tìm kiếm policy.
- **DashboardService**: lấy thống kê doanh số.

> Lưu ý: Giao tiếp đồng bộ thường dùng cho các nghiệp vụ cần phản hồi ngay.

## 4. Giao tiếp bất đồng bộ (RabbitMQ)
### 4.1 PolicyService publish event
- Khi policy được tạo, PolicyService **publish event** lên RabbitMQ.
- Event này được các service khác lắng nghe để cập nhật dữ liệu đọc (read model).

### 4.2 PolicySearchService subscribe event
- PolicySearchService **subscribe** các event từ PolicyService.
- Nó chuyển đổi dữ liệu và index vào **Elasticsearch** để hỗ trợ tìm kiếm nhanh.

### 4.3 DashboardService subscribe event
- DashboardService cũng **subscribe** các event bán policy.
- Nó lưu dữ liệu vào Elasticsearch và dùng aggregation để tạo thống kê.

> Lưu ý: Giao tiếp bất đồng bộ giúp tách rời service, giảm phụ thuộc và tăng khả năng mở rộng.

## 5. Service Discovery với Eureka
- Mỗi service đăng ký với Eureka khi khởi động.
- API Gateway dùng Eureka để tìm địa chỉ service thực tế.
- Khi service scale hoặc đổi địa chỉ, Gateway vẫn có thể tìm service qua Eureka.

## 6. Luồng nghiệp vụ mẫu (end-to-end)
### 6.1 Tạo offer và policy
1. Client gọi `POST /api/Offers` qua Gateway.
2. Gateway định tuyến đến PolicyService.
3. PolicyService gọi PricingService để tính giá (REST sync).
4. PolicyService lưu offer/policy.
5. PolicyService publish event lên RabbitMQ.
6. PolicySearchService và DashboardService nhận event và cập nhật Elasticsearch.

### 6.2 Tra cứu policy
1. Client gọi `GET /api/PolicySearch` qua Gateway.
2. Gateway định tuyến đến PolicySearchService.
3. PolicySearchService query Elasticsearch và trả kết quả.

## 7. Gợi ý học cho người mới
- Hiểu REST sync trước (PolicyService → PricingService).
- Sau đó tìm hiểu event-driven với RabbitMQ.
- Cuối cùng học thêm về API Gateway và Eureka.

---

## Tóm tắt nhanh
- **Gateway**: điểm vào duy nhất, xác thực + route + cache.
- **REST sync**: dùng khi cần phản hồi ngay (PolicyService ↔ PricingService).
- **RabbitMQ async**: dùng để broadcast sự kiện, cập nhật search & dashboard.
- **Elasticsearch**: lưu read model cho search và analytics.
- **Eureka**: quản lý service discovery.

