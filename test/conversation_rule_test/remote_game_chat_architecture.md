# Kiến trúc điều khiển game từ xa qua AI Chat

## 1. Bối cảnh

- Desktop app nằm tại:

~~~text
D:\Project\Auto\Auto_MU
~~~

- App có tab AI chat.
- Hiện tại chỉ người dùng trên cửa sổ desktop mới có thể chat với AI và ra lệnh cho game.
- Mỗi máy cài app cũng là máy chạy game tương ứng.
- Domain hiện có: mcp88.uk.
- Cloudflare Tunnel hiện có, nhưng không phải thành phần định tuyến chính của kiến trúc multi-machine.

## 2. Mục tiêu đã xác nhận

Hệ thống cần hỗ trợ:

- Nhiều máy cùng cài và chạy desktop app đồng thời.
- Mỗi máy có một game session riêng.
- Người dùng từ điện thoại, trình duyệt hoặc app khác có thể kết nối từ bất kỳ đâu.
- Client chỉ cần nhập mã phiên để vào đúng game session.
- Nhiều client có thể cùng tham gia một session.
- Message từ xa được đưa vào cùng luồng AI chat hiện tại của desktop app.
- AI và game executor vẫn chạy trên máy đang chạy game.
- Khi desktop app tắt, session của máy đó kết thúc.

## 3. Kiến trúc đã chốt

~~~text
Desktop App A ─┐
Desktop App B ─┼── WebSocket outbound ──> Cloudflare Worker
Desktop App C ─┘                              │
                                             ▼
                                      Durable Object
                                      theo từng session
                                             ▲
                                             │
Remote Client A ── WebSocket ────────────────┤
Remote Client B ── WebSocket ────────────────┘
~~~

### Nguyên tắc chính

- Worker là public entry point.
- Durable Object là điểm điều phối của từng session.
- Desktop app là WebSocket client, không phải public server Internet.
- Desktop app vẫn là nơi xử lý AI và điều khiển game.
- Remote client không kết nối trực tiếp tới máy game.
- Cloudflare Tunnel không thay thế Worker/Durable Object trong mô hình nhiều máy.

## 4. Vai trò của từng thành phần

### 4.1. Remote client

Remote client có thể là:

- Web client;
- app điện thoại;
- desktop app khác;
- client AI hoặc công cụ tự xây dựng khác.

Nhiệm vụ:

- kết nối tới domain công khai;
- gửi session code;
- tham gia session;
- gửi message;
- nhận phản hồi và trạng thái.

### 4.2. Cloudflare Worker

Worker là lớp public nhận request từ client và desktop app.

Nhiệm vụ:

- nhận HTTP/WebSocket request;
- xác thực request;
- nhận session code;
- tìm Durable Object tương ứng;
- chuyển tiếp request tới Durable Object;
- xử lý các endpoint quản lý session;
- trả lỗi khi session không tồn tại hoặc không hợp lệ.

Worker không trực tiếp điều khiển game.

### 4.3. Durable Object

Mỗi game session được điều phối bởi một Durable Object instance.

Nhiệm vụ:

- giữ WebSocket của desktop app;
- giữ WebSocket của các remote client;
- nhận diện vai trò desktop/client;
- relay message giữa desktop app và client;
- quản lý trạng thái session;
- xử lý desktop disconnect;
- hỗ trợ nhiều session độc lập cùng lúc;
- lưu metadata session nếu cần.

Durable Object không trực tiếp chạy game. Nó chỉ điều phối kết nối và message.

### 4.4. Desktop app

Desktop app gồm:

- AI chat hiện tại;
- WebSocket client kết nối tới Worker;
- session manager;
- message router;
- AI processor;
- game executor;
- feedback/status publisher.

Desktop app chủ động mở kết nối outbound tới Worker. Vì vậy máy game không cần mở public inbound port cho kiến trúc chính.

### 4.5. Game executor

Game executor vẫn chạy cục bộ trên máy game.

Nhiệm vụ:

- nhận command đã được AI hoặc command router xác nhận;
- thực thi hành động trong game;
- trả kết quả;
- báo lỗi hoặc trạng thái thực thi.

## 5. Luồng khởi động desktop app

~~~text
Desktop app khởi động
        ↓
Tạo session ID nội bộ
        ↓
Đăng ký session với Worker
        ↓
Nhận session code hiển thị cho người dùng
        ↓
Mở WebSocket outbound tới Worker
        ↓
Chờ remote client tham gia
~~~

Session code là mã để người dùng nhập. Session ID nội bộ phải là giá trị ngẫu nhiên, không nên dùng trực tiếp mã 4 số làm Durable Object ID.

Ví dụ:

~~~text
Mã hiển thị:       4821
Session ID nội bộ: session-7f2a9c...
~~~

## 6. Luồng client tham gia session

~~~text
Client mở remote client
        ↓
Nhập session code
        ↓
Kết nối tới Worker
        ↓
Worker xác thực code
        ↓
Worker chuyển request tới Durable Object
        ↓
Durable Object kiểm tra session
        ↓
Client được thêm vào session
~~~

Client không cần biết địa chỉ IP của máy game và không cần biết máy nào đang chạy session.

## 7. Luồng xử lý message

Ví dụ client gửi message: Mở rương.

~~~text
Client gửi message
        ↓
Worker nhận message
        ↓
Durable Object tìm desktop app của session
        ↓
Desktop app nhận message
        ↓
Message được đưa vào AI chat hiện tại
        ↓
AI phân tích và gọi tool nếu cần
        ↓
Game executor thực thi
        ↓
Desktop app tạo kết quả
        ↓
Durable Object relay kết quả
        ↓
