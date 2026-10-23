# Roadmap triển khai hệ thống điều khiển game từ xa trên Cloudflare

> Nội dung được định dạng lại từ phản hồi của AI Cloudflare. Các giới hạn, giá và hành vi dịch vụ cần được xác minh lại trước khi triển khai production.

## Thông tin tổng quan

| Hạng mục | Giá trị |
|---|---|
| Domain | control.mcp88.uk |
| Dịch vụ | Cloudflare Workers + Durable Objects |
| Storage | SQLite-backed Durable Objects |
| Realtime | WebSocket Hibernation |
| Kế hoạch dự kiến | Workers Free |
| Ngày tạo | 2026-09-30 |

## 1. Kiến trúc tổng thể

~~~text
Desktop App A ─┐
Desktop App B ─┼── WebSocket ──> control.mcp88.uk
Desktop App C ─┘                       │
                                      ▼
                                  Worker
                                      │
                                      ▼
                              GameSessionDO
                         (một DO cho mỗi session)
                                      ▲
                                      │
Remote Client ── WebSocket ───────────┘
~~~

### Vai trò

- Worker nhận WebSocket upgrade, đọc session code, định tuyến tới Durable Object và cung cấp REST endpoint.
- GameSessionDO giữ các WebSocket trong cùng session, relay message và lưu metadata bằng SQLite.
- WebSocket Hibernation cho phép Durable Object hibernate khi idle và thức dậy khi có message mới.

## 2. Xác nhận tính khả thi

| Câu hỏi | Trả lời dự kiến |
|---|---|
| Durable Objects có sẵn? | Có thể dùng trên Workers Free, với SQLite backend. |
| SQLite-backed Durable Objects? | Có, cần xác minh lại trạng thái GA và giới hạn hiện tại. |
| WebSocket Hibernation? | Có, được khuyến nghị cho realtime. |
| Dùng control.mcp88.uk? | Có thể gắn bằng Custom Domain nếu hostname không bị DNS record khác sử dụng. |
| Dùng cùng Cloudflare Tunnel? | Có thể cùng tồn tại nếu dùng subdomain hoặc route khác nhau. |
| Có mất phí? | Phụ thuộc free tier và mức sử dụng thực tế. |
| Cần Node.js và Wrangler? | Cần cho deploy bằng CLI; desktop app/client không cần Wrangler. |

## 3. Giới hạn cần xác minh

| Hạng mục | Giá trị trong phản hồi nhận được |
|---|---:|
| Durable Objects requests | 100.000/ngày |
| Durable Objects duration | 13.000 GB-s/ngày |
| WebSocket connections/DO | 32.768 |
| WebSocket message size | 32 MiB |
| Storage/object | 10 GB/object |
| Workers requests | 100.000/ngày |

Phản hồi đề xuất Workers Paid khoảng 5 USD/tháng khi vượt free tier. Không nên coi đây là giá cố định trước khi kiểm tra Pricing của Cloudflare.

## 4. Giai đoạn 1 — Scaffold và Wrangler

### Khởi tạo project

~~~bash
mkdir game-control-relay
cd game-control-relay
npm init -y
npm install --save-dev wrangler
npx wrangler init
~~~

### Cấu hình wrangler.jsonc

~~~json
{
  "name": "game-control-relay",
  "main": "src/index.ts",
  "compatibility_date": "2026-09-30",
  "durable_objects": {
    "bindings": [
      {
        "name": "GAME_SESSION",
        "class_name": "GameSessionDO"
      }
    ]
  },
  "migrations": [
    {
      "tag": "v1",
      "new_sqlite_classes": ["GameSessionDO"]
    }
  ],
  "routes": [
    {
      "pattern": "control.mcp88.uk",
      "custom_domain": true
    }
  ]
}
~~~

### Chạy và deploy

~~~bash
npx wrangler dev
npx wrangler deploy
~~~

### Checklist

- [ ] Cài Node.js và Wrangler.
- [ ] Cấu hình Durable Object binding và migration.
- [ ] Chạy local thành công.
- [ ] Deploy thành công lên Cloudflare.
- [ ] Kiểm tra Worker URL trên workers.dev.

