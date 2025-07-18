Chắc chắn rồi. Dưới đây là nội dung tổng hợp của các tài liệu thiết kế từ **Tuần 1-2**, được định dạng bằng Markdown.

Bạn có thể sao chép toàn bộ nội dung bên dưới, dán vào một trình soạn thảo văn bản (như VS Code, Notepad++, Sublime Text) và lưu lại với đuôi `.md` (ví dụ: `Athena_ThietKe_GiaiDoan1.md`) để dễ dàng chỉnh sửa, theo dõi và lưu trữ.

-----

# Hệ thống Athena - Tài liệu Thiết kế Giai đoạn 1

**Phiên bản: 1.0**
**Ngày: 18/07/2025**
**Thực hiện bởi: NVT AIoT**

Đây là tài liệu tổng hợp các quyết định về kiến trúc, cơ sở dữ liệu và API cho Giai đoạn 1 của dự án Athena.

-----

## 1\. Tài liệu Kiến trúc Hệ thống (System Architecture Document)

### 1.1. Tổng quan Kiến trúc

Hệ thống Athena được xây dựng dựa trên **kiến trúc Microservices**. Mỗi chức năng nghiệp vụ chính được đóng gói thành một dịch vụ độc lập, giao tiếp với nhau thông qua các API được định nghĩa rõ ràng.

  * **Triển khai:** Các dịch vụ được **container hóa bằng Docker** và điều phối bằng **Docker Compose** trong môi trường phát triển và triển khai ban đầu trên Ubuntu.
  * **Giao tiếp:**
      * **Client-to-Server (Bên ngoài):** Ứng dụng web (Admin UI) sẽ gọi đến **API Gateway** thông qua **RESTful API** (sử dụng JSON).
      * **Service-to-Service (Nội bộ):** Các microservice giao tiếp với nhau qua **gRPC** để tối ưu hiệu năng.
  * **Luồng dữ liệu:** `Thiết bị -> Protocol Gateway -> Data Ingestion Service -> Database`
  * **Luồng điều khiển:** `Admin UI -> API Gateway -> (Các dịch vụ nghiệp vụ) -> Database`

### 1.2. Sơ đồ Kiến trúc Giai đoạn 1

```
+------------------+      +---------------------+      +------------------------+
|   Thiết bị       |      |     Admin UI        |      |      Bên thứ 3        |
| (PLC, VĐK,...)   |      |      (React)        |      |      (Tương lai)       |
+------------------+      +---------------------+      +------------------------+
        |                        |                             |
        | (MQTT, ModbusTCP)      | (HTTPS/REST)                | (REST API)
        v                        v                             v
+-----------------------------------------------------------------------------+
|                               API Gateway (Traefik)                         |
|      (Routing, Load Balancing, SSL Termination, Authentication)             |
+-----------------------------------------------------------------------------+
        | (MQTT Passthrough)     | (gRPC)                      ^ (gRPC)
        v                        v                             |
+----------------------+   +-----------------------------+     |
| Protocol Gateway     |   | Identity & Access Mgmt (IAM)|     |
| (Go)                 |   | (Go)                        |     |
+----------------------+   +-----------------------------+     |
        | (gRPC)                       ^ | (gRPC)              | (gRPC)
        v                              | v                     |
+----------------------+   +-----------------------------+     |
| Data Ingestion       |   | Device Management           |-----+
| (Go)                 |   | (Go)                        |
+----------------------+   +-----------------------------+
        |                              |
        v                              v
+----------------------+   +-----------------------------+
|   InfluxDB           |   |   PostgreSQL                |
| (Time-series Data)   |   | (Metadata, Users, Devices)  |
+----------------------+   +-----------------------------+

```

### 1.3. Lựa chọn Công nghệ (Technology Stack)