Các client trong session nhận phản hồi
~~~

## 8. Vòng đời session

### Khi desktop app bật

1. Tạo session ID nội bộ.
2. Đăng ký session với Worker.
3. Nhận hoặc tạo session code.
4. Hiển thị code trên desktop app.
5. Mở WebSocket outbound.
6. Cho phép client tham gia.

### Khi desktop app tắt

1. Đóng WebSocket.
2. Worker/Durable Object nhận biết desktop disconnect.
3. Session chuyển sang inactive hoặc hết hạn.
4. Các client đang tham gia nhận thông báo.
5. Session code không còn sử dụng được.

## 9. Session code và bảo mật

Mã 4 chữ số được giữ lại cho prototype vì người dùng muốn nhập đơn giản.

Quy tắc tối thiểu:

- Mỗi lần app khởi động tạo session mới.
- Mã hết hạn khi app tắt hoặc session timeout.
- Giới hạn số lần nhập sai.
- Không dùng mã 4 số làm secret duy nhất.
- Dùng token nội bộ cho kết nối desktop.
- Dùng token hoặc quyền riêng cho client nếu cần kiểm soát người dùng.

Kiến trúc dữ liệu nên tách:

~~~text
user_code      = mã ngắn người dùng nhập
session_id     = ID ngẫu nhiên nội bộ
desktop_token  = token xác thực desktop app
client_token   = token xác thực client, nếu cần
~~~

## 10. Message protocol dự kiến

### Client gửi message tới desktop

~~~json
{
  "type": "chat_message",
  "request_id": "req-123",
  "session_code": "4821",
  "text": "Mở rương"
}
~~~

### Desktop gửi trạng thái

~~~json
{
  "type": "status",
  "request_id": "req-123",
  "state": "processing"
}
~~~

### Desktop gửi kết quả

~~~json
{
  "type": "command_result",
  "request_id": "req-123",
  "status": "completed",
  "message": "Đã mở rương"
}
~~~

### Desktop gửi lỗi

~~~json
{
  "type": "error",
  "request_id": "req-123",
  "code": "COMMAND_FAILED",
  "message": "Không thể thực hiện lệnh"
}
~~~

Mỗi message cần có request_id để client đối chiếu request và response.

## 11. Nhiều máy chạy đồng thời

~~~text
Mã 4821 → Durable Object/session của máy A
Mã 7350 → Durable Object/session của máy B
Mã 9142 → Durable Object/session của máy C
~~~

Các session phải độc lập:

- message của session A không được gửi sang máy B;
- client của session A không nhìn thấy message session B;
- desktop disconnect của A không ảnh hưởng B hoặc C;
- mỗi Durable Object quản lý đúng một session logic.

## 12. Vai trò của Cloudflare Tunnel

Cloudflare Tunnel hiện có thể tiếp tục dùng cho:

- truy cập quản trị riêng;
- debug hoặc health check;
- một service nội bộ khác;
- phương án fallback.

Nhưng kiến trúc multi-machine chính là:

~~~text
control.mcp88.uk → Cloudflare Worker
                           ↓
                    Durable Object
~~~

Không nên dùng cùng một hostname cho Worker và Tunnel nếu tạo xung đột route.

## 13. Những thành phần cần xây dựng

### Cloudflare side

- Worker public entry point.
- Durable Object class.
- Session registry.
- WebSocket relay.
- Authentication.
- Session expiry/cleanup.
- Rate limiting.

### Desktop app side

- WebSocket client.
- Session registration.
- Session code display.
- Reconnect logic.
- Message router.
- Kết nối message vào AI chat hiện tại.
- Gửi kết quả và trạng thái.

### Remote client side

- Form nhập session code.
- Kết nối WebSocket.
- Giao diện chat.
- Hiển thị trạng thái.
- Hiển thị lỗi.
- Reconnect khi mất mạng.

## 14. Các điểm cần khảo sát trong code hiện tại

1. Nơi tab AI chat nhận message từ UI.
2. Interface gửi message vào AI chat.
3. Nơi AI bắt đầu xử lý request.
4. Nơi AI gọi game tool hoặc game executor.
5. Cách nhận kết quả tool.
6. Cách hiển thị response lên UI.
7. Cách quản lý background thread hoặc async task.
8. Điểm phù hợp để thêm WebSocket client.
9. Cách tránh nhiều message xử lý xung đột.
10. Cách dừng kết nối khi desktop app đóng.

## 15. Phạm vi chưa chốt

- Mã phiên chính thức là 4 số hay nâng cấp sau.
- Có cần nhiều người dùng với quyền khác nhau không.
- Client được phép gửi chat tự do hay chỉ được gửi command allowlist.
- Có cần lưu lịch sử chat trên Cloudflare không.
- Có cần gửi screenshot/game state realtime không.
- Có cần Worker/Durable Object xử lý AI hay chỉ relay message.
- Cách xác thực desktop app và remote client production.

## 16. Quyết định kiến trúc hiện tại

- Hệ thống hỗ trợ nhiều desktop app chạy đồng thời.
- Mỗi desktop app gắn với một game session riêng.
- Desktop app chủ động kết nối outbound tới Cloudflare Worker.
- Worker là public entry point.
- Durable Object điều phối từng session và các WebSocket connection.
- Client chỉ cần nhập session code để tham gia.
- AI và game executor vẫn chạy trên máy desktop tương ứng.
- Cloudflare Tunnel không phải tuyến định tuyến chính.
- Không cần VPS riêng trong kiến trúc mục tiêu.
- Chi tiết deploy và giới hạn Cloudflare chưa được coi là đã kiểm chứng trong tài liệu này.