## 5. Giai đoạn 2 — Durable Object và WebSocket

### Cấu trúc file

~~~text
src/
├── index.ts
├── do/
│   └── game-session.ts
└── types.ts
~~~

### Worker entry point

~~~typescript
export { GameSessionDO } from "./do/game-session";

export default {
  async fetch(request, env): Promise<Response> {
    const url = new URL(request.url);

    if (request.headers.get("Upgrade") === "websocket") {
      const code = url.searchParams.get("code");
      if (!code) {
        return new Response("Missing session code", { status: 400 });
      }

      const id = env.GAME_SESSION.idFromName(code);
      return env.GAME_SESSION.get(id).fetch(request);
    }

    if (url.pathname === "/health") {
      return new Response("OK");
    }

    return new Response("Not found", { status: 404 });
  }
} satisfies ExportedHandler<Env>;
~~~

### Durable Object tối thiểu

~~~typescript
export class GameSessionDO extends DurableObject {
  async fetch(request: Request): Promise<Response> {
    if (request.headers.get("Upgrade") !== "websocket") {
      return new Response("Expected WebSocket", { status: 426 });
    }

    const pair = new WebSocketPair();
    const [client, server] = Object.values(pair);
    const role = new URL(request.url).searchParams.get("role") || "client";

    this.ctx.acceptWebSocket(server, [role]);
    return new Response(null, { status: 101, webSocket: client });
  }

  async webSocketMessage(ws: WebSocket, message: string | ArrayBuffer) {
    const payload =
      typeof message === "string" ? message : JSON.stringify(message);

    for (const peer of this.ctx.getWebSockets()) {
      if (peer !== ws) {
        peer.send(payload);
      }
    }
  }

  async webSocketClose(ws: WebSocket, code: number, reason: string) {
    ws.close(code, reason);
  }

  async webSocketError(ws: WebSocket, error: unknown) {
    console.error("WebSocket error:", error);
    ws.close(1011, "WebSocket error");
  }
}
~~~

### Test local bằng wscat

~~~bash
# Terminal 1 — desktop
wscat -c "ws://localhost:8787?code=ABC123&role=desktop"

# Terminal 2 — client
wscat -c "ws://localhost:8787?code=ABC123&role=client"
~~~

## 6. Giai đoạn 3 — Session Code và Authentication

### Endpoint dự kiến

~~~text
POST /api/session/create
WebSocket /?code=<session-code>&role=<desktop|client>
GET /health
~~~

### Authentication

| Kết nối | Phương pháp dự kiến |
|---|---|
| Desktop → Worker | desktopId hoặc token trong header |
| Client → Worker | Session code |
| WebSocket | Token ngắn hạn do Worker cấp |

Session cần có trạng thái active/inactive, timeout cleanup và thông báo khi desktop disconnect.

> Quyết định trước đó là dùng mã 4 chữ số cho MVP. Nếu giữ mã 4 số, cần rate limit, thời hạn và token nội bộ vì mã 4 số không đủ bảo mật khi public trên Internet.

## 7. Giai đoạn 4 — Tích hợp Desktop App Windows

### Công nghệ đề xuất

| Công nghệ | Thư viện |
|---|---|
| C#/.NET | System.Net.WebSockets |
| Python | websockets |
| Electron | ws |

### Luồng hoạt động

~~~text
App khởi động
    ↓
POST /api/session/create
    ↓
Nhận và hiển thị session code
    ↓
Mở WebSocket tới Worker
    ↓
Nhận message từ remote client
    ↓
Thực thi lệnh trong game
    ↓
Gửi feedback/status
~~~

Endpoint:

~~~text
wss://control.mcp88.uk?code=ABC123&role=desktop
~~~

Yêu cầu: auto-reconnect, exponential backoff, heartbeat, parse/validate message, allowlist lệnh và feedback về client.

## 8. Giai đoạn 5 — Remote Client

Khuyến nghị bắt đầu bằng Web client vì không cần cài app, dùng được trên điện thoại/máy tính và dễ kiểm thử WebSocket.