| Thành phần | Công nghệ được chọn | Lý do chính |
| :--- | :--- | :--- |
| **Ngôn ngữ Backend** | **Go (Golang)** | Hiệu năng cao, xử lý đồng thời tốt, triển khai đơn giản. |
| **Cơ sở dữ liệu Quan hệ** | **PostgreSQL** | Ổn định, mạnh mẽ, hỗ trợ tốt kiểu dữ liệu JSONB. |
| **Cơ sở dữ liệu Chuỗi thời gian** | **InfluxDB** | Chuyên dụng cho time-series, hiệu năng ghi/đọc cực cao. |
| **Giao tiếp nội bộ** | **gRPC** | Hiệu năng cao, định nghĩa dịch vụ rõ ràng qua `.proto`. |
| **Giao tiếp bên ngoài** | **RESTful API (JSON)** | Phổ biến, dễ tích hợp với các client. |
| **API Gateway** | **Traefik** | Tự động khám phá dịch vụ (service discovery) với Docker. |
| **Containerization** | **Docker & Docker Compose** | Tiêu chuẩn ngành, đảm bảo môi trường nhất quán. |
| **Frontend Framework** | **React (với Vite)** | Hệ sinh thái lớn, hiệu năng tốt, trải nghiệm dev nhanh. |

-----

## 2\. Tài liệu Thiết kế Cơ sở dữ liệu (Database Design Document)

### 2.1. Cơ sở dữ liệu PostgreSQL (Lưu dữ liệu cấu trúc)

Dùng cho dịch vụ **IAM** và **Device Management**.

**Sơ đồ quan hệ (ERD - bản đơn giản):**
`[users] --< [user_roles] >-- [roles]`
`[devices] --< [device_profiles]`

**Chi tiết các bảng (SQL DDL):**

```sql
-- Bảng lưu thông tin người dùng
CREATE TABLE "users" (
  "id" uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  "username" varchar(100) UNIQUE NOT NULL,
  "email" varchar(255) UNIQUE NOT NULL,
  "hashed_password" varchar(255) NOT NULL,
  "full_name" varchar(255),
  "is_active" boolean DEFAULT true,
  "created_at" timestamptz NOT NULL DEFAULT now(),
  "updated_at" timestamptz
);

-- Bảng lưu các vai trò trong hệ thống
CREATE TABLE "roles" (
  "id" serial PRIMARY KEY,
  "name" varchar(50) UNIQUE NOT NULL, -- Ví dụ: 'admin', 'editor', 'viewer'
  "description" text
);

-- Bảng trung gian để gán vai trò cho người dùng (quan hệ nhiều-nhiều)
CREATE TABLE "user_roles" (
  "user_id" uuid NOT NULL REFERENCES "users"("id") ON DELETE CASCADE,
  "role_id" integer NOT NULL REFERENCES "roles"("id") ON DELETE CASCADE,
  PRIMARY KEY ("user_id", "role_id")
);

-- Bảng lưu hồ sơ thiết bị, định nghĩa loại thiết bị và giao thức
CREATE TABLE "device_profiles" (
   "id" uuid PRIMARY KEY DEFAULT gen_random_uuid(),
   "name" varchar(255) UNIQUE NOT NULL,
   "protocol_type" varchar(50) NOT NULL, -- Ví dụ: 'MQTT', 'MODBUS_TCP'
   "description" text,
   "created_at" timestamptz NOT NULL DEFAULT now()
);

-- Bảng lưu thông tin các thiết bị vật lý
CREATE TABLE "devices" (
  "id" uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  "name" varchar(255) NOT NULL,
  "device_profile_id" uuid NOT NULL REFERENCES "device_profiles"("id"),
  "access_token" varchar(255) UNIQUE NOT NULL, -- Dùng để xác thực từ thiết bị
  "metadata" jsonb, -- Lưu các thông tin phụ, ví dụ: { "location": "Floor 1", "serial_number": "XYZ123" }
  "is_active" boolean DEFAULT true,
  "last_seen_at" timestamptz,
  "created_at" timestamptz NOT NULL DEFAULT now(),
  "updated_at" timestamptz
);
```

