# Service Boundary của nhóm

## 1. Thông tin nhóm

* Tên nhóm: 3B
* Lớp: CNTT 17-12
* Thành viên: Nguyễn Văn Thống,Lê Tiến Được,Nguyễn Xuân Hiệp,Phạm Đình Tuấn Anh
* Service nhóm phụ trách: Camera Stream
* Sản phẩm tổng thể của lớp: Smart Campus Ecosystem

---

## 2. Actor

Các actor tương tác với service:

* Camera IP (RTSP Stream)
* AI Vision Service
* Core Business Service
* Client (Web Dashboard / Monitoring System)

---

## 3. System Boundary

### Phần nhóm kiểm soát:

* Camera Stream Service
* Nhận luồng RTSP từ camera
* Buffer frame (ảnh)
* Xử lý phát hiện chuyển động (motion detection)
* Gửi dữ liệu sang AI Vision

### Phần nhóm chỉ tích hợp:

* AI Vision Service (xử lý nhận diện)
* Core Business Service
* Message Broker (RabbitMQ)
* API Gateway

---

## 4. Service Boundary

### Service có trách nhiệm:

* Nhận luồng RTSP từ Camera IP
* Chuyển đổi video thành frame
* Buffer và quản lý frame
* Phát hiện chuyển động (motion detection)
* Khi có chuyển động:

  * Gửi frame sang AI Vision Service
* Cung cấp API stream hoặc dữ liệu cho client

### Service KHÔNG làm:

* Không xử lý AI (YOLO, nhận diện người)
* Không lưu trữ dữ liệu dài hạn
* Không xử lý nghiệp vụ (business logic)
* Không gửi notification

---

## 5. Input / Output

### Input

* RTSP stream từ Camera IP
* Request từ client/API Gateway

### Output

* Frame ảnh (khi có chuyển động)
* Metadata (timestamp, event motion)
* Stream video (nếu có API)
* Dữ liệu gửi sang AI Vision Service

---

## 6. API dự kiến

| Method | Endpoint       | Mục đích                     |
| ------ | -------------- | ---------------------------- |
| GET    | /health        | Kiểm tra service             |
| GET    | /stream        | Xem video stream             |
| GET    | /frames        | Lấy frame hiện tại           |
| POST   | /detect-motion | Trigger kiểm tra chuyển động |
| POST   | /push-frame    | Gửi frame sang AI Vision     |

---

## 7. Phụ thuộc service khác

### Service này gọi đến:

* AI Vision Service (để nhận diện ảnh)
* Message Broker (RabbitMQ – gửi event)

### Service gọi đến service này:

* API Gateway
* Core Business Service
* Client (Dashboard)

---

## 8. Sơ đồ minh họa
flowchart LR
    %% ===== INPUT =====
    Camera[Camera IP\n(RTSP Stream)] -->|RTSP| Ingest

    %% ===== CAMERA STREAM SERVICE =====
    subgraph Camera_Stream_Service
        Ingest[Nhận luồng RTSP]
        Decode[Decode video (OpenCV)]
        Buffer[Frame Buffer]
        Detect{Có chuyển động?}
        Extract[Trích xuất Frame]
        Encode[Encode ảnh / clip]
        Publish[Gửi dữ liệu (Event/API)]

        Ingest --> Decode
        Decode --> Buffer
        Buffer --> Detect
        Detect -- Không --> Buffer
        Detect -- Có --> Extract
        Extract --> Encode
        Encode --> Publish
    end

    %% ===== OUTPUT =====
    Publish -->|Image/Event| AI[AI Vision Service]
    Publish -->|Metadata| Broker[(Message Broker\nKafka / RabbitMQ)]

    %% ===== AI PROCESS =====
    AI --> Result[Kết quả nhận diện]

    %% ===== CORE SYSTEM =====
    Result --> Core[Core Business Service]
    Core --> DB[(Database)]
    Core --> Notify[Notification Service]

    %% ===== API / CLIENT =====
    Ingest --> API[FastAPI Endpoint]
    API --> Client[Web / Mobile Client]

    %% ===== STYLE =====
    classDef input fill:#cce5ff,stroke:#333;
    classDef process fill:#fff3cd,stroke:#333;
    classDef decision fill:#ffe6e6,stroke:#333;
    classDef storage fill:#d4edda,stroke:#333;
    classDef ai fill:#e2ccff,stroke:#333;

    class Camera input;
    class Ingest,Decode,Buffer,Extract,Encode,Publish process;
    class Detect decision;
    class Broker,DB storage;
    class AI ai;