~~~html
<input id="code" placeholder="Session code" maxlength="6" />
<button onclick="connect()">Join</button>

<script>
  let ws;

  function connect() {
    const code = document.getElementById("code").value.toUpperCase();
    ws = new WebSocket(
      "wss://control.mcp88.uk?code=" + code + "&role=client"
    );

    ws.onmessage = (event) => {
      const message = JSON.parse(event.data);
      if (message.type === "status") {
        updateUI(message);
      }
    };
  }
</script>
~~~

Có thể deploy web client trên Cloudflare Pages với subdomain play.mcp88.uk.

## 9. Message protocol

### Client → Desktop

~~~json
{
  "type": "input",
  "action": "key_press",
  "key": "W",
  "timestamp": 1696123456789
}
~~~

~~~json
{
  "type": "input",
  "action": "mouse_click",
  "button": "left",
  "x": 100,
  "y": 200
}
~~~

### Desktop → Client

~~~json
{
  "type": "status",
  "state": "running",
  "fps": 60
}
~~~

~~~json
{
  "type": "error",
  "message": "Key not supported"
}
~~~

Screenshot lớn không nên nhúng trực tiếp vào JSON; nên chia chunk hoặc dùng R2.

## 10. Giai đoạn 6 — Deploy và giám sát

~~~bash
npx wrangler deploy
nslookup control.mcp88.uk
npx wrangler tail game-control-relay
~~~

### Kiểm thử production

1. Desktop app tạo session.
2. Desktop app nhận code.
3. Remote client kết nối bằng code.
4. Client gửi input.
5. Desktop app thực thi.
6. Desktop app gửi feedback.
7. Kiểm tra nhiều session song song.
8. Kiểm tra reconnect sau khi desktop restart.

## 11. Timeline dự kiến

| Giai đoạn | Nội dung | Thời gian |
|---:|---|---:|
| 1 | Scaffold và Wrangler | 1–2 ngày |
| 2 | Durable Object và WebSocket | 2–3 ngày |
| 3 | Session code và authentication | 2–3 ngày |
| 4 | Tích hợp desktop app | 3–5 ngày |
| 5 | Remote client | 2–3 ngày |
| 6 | Deploy và giám sát | 1–2 ngày |
| **Tổng** |  | **11–18 ngày** |

## 12. Lưu ý quan trọng

- Session code 6 ký tự A-Z/0-9 có không gian lớn hơn mã 4 số, nhưng vẫn cần kiểm tra collision.
- Code đơn thuần không đủ bảo mật; cần token auth, rate limit và thời hạn session.
- Desktop app và client đều cần reconnect với backoff: 1s, 2s, 4s, 8s, tối đa 30s.
- Nên có heartbeat khoảng mỗi 30s.
- Screenshot lớn nên chia nhỏ hoặc lưu qua R2.
- Cần xác minh lại giới hạn message, storage và chi phí theo plan hiện tại.

### Phân tách Cloudflare Tunnel và Worker

~~~text
control.mcp88.uk → Worker + Durable Objects
Subdomain khác   → Cloudflare Tunnel hiện tại
~~~

Không nên dùng chung một hostname cho Tunnel và Worker nếu tạo xung đột route.

## 13. Kiến trúc mục tiêu

~~~text
Máy A
  └── Desktop app
        ├── AI chat
        ├── WebSocket client
        ├── Session manager
        └── Game executor

Máy B
  └── Desktop app
        ├── AI chat
        ├── WebSocket client
        ├── Session manager
        └── Game executor

Remote clients
  └── Web client / mobile app

Tất cả kết nối tới:
  └── control.mcp88.uk
        └── Cloudflare Worker
              └── GameSessionDO theo session code
~~~

### Quyết định kiến trúc

- Nhiều desktop app có thể chạy đồng thời trên nhiều máy.
- Mỗi desktop app có một game session riêng.
- Client nhập session code để kết nối đúng máy.
- Worker là public entry point.
- Durable Object điều phối session và WebSocket.
- Desktop app xử lý AI và điều khiển game.
- Cloudflare Tunnel không thay thế Worker/Durable Object trong mô hình multi-machine.