### 2.2. Cơ sở dữ liệu InfluxDB (Lưu dữ liệu chuỗi thời gian)

Dùng cho dịch vụ **Data Ingestion**.

  * **Bucket:** `athena_main` (Một "database" trong InfluxDB)

  * **Data Retention Policy:** 1 năm (Dữ liệu cũ hơn 1 năm sẽ tự động bị xóa để tiết kiệm dung lượng).

  * **Cấu trúc dữ liệu (Line Protocol):**

    ```
    <measurement>,<tag_set> <field_set> <timestamp>
    ```

      * **Measurement:** `telemetry` (tên của "bảng" dữ liệu)
      * **Tag Set (Dữ liệu được index):**
          * `deviceId` (UUID của thiết bị, lấy từ PostgreSQL)
          * `profileId` (UUID của hồ sơ thiết bị)
          * `dataKey` (Tên của giá trị đo, ví dụ: `temperature`, `humidity`, `pressure`)
      * **Field Set (Giá trị đo được):**
          * `value_float` (dùng cho số thực)
          * `value_int` (dùng cho số nguyên)
          * `value_string` (dùng cho chuỗi)
          * `value_bool` (dùng cho boolean)
          * *Lưu ý: Chỉ một trong các trường `value_*` có giá trị tại một thời điểm.*
      * **Timestamp:** Thời gian ghi nhận dữ liệu (định dạng Unix nanosecond).

  * **Ví dụ một điểm dữ liệu:**
    `telemetry,deviceId=a1b2c3d4,profileId=e5f6g7h8,dataKey=temperature value_float=25.5 1678886400000000000`

-----

## 3\. Tài liệu Đặc tả API (API Specification)

### 3.1. API Bên ngoài (RESTful - cho Admin UI)

**Base URL:** `/api/v1`
**Authentication:** Sử dụng `Bearer Token` trong header `Authorization`. Token được lấy từ endpoint đăng nhập.

**Endpoints chính:**

  * **Auth Service:**
      * `POST /auth/login` - Body: `{ "username": "...", "password": "..." }` -\> Response: `{ "access_token": "..." }`
      * `POST /auth/refresh` - Làm mới token.
  * **Device Management Service:**
      * `POST /devices` - Tạo thiết bị mới.
      * `GET /devices` - Lấy danh sách thiết bị (hỗ trợ phân trang).
      * `GET /devices/{id}` - Lấy chi tiết một thiết bị.
      * `PUT /devices/{id}` - Cập nhật thiết bị.
      * `DELETE /devices/{id}` - Xóa thiết bị.
  * **User Management Service:**
      * `POST /users` - Tạo người dùng mới.
      * `GET /users` - Lấy danh sách người dùng.

### 3.2. API Nội bộ (gRPC)

Định nghĩa bằng Protocol Buffers (`.proto` files).

**Ví dụ `device.proto`:**

```protobuf
syntax = "proto3";

package device;

option go_package = "athena/pb/device";

// Dịch vụ quản lý thiết bị
service DeviceService {
  // Xác thực thiết bị bằng access token
  rpc AuthenticateDevice(AuthenticateDeviceRequest) returns (AuthenticateDeviceResponse);
}

message AuthenticateDeviceRequest {
  string access_token = 1;
}

message AuthenticateDeviceResponse {
  bool is_valid = 1;
  string device_id = 2; // UUID của thiết bị
  string profile_id = 3; // UUID của profile
}
```

**Ví dụ `iam.proto`:**

```protobuf
syntax = "proto3";

package iam;

option go_package = "athena/pb/iam";

// Dịch vụ quản lý định danh và truy cập
service IAMService {
  // Xác thực JWT token từ người dùng
  rpc ValidateToken(ValidateTokenRequest) returns (ValidateTokenResponse);
}

message ValidateTokenRequest {
  string token = 1;
}

message ValidateTokenResponse {
  bool is_valid = 1;
  string user_id = 2;
  repeated string roles = 3; // Danh sách vai trò của người dùng
}
```
